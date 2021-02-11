# Using optane memory to expand the capacity of a typical kdb market data system
[by Nick McDaid and Eoin Cunning](#authors)

## Abstract
- Wrote a test suite to examine the latency of a typical real time kdb solution. To compare and contrast the effects of different volumes of data and throughput rates among other variables.
- Wrote tools to easily configure the splitting of real time data between dram and filesystem backed memory.
- Compared the performance versus regular dram in these scenrios for several combinations of the system varibles mentioned.
- Found that there are scenarios where storing rdb data in optane is a viable option and in these case the addition of optane to an existing server should expand capicity where using traditional hardware would mean requiring to get additional servers.

### Table of Contents  

[Background](#Background)  
[Using filesystem backed memory](#FilesystemBackedMemory)  
[Testing](#testing)  
[Findings](#findings)  
[Conclusion](#conclusion)  
[About The Authors](#authors)  
[Appendix](#appen)

<a name="Background"/>

## Background
This blog assumes reader already has some knowledge of optane and persistent memory and how kdb has been adapted to interact with it. More information on this [here](https://code.kx.com/q/kb/optane).

While there has been research published in the kdb community showing the effectiveness of Optane as an extremely fast disk, ["performing up to almost 10 times faster than when run against high-performance NVMe"](https://kx.com/blog/overcoming-the-memory-challenge-with-optane). However there as yet, has been no published research using Optane as a volatile memory source. 

This blog post looks at running the realtime elements of a kdb+ market data stack on Optane Memory, mounted in DAX Enabled App Direct Mode. It also provides a few useful utilities for moving your data into Optane with minimal effort, and it documents the observed performance.

<a name="FilesystemBackedMemory"/>

## Filesystem backed memory
TODO could ad more explanation to app direct mode
https://thessdguy.com/intels-optane-two-confusing-modes-part-3-app-direct-mode/

Filesystem backed memory is a available in kdb by use of the [.m namespace](https://code.kx.com/q/ref/dotm/) added to kdb in version 4.0

From kdb release notes:
```
2019.10.22
NUC
memory can be backed by a filesystem, allowing use of dax-enabled filesystems (e.g. AppDirect) as a non-persistent memory extension for kdb+.
cmd line option -m path to use the filesystem path specified as a separate memory domain. This splits every thread's heap in to two:
  domain 0: regular anonymous memory (active and used for all allocs by default)
  domain 1: filesystem-backed memory
 .m namespace is reserved for objects in domain 1, however names from other namespaces can reference them too, e.g. a:.m.a:1 2 3
 \d .m changes current domain to 1, causing it to be used by all further allocs. \d .anyotherns sets it back to 0
 .m.x:x ensures the entirety of .m.x is in the domain 1, performing a deep copy of x as needed (objects of types 100-103h,112h are not copied and remain in domain 0)
 lambdas defined in .m set current domain to 1 during execution. This will nest since other lambdas don't change domains:
c  q)\d .myns
  q)g:{til x}
  q)\d .m
  q)w:{system"w"};f:{.myns.g x}
  q)\d .
  q)x:.m.f 1000000;.m.w` / x allocated in domain 1
 -120!x returns x's domain (currently 0 or 1), e.g 0 1~-120!'(1 2 3;.m.x:1 2 3)
 \w returns memory info for the current domain only:
  q)value each ("\\d .m";"\\w";"\\d .";"\\w")
-w limit (M1/m2) is no longer thread-local, but domain-local; cmdline -w, \w set limit for domain 0
mapped is a single global counter, same in every thread's \w
```
An important note for the concept for storing tables is:
```
q)t:.m.t:til 10  //define variable in .m then another variable that will point to same space in memory
q)t,:10          //append to t
q)-120!t         //t is in filesystem-backed memory domain
1
```
This means that with one simple change when defining schemas tables (or subset of columns of table) can be moved to filesystem backed memory without any other codes changes to the rest of the system. This was accomplished using the following two functions.
```
.mUtil.colsToDotM:{[t;cs] // ex .mUtil.colsToDotM[`trade;`price`size]
 /enusre cs is list
 cs,:();
 /pull out columns to move an assign in .m namespace
 (` sv `.m,t) set ?[t;();0b;cs!cs];
 /replace columns in table
 t set get[t],'.m[t];
 }

.mUtil.tblToDotM:.mUtil.colsToDotM[;()]
```
The difference then between running a DRAM rdb and appDirect rdb is starting it with the [-m option](https://code.kx.com/q/basics/cmdline/#-m-memory-domain) and running
```
.mUtil.tblToDotM each tables[];
```
After loading the standard schema file.

### Accessing file system backed memory - (this could be whole seperate blog)

As extra note we need to be aware When accessing the data from this new memory domain that if use our convential method and existing apis we will most likely at whatever stage we filter or aggregate or create new variables from that data it will be assigned a new spot in the memory.
There will be an inital over head for pulling the data from app direct instead of dram. After that once we cause kdb to assign data to new memory space it will get assigned in dram and everything from that point on will be as fast as it was in a dram rdb.

This is important as we have the ability to spin up many more instances of rdb, but when we query via api or otherwise new allocations will be made to dram. want to ensure your system doesnt try to sort whole quote table in all the rdbs at the same time as will quickly run out of dram.

However if we define code within the .m namespace then any intermediateis there created there will use the -m domain. 

So if we had a very memory intensive piece of code to run something like end of day we could define the code in .m namespace and run all of it without using any dram.
Proof:
```
q)n:1000000;.m.tab:tab:([]t:n?.z.n;s:n?`4;n?100f;n?100)
q)f:{`s`t xasc tab}
q)\d .m
q.m)f:{`s`t xasc tab}
q.m)\d .
q)\ts f[]
133 41944096
q)\ts .m.f[]
351 512
q)f[]~.m.f[]
1b
```

Whoever this is out of scope for what we are looking at here.

<a name="testing"/>

## Testing Framework
The Framework designed to test how optane chips can be deployed is a common market data capture solution. Based on the standard [tick setup](https://github.com/KxSystems/kdb-tick)

The traditional bottle neck of market data platforms has been the amount of DRAM on a server, which in turn determines how many RDB's a server can host. There are a number of ways to try and get around this limitation, such as directing users to aggregated services such as second or minute bucketed data where possible, however when a user requires tick by tick precision, there is no other option other routing the query to the raw data in the RDB.

To ensure a sufficent stress test, the system simulates the volume and velocity of market data processed during the market crash on March 9th 2020. We prestressed the RDB with 65 million records, and then sent 48,000 updates per second for the next 30 minutes, split between trades and quotes in a 1:10 ratio. A tickerplant consumed this feed and on a 50ms timer, distrubed the messages to an aggregation engine, which generated minute / daily aggregations and published these back to the tickerplant. The RDB consumed all incoming messages. Every 2 seconds our monitor process would query the RDB and measure:
- Max trade / quote latency (time between the tick being generated in the feed process and the tick being accessible via an RDB query)
- Max latency on aggregated trade / quote message (as above - but also including the time for the aggregation engine to do it's calculations and it's additional messaging hops)
- Time for an asof join of all trades seen for a single symbol and their prevailing quote information
- Various memory stats such as Percentage DRAM used and Percentage Optane memory used

The above was considered a "stack". We ran four stacks concurrently for our testing. Two fully hosted in DRAM and two with their RDBs hosted in Optane. 

Please find a more detailed description of the [arcitecture](#arcitecture) in the appendix.

<a name="findings"/>

## Findings

### On a server that maxed out with 2 DRAM rdbs we could run 10 appDirect rdbs!
we have mainly explored the capabilities of the standard tick systems abilities to keep latency low and process data while using pmem instead of dram. And expanding the capabilities of an existing server. We can run 10 optane rdbs at same time. And have very low mem usage. However you need to design to ensure you dont try to pull all the mem into dram durring queries.

### (AGG ENGINE IN PMEM = BAD)
It's worth nothing for our aggregation engines, we kept all the caches and required data in DRAM as Optane does have I/O restrictions and an engine which is constantly reading and writing small ticks would not be the optimal use case for the technology.

### Queries take longer so make sure extra services are worth it
Further exploration into the query performance of the optane rdbs should be looked at. could be 2-5x slower for raw pulling from dram compared to pmem but seeing how this would actually affect a api and if this slow down is completely offset by ability to run more rdbs.

### READ versus write
The general consesus was that pMem reads are super fast and closer to DRAM that writes 
but we seemed to run into issues with the reads more than writing the data.
This is possibly because generally kdb will write small amounts of data throughout the the pmem.
But queries can attempt to access all the data in pmem. .e.g max quote\`time 

<a name="conclusion"/>

## Conclusion
- It is possible to run rdbs with data stored in optane instead of dram. Code required for such changes is outlined.
- There is a trade off in performance for querying from persistent memory. can be seen to be 2-5x slower.
- Possible that just trying to convert a working rdb to optane. This slow down in read speed will cause queries to take longer and could result in rdb delaying processing of of new messages for tp.
- If a rdb is already seeing very high query volumes may not be advisable to move everything to optane mem.
- More realistic use case is if you have columns that are seldom queried even only adhocly that subset of columns could be moved to .m namespace freeing up ram for more rdbs.

While it isn't primarily marketed as a DRAM replacement technology, we found it was a very helpful addition in augmenting the volatile memory capacity of a server hosting realtime data.

### Steps for further investigation or post could involve
- enhancing query testing, have load balancer infront of rdbs and compare replacing 1 dram rdb with 3 optane rdbs. Ensure latency not an issue due to write contention and confirm that query impact not affect slow down mitagated by extra services available to run them.
- using optane for large static tables that take up alot of room in mem but are queried so often take to long to have written to disk. .e.d flatfile table you load into hdbs or reference data tables saved in gateways
- replay performance - we have seen write contention so be interesting to exploer how replay would work for pmem rdbs. This would probably stress test the write performance to the pmem mounts. May need to batch upds and and write to dram as intermediary

<a name="authors"/>

## About the authors
[Eoin Cunning](https://www.linkedin.com/in/eoin-cunning-b7195a67) is a kdb+ consultant, based in London, who has worked on several kx solutions projects and currently works on a market data system for a leading global market maker firm.

[Nick McDaid](https://www.linkedin.com/in/nmcdaid/) is a kdb+ consultant, based in London who has worked as a developer in multiple top tier banks and currently works as a devolper in a European hedge fund.

<a name="appen"/>

## Apendix

### Hardware

#### Server details 
```
|18:05:28|user@clx4:[q]> lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              56
On-line CPU(s) list: 0-55
Thread(s) per core:  1
Core(s) per socket:  28
Socket(s):           2
NUMA node(s):        2
Vendor ID:           GenuineIntel
CPU family:          6
Model:               85
Model name:          Genuine Intel(R) CPU 0000%@
Stepping:            6
CPU MHz:             2538.054
CPU max MHz:         4000.0000
CPU min MHz:         1000.0000
BogoMIPS:            4400.00
Virtualization:      VT-x
L1d cache:           32K
L1i cache:           32K
L2 cache:            1024K
L3 cache:            39424K
NUMA node0 CPU(s):   0-27
NUMA node1 CPU(s):   28-55
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc cpuid aperfmperf pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch cpuid_fault epb cat_l3 cdp_l3 invpcid_single ssbd mba ibrs ibpb stibp ibrs_enhanced tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm cqm mpx rdt_a avx512f avx512dq rdseed adx smap clflushopt clwb intel_pt avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 xsaves cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local dtherm ida arat pln pts hwp hwp_act_window hwp_epp hwp_pkg_req pku ospke avx512_vnni md_clear flush_l1d arch_capabilities
```
#### appDirect set up
```
# set all optane to appDirect (DID through bios may not be needed, require reboot)
ipmctl create -goal persistentmemorytype=appdirect

# creating namespace from persistent namespace (align good for pages )
ndctl create-namespace --mode=fsdax --region=0 --align=2M
ndctl create-namespace --mode=fsdax --region=1 --align=2M

# make a file system
mkfs -t xfs /dev/pmem0
mkfs -t xfs /dev/pmem1

# mount file system make writeable
mkdir /mnt/pmem0
mkdir /mnt/pmem1

# mount in direct access mode
mount -o dax /dev/pmem0 /mnt/pmem0/
mount -o dax /dev/pmem1 /mnt/pmem1/

chmod 777 /mnt/pmem0
chmod 777 /mnt/pmem1
```

#### Numa settings
The standard [recommendation](https://code.kx.com/q/kb/linux-production/) when using numa is to set --interleave=all                                                                                                                  
` numactl --interleave=all q `
but found better performance in aligning the numa nodes with the persistent memorary namespaces
`numactl -N 0 -m 0  q -q -m /mnt/pmem0/` and `numactl -N 1 -m 1  q -q -m /mnt/pmem1/`

<a name="arcitecture"/>

### Testing Framework Arcitecture

![fig - Arcitecture of kdb stack](figs/stack.png)

#### Feed
Data arrives from a feed. Normally this would be a feedhandler publishing data from exchanges or vendors. For consistent testing we have simulated this feed from another q process. This process generates random data and publishes down stream. For the rate of data to send we looked and the largest day of market activity in 2020. which durring its last half hour of trading before the close consisted of 80,000,000 quote msgs and 15,000,000 trades. Code can be viewed [here](../src/q/feed.q)

#### Tp
Standard kdb tp running in batch mode. Code can be viewed [here](../src/q/tp)

#### AggEngine
This process is the main departure for standard tick set up. This process subscribes to standard trade and quote tables and calculates running daily and minute level stacks for all symbols. These aggregated tables are then published to the rdb from which they could then be queried.
This process wwas added in order to have some kind of more complex event processs as well as standard rdb. This process will constantly have to read and write to memory. where generally only has to write as it appends data and only read for queries) Code can be viewed [here](../src/q/aggEngine)

#### Rdb
Standard rdb subscribes to tables from the tp. We also added option to prestress the memory before our half hour of testing again looking at the market fata on 2020.03.02 there were 650,000,000 quote msgs and 85,000,000 trades at 15:30 so we insert these volumes into the rdb at start up.
This aims to ensure that the ram is already somewhat saturdated. Full code available[here](../src/q/tp)

#### Monitor
The Monitor process connects to the rdb and collect performance stats on a timer. Main measurements are for latency of quote table this will track if messages getting queued from the tp, quote stats table if this falls behind indicates issue in aggEngine and the query time which measures how long it takes to run some typical rdb queries .e.g aj
On start up this process also kicks off the feed once having succsesfully connected to rdb to start testing run.
Once endTime has been reached the stats collected are aggregated and written to csv. Again link for full code available [here](../src/q/monitorPerf.q)
