---
title: "Our Home Server Build-Out Log Part 1: Speccing It Out"
date: 2022-10-31T17:00:00+11:00
draft: false
showSummary: true
summary: "We've wanted to build a home lab for a long time. Here's how it went."
series:
  - "Home Server Build-Out"
series_order: 1
---

# Part One: Specification

## Background

We've wanted to build a home lab for a long, long time, but never got around to it.

**(Selene)** However, a certain someone was complaining about disaster recovery and migrating workloads to us,
and that turned out to be just the push we needed to get this rolling.

Since we also want to do more blogging and community contribution, why not make this the inaugural post on our new blog?

**(Lorelai)** So, let's get this show on the road, shall we?

## The Use-Case

So what did we need from this server? After talking over with friends, the following use-cases came to mind:
 - Storage Cluster (ZFS of some kind)
 - Foundry VTT Server
 - [chibisafe](https://github.com/chibisafe/chibisafe) Server
 - Modded Minecraft Server ?
 - Generic cloud host for friends

## The Spec
We wanted this server to be the basis for a home lab that could later be expanded, so we want it to have a decent amount of oomph.
After deliberating on it for a while, we decided on the following:

| Part        | Spec                                                        |
| ----------- | ----------------------------------------------------------- |
| CPU         | Min. 16 core/32 thread                                      |
| Motherboard | Must support large amounts of RAM and have BMC/IPMI         |
| RAM         | Moderate speed, 128GB to start, more to be added            |
| Storage     | At least 32 TB usable, 2 redundant disks min., 1 hot spare  |
| Networking  | Min. 2 ✕ 1Gbps management, 2 ✕ 40Gbps data                  |

### But Why?

**(Selene)** I'm no techie, but this all seems rather excessive for a home server. Especially compared to people we know.

**(Ashe)** You're not wrong there, Selene. I could wax lyrical about how I want this server to last a long time and be
useful for many different things and so on and so forth, but if we're being honest, the main reason is "Tech shiny!"

**(Tammy)** So the same reason we've always gotten high spec PCs, really.


## Fulfilling the Spec
Now that we have a spec, we need to figure out the best way to fulfill it.

### The Case
This would normally be something to consider carefully, but a good friend was getting rid of a 3U server case with
12 3.5" bays, so we'll nick that off of him, and move on. He also donated us a few
[Arctic P12s](https://www.arctic.de/en/P12/ACFAN00118A) for cooling as they'd be quieter than the jet engines
that came with the case.

### The PSU
We have a spare 750W Silverstone 80+ Platinum PSU lying around the house from an old PC, so we'll just repurpose that here.

### The Storage
It's probably strange to start with the storage, but we'll need to know how much storage we have to spec out
the ZFS instance properly.

#### The Goal
The goal is to have a scaleable template that can be expanded without breaking the bank each time,
while maintaining favourable failure characteristics. We'll budget out 3000 AUD for each expansion.

**(Doll)** That sounds like a lot, Miss...

**(Ashe)** It's a good chunk, yes. But this should, ideally, only happen every few years, so we can amortise this
over the life of the storage.

We have 12 3.5" slots in our 3U compute case, and most storage arrays are either 24 or 36 bay, so planning around
12 unit slices seems logical.

If we round up and call it 3600 AUD per slice, we get ~300 AUD per HDD.
At time of writing, the best $/GB is 16TB Seagate Exos drives. We picked up 6 a year ago when exchange rates
weren't so miserable, and will be picking up another 6 to round out this cluster after the new year.
This overshoots our budget by about 25%, but there's little to be done about exchange rates.

We can, however, expect these to fall in price over time,
so this will likely be within budget by the time we need another slice.


#### RAID Layout
[dRAID](https://openzfs.github.io/openzfs-docs/Basic%20Concepts/dRAID%20Howto.html) came to OpenZFS a while back,
and has improved resilver times compared to Z1/Z2/Z3, as well as allowing for much more granular redundancy design,
so we'll be using that.

The Seagate Exos drives are specc'd at ~200MB/s peak write rate, which gives us a resilver time of just over 22 hours in
traditional Z1/Z2/Z3. And that's 22 hours of every other drive in the array getting hammered. This is the main reason
we're going to be using dRAID. Under dRAID, we specify a number of data, parity, and spare slices, which are distributed
amongst the physical drives.

In our case, we want at least one, maybe two, hot-spare slices to make best use of dRAID.
This leaves us with 10 or 11 slices to play with.

With 11 usable slices our best bet would probably be 8/3/1 (8 data, 3 parity, 1 hot-spare), where as with 10 usable slices,
we'd probably want to go for either 7/3/2, or 8/2/2.

7/3/2 is certainly the safest set-up, as it gives us 3 parity slices and two hot-spares, potentially allowing for
5 HDD failures before data-loss occurs. However this is over-conservative for the pool that we'll be building out.
This pool won't be used for critical data, so we can afford to be a little more aggressive
with our dRAID setup.
That leaves us with 8/2/2, or 8/3/1.

These can both tolerate 4 sequential drive failures, however 8/2/2 has a slightly higher write and resilver speed
at the cost of tolerating less concurrent failures (2 at a time vs 3 at  a time).

With how large spinning rust drives can get these days (18 - 20 TB each), two parity slices feels a little sketchy, so
we'll go with 8/3/1 as our template.

#### ZIL/L2ARC
We'll also need a ZIL and L2ARC device. Sun's old documentation recommends mirroring this device for resilience and
reliability reasons. That seems entirely reasonable, so let's do that, too.
We can also use these drives as a read cache to improve latency on popular data.
We'll want something with relatively high IOPS for this, as well as good random IO speeds.
In theory, we could go hog wild and get Intel Optane and never worry about it again, but that's...
prohibitively expensive to put it lightly.

Balancing cost and performance, 2 ✕ 500GB Seagate FireCuda 530s does the job well. Relatively high endurance,
good random IO performance, and not too expensive. Two of these sets us back 600 AUD.

#### Final Storage Configuration
Our final storage array consists of 12 ✕ 16 TB Seagate Exos Drives, and 2 ✕ 500GB Seagate FireCuda 530s.
With 8 data slices, 3 parity slices, and 1 hot spare slice, we should end up with approximately 110 TB of usable space.

{{< alert >}}
**(Ashe)** It is worth mentioning here that dRAID does not allow for variable stripe width due to how sequential
resilvering works, so compression ratios and real vs on-disk utilisation may suffer.
{{< /alert >}}

### The CPU
We're going to make an executive decision and go with an AMD EPYC CPU, because we've always wanted to use one.
That said, we have a few options:

| Model    | Generation | RRP (USD) | Cores (threads) |  Clock (GHz)  | L3 Cache (MB) | TDP (W) |
| -------- |:----------:|:---------:|:---------------:|:-------------:|:-------------:|:-------:|
| 7272     |    Rome    |    625    |     12 (24)     |   2.9 - 3.2   |      64       |   120   |
| 7302     |    Rome    |    978    |     16 (32)     |   3.0 - 3.3   |      128      |   155   |
| **7352** |  **Rome**  | **1350**  |   **24 (48)**   | **2.3 - 3.2** |    **128**    | **155** |
| 7402     |    Rome    |   1783    |     24 (48)     |  2.8 - 3.35   |      128      |   180   |
| 7F72     |    Rome    |   2450    |     24 (48)     |   3.5 - 3.9   |      192      |   240   |
| 7452     |    Rome    |   2025    |     32 (64)     |  2.35 - 3.35  |      128      |   155   |
| 7453     |   Milan    |   1570    |     28 (56)     |  2.75 - 3.45  |      64       |   225   |
| 7413     |   Milan    |   1825    |     24 (48)     |  2.85 - 4.00  |      128      |   200   |
| 7513     |   Milan    |   2840    |     32 (64)     |  2.6 - 3.65   |      128      |   200   |

The odd one out here is the `7F72`, which is a frequency-optimised model, designed for maximum performance per core
to get around per-core licensing issues in enterprise applications. While cool, it being nearly double the price
of the comparable `7352` puts it outside our budget for this particular build.

Again, balancing price and performance, we've decided to go with a `AMD EPYC 7352`, as 24/48 exceeds our spec, and doesn't break
the bank while doing so. We miss out on some of the performance of the Milan line, but that's acceptable here.
The SP3 socket also allows us to upgrade to Milan(-X) down the line if we need more performance (with a BIOS update).

Shipped to a friend, this sets us back ~1500 AUD.

### The Motherboard

With our CPU chosen, we need a motherboard that fulfills our purposes.
Of the readily-available options, we are looking for something with an
IPMI/BMC, and dual Ethernet interfaces onboard, as our data port requirement can be fulfilled by a PCIe network card.

8 SATA ports under one SATA controller would be nice, as it makes configuring passthrough for ZFS easier, but is not
essential.

The `AsRock Rack ROMED8U-2T` serves our purposes perfectly:
 - [X] &nbsp;&nbsp;8 ✕ DIMM slots
 - [X] &nbsp;&nbsp;8 ✕ SATA (2 ✕ mini-SAS)
 - [X] &nbsp;&nbsp;Some amount of PCIe Slots (3)
 - [X] &nbsp;&nbsp;2 ✕ 10GbE + 1 ✕ IPMI

We don't *need* 10GbE for our management network, but this allows us to build this out with 2 ✕ 10GbE for data, and
upgrade to 2 ✕ 40GbE later, which may be what we end up doing. Notably, this leaves us without separate management and
data interfaces.

**(Selene)** Is that a problem?

**(Ashe)** Not really, no. It makes securing the network *slightly* trickier as we can't airgap the management network,
but it's not the end of the world. We intend to get a PCIe Ethernet card at some point anyway, so this'll
be temporary.

**(Octavia)** Famous last words.

New from Newegg, this sets us back ~1100 AUD.

### The RAM
There are two ways we can estimate how much RAM we'll need.
 - Total RAM required by all planned VMs
 - RAM per vCPU

#### RAM by VM

1. Foundry Server
	* 8GB will be plenty for this
2. Chibisafe
	* Chibisafe prides itself on running lean, so we should be able to go down to as low as 1 or 2 GB
3. ZFS Storage Cluster
	* The extremely conservative guideline (as published by Sun back in the day) is 1GB RAM per TB of storage.
      This guideline was published with the idea that at this level you should *never* encounter any bottlenecks or issues.
	* We do not need such a strict performance guarantee.
	* We should be able to halve or even quarter this and not encounter bottlenecks.
	* We'll initially provision 32GB, and adjust as necessary.
4. Modded Minecraft Server
	* 16 - 32 GB is the rough ballpark for good performance with 8 - 16 players
	* This is likely overkill for our server, so we can dial it back to 12GB with some GC tuning on the server end

So that totals up to 8 + 2 + 32 + 12 = 54 GB.

We want to allow room for growth and for friends to start their own VMs, so the next logical stepping stones are 64 or 128 GB.

#### Total Required RAM
We have 48 vCPUs in our current setup (with no overcommit, but more on that in the next blog post).
Very broadly, most environments that we've had exposure to allocate approximately 4GB/vCPU, and adjust based on how
CPU-hungry or memory-hungry a particular workload is.
Since we're expecting mixed workloads (from friends), we'll follow that same guideline.
We'll need at least 4 ✕ 48 = 192GB. With 8 slots on the mobo,
that means we'll have to use 32GB modules (or 64GB modules if we feel like going overboard).

Our motherboard comes with 8 RAM slots and the Rome EPYC CPUs support octo-channel RAM, so we'd get
the best performance from fully populating the RAM slots.

This however makes upgrading painful, so we're going to go with 4 ✕ 32GB @ 2400MHz for 128GB total for now, which sets
us back ~1000 AUD.

## The Damage
All up, we've settled on:
|   Component    |                    Selected Part                     |   Cost (AUD)    | Running Total (AUD) |
|:--------------:|:----------------------------------------------------:|:---------------:|:-------------------:|
| 3U Server Case |                   Who Even Knows?                    |      FREE!      |          0          |
|      PSU       |   SilverStone 750W Platinum <br /> (SST-SX750-PT)    |      FREE!      |          0          |
|      CPU       |      AMD EPYC 7352 <br /> (24 core / 48 thread)      |      1500       |        1500         |
|      RAM       |      4 ✕ 32GB@2400MHz <br /> (M393A4K40CB1-CRC)      | 4 ✕ 250 = 1000  |        2500         |
|  Mass Storage  |    12 ✕ 16TB Seagate Exos <br /> (1ST16000NM001G)    | 12 ✕ 400 = 4800 |        7300         |
| ZIL/L2ARC SSDs | 2 ✕ 500GB Seagate FireCuda 530 <br /> (ZP500GM3A013) |  2 ✕ 300 = 600  |        7900         |
|  Motherboard   |                AsRock Rack ROMED8U-2T                |      1100       |        9000         |

So just shy of 10,000 AUD. It's a hefty price tag, but worth it in our opinion.

## Next Time!

With all that decided upon, ordered, all we have to do now is wait for it to arrive.

Join us again next time for the actual build log!

**(Doll)** This one is super excited to see Miss' build the nyoomy server!
