# Apache Cassandra on Azure VMs Performance Experiments
September 2019

## Overview
Many customers run Apache Cassandra in Azure and are looking for experiment-based guidance for tuning Cassandra on Azure VMs. This repo summarizes learnings from performing various relative performance tests of Apache Cassandra on different Azure VM configurations to answer a few common questions such as: 
* What stripe/chunk size should I use for Cassandra data disks?
* What is the impact of data disk caching? 
* Will my cluster performance scale linearly if I add another Cassandra data center to my ring in another Azure region?
* Is there a performance difference between `ext4` and `xfs` filesystems?

The goal of this repo is to share interesting learnings to help increase knowledge around how Apache Cassandra will behave and perform on various Azure VM configurations. It is not meant to be used as a benchmark of Cassandra on Azure, but rather as a summary of observations and conclusions from micro-scale tests comparing relative performance observed when tuning Azure VM and Cassandra configurations.

## Test setup and methodology
The Azure Cosmos DB service offers *"Cassandra to Cosmos Exchange (CCX)"* which is designed to enable large Enterprise customers to use hybrid Cassandra migrations or runtime scenarios where Cosmos DB augments Cassandra clusters. *CCX* provides specially-designated Azure VM SKUs: `Special_CCX_DS14_v2` and `Special_CCX_DS13_v2`. These Azure VM sizes are performance-equivalent to the usual Azure `Standard_DS14_v2` and `Standard_DS13_v2` [VM sizes](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/sizes-memory#dsv2-series-11-15) but are hosted on the same infrastructure as Cosmos DB itself, thereby providing the Cosmos DB service team increased control over the updates and maintenance of these compute clusters. 

By default, you will **not** see `Special_CCX` VM SKUs in your subscription. If you are interested in the *CCX offer*, please email askcosmosdb@microsoft.com with a description of your Cassandra on Azure scenario.

In these tests, both the `Special_CCX_DS14_v2` and `Standard_DS14_v2` sizes were used interchangeably, depending on the Azure region where the Cassandra rings were deployed. Therefore, expect to see comparable performance numbers, even if using `Standard_DS14_v2` VMs.

Apache Cassandra's (version 3.11.4) standard `cassandra-stress` tool is used throughout all tests.

### Test setup
| Parameter | Value | Comments |
| ------- | ------- | ------- |
| Cassandra Node VMs | 6 x DS14_v2| VM disk throttles are: uncached 51k IOPS/768MBps, cached 64k/576MBps. Was also tested using Special_CCX_DS14_v2 VM size which has performance identical to DS14_v2. |
| Data disks | 4 x P30 (1TB) | P30 disk throttle is 5k/200MBps |
| Disk Caching | None and ReadOnly | For many VMs the throughput throttle for uncached disks is higher than cached due to limits of the *Azure Blob Cache* cache. Average latency is higher with uncached disks since all IOs go to backend storage. Tested both DiskCaching=None and DiskCaching=ReadOnly with empty and full host cache. See [Disk Caching learnings](docs/cassandra-azure-vm-data-disk-caching.md). |
| Accelerated Networking | Enabled | Accelerated Networking decreases network latency from ~250us to ~50us and improves throughput, allowing DS14_v2 to reach ~11Gbps (1.2GB/s) with single-stream iperf3 |
| Linux distro and version | CentOS 7.5 | A modern Linux distro is required for latest drivers and the ability to use Accelerated Networking. Latest versions of Ubuntu 16.x and 18.x could also be used. The main reason we used CentOS 7.5 was because it matched one of specific customer scenarios. |
| Cassandra Version | Apache Cassandra 3.11.4 | Repo base URL ([link](https://www.apache.org/dist/cassandra/redhat/311x/)) |
| mdadm chunk size | 256KB, 128KB, 64KB, 4KB | The default value in mdadm is 256k and was tested initially. Also tested 4k (outlier), 64k, and 128k chunk sizes to see if there was any corresponding difference in performance. See [Chunk Sizes learnings](docs/cassandra-mdadm-chunk-sizes.md). |
| commitlog | Local SSD, Premium Disk | Specific customer scenario was using local/ephemeral SSD, likely because of assumption that it provides fastest writes due to low latency of local disk. However, since these local disks are not durable, a VM host crash could cause the commitlog on local/ephemeral disk to be lost. Therefore, we also tested writes with commitlog on attached Premium data disks to assess relative impact to performance. |
| SSTables | RAID 0, XFS | mdadm RAID 0 with various chunk sizes above, XFS filesystem with the default 4096 byte block size. |
| Cassandra Java Garbage Collection | UseG1GC | Default is CMS GC, but during testing we noticed syscpu on some nodes spiking to 90% and the node becoming unresponsive for some time, presumably during GC pauses. With G1GC, high syscpu is much less frequent, but still occurs sometimes. This requires further investigation. Early straces show Java process doing lots of futex and epoll_wait syscalls during the pause. |
| Cassandra-Stress client VMs | 1 x DS14_v2 (usually with 256KB for writes and 128 for reads) | Client VM is deployed in the same VNet and on a separate subnet. It connects to all 6 Cassandra nodes with up to 128 connections per node and uses 128 threads with no throttling. Threads could be increased higher which may increase the ops/sec for some scenarios, including with more clients.  That said, in most scenarios we tested, client CPU and network were not the bottleneck. |

For more details see [setup of Azure VMs used in Cassandra tests](docs/cassandra-vm-setup.md).

## Learnings and Observations
* [General validation of baseline VM performance](docs/baseline-vm-perf.md)
* [Observations on Cassandra usage of Linux page caching](docs/cassandra-linux-page-caching.md)
* [Comparing impact of disk read-ahead settings](docs/cassandra-read-ahead.md)
* [Comparing relative performance of various Cassandra document sizes](docs/cassandra-doc-sizes.md)
* [Comparing Azure VM data disk caching configurations](docs/cassandra-azure-vm-data-disk-caching.md)
* [Measuring impact of mdadm chunk sizes](docs/cassandra-mdadm-chunk-sizes.md)
* [Comparing relative performance of various replication factors](docs/cassandra-replication-factors.md)
* [Measuring impact of multi-DC replication](docs/cassandra-multi-dc-azure-regions.md)
* [Observations on hinted handoff in cross-region replication](docs/cassandra-hinted-handoff-cross-region.md)
* [Comparing performance of Azure local/ephemeral vs attached/persistent disks](docs/cassandra-local-attached-disks.md)
* [Observations on ext4 and xfs filesystems and compressed commitlogs](docs/cassandra-commitlogs-xfs-ext4.md)

## Appendix

* [Sample for deploying Apache Cassandra on Azure IaaS and running cassandra-stress tests](https://github.com/azure-Samples/cassandra-on-azure-vms-deployment-and-stress-test) (GitHub)

