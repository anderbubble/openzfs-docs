===================
Introduction to ZFS
===================


History
=======

ZFS was originally developed by Sun Microsystems for the Solaris
operating system. The source code for ZFS was released under the CDDL
as part of the OpenSolaris operating system and was subsequently
ported to other platforms.

OpenSolaris was discontinued in 2010, and it is from this last release
that illumos was forked. Official announcement of The OpenZFS project
was announced in 2013 to serve as an upstream for illumos, BSD, Linux,
and other ports of ZFS. Further development of ZFS on Solaris is not
open source.

OpenZFS is the truly open source successor to the ZFS
project. Development thrives as a part of illumos, which has added
many features and performance improvements. New OpenZFS features and
fixes are regularly pulled in from illumos, to all ports to other
platforms, and vice versa.


Architecture
============

ZFS itself is composed of three principle layers. The bottom layer is
the Storage Pool Allocator, which handles organizing the physical
disks into storage. The middle layer is the Data Management Unit,
which uses the storage provided by the SPA by reading and writing to
it transactionally in an atomic manner. The top layer is the dataset
layer, which translates between operations on the filesystems and
block devices (zvols) provided by the pool into operations in the DMU.


Storage pools
=============

The basic unit of storage in ZFS is the pool and from it, we obtain
datasets that can be either mountpoints (a mountable filesystem) or
block devices. The ZFS pool is a full storage stack capable of
replacing RAID, partitioning, volume management, fstab/exports files
and traditional single-disk file systems such as UFS and XFS. This
allows the same tasks to be accomplished with less code, greater
reliability and simplified administration.

The creation of a usable filesystem with redundancy from a set of
disks can be accomplished with 1 command and this will be persistent
upon reboots. This is because a ZFS pool will always have a mountable
filesystem called the root dataset, which is mounted at pool
creation. At creation, a pool is imported into the system, such that
an entry in the ``zpool.cache`` file is created. At time of import or
creation, the pool stores the system's unique hostid and for the
purposes of supporting multipath, import into other systems will fail
unless forced.

Administration of ZFS is performed through the zpool and zfs
commands. To create a pool, you can use ``zpool create poolname
...``. The part following the pool name is the vdev tree
specification. The root dataset will be mounted at ``/poolname`` by
default unless ``zpool create -m none`` is used. Other mountpoint
locations can be specified by writing them in place of ``none``.

Any file or disk vdevs before a top level vdev keyword is specified
will be a top level vdev. Any after a top level vdev keyword will be a
child of that vdev. Non-equal sized disks or files inside a top level
vdev will restrict its storage to the smaller of the two.

Additional information can be found in the zpool man page on your
platform. This is section 1m on Illumos and section 8 on Linux.


Virtual devices (vdevs)
=======================

The organization of disks in a pool by the SPA is a tree of vdevs or
virtual devices. At the top level of the tree is the root vdev. Its
immediate children can be any vdev type other than itself. The main
types of vdevs are:

- mirror (n-way mirrors supported)
- raidz
  - raidz1 (1-disk parity, similar to RAID 5)
  - raidz2 (2-disk parity, similar to RAID 6)
  - raidz3 (3-disk parity, no RAID analog)
- disk
- file (not recommended for production due to another filesystem
  adding unnecessary layering)

Any number of these can be children of the root vdev, which are called
top-level vdevs. Furthermore, some of these may also have children,
such as mirror vdevs and raidz vdevs. The commandline tools do not
support making mirrors of raidz or raidz of mirrors, although such
configurations are used in developer testing.

The use of multiple top level vdevs will affect IOPS in an additive
manner where total IOPS will be the sum of the top level
vdevs. Consequently, the loss of any main top level vdev will result
in the loss of the entire pool, such that proper redundancy must be
used on all top level vdevs.

The smallest supported file vdev or disk vdev size is 64MB (2^16
bytes) while the largest depends on the platform, but all platforms
should support vdevs of at least 16EB (2^64 bytes).

There are also three special device types.

- spare
- cache
- log

The spare devices are used for replacement when a drive fails,
provided that the pool's autoreplace property is enabled and your
platform supports that functionality. It will not replace a cache
device or log device.

The cache devices are used for extending ZFS's in-memory data cache,
which replaces the page cache with the exception of mmap(), which
still uses the page cache on most platforms. The algorithm used by ZFS
is the Adaptive Replacement Cache algorithm, which has a higher hit
rate than the Last Recently Used algorithm used by the page cache. The
cache devices are intended to be used with flash devices. The data
stored in them is non-persistent, so cheap devices can be used.

The log devices allow ZFS Intent Log records to be written to
different devices, such as flash devices, to increase performance of
synchronous write operations, before they are written to main
storage. These records are used unless the sync=disabled dataset
property is set. In either case, the synchronous operations' changes
are held in memory until they are written out on the next transaction
group commit. ZFS can survive the loss of a log device as long as the
next transaction group commit finishes successfully. If the system
crashes at the time that the log default is lost, the pool will be
faulted. While it can be recovered, whatever synchronous changes made
in the current transaction group will be lost to the datasets stored
on it.


Datasets
========

file system

volume

snapshot

bookmark


Data integrity
==============

ZFS has multiple mechanisms by which it attempts to provide data integrity:

- Committed data is stored in a merkle tree that is updated atomically
  on each transaction group commit
- The merkle tree uses 256-bit checksums stored in the block pointers
  to protect against misdirected writes, including those that would be
  likely to collide for weaker checksums. sha256 is a supported
  checksum for cryptographically strong guarantees, although the
  default is fletcher4.
- Each disk/file vdev contains four disk labels (two on each end) so
  that the loss of data at either end from a head drop does not wipe
  the labels.
- The transaction group commit uses two stages to ensure that all data
  is written to storage before the transaction group is considered
  committed. This is why ZFS has two labels on each end of each
  disk. A full head sweep is required on mechanical storage to perform
  the transaction group commit and flushes are used to ensure that the
  latter half does not occur before anything else.
- ZIL records storing changes to be made for synchronous IO are self
  checksumming blocks that are read only on pool import if the system
  made changes before the last transaction group commit was made.
- All metadata is stored twice by default, with the object containing
  the pool's state at a given transaction group to which the labels
  point being written three times. An effort is made to store the
  metadata at least 1/8 of a disk apart so that head drops do not
  result in irrecoverable damage.
- The labels contain an uberblock history, which allows rollback of
  the entire pool to a point in the near past in the event of a worst
  case scenario. The use of this recovery mechanism requires special
  commands because it should not be needed.
- The uberblocks contain a sum of all vdev GUIDs. Uberblocks are only
  considered valid if the sum matches. This prevents uberblocks from
  destroyed old pools from being be mistaken as being valid
  uberblocks.
- N-way mirroring and up to 3 levels of parity on raidz are supported
  so that increasingly common 2-disk failures[1] that kill RAID 5 and
  double mirrors during recovery do not kill a ZFS pool when proper
  redundancy is used.

Misinformation has been circulated that ZFS data integrity features
are somehow worse than those of other filesystems when ECC RAM is not
used. This is not the case: all software needs ECC RAM for reliable
operation and ZFS is no different from any other filesystem in that
regard.


Example
=======

A simple example here creates a new single-device pool "tank" using
the single disk ``sda`` and mounts this pool at ``/mnt/tank``.

.. code-block:: sh

   zpool create -o mountpoint=/mnt/tank tank sda
   zfs mount tank
