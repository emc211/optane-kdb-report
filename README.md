# check for TODOs
# Expanding Capacity of real time kdb tick stack with optane

## Abstract
- summary

Table of Contents  

[Intro](#head1)  
[Hardware](#head2)  

<a name="head1"/>

## Intro/Background

- previous posts how this is diffetent
- description of test case

<a name="head2"/>

## Hardware

-server details TODO evgueny provide details

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
### appDirect set up 
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

#This will mount in direct axcess mode
mount -o dax /dev/pmem0 /mnt/pmem0/
mount -o dax /dev/pmem1 /mnt/pmem1/

# sector mode (maybe better to specidfy sector mode)
mount /dev/pmem0 /mnt/pmem0/
mount /dev/pmem1 /mnt/pmem1/

chmod 777 /mnt/pmem0
chmod 777 /mnt/pmem1
```

### Numa settings 
The standard [recommendation](https://code.kx.com/q/kb/linux-production/) when using numa is to set --interleave=all                                                                                                                  
` numactl --interleave=all q ` 
but found better performance in aligning the numa nodes with the persistent memorary namespaces
`numactl -N 0 -m 0  q -q -m /mnt/pmem0/` and `numactl -N 1 -m 1  q -q -m /mnt/pmem1/`
TODO link to fuller explanation


## Testing Framework - description of testing stack 
The Framework designed to test how optane chips can be deployed is a common market data capture solution. Based on the standard [tick setup](https://github.com/KxSystems/kdb-tick)

### Feed
Data arrives from a feed. Normally this would be a feedhandler publishing data from exchanges or vendors. For consistent testing we have simulated this feed from another q process. This process generates random data and publishes down stream. For the rate of data to send we looked and the largest day of market activity in 2020. which durring its last half hour of trading before the close consisted of 80,000,000 quote msgs and 15,000,000 trades 

### Tp
Standard kdb tp running in batch mode.

### AggEngine
This process is the main departure for standard tick set up. This process subscribes to standard trade and quote tables and calculates running daily and minute level stacks for all symbols. These aggregated tables are then published to the rdb from which they could then be queried. 
This process wwas added in order to have some kind of more complex event processs as well as standard rdb. This process will constantly have to read and write to memory. where generally only has to write as it appends data and only read for queries)

### Rdb
Standard rdb subsribes to tables from the tp. We also added option to prestress the memory before our half hour of testing again looking at the market fata on 2020.03.02 there were 650,000,000 quote msgs and 85,000,000 trades at 15:30 so we insert these volumes into the rdb at start up.
This aims to ensure that the ram is already somewhat saturdated.

### Monitor
The Monit

![fig.1- Arcitecture of kdb stack](figs/stack.png)

Wrapper scripts to start these stacks were then implemented

## Findings/Results

This test was run with ticker plant in a 1s batch publish mode. And Querys hitting the rdbs every 2 seconds.

![fig.2 - Compare max quote time](figs/compMqtl.png)

In this case although we see spikes in the maximium latency for the appDirect rdb.
The latency remains very small through out. And is able to keep ingesting data from the tp without falling behind.

The issue is whenever we decrease the batching and the query the rdb more regularly. Simulating a more latency sensitive system
Here the tp was publish in 50ms batchs and querys running on rdbs every 1 second.

![fig.3 - Compare max quote time](figs/compMqtl2.png)

We can see now that the appDirect rdb falls very far behind compared to the DRAM rdb.

## Conclusion

