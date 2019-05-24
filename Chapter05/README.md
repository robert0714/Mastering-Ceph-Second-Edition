# RADOS POOLS AND CLIENT ACCESS
Ceph provides a variety of different pool types and configurations. It also supports several different data-storage types to offer storage to clients. This chapter will look at the differences between replicated and erasure-coded pools, giving examples of the creation and maintenance of both. We will then move on to how to use these pools for the three data-storage methods: RADOS Block Device (RBD), object, and CephFS. Finally, we will finish with a look at how to take snapshots of the different types of storage methods. The following topics are covered in this chapter:

* Pools
* Ceph storage types

## Pools
RADOS pools are the core part of a Ceph cluster. Creating a RADOS pool is what drives the creation and distribution of the placement groups, which themselves are the autonomous part of Ceph. Two types of pools can be created, replicated, and erasure-coded, offering different usable capacities, durability, and performance. RADOS pools can then be used to provide different storage solutions to clients via RBS, CephFS, and RGW, or they can be used to enable tiered performance overlaying other RADOS pools.

### Replicated pools
Replicated RADOS pools are the default pool type in Ceph; data is received by the primary OSD from the client and then replicated to the remaining OSDs. The logic behind the replication is fairly simple and requires minimal processing to calculate and replicate the data between OSDs. However, as the data is replicated in whole, there is a large write penalty, as the data has to be written multiple times across the OSDs. By default, Ceph will use a replication factor of 3x, so all data will be written three times; this does not take into account any other write amplification that may be present further down in the Ceph stack. This write penalty has two main drawbacks: It obviously puts further I/O load on your Ceph cluster, as there is more data to be written, and in the case of SSDs, these extra writes will wear out the flash cells more quickly. However, as we will see in the section, Erasure-code pools, for smaller I/Os, the simpler replication strategy actually results in lower total required operations—there is always a fixed 3x write penalty, no matter the I/O size.

It should also be noted that although all replicas of an object are written to during a client write operation, when an object is read, only the primary OSD holding a copy of the object is involved. A client also only sends the write operation to the primary OSD, which then sends the operation to the remaining replicas. There are a number of reasons for this behavior, but they largely center around ensuring the consistency of reads.

As mentioned, the default replication size is 3, with a required minimum size of two replicas to accept client I/O. Decreasing either of these values is not recommended, and increasing them will likely have minimal effects on increasing data durability, as the chance of losing three OSDs that all share the same PG is highly unlikely. As Ceph will prioritize the recovery of PGs that have the fewest copies, this further minimizes the risk of data loss, therefore, increasing the number of replica copies to four is only beneficial when it comes to improving data availability, where two OSDs sharing the same PG can be lost and allow Ceph to keep servicing client I/O. However, due to the storage overhead of four copies, it would be recommended to look at erasure coding at this point. With the introduction of NVMes, which due to their faster performance reduce rebuild times, using a replica size of 2 can still offer reasonable data durability.

To create a replicated pool, issue a command, such as the one in the following example:

```
ceph osd pool create MyPool 128 128 replicated

```

This would create a replicated pool with 128 placement groups, called MyPool.

### Erasure code pools
Ceph's default replication level provides excellent protection against data loss by storing three copies of your data on different OSDs. However, storing three copies of data vastly increases both the purchase cost of the hardware and the associated operational costs, such as power and cooling. Furthermore, storing copies also means that for every client write, the backend storage must write three times the amount of data. In some scenarios, either of these drawbacks may mean that Ceph is not a viable option.

Erasure codes are designed to offer a solution. Much like how RAID 5 and 6 offer increased usable storage capacity over RAID 1, erasure coding allows Ceph to provide more usable storage from the same raw capacity. However, also like the parity-based RAID levels, erasure coding brings its own set of disadvantages.

#### What is erasure coding?
Erasure coding allows Ceph to achieve either greater usable storage capacity or increase resilience to disk failure for the same number of disks, versus the standard replica method. Erasure coding achieves this by splitting up the object into a number of parts and then also calculating a type of cyclic redundancy check (CRC), the erasure code, and then storing the results in one or more extra parts. Each part is then stored on a separate OSD. These parts are referred to as K and M chunks, where K refers to the number of data shards and M refers to the number of erasure code shards. As in RAID, these can often be expressed in the form K+M, or 4+2, for example.

In the event of an OSD failure that contains an object's shard, which is one of the calculated erasure codes, data is read from the remaining OSDs that store data with no impact. However, in the event of an OSD failure that contains the data shards of an object, Ceph can use the erasure codes to mathematically recreate the data from a combination of the remaining data and erasure code shards.

#### K+M
The more erasure code shards you have, the more OSD failures you can tolerate and still successfully read data. Likewise, the ratio of K to M shards each object is split into has a direct effect on the percentage of raw storage that is required for each object.

A 3+1 configuration will give you 75% usable capacity but only allows for a single OSD failure, and so would not be recommended. In comparison, a three-way replica pool only gives you 33% usable capacity.

4+2 configurations would give you 66% usable capacity and allows for two OSD failures. This is probably a good configuration for most people to use.

At the other end of the scale, 18+2 would give you 90% usable capacity and still allow for two OSD failures. On the surface, this sounds like an ideal option, but the greater total number of shards comes at a cost. A greater number of total shards has a negative impact on performance and also an increased CPU demand. The same 4 MB object that would be stored as a whole single object in a replicated pool would now be split into 20 x 200-KB chunks, which have to be tracked and written to 20 different OSDs. Spinning disks will exhibit faster bandwidth, measured in MBps with larger I/O sizes, but bandwidth drastically tails off at smaller I/O sizes. These smaller shards will generate a large amount of small I/O and cause an additional load on some clusters.

Also, it's important not to forget that these shards need to be spread across different hosts according to the CRUSH map rules: no shard belonging to the same object can be stored on the same host as another shard from the same object. Some clusters may not have a sufficient number of hosts to satisfy this requirement. If a CRUSH rule cannot be satisfied, the PGs will not become active, and any I/O destined for these PGs will be halted, so it's important to understand the impact on a cluster's health of making CRUSH modifications.

Reading back from these high-chunk pools is also a problem. Unlike in a replica pool, where Ceph can read just the requested data from any offset in an object, in an erasure pool, all shards from all OSDs have to be read before the read request can be satisfied. In the 18+2 example, this can massively amplify the amount of required disk read ops, and average latency will increase as a result. This behavior is a side-effect that tends to only cause a performance impact with pools that use a lot of shards. A 4+2 configuration in some instances will get a performance gain compared to a replica pool, from the result of splitting an object into shards. As the data is effectively striped over a number of OSDs, each OSD has to write less data, and there are no secondary and tertiary replicas to write.

Erasure coding can also be used to improve durability rather than to maximize available storage space. Take, for example, a 4+4 pool: it has a storage efficiency of 50%, so it's better than a 3x replica pool, yet it can sustain up to four OSD losses without data loss.

#### How does erasure coding work in Ceph?
As with replication, Ceph has a concept of a primary OSD, which also exists when using erasure-coded pools. The primary OSD has the responsibility of communicating with the client, calculating the erasure shards, and sending them out to the remaining OSDs in the PG set. This is illustrated in the following diagram:


If an OSD in the set is down, the primary OSD can use the remaining data and erasure shards to reconstruct the data, before sending it back to the client. During read operations, the primary OSD requests all OSDs in the PG set to send their shards. The primary OSD uses data from the data shards to construct the requested data, and the erasure shards are discarded. There is a fast read option that can be enabled on erasure pools, which allows the primary OSD to reconstruct the data from erasure shards if they return quicker than data shards. This can help to lower average latency at the cost of a slightly higher CPU usage. The following diagram shows how Ceph reads from an erasure-coded pool:

## Ceph storage types
Although Ceph provides basic object storage via the RADOS layer, on its own this is not very handy, as the scope of applications that could consume RADOS storage directly is extremely limited. Therefore, Ceph builds on the base RADOS capabilities and provides higher-level storage types that can be more easily consumed by clients.

### RBD
RBD for short, is how Ceph storage can be presented as standard Linux block devices. RBDs are composed of a number of objects, 4 MB by default, which are concatenated together. A 4 GB RBD would contain a 1,000 objects by default.

### CephFS
CephFS is a POSIX-compatible filesystem that sits on top of RADOS pools. Being POSIX-compliment means that it should be able to function as a drop-in replacement for any other Linux filesystem and still function as expected. There is both a kernel and userspace client to mount the filesystem on to a running Linux system. The kernel client, although normally faster, tends to lag behind the userspace client in terms of supported features and will often require you to be running the latest kernel to take advantage of certain features and bug fixes. A CephFS filesystem can also be exported via NFS or Samba to non-Linux-based clients, both software have direct support for talking to CephFS. This subject will be covered in more detail in the next chapter.

CephFS stores each file as one or more RADOS objects. If an object is larger than 4 MB, it will be striped across multiple objects. This striping behavior can be controlled by the use of XATTRs, which can be associated with both files and directories, and can control the object size, stripe width, and stripe count. The default striping policy effectively concatenates multiple 4 MB objects together, but by modifying the stripe count and width, a RAID 0 style striping can be achieved.

### RGW
The RADOS Gateway (RGW) presents the Ceph native object store via a S3 or swift-compatible interface, which are the two most popular object APIs for accessing object storage, with S3 being the dominant one, mainly due to the success of Amazon's AWS S3. This section of the book will primarily focus on S3. 

RGW has recently been renamed to Ceph Object Gateway although both the previous names are still widely used.
The radosgw component of Ceph is responsible for turning S3 and swift API requests into RADOS requests. Although it can be installed alongside other components, for performance reasons, it's recommended to be installed on a separate server. The radosgw components are completely stateless and so lend themselves well to being placed behind a load balancer to allow for horizontal scaling.

Aside from storing user data, the RGW also requires a number of additional RADOS pools to store additional metadata. With the exception of the index pool, most of these pools are very lightly utilized and so can be created with a small amount of PGs, around 64 is normally sufficient. The index pools helps with the listing of bucket contents and so placing the index pool on SSDs is highly recommended. The data pool can reside on either spinning disks or SSDs, depending on the type of objects being stored, although object storage tends to be a fairly good match for spinning disks. Quite often, clients are remote and the latency of WAN connections offsets a lot of the gains to be had from SSDs. It should be noted that only the data pool should be placed on erasure-coded pools.

Handily, RGW will create the required pools the first time it tries to access them, reducing the complexity of installation somewhat. However, pools are created with their default settings, and it may be that you wish to create an erasure-coded pool for data-object storage. As long as no access has been made to the RGW service, the data pool should not exist after creation, and it can therefore be manually created as an erasure pool. As long as the name matches the intended pool name for the RGW zone, RGW will use this pool on first access, instead of trying to create a new one.

#### Deploying RGW
We will use the Ansible lab deployed in Chapter 2, Deploying Ceph with Containers, to deploy a RGW.

First, edit the /etc/ansible/hosts file and add the rgws role to the mon3 VM:

```
vagrant@ansible:~$  sudo vi /etc/ansible/hosts

[mons] 
mon1 
mon2 
mon3

[mgrs]
mon1 
 
[osds] 
osd1 
osd2 
osd3 
 
[ceph:children] 
mons 
osds
mgrs

[rgws]
mon3 

```
We also need to update the /etc/ansible/group_vars/ceph file to add the radosgw_address variable; it will be set to [::], which means bind to all IPv4 and IPv6 interfaces:

```
vagrant@ansible:~$  sudo vi /etc/ansible/group_vars/ceph

ceph_origin: 'repository'
ceph_repository: 'community'
ceph_mirror: http://download.ceph.com
ceph_stable: true # use ceph stable branch
ceph_stable_key: https://download.ceph.com/keys/release.asc
ceph_stable_release: mimic # ceph stable release
ceph_stable_repo: "{{ ceph_mirror }}/debian-{{ ceph_stable_release }}"
monitor_interface: eth1 #Check ifconfig
public_network: 10.100.0.0/24
journal_size: 1024
ragosgw_address: "[::]"
```

Now run the Ansible playbook again:

```
vagrant@ansible:~$ cd /etc/ansible 
vagrant@ansible:/etc/ansible$ ansible-playbook -K site.yml

```

After running, you should see it has successfully deployed the RGW component:

```bash

INSTALLER STATUS *************************************************************************************************************************************************************************************************************************************************************************
Install Ceph Monitor        : Complete (0:01:26)
Install Ceph Manager        : Complete (0:00:20)
Install Ceph OSD            : Complete (0:00:53)
Install Ceph RGW            : Complete (0:00:19)

```

Viewing the Ceph status from a monitor node, we can check that the RGW service has registered with the Ceph cluster and is operational:

```bash

vagrant@mon3:~$ sudo ceph -s
  cluster:
    id:     2a50b3fc-a69a-48ec-a1c9-556688b44d2e
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum mon1,mon2,mon3
    mgr: mon1(active)
    osd: 3 osds: 3 up, 3 in
    rgw: 1 daemon active
 
  data:
    pools:   4 pools, 32 pgs
    objects: 219  objects, 1.1 KiB
    usage:   3.0 GiB used, 26 GiB / 29 GiB avail
    pgs:     32 active+clean


```

Now that the RGW is active, a user account is required to interact with the S3 API, and this can be created using the radosgw-admin tool shown as follows:

```bash

vagrant@mon3:~$ sudo radosgw-admin user create --uid=robert0714 --display-name="Robert Lee" --email=robert0714@gmail.com
{
    "user_id": "robert0714",
    "display_name": "Robert Lee",
    "email": "robert0714@gmail.com",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "robert0714",
            "access_key": "2BEC31S1HB2CU84D8ZGB",
            "secret_key": "1lXm4xf4yGvqjoZvWTnlCMGudfB2vCFsJ630IkNR"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}


```
