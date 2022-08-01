# Guide for Tuning Cassandra on 4th Generation Intel® Xeon® Scalable Processors 

## Introduction

This guide is for users who are already familiar with Cassandra.  It provides recommendations for configuring hardware and software that will provide the best performance in most situations. However, please note that we rely on the users to carefully consider these settings for their specific scenarios, since OVS and DPDK can be deployed in multiple ways. 

*(Describe Cassandra)*	

*(Describe pre-requisites)*
	
4th Generation Intel® Xeon® Scalable processors lead the industry in delivering workload-optimized platforms with built-in AI acceleration.  They provide seamless performance to speed the transformative impact of data from the intelligent edge to the multi-cloud. Improvements of particular interest to this workload are: 

- Enhanced Performance
- Enhanced Intel® Deep Learning Boost with VNNI
- More Intel® Ultra Path Interconnect
- Increased DDR4 Memory Speed & Capacity
- Intel® Advanced Vector Extensions
- Intel® Security Essentials and Intel® Security Libraries for Data Center 
- Intel® Speed Select Technology
- Support for Intel® Optane™ Persistent Memory 200 series

Tested hardware and software for this tuning guide include:

### Server Configuration

#### Hardware

The configuration described in this article is based on 4th Generation Intel® Xeon® processor hardware. The server platform, memory, hard drives, and network interface cards can be determined according to your usage requirements.

| Hardware | Model |
|----------------------------------|------------------------------------|
| Server Platform Name/Brand/Model | Intel® Server System  |
| CPU | Intel® Xeon® ???  | 
| BIOS | version # ???| 
| Memory | ??? | 
| Storage/Disks | ??? | 
| NIC (if it applies) | Intel® Ethernet Controller ??? | 

#### Software

| Software | Version |
|------------------|-------------|
| Operating System | Ubuntu 20.04 or later  | 
| Kernel | 5.13 or later | 
| Cassandra version | 3.11.x and 4.0.x |
| JDK 8 version | 1.8.190 or later on 3.x |
| JDK 11 and 14 version | Ubuntu openjdk versions |

			
## Hardware Tuning

*(Enter any other information on general system setup and tuning.)*

### BIOS Settings

*(Use this section to describe BIOS settings that should be changed in order to improve workload performance.)*

Begin by resetting your BIOS to default setting, then follow the suggestions below for changes to the default:

| Configuration Item | Recommended Value
|---------------------|------------------|
| Advanced/Power & Performance/CPU P State Control/CPU P State Control/Enhanced Intel SpeedStep® Tech | ??? |
| Advanced/Power & Performance/CPU Power and Performance Policy | ??? |
| Advanced/Memory Configuration/SNC (Sub-NUMA Clusters) | ??? |
| Advanced/Memory Configuration/Page Policy | ??? |
| Advanced/UPI Configuration/XPT Prefetch | ??? |
| Advanced/Processor Configuration/Direct-to-UPI (D2K) | ??? |

If no specific settings apply, then use this text: (No specific BIOS setting for this this workload – just use the defaults)

*If you recommend a BIOS setting, give an explanation of what the setting does and why it matters to the workload.  Here is an example from the HPC Cluster Tuning Guide:*

### Description of Settings *(Update or remove any of these)*

#### Sub-NUMA Cluster (SNC) 

SNC is a feature that provides similar localization benefits as Cluster-On-Die (COD), a feature found in previous processor families, without some of COD’s downsides. SNC breaks up the last level cache (LLC) into disjoint clusters based on address range, with each cluster bound to a subset of the memory controllers in the system. SNC improves average latency to the LLC and is a replacement for the COD feature found in previous processor families.

#### Direct-to-UPI (D2K)

D2U is a latency-saving feature for remote read transactions. With D2U enabled, the IMC will send the data directly to the UPI instead of going through the Caching and Home Agent (CHA), reducing latency. Keep enabled, although workloads that are highly NUMA-optimized or that use high levels of memory bandwidth are less likely to be affected by disabling D2U.

#### XPT (eXtended Prediction Table) Prefetch

Extended prediction table (XPT) Prefetch is a new capability that is designed to reduce local memory access latency. XPT Prefetch is an “LLC miss predictor” in each core that will issue a speculative DRAM read request in parallel to an LLC lookup, but only when XPT predicts a “miss” from the LLC lookup.
For more information, refer to the BIOS Setup Utility User Guide for the Intel® Server Board D50TNP and M50CYP Family.

### Memory Configuration/Settings

(Use this section to describe the optimum memory configuration.  Here are some questions to consider:  How many DIMMS per channel?  How many channels are used?  Is PMem appropriate for this workload?  If so how should it be configured?)

Example: At least 1 DIMM per memory channel needs to be populated. Lower cpu-utilization could be an issue if there are no DIMMs in a memory channel because of the contention from database cache.

If no specific settings apply, then use this text: (No specific workload setting)

### Storage/Disk Configuration/Settings

*(Are there any specific settings or recommendations for storage media?)*

If no specific suggestions apply, then use this text: (No specific workload setting)

### Network Configuration/Setting

*(Are there any specific settings or recommendations for the network setup?  Does your workload use multiple systems? Any advice on how many clients? For example, how much CPU and memory will they need and how they should it be setup?).* 

*If no specific suggestions apply, then use this text: (No specific workload setting for this topic)*

## Software Tuning 

Software configuration tuning is essential.  The general purpose, default settings are almost never yield the best performance for specialized workloads.

The following are best known practices shared within the Intel Java Performance Optimization team when testing Cassandra.

### Nomenclature

This section uses the following distinctions: 

•	Client side - applies only to client system running the load generator to stress test Cassandra
•	Server side - applies only to server system running one or more Cassandra processes
•	Cloud note - describes the differences between bare-metal and cloud instances
•	v3.x - applies only to 3.x version of Cassandra
•	v4.x - applies only to 4.x version of Cassandra

### Typical Cassandra Configuration:

Cy note:  Could this be a diagram?

client system < -----------Network--------- > server system(s) ------ > database 
(load generator)                                      (Cassandra Servers)     (local storage)

### Cassandra Directories and Files of Interest:

If multiple Cassandra processes are running on a server, there will be at least one CASSANDRA_SERVER_HOME# directory per process.  Server side has the entire Cassandra directory:

```
<CASSANDRA_SERVER_HOME#> 
|
|__bin
|    |__cassandra (startup cassandra script)
|    |__nodetool (node status and compaction stats)
| 
|__ conf
|    |__cassandra.yaml (many tuning parameters)
|    |__cassandra-env.sh (port # settings)
|    |__jvm.options (3.x all java setting)
|    |__jvm-server.options (4.x java general settings)
|    |__jvm8-server.options (4.x java 8 specific settings)
|    |__jvm11-sever.options (4.x java >8 specific settings)
|
|__tools
|    |__cqlstress-insanity-example.yaml (defines schema)
|    |
|    |__bin
|          |__cassandra-stress (load generator)
```

Client side only requires the workload generator:

```
<CASSANDRA_CLIENT_HOME> 
|
|
|__tools (used when running separate client/server)
|    |__cqlstress-insanity-example.yaml
|    |
|    |__bin
|          |__cassandra-stress

```

### Modifications to Linux Kernel/General settings on the Server Side

1. Change the Cassandra production level per these recommendations:  https://docs.datastax.com/en/dse/6.8/dse-dev/datastax_enterprise/config/configRecommendedSettings.html

Make sure to set the read_ahead kernel from 128(64KB) to 8(4KB) 


2. JAVA_HOME must be set for Cassandra to work properly.  If “echo $JAVA_HOME” does not point to a directory then set the path

Open ~/.bashrc file and add these file 2 lines as below:

```
	export JAVA_HOME=Path_to_Java_installation_folder
	export PATH=$JAVA_HOME/bin:$PATH
```

### Modifications to Cassandra Schema File on the Client Side

Cassandra comes with four data layout files or schemas.  The insanity schema represents the data environment for most customers. 

#### Insanity Schema

Locate the insanity schema at:  <CASSANDRA_CLIENT_HOME>/tools/cql-insanity-example.yaml.  

Change the compaction strategy that is specified when creating a new table using cassandra-stress in the file passed as “profile”.  Use the recommendations from DataStax for best overall performance.

From:

```LeveledCompactionStrategy```

To:

```SizeTieredCompactionStrategy``

By default Cassandra uses LZ4 compression with 64KB chunk block size (v3.x Cassandra) or 16KB chunk block size (v4.x Cassandra) for baseline performance.  

Default Insanity Schema:

```
) WITH compaction = { 'class':'SizeTieredCompactionStrategy' }
AND compression = { 'class' : 'LZ4Compressor', 'chunk_length_in_kb' : 64 }#for v3.x
AND compression = { 'class' : 'LZ4Compressor', 'chunk_length_in_kb' : 16 }#for v4.x
AND comment='A table of many types to test wide rows and collections'
```

### Modification to the Cassandra YAML Configuration File on Server Side

Locate and modify this file:  <CASSANDRA_SERVER_HOME#>/conf/cassandra.yaml

| Setting | Value |
|---------|-------|
| cluster_name |  <name_for_cluster> |
| seeds |  <your server's IP> |
| listen_address |  <your server’s IP> |
| rpc_address |  <your server’s IP> |
| data_file_directories |  <database_storage > (example /mnt/cass_db1) |
| commitlog_directory |  (example /mnt/cass_db1/commitlog) |
| cdc_raw_directory |  (example /mnt/cass_db1/cdc_raw) |
| concurrent_reads |  <#number of threads for reads>, optimal values of this is 2X-3X the number of VCPUs allocated for this process. |
| concurrent_writes |  There are negative effects in increasing this value beyond the number of VCPU (virtual CPU).  The root cause comes from a known software issue  https://issues.apache.org/jira/browse/CASSANDRA-13896.  As a workaround, use the same value as VCPU |


### Modification to Cassandra jvm Files on the Server Side

Apply the following to <CASSANDRA_SERVER_HOME #>/conf/jvm* file changes according to version supported.  Note: Cassandra 3.x only supports JDK 8 and under, Cassandra 4.0.x supports JDK versions 8 through 14.  There are compile issues with Java 15 or above:
https://issues.apache.org/jira/browse/CASSANDRA-16895

```
# comment out NUMA
#-XX:+UseNUMA

# we will use G1GC, so comment out all 
# CMS garbage collector for 3.X
	
### CMS Settings
#-XX:+UseParNewGC
#-XX:+UseConcMarkSweepGC
#-XX:+CMSParallelRemarkEnabled
#-XX:SurvivorRatio=8
#-XX:MaxTenuringThreshold=1
#-XX:CMSInitiatingOccupancyFraction=75
#-XX:+UseCMSInitiatingOccupancyOnly
#-XX:CMSWaitDuration=10000
#-XX:+CMSParallelInitialMarkEnabled
#-XX:+CMSEdenChunksRecordAlways
#-XX:+CMSClassUnloadingEnabled

# Use G1GC
-XX:+UseG1GC

# 
# This is the Cassandra process heap size 
# NOTE: the total number of CassandraProcesses*ProcessHeapSize # should take between 25-50% of total system memory, we have 
# not seen any benefits higher than 64GB with G1GC
# PERFORMANCE NOTE: do not use 32G-38G, explained here: 
# http://java-performance.info/over-32g-heap-java/
#  
-Xms31G
-Xmx31G
Modification to the Cassandra-env file on the Server Side:
No modification are needed to this file if running one Cassandra process on the server.  When more than one is used, each Cassandra process must have a unique JMX_PORT, hence the following file should be modified, <CASSANDRA_SERVER_HOME#>/conf/cassandra-env.sh: 
JMX_PORT=”7199” # 7199 for instance 1, JMX_PORT=”7299” for instance 2, etc. 
Note: The following port numbers must be opened: 7000, 7001, 7199, 9042, 9160. Commands to open ports:
•	sudo firewall-cmd  --list-all
•	sudo firewall-cmd  --zone=public --add-port=7000/tcp --permanent
•	sudo firewall-cmd --reload
•	<repeat for the rest of the ports>
=================================================================
Performance Option:
Cassandra scales well till about 40 VCPU (physical and hyperthreaded cores), hence for very large system with >40 VCPU or multiple CPU sockets, multiple Cassandra processes are recommended to use resources efficiently. Example below is running four individual Cassandra processes each to an nvme device:
```

### NUMA considerations

Cloud Note: Cloud providers typically do not let you modify NUMA and may only have one numa node for the entire instance hence this NUMA section can be skipped for the cloud.

Each Cassandra instance can be pinned to distinct CPU and NUMA memory region if 4 NUMA nodes are supported, each Cassandra process is pinned to ½ socket and ½  memory on the socket.   The cassandra file in cassandra/bin directory of each of the four Cassandra process was modified as highlighted below: 

```
<CASSANDRA_HOME_INST0>/bin/cassandra file compute and memory bind to NUMA 0:
NUMACTL_ARGS=””
if which numactl >/dev/null 2>/dev/null && numactl $NUMACTL_ARGS ls / >/dev/null 2>/dev/null
then
     NUMACTL=" numactl -m 0 -N 0 $NUMACTL_ARGS"
else
     NUMACTL=""
fi

<CASSANDRA_HOME_INST1>/bin/cassandra file compute and memory bind to NUMA 1:
NUMACTL_ARGS=””
if which numactl >/dev/null 2>/dev/null && numactl $NUMACTL_ARGS ls / >/dev/null 2>/dev/null
then
     NUMACTL=" numactnvl -m 1 -N 1  $NUMACTL_ARGS"
else
     NUMACTL=""
fi

<CASSANDRA_HOME_INST2>/bin/cassandra file compute and memory bind to NUMA 2:
NUMACTL_ARGS=””
if which numactl >/dev/null 2>/dev/null && numactl $NUMACTL_ARGS ls / >/dev/null 2>/dev/null
then
     NUMACTL=" numactl -m 2 -N 2 $NUMACTL_ARGS"
else
     NUMACTL=""
fi

<CASSANDRA_HOME_INST3>/bin/cassandra file compute and memory bind to NUMA 3:
NUMACTL_ARGS=””
if which numactl >/dev/null 2>/dev/null && numactl $NUMACTL_ARGS ls / >/dev/null 2>/dev/null
then
     NUMACTL=" numactl -m 3 -N 3  $NUMACTL_ARGS"
else
     NUMACTL=""
fi
```

To find the specific numa details of your system run the following command, you may need to change the number of NUMA node supported in the System BIOS:

```
      numactl -H
```

### Optional Performance: Including multiple network connection on one Network card

Each Cassandra instance requires a unique IP Address.  Thus to support multiple instances on a Network port, IP aliasing is used and add the number of network addresses to one NIC port:

Network Alias (one IP address per instance)	ifconfig eth0:0 <new ip addr for Cassandra2> up
ifconfig eth0:1 <new ip addr for Cassandra3> up
..
Ifconfig eth0:n <new ip addr for CassandraN> up


### Running the Cassandra process on the Server Side

```
<CASSANDRA_SERVER_HOME#>/bin/cassandra -R
Stopping the Cassandra process on the Server Side:

killall –w <pid for cassandra>

or to stop all the Cassandra processes: 

killall –w java 
```
 
### Cassandra-stress initial table creation

Cassandra requires you to build a dataset on the server in order to read and write to the dataset.  To build your own dataset on one Cassandra process:

```
<CASSANDRA_HOME>/tools/bin/cassandra-stress user profile=<CASSANDRA_HOME>/tools/cqlstress-xyz.yaml ops\(insert=1\) no-warmup cl=ONE n=<number of entries in the DB> -mode native cql3 –pop seq=1..<number of entries in DB> -node <your server’s IP> -rate threads=<load>
```

user profile Designate the YAML file to use with cassandra-stress. 

no-warmup Do not warmup the process, do a cold start.

cl=ONE – consistency level. A write must be written to the commit log and memtable of at least one replica node. This is for one node cluster case.   If multiple nodes are used in a cluster for example you can make multiple copies of the data.

n= number of entries in database

pop seq=1..<number of entries> – sequentially distributed entry  

node - specify port for connecting Cassandra nodes.

rate threads – # of outstanding write commands on the server

To accelerate compaction (background database cleanup), modify <CASSANDRA_HOME>/conf/cassandra.yaml before starting Cassandra 

| Settings | Value|
|------|------|
| concurrent_compactors | <#physical cores> |
| compaction_throughput_mb_per_sec | 0 |


Remember to change these settings back to default after compaction is done:

•	After the initial table has been created, make sure to wait until the compaction have finished before stopping Cassandra

•	Flush Cassandra memory buffers to disk: <CASSANDRA_HOME>/bin/nodetool flush

•	Wait until all compaction jobs are done: <CASSANDRA_HOME>/bin/nodetool compactionstats

•	Once compaction is done, it’s safe to turn Cassandra off and save a copy of the Cassandra data directory as a backup if needed

### Cassandra-Stress Benchmark parameters for reading a node

<CASSANDRA_HOME>/tools/bin/cassandra-stress user profiile=<CASSANDRA_HOME>/tools/cqlstress-xyz.yaml ops\(simple1=1\) no-warmup cl=ONE duration=#s<number of seconds> -mode native cql3 –pop dist=uniform\(1..<number of entries in DB>\) -node <your server’s IP> -rate threads= number_of_client_threads

### Cassandra-Stress Benchmark parameters for write on one node

<CASSANDRA_HOME>/tools/bin/cassandra-stress user profile=<cassandra_directory>/tools/cqlstress-xyz.yaml ops\(insert=1\) no-warmup cl=ONE duration=#s<number of seconds> -mode native cql3 –pop dist=uniform\(1..<number of entries in DB>\) -node <your server’s IP> -rate threads=number_of_client_threads

### Cassandra-Stress Benchmark parameters for mix 80/20 (80% read, 20% write) on one node

<CASSANDRA_HOME>/tools/bin/cassandra-stress user profiile=<CASSANDRA_HOME>/tools/cqlstress-xyz.yaml ops\(insert=20,simple1=80\) no-warmup cl=ONE duration=#s<number of seconds> -mode native cql3 –pop dist=uniform\(1..<number of entries in DB>\) -node <yur server’s IP> -rate threads= number_of_client_threads

### Optional Tools for debugging Performance issues

The following tool captures configuration details from the system:  
https://github.intel.com/ssgcce/svrinfo

The following tool captures CPU/Memory/Disk/Network performance details using Linux’s performance monitor tools:
https://github.com/intel-hadoop/PAT

## FAQ

1.	Where can I download Cassandra?
As of mid 2022, the current official release version is 3.11.x and 4.0.x. Download binaries and source code from http://archive.apache.org/dist/cassandra/  GitHub is also a good resource place https://github.com/apache/cassandra, but you will need to build from source.

2.	Where can get kernel different than one that comes with OS? 
https://www.kernel.org/

3	How often do I need to change/update Cassandra?
Stick with one that works for you and your customers are using. Change only where significant improvement in performance between two versions is or required by your customer. 

3.	Is there a minimum resource per Cassandra process?
It is recommend to run at minimum 8 VCPU and 32GB of memory per Cassandra Server process.  We typically run with more than twice this for optimal performance.  We also run databases where the database capacity is 2x-8x the size of DRAM.  This ensures we exercise both DRAM and Disk like in the typical Cassandra use case.

4.	What should I use as heap size?
We are following DataStax (key contributor to Cassandra) recommendations “For a node using G1, the Cassandra community recommends a MAX_HEAP_SIZE as large as possible, up to 64 GB.”  

5.	What are storage requirements?
In general the faster storage than better unless you want to do comparisons between different drive types. Storage size need to be appropriate to your datasize. In our set up we are using SSDs for the dataset.  Cassandra is IO intensive and a Cassandra process will be IO bound if there is only one standard flash nvme devices per Cassandra process of >20 VCPUs. 

6.	Do I need to rebuild the dataset when I change configuration (drives, CPUs, or mem)?
No. If you have working dataset on one drive and need to get the same dataset to different type of drives, just copy it.

7.	What are network requirements?
1Gbe NIC will limit your throughput, 10Gbe or higher is recommend.

8.	Can I have client and server on the same machine? 
Yes. Your data packages will not be transfer over network. However, if you want to reproduce and report Cassandra performance real use cases, you want to have client and server on different machines.   

9.	How to specify how many entries I want to feed into my data base to get certain 
dataset size?
If you use recommended cqlstress-insanity-example.yaml file to create your data set, it 
will create files of approx. 670 bytes per compressed partition on disk. Here are some estimates:
For n=600 Million partitions it will built a ~400GB compressed dataset on disk


## Sample script for setting up environment with 4 NVME deices

```
ulimit -n 1000000
ulimit -l unlimited

sysctl -w \
net.ipv4.tcp_keepalive_time=60 \
net.ipv4.tcp_keepalive_probes=3 \
net.ipv4.tcp_keepalive_intvl=10
sysctl -w \
net.core.rmem_max=16777216 \
net.core.wmem_max=16777216 \
net.core.rmem_default=16777216 \
net.core.wmem_default=16777216 \
net.core.optmem_max=40960 \
net.ipv4.tcp_rmem='4096 87380 16777216' \
net.ipv4.tcp_wmem='4096 65536 16777216'

for CPUFREQ in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
do
            [ -f $CPUFREQ ] || continue
            echo -n performance > $CPUFREQ
done

echo 0 > /proc/sys/vm/zone_reclaim_mode
swapoff --all
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

touch /var/lock/subsys/local
echo none > /sys/block/nvme1n1/queue/scheduler
echo none > /sys/block/nvme2n1/queue/scheduler
echo none > /sys/block/nvme3n1/queue/scheduler
echo none > /sys/block/nvme4n1/queue/scheduler

echo 0 > /sys/class/block/nvme1n1/queue/rotational
echo 0 > /sys/class/block/nvme2n1/queue/rotational
echo 0 > /sys/class/block/nvme3n1/queue/rotational
echo 0 > /sys/class/block/nvme4n1/queue/rotational

echo 8 > /sys/class/block/nvme1n1/queue/read_ahead_kb
echo 8 > /sys/class/block/nvme2n1/queue/read_ahead_kb
echo 8 > /sys/class/block/nvme3n1/queue/read_ahead_kb
echo 8 > /sys/class/block/nvme4n1/queue/read_ahead_kb

ifconfig eno1:1 134.134.101.218 up # need multiple IP addresses Process2
ifconfig eno1:2 134.134.101.219 up # need multiple IP addresses Process3
ifconfig eno1:3 134.134.101.220 up # need multiple IP addresses Process4
```

## Best Practices for Testing and Verification (optional section)

*In this section you can list any BKMs you have for running the workload. Example below is from the WorkPress Tuning Guide:*

## Conclusion

*Use this section as a brief wrap-up for the guide.* 

## References (optional, as needed)

## Feedback (required section)

We value your feedback. If you have comments (positive or negative) on this guide or are seeking something that is not part of this guide, please reach out and let us know what you think. 
 
