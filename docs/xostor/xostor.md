
# XOSTOR documentation

## LINSTOR/DRBD global documentation in a XCP-ng context

### What it this?

`LINSTOR` is an open source software developed by `LINBIT`. It was designed to manipulate a set of resources on several machines in order to replicate them via `DRBD` block devices while allowing high performance. We use it in our XCP-ng `LinstorSR` driver to support volume replication in a XCP-ng pool.

### How does it work?

Several components are necessary to bring LINSTOR mechanisms to life:

- A controller, a daemon that receives commands across a pool's network. It allows us to manipulate the volumes, the network used, the configuration... For each XCP-ng pool, there can only be one controller running on a machine. This controller can run on a master or slave. This is independent of the host type.
- A satellite on each machine of the pool, the satellites are there to send commands to the controllers. They are also used to send the host state to the controller.

Communication between satellites and controller is done via TCP/IP. By default, a new LINSTOR SR created will use the XAPI management interface.

A python API is available to communicate with LINSTOR. We actually use it in our driver. Otherwise a CLI tool is available: `linstor`.

### DRBD

We explained previously what an XCP-ng pool looks like with LINSTOR: a controller and satellites. It's good but it's not enough for us to explain what the volumes are based on.
In fact each volume created via a LINSTOR command appears in the form of a `DRBD` (Distributed Replicated Block Device). Like LINSTOR, this tool is also developed by LINBIT and is officially available in the linux kernel.

Basically, DRBD is a solution to share a resource between multiple machines with a replication parameter. Quite simply, in XCP-ng it's a `/dev/drbdXXX` volume accessible on several hosts. Here `XXXX` is a `device minor number`. For each XCP-ng VDI, you have a DRBD volume.

#### Resources & volumes

Like said DRBD has a `device minor number` but also a `resource name` in order to have a more understandable name of what a resource corresponds to.

The paths of a DRBD resource always have the same pattern:
```
> realpath /dev/drbd/by-res/<RESOURCE_NAME>/<VOLUME_ID>
/dev/drbd<DRBD_MINOR>
```

You may notice the use of a `<VOLUME_ID>` here.
In fact a DRBD is a set of volumes: a group. Each volume of the same DRBD resource shares the same attributes. Several volumes can be useful for breaking down information while sharing the context of the same DRBD resource. In our case, we only use one volume for each DRBD resource. In other words our `VOLUME_ID` will always be 0. So it will not be a surprise, but by abuse of language, it's therefore possible that I use the term resource and volume in the same way in this documentation.

#### Roles/locks

We know that the same DRBD path `/dev/drbdXXX` can be accessible on several machines. It is necessary here to introduce the notion of role, because there are barriers to this shared access...

A resource can be `Primary` or `Secondary`:
- A `Primary` is a DRBD accessible on a host for READ and WRITE operations.
- A `Secondary` is just a DRBD that receives requests from the `Primary`, it's used to write a replication of the data and also to improve reading performance. The data of this volume can't be written or read by process. Only the DRBD kernel module can access it. In fact we can see a DRBD Primary as a lock on a resource. This lock is global to a machine, not specific to a process.

Initially all instances of a DRBD are in the Secondary role. There are two main ways for a DRBD to become Primary:
- A DRBD command allows us to take the Primary role if the resource is indeed Secondary on all machines: `drbdadm primary <RESOURCE_NAME>`. Then this same command can be used with the `secondary` parameter when we no longer need to read or write to this resource: `drbdadm secondary <RESOURCE_NAME>`.
- Using the default configuration of a DRBD resource, a simple call to a C function like `fopen("/dev/drbd1001", "r+")` is enough to obtain `Primary` access to a resource. If the resource is opened on another machine, an `EROFS` errno code is returned. If the resource contains a partition, a call to the `mount` command also allows us to obtain a lock.

#### Diskless and Diskful

Usually when a resource is replicated through DRBD we assume that it only exists in one or even 2 or 3 copies. In other words, a host may not have a copy of a resource locally. So is it still possible to access the data of a resource in this type of case? The answer is of course yes. To do this we still use a device like `/dev/drbdXXXX`. The subtlety here is that when we write or read its data, we send network requests to hosts that have a replication of this volume. We call this type of volume: a `diskless`. Obviously a DRBD which has local data is called `diskful`.

And of course like a diskful, when a diskless is opened, it takes a primary lock.

#### Where is the data stored?

There is a layer below the DRBDs which is used to store our data, a lower-level storage.
In our case we use an `LVM group` built on one or more physical disks. It should be noted that each machine in the pool doesn't need a physical disk to be able to use LINSTOR/DRBD. But we recommend using the same types of drives on every machine that has them.

### Concepts of LINSTOR

We saw that DRBD had the concept of resources and volumes. In LINSTOR these same notions exist and relate to the same thing.
But other notions are grafted on top of DRBD.

#### Node

A node is simply an object that contains important information about a host. It contains important information like:
  - Name: it must be identical to the hostname.
  - Type: controller, satellite, combined, auxiliary. In our case we always use combined nodes which can be controller and/or satellite simply because if the current controller of a pool has problem, we want to be able to start one elsewhere. Without this we would no longer be able to launch LINSTOR commands.
  - IP address and port: used by satellites/controller. By default we use the XAPI management IP.

CLI example:
```
> linstor node list
╭──────────────────────────────────────────────────────────╮
┊ Node    ┊ NodeType ┊ Addresses                  ┊ State  ┊
╞══════════════════════════════════════════════════════════╡
┊ r620-s1 ┊ COMBINED ┊ 172.16.210.14:3366 (PLAIN) ┊ Online ┊
┊ r620-s2 ┊ COMBINED ┊ 172.16.210.15:3366 (PLAIN) ┊ Online ┊
┊ r620-s3 ┊ COMBINED ┊ 172.16.210.16:3366 (PLAIN) ┊ Online ┊
╰──────────────────────────────────────────────────────────╯
```

#### Storage pool

A storage pool is the LINSTOR object that represents the physical storage layer of a pool node. In our case it is an LVM layer below DRBD.

CLI example:
```
> linstor storage-pool list
╭────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool                      ┊ Node    ┊ Driver   ┊ PoolName                  ┊ FreeCapacity ┊ TotalCapacity ┊ CanSnapshots ┊ State ┊ SharedName                               ┊
╞════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ DfltDisklessStorPool             ┊ r620-s1 ┊ DISKLESS ┊                           ┊              ┊               ┊ False        ┊ Ok    ┊ r620-s1;DfltDisklessStorPool             ┊
┊ DfltDisklessStorPool             ┊ r620-s2 ┊ DISKLESS ┊                           ┊              ┊               ┊ False        ┊ Ok    ┊ r620-s2;DfltDisklessStorPool             ┊
┊ DfltDisklessStorPool             ┊ r620-s3 ┊ DISKLESS ┊                           ┊              ┊               ┊ False        ┊ Ok    ┊ r620-s3;DfltDisklessStorPool             ┊
┊ xcp-sr-linstor_group_thin_device ┊ r620-s1 ┊ LVM_THIN ┊ linstor_group/thin_device ┊   859.10 GiB ┊    931.28 GiB ┊ True         ┊ Ok    ┊ r620-s1;xcp-sr-linstor_group_thin_device ┊
┊ xcp-sr-linstor_group_thin_device ┊ r620-s2 ┊ LVM_THIN ┊ linstor_group/thin_device ┊   829.77 GiB ┊    931.28 GiB ┊ True         ┊ Ok    ┊ r620-s2;xcp-sr-linstor_group_thin_device ┊
┊ xcp-sr-linstor_group_thin_device ┊ r620-s3 ┊ LVM_THIN ┊ linstor_group/thin_device ┊   758.99 GiB ┊    931.28 GiB ┊ True         ┊ Ok    ┊ r620-s3;xcp-sr-linstor_group_thin_device ┊
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

In XCP-ng we only support two drivers: `LVM` (thick provisioning) or `LVM_THIN`.
In this example the name of the storage pool used on each node is: `xcp-sr-linstor_group_thin_device`.

You will note the presence of a special storage pool here: `DfltDisklessStorPool`. It's used to manage diskless DRBDs.

#### Resource group

A resource group is another LINSTOR object, it's like a container of resources. The properties of a group have an influence on the managed resources. The most important property is the `PlaceCount`. It defines for each resource how many copies must be present in the pool.

Also a change on a property has a direct impact on the existing resources, for example if you increase a place count, new duplications are created on other hosts.

CLI example:
```
> linstor rg list
╭────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceGroup                    ┊ SelectFilter                                     ┊ VlmNrs ┊ Description ┊
╞════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ DfltRscGrp                       ┊ PlaceCount: 2                                    ┊        ┊             ┊
╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
┊ xcp-ha-linstor_group_thin_device ┊ PlaceCount: 3                                    ┊ 0      ┊             ┊
┊                                  ┊ StoragePool(s): xcp-sr-linstor_group_thin_device ┊        ┊             ┊
┊                                  ┊ DisklessOnRemaining: False                       ┊        ┊             ┊
╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
┊ xcp-sr-linstor_group_thin_device ┊ PlaceCount: 2                                    ┊ 0      ┊             ┊
┊                                  ┊ StoragePool(s): xcp-sr-linstor_group_thin_device ┊        ┊             ┊
┊                                  ┊ DisklessOnRemaining: False                       ┊        ┊             ┊
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

In an XCP-ng context you can note that we use a single storage pool (xcp-sr-linstor_group_thin_device) containing two resource groups:
- xcp-ha-linstor_group_thin_device
- xcp-sr-linstor_group_thin_device

The second group is used for all volumes created by a user. For the first it's a little more subtle, we use it for the heartbeat volume required by the HA and also for the drbd1000 volume which contains the LINSTOR database, this same base which contains the information that we display via the `linstor` command. These volumes are extremely vital for the survival of an SR, so the replication here is forced to 3. While for the second resource group the replication count remains configurable (between 1 and 3).

#### Resource and volume + definitions

Although I said that resource and volume exist in LINSTOR in the same way as in DRBD, there is a small distinction here. The resource and volume info also has a `definition`. A resource definition is just there to describe a resource regardless of the number of replications.

Examples of resource and volume definitions:
```
> linstor rd list
╭───────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceName                                    ┊ Port ┊ ResourceGroup                    ┊ State ┊
╞═══════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ xcp-persistent-database                         ┊ 7000 ┊ xcp-ha-linstor_group_thin_device ┊ ok    ┊
┊ xcp-persistent-ha-statefile                     ┊ 7001 ┊ xcp-ha-linstor_group_thin_device ┊ ok    ┊
┊ xcp-persistent-redo-log                         ┊ 7002 ┊ xcp-ha-linstor_group_thin_device ┊ ok    ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ 7003 ┊ xcp-sr-linstor_group_thin_device ┊ ok    ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ 7010 ┊ xcp-sr-linstor_group_thin_device ┊ ok    ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ 7021 ┊ xcp-sr-linstor_group_thin_device ┊ ok    ┊
╰───────────────────────────────────────────────────────────────────────────────────────────────────╯
> linstor vd list
╭───────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceName                                    ┊ VolumeNr ┊ VolumeMinor ┊ Size       ┊ Gross ┊ State ┊
╞═══════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ xcp-persistent-database                         ┊ 0        ┊ 1000        ┊ 1 GiB      ┊       ┊ ok    ┊
┊ xcp-persistent-ha-statefile                     ┊ 0        ┊ 1001        ┊ 2 MiB      ┊       ┊ ok    ┊
┊ xcp-persistent-redo-log                         ┊ 0        ┊ 1002        ┊ 256 MiB    ┊       ┊ ok    ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ 0        ┊ 1003        ┊ 2.01 GiB   ┊       ┊ ok    ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ 0        ┊ 1010        ┊ 100.21 GiB ┊       ┊ ok    ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ 0        ┊ 1021        ┊ 100.25 GiB ┊       ┊ ok    ┊
╰───────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

As noted above, in XCP-ng we always use a single volume for a resource, so the `VolumeNr` column only contains 0s.
Of course we can retrieve DRBD paths with these info.
For example for the HA statefile volume we have:
```
/dev/drbd/by-res/xcp-persistent-ha-statefile/0
/dev/drbd1001
```

Examples of resources and volumes:
```
> linstor r list
╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceName                                    ┊ Node    ┊ Port ┊ Usage  ┊ Conns ┊      State ┊ CreatedOn           ┊
╞══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ xcp-persistent-database                         ┊ r620-s1 ┊ 7000 ┊ InUse  ┊ Ok    ┊   UpToDate ┊ 2023-06-13 18:40:06 ┊
┊ xcp-persistent-database                         ┊ r620-s2 ┊ 7000 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2023-06-13 18:40:04 ┊
┊ xcp-persistent-database                         ┊ r620-s3 ┊ 7000 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2023-06-13 18:40:04 ┊
┊ xcp-persistent-ha-statefile                     ┊ r620-s1 ┊ 7001 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:18 ┊
┊ xcp-persistent-ha-statefile                     ┊ r620-s2 ┊ 7001 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:14 ┊
┊ xcp-persistent-ha-statefile                     ┊ r620-s3 ┊ 7001 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:18 ┊
┊ xcp-persistent-redo-log                         ┊ r620-s1 ┊ 7002 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:27 ┊
┊ xcp-persistent-redo-log                         ┊ r620-s2 ┊ 7002 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:23 ┊
┊ xcp-persistent-redo-log                         ┊ r620-s3 ┊ 7002 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:28 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s1 ┊ 7003 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-19 14:59:57 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s2 ┊ 7003 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-19 14:59:57 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s3 ┊ 7003 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2023-10-19 14:59:56 ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ r620-s1 ┊ 7010 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-24 10:42:40 ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ r620-s3 ┊ 7010 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-24 10:42:40 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s1 ┊ 7021 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:11 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s2 ┊ 7021 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2024-02-16 15:42:10 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s3 ┊ 7021 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:11 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s1 ┊ 7012 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 13:44:05 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s2 ┊ 7012 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 13:44:05 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s3 ┊ 7012 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2024-02-16 13:44:04 ┊
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
> linstor v list
╭───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ Node    ┊ Resource                                        ┊ StoragePool                      ┊ VolNr ┊ MinorNr ┊ DeviceName    ┊  Allocated ┊ InUse  ┊      State ┊
╞═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ r620-s1 ┊ xcp-persistent-database                         ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1000 ┊ /dev/drbd1000 ┊  67.54 MiB ┊ InUse  ┊   UpToDate ┊
┊ r620-s2 ┊ xcp-persistent-database                         ┊ DfltDisklessStorPool             ┊     0 ┊    1000 ┊ /dev/drbd1000 ┊            ┊ Unused ┊   Diskless ┊
┊ r620-s3 ┊ xcp-persistent-database                         ┊ DfltDisklessStorPool             ┊     0 ┊    1000 ┊ /dev/drbd1000 ┊            ┊ Unused ┊   Diskless ┊
┊ r620-s1 ┊ xcp-persistent-ha-statefile                     ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1001 ┊ /dev/drbd1001 ┊      1 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s2 ┊ xcp-persistent-ha-statefile                     ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1001 ┊ /dev/drbd1001 ┊      1 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s3 ┊ xcp-persistent-ha-statefile                     ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1001 ┊ /dev/drbd1001 ┊      1 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s1 ┊ xcp-persistent-redo-log                         ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1002 ┊ /dev/drbd1002 ┊   2.50 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s2 ┊ xcp-persistent-redo-log                         ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1002 ┊ /dev/drbd1002 ┊   2.50 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s3 ┊ xcp-persistent-redo-log                         ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1002 ┊ /dev/drbd1002 ┊   2.50 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s1 ┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1003 ┊ /dev/drbd1003 ┊ 163.47 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s2 ┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1003 ┊ /dev/drbd1003 ┊ 163.47 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s3 ┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ DfltDisklessStorPool             ┊     0 ┊    1003 ┊ /dev/drbd1003 ┊            ┊ Unused ┊ TieBreaker ┊
┊ r620-s1 ┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1010 ┊ /dev/drbd1010 ┊ 728.63 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s3 ┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1010 ┊ /dev/drbd1010 ┊ 728.63 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s1 ┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1021 ┊ /dev/drbd1021 ┊  10.27 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s2 ┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ DfltDisklessStorPool             ┊     0 ┊    1021 ┊ /dev/drbd1021 ┊            ┊ Unused ┊   Diskless ┊
┊ r620-s3 ┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1021 ┊ /dev/drbd1021 ┊  10.27 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s1 ┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1012 ┊ /dev/drbd1012 ┊  10.27 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s2 ┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ xcp-sr-linstor_group_thin_device ┊     0 ┊    1012 ┊ /dev/drbd1012 ┊  10.27 MiB ┊ Unused ┊   UpToDate ┊
┊ r620-s3 ┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ DfltDisklessStorPool             ┊     0 ┊    1012 ┊ /dev/drbd1012 ┊            ┊ Unused ┊ TieBreaker ┊
╰───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

### State of a DRBD/LINSTOR resource

A DRBD/LINSTOR resource can be in several states, I will document the main ones here:
- UpToDate: No issue with the resource!
- DUnknown: There is a communication issue to get the resource state. Maybe a bad IP, satellite or another network issue.
- Inconsistent: When a new resource is created, we can see this state on the replications. It can also be visible during a new synchronization.
- Diskless: The resource doesn't have data locally but have a DRBD path to read/write using the network.
- TieBreaker: In order to protect against loss of quorum, each diskful & diskless DRBD acts as a tie breaker. So no reason to display this state using `linstor r list` command? Wrong. There is a specific case where this state is explicitly visible: when a resource is not available on a host (so no diskful nor diskless => no `/dev/drbdXXXX` path).

For more info: https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/#s-disk-states

#### drbd-reactor

Several services are necessary for the proper functioning of LINSTOR on an XCP-ng, the most trivial are of course: `linstor-controller` and `linstor-satellite`. However it's important to note one thing: it's more than forbidden to manually start a controller on a pool where there is already a configured SR. There is a reason for this and it's called high availability: a controller must always be available. If a host that was running the controller is rebooted, another machine will start another controller.

How does it work in practice? Well we use a daemon called `drbd-reactor` which is implemented to react to DRBD events and produce responses using scripts.

On XCP-ng it has a modified configuration so that it can always be restarted in the event of a problem:
```
/etc/systemd/system/drbd-reactor.service.d/override.conf
```

So, how is it configured? We use several things:

- A modification of the satellite configuration:
```
> cat /etc/systemd/system/linstor-satellite.service.d/override.conf
[Service]
Environment=LS_KEEP_RES=^xcp-persistent*

[Unit]
After=drbd.service
```

As noted in the shared database section, the controller uses a DRBD volume containing the LINSTOR database instead of a simple local folder on a host. This database must be accessible after a reboot, that is to say that `/dev/drbd/by-res/xcp-persistent-database/0` must be accessible on at least one machine (normally 3 since this volume is always replicated 3 times). A path like that is generated using a DRBD resource config file. We don't create them ourselves, these configurations are created by LINSTOR itself and they are not persistent, they are recreated each time the controller is started after a pool reboot. The only way to have a DRBD config file preserved at boot is to use the `LS_KEEP_RES` environment variable to indicate it to LINSTOR.

- A `drbd-reactor` configuration that uses the linbit promoter plugin:
```
> cat /etc/drbd-reactor.d/sm-linstor.toml
[[promoter]]

[promoter.resources.xcp-persistent-database]
start = [ "var-lib-linstor.service", "linstor-controller.service" ]
```
To quickly summarize: this plugin allows us to run services when a resource is not primary and with a quorum. In our case we start `var-lib-linstor.service` which is used to mount the database `/dev/drbd1000` on `/var/lib/linstor` and if successful we can start the controller. Of course only one host can manage to mount the database since it can only have one primary on the resource.

For more info regarding the promoter plugin: https://github.com/LINBIT/drbd-reactor/blob/0620ff875370ad26a61c149fa6f02a19d66f45cd/doc/promoter.md

- The `/etc/systemd/system/var-lib-linstor.service` config file.

No major things to say here, just open it for more details.
## Howto and questions

## Installation

### What are the prerequisites to install LINSTOR?

- At least 3 hosts, why? Quite simply because DRBD uses a quorum algorithm that requires at least 3 reachable machines in order to correctly replicate resources. Otherwise there is a risk of split-brain. And another important point: we support a maximum of 7 machines per pool.
- A dedicated 10G network interface, it's possible to use the same interface as the management one (XAPI) but it's recommended to use a specific interface.
- At a minimum the pool must contain at least one disk on a single machine (case without replication). Otherwise any number of disks can be used on a machine BUT we recommend that for each machine that has disks to use the same model and number: a certain homogeneity is required.
- Currently only XCP-ng 8.2.1 is supported.
- The replication/place count must be equal to 1, 2 or 3.

### Installation using the CLI

__TODO__

## The linstor command does not work!!!

If you get the following output, this is not necessarily abnormal:
```
> linstor r list
Error: Unable to connect to linstor://localhost:3370: [Errno 99] Cannot assign requested address
```

As already explained, there can only be one controller running on a pool at a certain time. If you get this error, there are two explanations:
- There is indeed a problem and the `linstor-controller` service is not started anywhere.
- Or you must rerun the command on another machine.

Small tip: It's possible to give the list of IPs to the command so as not to manually connect to another host:
```
linstor --controllers=<IP_LIST> r list
```

Example on a pool with 3 machines:
```
linstor --controllers=172.16.210.84,172.16.210.85,172.16.210.86 r list
```

Important: the `--controllers` param must always be just after the command name and before the action.

## How to list LINSTOR resources? And how to interpret this output?

Command:
```
linstor r list
```

Output example:
```
╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceName                                    ┊ Node    ┊ Port ┊ Usage  ┊ Conns ┊      State ┊ CreatedOn           ┊
╞══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ xcp-persistent-database                         ┊ r620-s1 ┊ 7000 ┊ InUse  ┊ Ok    ┊   UpToDate ┊ 2023-06-13 18:40:06 ┊
┊ xcp-persistent-database                         ┊ r620-s2 ┊ 7000 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2023-06-13 18:40:04 ┊
┊ xcp-persistent-database                         ┊ r620-s3 ┊ 7000 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2023-06-13 18:40:04 ┊
┊ xcp-persistent-ha-statefile                     ┊ r620-s1 ┊ 7001 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:18 ┊
┊ xcp-persistent-ha-statefile                     ┊ r620-s2 ┊ 7001 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:14 ┊
┊ xcp-persistent-ha-statefile                     ┊ r620-s3 ┊ 7001 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:18 ┊
┊ xcp-persistent-redo-log                         ┊ r620-s1 ┊ 7002 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:27 ┊
┊ xcp-persistent-redo-log                         ┊ r620-s2 ┊ 7002 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:23 ┊
┊ xcp-persistent-redo-log                         ┊ r620-s3 ┊ 7002 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-03-20 16:01:28 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s1 ┊ 7003 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-19 14:59:57 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s2 ┊ 7003 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-19 14:59:57 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s3 ┊ 7003 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2023-10-19 14:59:56 ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ r620-s1 ┊ 7010 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-24 10:42:40 ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ r620-s3 ┊ 7010 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-24 10:42:40 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s1 ┊ 7021 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:11 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s2 ┊ 7021 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2024-02-16 15:42:10 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s3 ┊ 7021 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:11 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s1 ┊ 7012 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 13:44:05 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s2 ┊ 7012 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 13:44:05 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s3 ┊ 7012 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2024-02-16 13:44:04 ┊
┊ xcp-volume-289c3fb5-67ec-49b1-b5a8-71ebe379ee2a ┊ r620-s1 ┊ 7023 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2024-02-12 23:32:25 ┊
┊ xcp-volume-289c3fb5-67ec-49b1-b5a8-71ebe379ee2a ┊ r620-s2 ┊ 7023 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-12 23:32:26 ┊
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

Remarks regarding special volumes:
- `xcp-persistent-database` is an important resource that contains the LINSTOR database, i.e. the list of volumes, nodes, etc. It's this DRBD resource that is mounted on `/var/lib/linstor` before the controller starts.
- `xcp-persistent-ha-statefile` & `xcp-persistent-redo-log` are special volumes used by the HA.
- All other resources start with `xcp-volume-<UUID>` where UUID is an internal identifier different from the XAPI VDI UUIDs.

### How to get a quick resume of the resource status of a pool?

You can use the following commands to quickly find out the status of a pool:
```
linstor n list
linstor r list
```

They allow you to know the state of nodes and resources. Overall, if you don't see words in RED, it's probably because the data is not damaged. However, this does not mean that there are not problems elsewhere. Also, there is an even more useful command to avoid future problems:
```
linstor advise r
```

This command allows you to get recommendations for resources that might have problems later.
For example you can have output like this:
```
╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ Resource                                        ┊ Issue                                                                  ┊ Possible fix                                                                                  ┊
╞══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ xcp-volume-8791c6e9-a17b-4eec-878a-3e1bdd3d2273 ┊ Resource has 2 replicas but no tie-breaker, could lead to split brain. ┊ linstor rd ap --drbd-diskless --place-count 1 xcp-volume-8791c6e9-a17b-4eec-878a-3e1bdd3d2273 ┊
┊ xcp-volume-6818e3f6-a129-4b76-b330-33dc86e7cf02 ┊ Resource has 2 replicas but no tie-breaker, could lead to split brain. ┊ linstor rd ap --drbd-diskless --place-count 1 xcp-volume-6818e3f6-a129-4b76-b330-33dc86e7cf02 ┊
┊ xcp-volume-98076e75-63e3-4582-80e6-79fe698308e3 ┊ Resource has 2 replicas but no tie-breaker, could lead to split brain. ┊ linstor rd ap --drbd-diskless --place-count 1 xcp-volume-98076e75-63e3-4582-80e6-79fe698308e3 ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ Resource has 2 replicas but no tie-breaker, could lead to split brain. ┊ linstor rd ap --drbd-diskless --place-count 1 xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊
┊ xcp-volume-ac5df7a3-9cc2-45ee-98e3-d86cbd235c72 ┊ Resource has 2 replicas but no tie-breaker, could lead to split brain. ┊ linstor rd ap --drbd-diskless --place-count 1 xcp-volume-ac5df7a3-9cc2-45ee-98e3-d86cbd235c72 ┊
┊ xcp-volume-e480eec9-97c3-44ea-b070-1c1dff72aacd ┊ Resource has 2 replicas but no tie-breaker, could lead to split brain. ┊ linstor rd ap --drbd-diskless --place-count 1 xcp-volume-e480eec9-97c3-44ea-b070-1c1dff72aacd ┊
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

### Map LINSTOR resource names to XAPI VDI UUIDs

The output of `linstor r list` is rather problematic for understanding the relationships between LINSTOR volumes and VDI UUIDs:
```
╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceName                                    ┊ Node    ┊ Port ┊ Usage  ┊ Conns ┊      State ┊ CreatedOn           ┊
╞══════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ xcp-persistent-database                         ┊ r620-s1 ┊ 7000 ┊ InUse  ┊ Ok    ┊   UpToDate ┊ 2023-06-13 18:40:06 ┊
┊ xcp-persistent-database                         ┊ r620-s2 ┊ 7000 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2023-06-13 18:40:04 ┊
┊ xcp-persistent-database                         ┊ r620-s3 ┊ 7000 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2023-06-13 18:40:04 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s1 ┊ 7003 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-19 14:59:57 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s2 ┊ 7003 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-19 14:59:57 ┊
┊ xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf ┊ r620-s3 ┊ 7003 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2023-10-19 14:59:56 ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ r620-s1 ┊ 7010 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-24 10:42:40 ┊
┊ xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967 ┊ r620-s3 ┊ 7010 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2023-10-24 10:42:40 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s1 ┊ 7021 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:11 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s2 ┊ 7021 ┊ Unused ┊ Ok    ┊   Diskless ┊ 2024-02-16 15:42:10 ┊
┊ xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815 ┊ r620-s3 ┊ 7021 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:11 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s1 ┊ 7012 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 13:44:05 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s2 ┊ 7012 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 13:44:05 ┊
┊ xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4 ┊ r620-s3 ┊ 7012 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2024-02-16 13:44:04 ┊
┊ xcp-volume-289c3fb5-67ec-49b1-b5a8-71ebe379ee2a ┊ r620-s1 ┊ 7023 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2024-02-12 23:32:25 ┊
┊ xcp-volume-289c3fb5-67ec-49b1-b5a8-71ebe379ee2a ┊ r620-s2 ┊ 7023 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-12 23:32:26 ┊
┊ xcp-volume-289c3fb5-67ec-49b1-b5a8-71ebe379ee2a ┊ r620-s3 ┊ 7023 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-12 23:32:26 ┊
┊ xcp-volume-35c8b8e3-a1b7-4372-8ee8-43969221fabc ┊ r620-s1 ┊ 7008 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:04 ┊
┊ xcp-volume-35c8b8e3-a1b7-4372-8ee8-43969221fabc ┊ r620-s2 ┊ 7008 ┊ Unused ┊ Ok    ┊   UpToDate ┊ 2024-02-16 15:42:04 ┊
┊ xcp-volume-35c8b8e3-a1b7-4372-8ee8-43969221fabc ┊ r620-s3 ┊ 7008 ┊ Unused ┊ Ok    ┊ TieBreaker ┊ 2024-02-16 15:42:04 ┊
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

For this I developed a small tool which allows us to better understand the mapping of UUIDs: `linstor-kv-tool`.

Usage:
```
linstor-kv-tool --dump-volumes -u <HOSTNAME> -g <GROUP_NAME> | grep '/volume-name":'
```

- `<HOSTNAME>` is the host where the controller is running. If you are connected to the right host, you can use `localhost`.
- `<GROUP_NAME>` is the LINSTOR group used by the pool.

If you're not sure which group to use, you can easily get it via:
```
linstor resource-group list
```

So you must have a similar output:
```
╭────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ ResourceGroup                    ┊ SelectFilter                                     ┊ VlmNrs ┊ Description ┊
╞════════════════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ DfltRscGrp                       ┊ PlaceCount: 2                                    ┊        ┊             ┊
╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
┊ xcp-ha-linstor_group_thin_device ┊ PlaceCount: 3                                    ┊ 0      ┊             ┊
┊                                  ┊ StoragePool(s): xcp-sr-linstor_group_thin_device ┊        ┊             ┊
┊                                  ┊ DisklessOnRemaining: False                       ┊        ┊             ┊
╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
┊ xcp-sr-linstor_group_thin_device ┊ PlaceCount: 2                                    ┊ 0      ┊             ┊
┊                                  ┊ StoragePool(s): xcp-sr-linstor_group_thin_device ┊        ┊             ┊
┊                                  ┊ DisklessOnRemaining: True                        ┊        ┊             ┊
╰────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```
The group name is the value of `StoragePool(s)`.

Output example:
```
> linstor-kv-tool --dump-volumes -u localhost -g xcp-sr-linstor_group_thin_device | grep '/volume-name":'
  "xcp/volume/0e7a20d9-04ce-4306-8229-02a76a407ae6/volume-name": "xcp-volume-289c3fb5-67ec-49b1-b5a8-71ebe379ee2a",
  "xcp/volume/2369e1b9-2166-4e3a-b61d-50d539fce51b/volume-name": "xcp-volume-13f2bcc0-c010-4637-9d10-d670f68a4ff4",
  "xcp/volume/28e44535-b463-4606-9f35-b8d701640f9d/volume-name": "xcp-volume-35c8b8e3-a1b7-4372-8ee8-43969221fabc",
  "xcp/volume/296fcf08-fb39-42eb-81b5-6208b8425f1b/volume-name": "xcp-volume-01611aa8-688b-470a-92ee-21c56de16cdf",
  "xcp/volume/5798a276-b3eb-45f1-bdda-4abc296b25dc/volume-name": "xcp-volume-10d2c269-35ef-4948-b7f0-dcd8db9b7815",
  "xcp/volume/b947f6ae-3403-4e07-b002-d159b11d988f/volume-name": "xcp-volume-07f73f51-95ea-4a82-ba03-ef65dfbfb967"
```

Output format:
```
  "xcp/volume/<VDI_UUID>/volume-name": "<LINSTOR_VOLUME_NAME>"
```

### How a LINSTOR SR capacity is calculated?

If you can't create a VDI greater than the displayed size in the XO SR view, don't worry:
- There are two important things to remember: the maximum size of a VDI that can be created is not necessarily equal to the capacity of the SR. The SR capacity in the XOSTOR context is the maximum size that can be used to store all VDI data.
- Exception: if the replication count is equal to the number of hosts, the SR capacity is equal to the max VDI size, i.e. the capacity of the smallest disk in the pool.

We use this formula to compute the SR capacity:
```
sr_capacity = smallest_host_disk_capacity * host_count / replication_count
```

For example if you have a pool of 3 hosts with a replication count of 2 and a disk of 200 GiB on each host, the capacity of the SR is equal to 300 GiB using the formula. Notes:
- You can't create a VDI greater than 200 GiB because the replication is not block based but volume based.
- If you create a volume of 200 GiB (400 of the 600 GiB are physically used) and the remaining disk can't be used because it becomes impossible to replicate on two different disks.
- If you create 3 volumes of 100 GiB: the SR becomes fully filled. In this case you have 300 GiB of unique data and a replication of 300 GiB.

### How to use a specific network for DRBD requests?

To use a specific network to handle the DRBD traffic, a new interface must be created for each host:
```
linstor node interface create <NODE_NAME> <INTERFACE_NAME> <IP>
```

`<INTERFACE_NAME>` is just a user choice, choose any name.

Then to set this new interface as active, use:
```
linstor node set-property <NODE_NAME> PrefNic <INTERFACE_NAME>
```

Repeat this command for all nodes.

### How do I change a hostname/node name?

Reminder: A node name must always equal a host name. So when you change a hostname, a node name must be changed and vice versa.

There is no simple helper to do that using the CLI. Here are the steps to follow:

1. You must create a new node using:
```
linstor node create --node-type Combined <NODE_NAME> <IP>
```

2. Then you must evacuate the old node to preserve the replication count:
```
linstor node evacuate <OLD_NAME>
```

3. Next, you must change your hostname and restart the services on each host:
```
systemctl stop linstor-controller
systemctl restart linstor-satellite
```

4. Finally you can delete the node and create a storage pool for the new node:
```
linstor node delete <OLD_NAME>
```

```
# For thin:
linstor storage-pool create lvmthin <NODE_NAME> <SP_NAME> <VG_NAME>

# For thick:
linstor storage-pool create lvm <NODE_NAME> <SP_NAME> <VG_NAME>

# Example:
# linstor storage-pool create lvmthin r620-s4 xcp-sr-linstor_group_thin_device linstor_group/thin_device
```

To verify the SP, you can use:
```
linstor sp list
```

5. After that you must recreate the diskless/diskful resources if necessary. Use `linstor advise r` to see the commands to execute.

### How to use a specific network for satellites?

Clearly I don't recommend doing this. In order to guarantee a certain robustness of the pool, it's better to use the management interface. But if you are sure of what you are doing:

```
linstor node interface modify <NODE_NAME> <INTERFACE_NAME> --active
```

Documentation: https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/#s-managing_network_interface_cards

### What to do if unable to mount the database volume (drbd1000) after bad network interface configuration or IP change?

#### Reset resource file config

First check if the node list is ok using `linstor node list` on the node where is running the controller.
If there is no controller, use `drbdsetup status xcp-persistent-database` to see the state of the database on each host. In the case of split-brain, use the corresponding documentation.
Otherwise to change the IPs manually, open this file on a pool machine:
```
nano /var/lib/linstor.d/xcp-persistent-database.res
```

You must have a similar configuration in the file:
```
    connection
    {
        host r620-s1 address ipv4 172.16.210.83:7000;
        host r620-s2 address ipv4 172.16.210.84:7000;
    }

    connection
    {
        host r620-s1 address ipv4 172.16.210.83:7000;
        host r620-s3 address ipv4 172.16.210.85:7000;
    }
```

For each entry, modify the IPs to use the XAPI management interface of each host name.
Save and repeat this modification on each host.

Restart `drbd-reactor` using this on all machines:
```
systemctl restart drbd-reactor
```

Reboot if the controller cannot be started.

After this point, the controller must be running.
Find it and list the resources:
```
linstor r list
```

If the array is empty, execute:
```
systemctl stop linstor-controller
```

The controller will be restarted on the current machine or on another one.
Re-check the resource list.


#### Reset active satellite connection

Verify the node list:
```
linstor n list
```

If you have a similar result, it's still bad:
```
╭───────────────────────────────────────────────────────────╮
┊ Node    ┊ NodeType ┊ Addresses                  ┊ State   ┊
╞═══════════════════════════════════════════════════════════╡
┊ r620-s1 ┊ COMBINED ┊ 172.16.210.83:3366 (PLAIN) ┊ Unknown ┊
┊ r620-s2 ┊ COMBINED ┊ 172.16.210.84:3366 (PLAIN) ┊ Unknown ┊
┊ r620-s3 ┊ COMBINED ┊ 172.16.210.85:3366 (PLAIN) ┊ Unknown ┊
╰───────────────────────────────────────────────────────────╯
```

Verify the interfaces using:
```
linstor node interface list r620-s1
```

In the first column if you don't have "StltCon" for the default interface (or if this interface is missing), the active connection must be reset:
```
╭────────────────────────────────────────────────────────────────╮
┊ r620-s1 ┊ NetInterface ┊ IP            ┊ Port ┊ EncryptionType ┊
╞════════════════════════════════════════════════════════════════╡
┊ +        ┊ default      ┊ 172.16.210.83 ┊ 3366 ┊ PLAIN         ┊
╰────────────────────────────────────────────────────────────────╯
```

Try using this command for each host interface (replace `r620-s1` using the right hostnames):
```
linstor node interface modify r620-s1 default --active
```

Verify the interfaces of each node a new time.


#### H2 modification to reset active connection and use default interface

Ok, so... If you can't modify the connections using:
```
linstor node interface modify <HOSTNAME> default --active
```

You probably have a similar error:
```
ERROR:
Description:
    Modification of netInterface 'default' on node 'r620-s1' failed due to an unknown exception.
Details:
    Node: 'r620-s1', NetIfName: default'
Show reports:
    linstor error-reports show 660585D7-00000-000000
```

In this situation the LINSTOR database must be manually modified.
Copy the database in another directory:
```
mkdir /root/linstor-db/
cp /var/lib/linstor/linstordb.mv.db /root/linstor-db/linstordb.mv.db.backup
cp /root/linstor-db/linstordb.mv.db.backup /root/linstor-db/linstordb.mv.db
```

Connect to the DB backup using:
```
java -cp /usr/share/linstor-server/lib/h2*.jar org.h2.tools.Shell -url jdbc:h2:/root/db/linstordb -user linstor -password linstor
# Note: you can also use a web interface with:
# java -jar /usr/share/linstor-server/lib/h2-1.4.197.jar -web -webAllowOthers -tcp -tcpAllowOthers
# But the server must be reachable without the support tunnel or using additional bind commands, so easier to use the CLI...
```

##### Modify node properties

Now we can execute few commands to verify the database state, first check the node properties:

```
SELECT * FROM LINSTOR.PROPS_CONTAINERS WHERE PROPS_INSTANCE LIKE '/NODE%';
```

With a valid configuration, you must have something like that:
```
PROPS_INSTANCE | PROP_KEY        | PROP_VALUE
/NODES/R620-S1 | CurStltConnName | default
/NODES/R620-S1 | NodeUname       | r620-s1
/NODES/R620-S1 | PrefNic         | default
/NODES/R620-S2 | CurStltConnName | default
/NODES/R620-S2 | NodeUname       | r620-s2
/NODES/R620-S2 | PrefNic         | default
/NODES/R620-S3 | CurStltConnName | default
/NODES/R620-S3 | NodeUname       | r620-s3
/NODES/R620-S3 | PrefNic         | default
(9 rows, 4 ms)
```

In this context:
- `CurStltConnName` is the satellite connection used by a node.
- `PrefNic` is the preferred network used by a DRBD resource on this node.

Our goal here is to reset the satellite and DRBD connections to use the `default` interface.


If the the `CurStltConnName` is not equal to `default` for each node, use:
```
UPDATE LINSTOR.PROPS_CONTAINERS
SET PROP_VALUE = 'default'
WHERE PROPS_INSTANCE LIKE '/NODE%' AND PROP_KEY = 'CurStltConnName';
```

Same for the preferred NIC:
```
UPDATE LINSTOR.PROPS_CONTAINERS
SET PROP_VALUE = 'default'
WHERE PROPS_INSTANCE LIKE '/NODE%' AND PROP_KEY = 'PrefNic';
```

If a node entry is missing (for `NodeUname`, `CurStltConnName` or `PrefNic`), you can
add the missing info. For example to configure a `r620-s4` node:
```
INSERT INTO LINSTOR.PROPS_CONTAINERS
VALUES ('/NODES/R620-S4', 'NodeUname', 'r620-s4');
INSERT INTO LINSTOR.PROPS_CONTAINERS
VALUES ('/NODES/R620-S4', 'CurStltConnName', 'default');
INSERT INTO LINSTOR.PROPS_CONTAINERS
VALUES ('/NODES/R620-S4', 'PrefNic', 'default');
```

Warning: Please note the value of `PROPS_INSTANCE` must be in capital letters, here: `/NODES/R620-S4`.


And to remove a bad node, for example `r620-s4`:
```
DELETE FROM LINSTOR.PROPS_CONTAINERS
WHERE PROPS_INSTANCE = '/NODES/R620-S4';
```

##### Modify interface IPs

Now we must have a default config set for each node.
The last thing is to verify the IPs of the default interfaces.

To list them:
```
SELECT * FROM LINSTOR.NODE_NET_INTERFACES;
```

Output example:
```
UUID                                 | NODE_NAME | NODE_NET_NAME | NODE_NET_DSP_NAME | INET_ADDRESS  | STLT_CONN_PORT | STLT_CONN_ENCR_TYPE
25bf886f-2325-42df-a888-c7c88f48d722 | R620-S1   | DEFAULT       | default           | 192.16.210.14 | 3366           | PLAIN
bb739364-2aeb-421c-96fa-c1eae934b192 | R620-S3   | DEFAULT       | default           | 172.16.210.16 | 3366           | PLAIN
62f19c69-77b5-43cf-ba68-0a68149ab7ff | R620-S2   | DEFAULT       | default           | 172.16.210.15 | 3366           | PLAIN
e0d79792-b20c-4dd9-a76c-cd7921a70f05 | R620-S1   | STORAGE       | storage           | 192.168.1.48  | null           | null
19fc1fcb-507c-4524-b6fe-a697a4a068ca | R620-S2   | STORAGE       | storage           | 192.168.1.49  | null           | null
b2d8e90c-7ce2-46aa-bd50-c5dd3bdae85b | R620-S3   | STORAGE       | storage           | 192.168.1.50  | null           | null
```

If an IP is incorrect for the default interface, you can modify it. For the `R620-S1` node, it would look like this:
```
UPDATE LINSTOR.NODE_NET_INTERFACES
SET INET_ADDRESS = '172.16.210.14'
WHERE UUID = '25bf886f-2325-42df-a888-c7c88f48d722';
```

After all these changes, you can exit, if SQL transactions have been used, do not forget the command:
```
COMMIT;
```

Then you can override the LINSTOR database:
```
cp /root/linstor-db/linstordb.mv.db /var/lib/linstor/linstordb.mv.db
```

And finally you can stop the controller on the host that is currently running.
`drbd-reactor` will restart it and you should have a valid database again.

Check using:
```
linstor n list
linstor r list
```

### What to do when a node is in an EVICTED state?

Ok first, why a node can be in this state? In fact using the default LINSTOR configuration and if a satellite is offline during 60 minutes, the controller marks this node as EVICTED. Then the DRBD resources are replicated on remaining nodes. Why? It's simply a protection against data loss. If the evicted machine must be re-imported, a simple command is generally sufficient:
```
linstor node restore <NODE_NAME>
```

If the machine must be removed:
```
linstor node lost <NODE_NAME>
```
Then the machine must be removed from the pool using XAPI commands.
Note: iptables config must also be modified to remove LINSTOR ports rules.

For more info:
- https://linbit.com/blog/linstors-auto-evict/
- https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/#s-linstor-auto-evict
