---
title: "Rootless Containers on Alpine"
date: 2022-10-12T22:17:15+11:00
draft: false
showSummary: true
summary: "We recently murdered a server's terminal via `do_distro_upgrade`, and thought it'd be a good time to learn more about containers and Alpine."
---

# Rootless Containers on Alpine Linux

## Background
**(Ashe)**
So. We recently murdered a server's terminal via `do_distro_upgrade`.

**(Tammy)** Was it really that bad?

**(Ashe)** Yes.

```
%  man 7z                               
WARNING: terminal is not fully functional
-  (press RETURN)%
```
It was in fact *that bad*. So we figured, well, we can spend a few hours, days, whatever fixing this...

**(Tammy)** Or we could just build a new server!

**(Ashe)** Right.
So, after asking some friends about their opinions, we settled on Alpine Linux. And why not also migrate all of our pm2 workloads to containers while we're at it? We've been meaning to learn more about containers for a while now.

So off we go!

## Prep Work

We need a few things before we actually set up rootless containers. We'll be following along with the [Official Rootless Containers Tutorial](https://rootlesscontaine.rs/getting-started/common/), making adjustments as necessary.

### Login Information

Most Rootless Container implementations use `$XDG_RUNTIME_DIR` to find the user's ID and where their runtime lives (usually some subdir of `/run/user/`).
Systemd-based Linux distros will handle this automatically, but Alpine uses [OpenRC](https://wiki.alpinelinux.org/wiki/OpenRC), which does not do this automatically.

While Alpine doesn't provide a tutorial for Rootless Containers, we can adapt some of the prep work done for [Wayland](https://wiki.alpinelinux.org/wiki/Wayland) to get OpenRC to set `$XDG_RUNTIME_DIR` for us.

We just create `/etc/profile.d/xdg_runtime_dir.sh` like so:
```sh
if test -z "${XDG_RUNTIME_DIR}"; then
  export XDG_RUNTIME_DIR=/tmp/$(id -u)-runtime-dir
  if ! test -d "${XDG_RUNTIME_DIR}"; then
    mkdir "${XDG_RUNTIME_DIR}"
    chmod 0700 "${XDG_RUNTIME_DIR}"
  fi
fi
```
And, log out and then back in...
```sh
~ ❯ env
[...]
XDG_RUNTIME_DIR=/tmp/1000-runtime-dir
[...]
```

With that done, we can move onto our next steps.

### Sysctl
There's some sysctl config required for older distros, but this isn't required for Alpine.

### User Namespace Configuration
Rootless Containers use User Namespaces, subUIDs, and subGIDs, so we'll need to have those working. 
The apk package `shadow-subids` provides that functionality for us.
```
~ ❯ apk info shadow-subids
shadow-subids-4.10-r3 description:
Utilities for using subordinate UIDs and GIDs

shadow-subids-4.10-r3 webpage:
https://github.com/shadow-maint/shadow

shadow-subids-4.10-r3 installed size:
140 KiB
```

### Sub-ID Counts
Rootless Containers generally expect `/etc/subuid` and `/etc/subgid` to contain at least 65,536 sub-IDs for each user.
`shadow-subids` doed create these files for us, but leaves them empty by default, so let's go ahead and do that.
The [page on subIDs](https://rootlesscontaine.rs/getting-started/common/subuid/) provides a handy Python script to do that for us, which we'll edit slightly so it's not writing directly to system files:
```python
f = open("subuid", "w")
for uid in range(1000, 65536):
  f.write("%d:%d:65536\n" %(uid,uid*65536))
f.close()

f = open("subgid", "w")
for uid in range(1000, 65536):
  f.write("%d:%d:65536\n" %(uid,uid*65536))
f.close()
```
This is probably overkill for our use-case, but that's also fine.

**(Doll)** So this one just runs script and copies to /etc/?

**(Ashe)** Yes Doll, that's right.
With that done, we can move onto the last prep step.

### CGroups V2
To limit resources that a container can use, we need to enable CGroups V2. In OpenRC, this can be done by changing some options in `/etc/rc.conf`.

To enable CGroups in general, we need to set `rc_controller_cgroups` to `YES`
```sh
# This switch controls whether or not cgroups version 1 controllers are
# individually mounted under
# /sys/fs/cgroup in hybrid or legacy mode.
rc_controller_cgroups="YES"
```
From here, we can enable CGroups V2 by setting `rc_cgroup_mode` to `unified`
```sh
# This sets the mode used to mount cgroups.
# "hybrid" mounts cgroups version 2 on /sys/fs/cgroup/unified and
# cgroups version 1 on /sys/fs/cgroup.
# "legacy" mounts cgroups version 1 on /sys/fs/cgroup
# "unified" mounts cgroups version 2 on /sys/fs/cgroup
rc_cgroup_mode="unified"
```

**(Doll)**: Doll confused.

**(Ashe)** So was I, for a bit. Despite what `rc.conf` says, cgroups V2 does *not* seem to be enabled on Alpine 
unless `rc_cgroup_mode` is set to `unified`. The [https://wiki.alpinelinux.org/wiki/OpenRC#cgroups\_v2](Alpine Wiki)
seems to agree here, but isn't super clear. We'll find out if this is sufficient.


Next step is configuring the controllers we want to use:
```sh
# This is a list of controllers which should be enabled for cgroups version 2
# when hybrid mode is being used.
# Controllers listed here will not be available for cgroups version 1.
rc_cgroup_controllers="cpuset cpu io memory hugetlb pids"
```
Finally, we can add cgroups to a runlevel so that it's started automatically at boot:
```sh
rc-update add cgroups
```
From here, we can reboot, and continue on. If you don't want to reboot, you can start the cgroup service manually:
```sh
rc-service cgroups start
```


## Configuring the Rootless containerd service
We'll be using nerdctl as our containerd controller of choice. It comes with a rootless containerd.service, but since Alpine doesn't use systemd, we'll have to adapt this into an rc service.

We can adapt the [install script](https://github.com/containerd/nerdctl/blob/48f189a53a24c12838433f5bb5dd57f536816a8a/extras/rootless/containerd-rootless-setuptool.sh) nerdctl provides to our purposes.
