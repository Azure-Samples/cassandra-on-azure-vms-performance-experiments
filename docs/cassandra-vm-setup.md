# Setup of Azure VMs used in Cassandra tests

For Cassandra test clusters, we deployed 6 "OpenLogic:CentOS:7.5:latest" Special_CCX_DS14_v2/Standard_DS14_v2 VMs into an Availability Set, each VM had 4 attached P30 (1TB) premium managed disks (caching was either None or ReadOnly depending on the tests), and Accelerated Networking enabled. For client VM, we used Standard_DS14_v2 in the same VNet but in a separate subnet (i.e. /vnet1/default for servers and /vnet1/clients for clients).

After deploying the VMs, we updated CentOS, installed generic Linux perf test tools, and setup RAID0 stripe set:

```
yum update -y

# Install perf test tools
yum install -y epel-release
yum install -y dstat iperf3 fio qperf sockperf ioping pssh

# Create mdadm stripe set (RAID 0) with chunk size 128 (we tested various chunk sizes)
mdadm --verbose --create /dev/md0 --metadata=1.2 --chunk=128 --level=raid0 --raid-devices=4 /dev/sd[cdef]

mdadm --verbose --detail --scan > /etc/mdadm.conf

mdadm --verbose --detail /dev/md0

# Set persistent read-ahead and rotational setting via udev rules
cat > /etc/udev/rules.d/67-azure-setra.rules << EOL
SUBSYSTEM=="block", ACTION=="add|change", KERNEL=="sd[a-z]|md*", ATTR{queue/rotational}="0", ATTR{queue/read_ahead_kb}="8"
EOL

for x in /sys/block/sd[a-z]
do
    udevadm test --action=add $x
done

for x in /sys/block/md*
do
    udevadm test --action=add $x
done

# For some reason, when VM is rebooted the udev rules do not apply as expected probably due to a timing issue
# As a workaround, add delayed partprobe call to /etc/rc.d/rc.local to force udev rule to detect "change"
echo "sleep 60" >> /etc/rc.d/rc.local
echo "partprobe" >> /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local

# Create XFS filesystem on the RAID0 device to use for Cassandra data (add -f flag to force recreate the filesystem if it already exists on the device)
mkfs.xfs /dev/md0

# Update /etc/fstab to mount /dev/md0 as /data with minimal mount options
mkdir -p /data
echo '/dev/md0  /data xfs defaults,nofail   0 0' >> /etc/fstab

# Mount all
mount -a

# List block devices to confirm /data is mounted
lsblk
``` 

In our tests, we used Apache Cassandra's version 3.11.4 installed via yum:

```
# Install OpenJDK 1.8.0
yum install -y java-1.8.0-openjdk
java -version

# Configure Cassandra 3.11 yum repo
cat > /etc/yum.repos.d/cassandra.repo << EOL
[cassandra]
name=Apache Cassandra
baseurl=https://www.apache.org/dist/cassandra/redhat/311x/
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://www.apache.org/dist/cassandra/KEYS
EOL

# Install Cassandra 3.11
yum install -y cassandra

# Refresh services
systemctl daemon-reload
```

We used GossipingPropertyFileSnitch to configure each Cassandra node in a "rack" determined by that VM's "fault domain" within the Availability Set. Fault domain was obtained from inside of the VM using [Azure Instance Metadata Service](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/instance-metadata-service) endpoint. 

Script below includes specific **cassandra.yaml** and **jvm.options** configurations used in most tests.

Due to a specific customer scenario, in most tests, commitlog was on the ephemeral/local disk (i.e. mounted as /mnt/resource on CentOS 7.x in Azure). However, this is not-durable since if host crashes and/or VM gets migrated to another host, commitlog on the local/ephemeral disk is lost.

```
#!/bin/sh

DATE=`date '+%Y%m%d%H%M%S'`

CLUSTER_NAME="your_cluster_name"

# Get VM private IP address from the Azure Instance Metadata Service endpoint
LISTEN_ADDRESS=$(curl -s -H Metadata:true "http://169.254.169.254/metadata/instance/network/interface/0/ipv4/ipAddress/0/privateIpAddress?api-version=2017-08-01&format=text")

# Get VM fault domain from the Azure Instance Metadata Service endpoint
FAULT_DOMAIN=$(curl -s -H Metadata:true "http://169.254.169.254/metadata/instance/compute/platformFaultDomain?api-version=2017-08-01&format=text")

# Include IP addresses of seed nodes with at least one per data center
SEEDS="10.0.0.4"

# Data Center
DC="dc1"

# Create directories
DATA_PATH=/data/cassandra
COMMITLOG_PATH=/mnt/resource/cassandra

mkdir -p $DATA_PATH
chown cassandra:cassandra $DATA_PATH

mkdir -p $COMMITLOG_PATH
chown cassandra:cassandra $COMMITLOG_PATH

# Configure snitch file
cat > /etc/cassandra/conf/cassandra-rackdc.properties << EOL
dc=${DC}
rack=rack${FAULT_DOMAIN}
EOL

# Backup previous cassandra.yaml
cp /etc/cassandra/conf/cassandra.yaml /etc/cassandra/conf/cassandra-backup-$DATE.yaml

# Create new cassandra.yaml
cat > /etc/cassandra/conf/cassandra.yaml << EOL
cluster_name: ${CLUSTER_NAME}
num_tokens: 64 # original is 256
hinted_handoff_enabled: true
max_hint_window_in_ms: 10800000
hinted_handoff_throttle_in_kb: 102400 # original is 1024, setting to 100x (probably too much) larger value for faster replay of hints in multi-dc
max_hints_delivery_threads: 32 # original is 2, setting to larger value for faster replay of hints in multi-dc
batchlog_replay_throttle_in_kb: 1024
authenticator: AllowAllAuthenticator
authorizer: AllowAllAuthorizer
permissions_validity_in_ms: 2000
partitioner: org.apache.cassandra.dht.Murmur3Partitioner
disk_failure_policy: stop
commit_failure_policy: stop
key_cache_size_in_mb: 1024 # original is empty=auto=min(5% of Heap, 100MB)
key_cache_save_period: 14400
row_cache_size_in_mb: 0
row_cache_save_period: 0
counter_cache_size_in_mb:
counter_cache_save_period: 7200
commitlog_sync: periodic
commitlog_sync_period_in_ms: 10000
#commitlog_compression: # default is off, but in the ext4/xfs tests we tested commitlog compression since it improves performance
# - class_name: LZ4Compressor
commitlog_segment_size_in_mb: 32
seed_provider:
 - class_name: org.apache.cassandra.locator.SimpleSeedProvider
   parameters:
    - seeds: ${SEEDS}
concurrent_reads: 64 # original is 32
concurrent_writes: 64 # original is 32
concurrent_counter_writes: 64 # original is 32
memtable_allocation_type: offheap_objects # original is heap_buffers
index_summary_capacity_in_mb:
index_summary_resize_interval_in_minutes: 60
trickle_fsync: true # original is false
trickle_fsync_interval_in_kb: 10240
storage_port: 7000
ssl_storage_port: 7001
listen_address: ${LISTEN_ADDRESS}
start_native_transport: true
native_transport_port: 9042
start_rpc: true # original is false
rpc_address: 0.0.0.0 # original is localhost
rpc_port: 9160
rpc_keepalive: true
rpc_server_type: sync
thrift_framed_transport_size_in_mb: 15
incremental_backups: false
snapshot_before_compaction: false
auto_snapshot: true
tombstone_warn_threshold: 1000
tombstone_failure_threshold: 100000
column_index_size_in_kb: 64
batch_size_warn_threshold_in_kb: 5
unlogged_batch_across_partitions_warn_threshold: 10
compaction_throughput_mb_per_sec: 128 # original is 16
compaction_large_partition_warning_threshold_mb: 100
sstable_preemptive_open_interval_in_mb: 50
read_request_timeout_in_ms: 5000
range_request_timeout_in_ms: 10000
write_request_timeout_in_ms: 2000
counter_write_request_timeout_in_ms: 5000
cas_contention_timeout_in_ms: 1000
truncate_request_timeout_in_ms: 60000
request_timeout_in_ms: 10000
cross_node_timeout: false
endpoint_snitch: GossipingPropertyFileSnitch # original is SimpleSnitch
dynamic_snitch_update_interval_in_ms: 100
dynamic_snitch_reset_interval_in_ms: 600000
dynamic_snitch_badness_threshold: 0.1
request_scheduler: org.apache.cassandra.scheduler.NoScheduler
server_encryption_options:
  internode_encryption: none
  keystore: conf/.keystore
  keystore_password: cassandra
  truststore: conf/.truststore
  truststore_password: cassandra
client_encryption_options:
  enabled: false
  optional: false
  keystore: conf/.keystore
  keystore_password: cassandra
internode_compression: all # original is dc
inter_dc_tcp_nodelay: false
memtable_flush_writers: 2 # original is commented out
concurrent_compactors: 4 # original is commented out
otc_coalescing_strategy: DISABLED # original is commented out
phi_convict_threshold: 12 # original is commented out
commitlog_directory: ${COMMITLOG_PATH}/commitlog
saved_caches_directory: ${COMMITLOG_PATH}/saved_caches
hints_directory: ${DATA_PATH}/hints
cdc_raw_directory: ${DATA_PATH}/cdc
file_cache_size_in_mb: 2048 # original is commented out or 512
data_file_directories:
 - ${DATA_PATH}/data
broadcast_rpc_address: ${LISTEN_ADDRESS}
EOL

# Backup previous jvm.options
cp /etc/cassandra/conf/jvm.options /etc/cassandra/conf/jvm-backup-$DATE.options

# Create new jvm.options primarily to configure G1GC garbage collection instead of the default CMS GC
cat > /etc/cassandra/conf/jvm.options << EOL
-ea
-XX:+UseThreadPriorities
-XX:ThreadPriorityPolicy=42
-XX:+HeapDumpOnOutOfMemoryError
-Xss256k
-XX:StringTableSize=1000003
-XX:+AlwaysPreTouch
-XX:-UseBiasedLocking
-XX:+UseTLAB
-XX:+ResizeTLAB
-XX:+UseNUMA
-XX:+PerfDisableSharedMem
-Djava.net.preferIPv4Stack=true
-XX:+UseG1GC
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintHeapAtGC
-XX:+PrintTenuringDistribution
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintPromotionFailure
#-XX:PrintFLSStatistics=1
#-Xloggc:/var/log/cassandra/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=10M
EOL
```

Start Cassandra service on the seed node first, wait for it to start by tailing the log, and then start the service on the rest of the nodes one-by-one waiting for each one to bootstrap:

```
service cassandra start
tail -f /var/log/cassandra/system.log

nodetool status
```

Between some of the read tests, we used the following approach to restart Cassandra service and clear Linux page caches (see [more](cassandra-linux-page-caching.md))

```
nodetool flush 
service cassandra stop
free; sync; echo 3 > /proc/sys/vm/drop_caches; free
service cassandra start
sleep 15
nodetool status
```

## Next

Return to [Learnings and Observations](../README.md#learnings-and-observations) table of contents