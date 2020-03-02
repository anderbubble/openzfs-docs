====================
 Performance tuning
====================


Basic concepts
==============

Adaptive Replacement Cache
--------------------------

Operating systems traditionally use RAM as a cache to avoid the
necessity of waiting on disk IO, which is comparatively extremely
slow. This concept is called page replacement. Until ZFS, virtually
all filesystems used the Least Recently Used (LRU) page replacement
algorithm in which the least recently used pages are the first to be
replaced. Unfortunately, the LRU algorithm is vulnerable to cache
flushes, where a brief change in workload that occurs occasionally
removes all frequently used data from cache. The Adaptive Replacement
Cache (ARC) algorithm was implemented in ZFS to replace LRU. It solves
this problem by maintaining four lists:

- A list for recently cached entries.
- A list for recently cached entries that have been accessed more than
  once.
- A list for entries evicted from #1.
- A list of entries evicited from #2.

Data is evicted from the first list while an effort is made to keep
data in the second list. In this way, ARC is able to outperform LRU by
providing a superior hit rate.

In addition, a dedicated cache device (typically an SSD) can be added
to the pool, with ``zpool add poolname cache devicename``. The cache
device is managed by the Level 2 ARC (L2ARC) which scans entries that
are next to be evicted and writes them to the cache device. The data
stored in ARC and L2ARC can be controlled via the ``primarycache`` and
``secondarycache`` ZFS properties respectively, which can be set on
both zvols and datasets. Possible settings are ``all``, ``none``, and
``metadata``. It is possible to improve performance when a zvol or
dataset hosts an application that does its own caching by caching only
metadata. One example is PostgreSQL. Another would be a virtual
machine using ZFS.


Alignment shift (ashift)
------------------------

Top-level vdevs contain an internal property called ashift, which
stands for alignment shift. It is set at vdev creation and is
otherwise immutable. It can be read using the ``zdb`` command. It is
calculated as the maximum base 2 logarithm of the physical sector size
of any child vdev and it alters the disk format such that writes are
always done according to it. This makes 2^ashift the smallest possible
IO on a vdev. Configuring ashift correctly is important because
partial sector writes incur a penalty where the sector must be read
into a buffer before it can be written.

ZFS will attempt to ensure proper alignment by extracting the physical
sector sizes from the disks. ZFS makes the implicit assumption that
the sector size reported by drives is correct and calculates ashift
based on that. The largest sector size will be used per top-level vdev
to avoid partial sector modification overhead is eliminated. This will
not be correct when drives misreport their physical sector sizes.

In an ideal world, physical sector size is always reported correctly
and therefore, this requires no attention. Unfortunately, this is not
the case. The sector size on all storage devices was 512-bytes prior
to the creation of flash-based solid state drives. Some operating
systems, such as Windows XP, were written under this assumption and
will not function when drives report a different sector size.

Flash-based solid state drives came to market around 2007. These
devices report 512-byte sectors, but the actual flash pages, which
roughly correspond to sectors, are never 512-bytes. The early models
used 4096-byte pages while the newer models have moved to an 8192-byte
page. In addition, "Advanced Format" hard drives have been created
which also use a 4096-byte sector size. Partial page writes suffer
from similar performance degradation as partial sector writes. In some
cases, the design of NAND-flash makes the performance degradation even
worse, but that is beyond the scope of this description.

Reporting the correct sector sizes is the responsibility the block
device layer. This unfortunately has made proper handling of devices
that misreport drives different across different platforms. The
respective methods are as follows:

- ``sd.conf`` on illumos
- ``gnop`` on FreeBSD
- ``-o ashift=`` on Linux and potentially other OpenZFS
  implementations

``-o ashift=` is convenient, but it is flawed in that the creation of
pools containing top level vdevs that have multiple optimal sector
sizes require the use of multiple commands. A newer syntax that will
rely on the actual sector sizes has been discussed as a cross platform
replacement and will likely be implemented in the future.

In addition, a database of drives known to misreport sector sizes is
used to automatically adjust ashift without the assistance of the
system administrator. This approach is unable to fully compensate for
misreported sector sizes whenever drive identifiers are used
ambiguously (e.g. virtual machines, iSCSI LUNs, some rare SSDs), but
it does a great amount of good. The format is roughly compatible with
illumos' ``sd.conf``, and it is expected that other implementations
will integrate the database in future releases. Strictly speaking,
this database does not belong in ZFS, but the difficulty of patching
the Linux kernel (especially older ones) necessitated that this be
implemented in ZFS itself for Linux. The same is true for
MacZFS. However, FreeBSD and illumos are both able to implement this
in the correct layer.


Compression
-----------

Internally, ZFS allocates data using multiples of the device's sector
size, typically either 512 bytes or 4KB (see above). When compression
is enabled, a smaller number of sectors can be allocated for each
block. The uncompressed block size is set by the ``recordsize`` file
system property (defaults to 128KB) or the ``volblocksize`` volume
property (defaults to 8KB).

The following compression algorithms are available:

LZ4
   New algorithm added after feature flags were created. It is
   significantly superior to LZJB in all metrics tested. It is new
   default compression algorithm (compression=on) in OpenZFS[1], but
   not all platforms have adopted the commit changing it yet.

LZJB
   Original default compression algorithm (compression=on) for ZFS. It
   was created to satisfy the desire for a compression algorithm
   suitable for use in filesystems. Specifically, that it provides
   fair compression, has a high compression speed, has a high
   decompression speed and detects incompressible data detection
   quickly.

GZIP (1 through 9)
   Classic Lempel-Ziv implementation. It provides high compression,
   but it often makes IO CPU-bound.

ZLE (Zero Length Encoding)
   A very simple algorithm that only compresses zeroes.

If you want to use compression and are uncertain which to use, use
LZ4. It averages a 2.1:1 compression ratio while gzip-1 averages
2.7:1, but gzip is much slower. Both figures are obtained from testing
by the LZ4 project on the Silesia corpus. The greater compression
ratio of gzip is usually only worthwhile for rarely accessed data.


RAID-Z stripe width
-------------------

Choose a RAID-Z stripe width based on your IOPS needs and the amount
of space you are willing to devote to parity information. If you need
more IOPS, use fewer disks per stripe. If you need more usable space,
use more disks per stripe. Trying to optimize your RAID-Z stripe width
based on exact numbers is irrelevant in nearly all cases.


Dataset recordsize
------------------

ZFS datasets use a default internal recordsize of 128KB. The dataset
recordsize is the basic unit of data used for internal copy-on-write
on files. Partial record writes require that data be read from either
ARC (cheap) or disk (expensive). recordsize can be set to any power of
2 from 512 bytes to 128 kilobytes. Software that writes in fixed
record sizes (e.g. databases) will benefit from the use of a matching
recordsize.

Changing the recordsize on a dataset will only take effect for new
files. If you change the recordsize because your application should
perform better with a different one, you will need to recreate its
files. A ``cp`` followed by a ``mv`` on each file is
sufficient. Alternatively, ``send`` / ``recv`` should recreate the
files with the correct recordsize when a full receive is done.


zvol volblocksize
-----------------

Zvols have a volblocksize property that is analogous to record
size. The default size is 8KB, which is the size of a page on the
SPARC architecture. Workloads that use smaller sized IOs (such as swap
on x86 which use 4096-byte pages) will benefit from a smaller
volblocksize.


Deduplication
-------------

Deduplication uses an on-disk hash table, using extensible hashing as
implemented in the ZAP (ZFS Attribute Processor). Each cached entry
uses slightly more than 320 bytes of memory. The DDT code relies on
ARC for caching the DDT entries, such that there is no double caching
or internal fragmentation from the kernel memory allocator. Each pool
has a global deduplication table shared across all datasets and zvols
on which deduplication is enabled. Each entry in the hash table is a
record of a unique block in the pool. (Where the block size is set by
the ``recordsize`` or ``volblocksize`` properties.)

The hash table (also known as the deduplication table, or DDT) must be
accessed for every dedup-able block that is written or freed
(regardless of whether it has multiple references). If there is
insufficient memory for the DDT to be cached in memory, each cache
miss will require reading a random block from disk, resulting in poor
performance. For example, if operating on a single 7200RPM drive that
can do 100 io/s, uncached DDT reads would limit overall write
throughput to 100 blocks per second, or 400KB/s with 4KB blocks.

The consequence is that sufficient memory to store deduplication data
is required for good performance. The deduplication data is considered
metadata and therefore can be cached if the ``primarycache`` or
``secondarycache`` properties are set to ``metadata``. In addition,
the deduplication table will compete with other metadata for metadata
storage, which can have a negative effect on performance. Simulation
of the number of deduplication table entries needed for a given pool
can be done using the -D option to zdb. Then a simple multiplication
by 320-bytes can be done to get the approximate memory
requirements. Alternatively, you can estimate an upper bound on the
number of unique blocks by dividing the amount of storage you plan to
use on each dataset (taking into account that partial records each
count as a full recordsize for the purposes of deduplication) by the
recordsize and each zvol by the volblocksize, summing and then
multiplying by 320-bytes.


Metaslab allocator
------------------

ZFS top level vdevs are divided into metaslabs from which blocks can
be independently allocated so allow for concurrent IOs to perform
allocations without blocking one another.

By default, the selection of a metaslab is biased toward lower LBAs to
improve performance of spinning disks, but this does not make sense on
solid state media. This behavior can be adjusted globally by setting
the ZFS module's global ``metaslab_lba_weighting_enabled`` tuanble to
``0``. This tunable is only advisable on systems that only use solid
state media for pools.

The metaslab allocator will allocate blocks on a first-fit basis when
a metaslab has more than or equal to 4 percent free space and a
best-fit basis when a metaslab has less than 4 percent free space. The
former is much faster than the latter, but it is not possible to tell
when this behavior occurs from the pool's free space. However, the
command ``zdb -mmm $POOLNAME`` will provide this information.


Pool geometry
-------------

If small random IOPS are of primary importance, mirrored vdevs will
outperform raidz vdevs. Read IOPS on mirrors will scale with the
number of drives in each mirror while raidz vdevs will each be limited
to the IOPS of the slowest drive.

If sequential writes are of primary importance, raidz will outperform
mirrored vdevs. Sequential write throughput increases linearly with
the number of data disks in raidz while writes are limited to the
slowest drive in mirrored vdevs. Sequential read performance should be
roughly the same on each.

Both IOPS and throughput will increase by the respective sums of the
IOPS and throughput of each top level vdev, regardless of whether they
are raidz or mirrors.


Whole disks vs partitions
-------------------------

ZFS will behave differently on different platforms when given a whole
disk.

ZFS will also attempt minor tweaks on various platforms when whole
disks are provided. On Illumos, ZFS will enable the disk cache for
performance. It will not do this when given partitions to protect
other filesystems sharing the disks that might not be tolerant of the
disk cache, such as UFS. On Linux, the IO elevator will be set to noop
to reduce CPU overhead. ZFS has its own internal IO elevator, which
renders the Linux elevator redundant. The Performance Tuning page
explains this behavior in more detail.

On illumos, ZFS attempts to enable the write cache on a whole
disk. The illumos UFS driver cannot ensure integrity with the write
cache enabled, so by default Sun/Solaris systems using UFS file system
for boot were shipped with drive write cache disabled (long ago, when
Sun was still an independent company). For safety on illumos, if ZFS
is not given the whole disk, it could be shared with UFS and thus it
is not appropriate for ZFS to enable write cache. In this case, the
write cache setting is not changed and will remain as-is. Today, most
vendors ship drives with write cache enabled by default.

On Linux, the Linux IO elevator is largely redundant given that ZFS
has its own IO elevator, so ZFS sets the IO elevator to noop to avoid
unnecessary CPU overhead.

ZFS also creates a GPT partition table own partitions when given a
whole disk under illumos on x86/amd64 and on Linux. This is mainly to
make booting through UEFI possible because UEFI requires a small FAT
partition to be able to boot the system. The ZFS driver will be able
to tell the difference between whether the pool had been given the
entire disk or not via the whole_disk field in the label.

This is not done on FreeBSD. Pools created by FreeBSD will always have
the whole_disk field set to true, such that a pool imported on another
platform that was created on FreeBSD will always be treated as the
whole disks were given to ZFS.


General recommendations
=======================


Alignment shift
---------------

Make sure that you create your pools such that the vdevs have the
correct alignment shift for your storage device's size. if dealing
with flash media, this is going to be either 12 (4K sectors) or 13 (8K
sectors). For SSD ephemeral storage on Amazon EC2, the proper setting
is 12.


Free space
----------

Keep pool free space above 10% to avoid many metaslabs from reaching
the 4% free space threshold to switch from first-fit to best-fit
allocation strategies. When the threshold is hit, the metaslab
allocator becomes very CPU intensive in an attempt to protect itself
from fragmentation. This reduces IOPS, especially as more metaslabs
reach the 4% threshold.

The recommendation is 10% rather than 5% because metaslabs selection
considers both location and free space unless the global
``metaslab_lba_weighting_enabled`` tunable is set to ``0``. When that
tunable is 0, ZFS will consider only free space, so the the expense of
the best-fit allocator can be avoided by keeping free space above
5%. That setting should only be used on systems with pools that
consist of solid state drives because it will reduce sequential IO
performance on mechanical disks.


LZ4 compression
---------------

Set ``compression=lz4`` on your pools' root datasets so that all
datasets inherit it unless you have a reason not to enable
it. Userland tests of LZ4 compression of incompressible data in a
single thread has shown that it can process 10GB/sec, so it is
unlikely to be a bottleneck even on incompressible data. The reduction
in IO from LZ4 will typically be a performance win.


Pool geometry
-------------

Do not put more than ~16 disks in raidz. The rebuild times on
mechanical disks will be excessive when the pool is full.


Synchronous I/O
---------------

If your workload involves fsync or O_SYNC and your pool is backed by
mechanical storage, consider adding one or more SLOG devices. Pools
that have multiple SLOG devices will distribute ZIL operations across
them.

To ensure maximum ZIL performance on NAND flash SSD-based SLOG
devices, you should also overprovison spare area to increase IOPS. You
can do this with a mix of a secure erase and a partition table trick,
such as the following:

1. Run a secure erase on the NAND-flash SSD.
2. Create a partition table on the NAND-flash SSD.
3. Create a 4GB partition.
4. Give the partition to ZFS to use as a log device.

If using the secure erase and partition table trick, do not use the
unpartitioned space for other things, even temporarily. That will
reduce or eliminate the overprovisioning by marking pages as dirty.

Alternatively, some devices allow you to change the sizes that they
report.This would also work, although a secure erase should be done
prior to changing the reported size to ensure that the SSD recognizes
the additional spare area. Changing the reported size can be done on
drives that support it with ``hdparm -N <sectors>`` on systems that
have laptop-mode-tools.

The choice of 4GB is somewhat arbitrary. Most systems do not write
anything close to 4GB to ZIL between transaction group commits, so
overprovisioning all storage beyond the 4GB partition should be
alright. If a workload needs more, then make it no more than the
maximum ARC size. Even under extreme workloads, ZFS will not benefit
from more SLOG storage than the maximum ARC size. That is half of
system memory on Linux and 3/4 of system memory on illumos.


Whole disks
-----------

Whole disks should be given to ZFS rather than partitions. If you must
use a partition, make certain that the partition is properly aligned
to avoid read-modify-write overhead. See the section on Alignment
Shift for a description of proper alignment. Also, see the section on
Whole Disks versus Partitions for a description of changes in ZFS
behavior when operating on a partition.

Single disk RAID 0 arrays from RAID controllers are not equivalent to
whole disks.


Bit Torrent
===========

Bit torrent performs 16KB random reads/writes. The 16KB writes cause
read-modify-write overhead. The read-modify-write overhead can reduce
performance by a factor of 16 with 128KB record sizes when the amount
of data written exceeds system memory. This can be avoided by using a
dedicated dataset for bit torrent downloads with recordsize=16KB.

When the files are read sequentially through a HTTP server, the random
nature in which the files were generated creates fragmentation that
has been observed to reduce sequential read performance by a factor of
two on 7200RPM hard disks. If performance is a problem, fragmentation
can be eliminated by rewriting the files sequentially in either of two
ways:

The first method is to configure your client to download the files to
a temporary directory and then copy them into their final location
when the downloads are finished, provided that your client supports
this.

The second method is to use send/recv to recreate a dataset
sequentially.

In practice, defragmenting files obtained through bit torrent should
only improve performance when the files are stored on magnetic storage
and are subject to significant sequential read workloads after
creation.


InnoDB (MySQL)
==============

Make separate datasets for InnoDB's data files and log files. Set
recordsize=16K on InnoDB's data files to avoid expensive partial
record writes and leave ``recordsize=128K`` on the log files. Set
primarycache=metadata on both to prefer InnoDB's caching. Set
``logbias=throughput`` on the data to stop ZIL from writing twice.

Set ``skip-innodb_doublewrite`` in ``my.cnf`` to prevent innodb from
writing twice. The double writes are a data integrity feature meant to
protect against corruption from partially-written records, but those
are not possible on ZFS. It should be noted that Percona’s blog had
advocated using an ext4 configuration where double writes were turned
off for a performance gain, but later recanted it because it caused
data corruption. Following a well timed power failure, an in place
filesystem such as ext4 can have half of a 8KB record be old while the
other half would be new. This would be the corruption that caused
Percona to recant its advice. However, ZFS’ copy on write design would
cause it to return the old correct data following a power failure (no
matter what the timing is). That prevents the corruption that the
double write feature is intended to prevent from ever happening. The
double write feature is therefore unnecessary on ZFS and can be safely
turned off for better performance.

On Linux, the driver's AIO implementation is a compatibility shim that
just barely passes the POSIX standard. InnoDB performance suffers when
using its default AIO codepath. Set ``innodb_use_native_aio=0`` and
``innodb_use_atomic_writes=0`` in ``my.cnf`` to disable AIO. Both of
these settings must be disabled to disable AIO.


PostgreSQL
==========

Make separate datasets for PostgreSQL's data and WAL. Set
``recordsize=8K`` on both to avoid expensive partial record
writes. Set ``logbias=throughput`` on PostgreSQL's data to avoid
writing twice.


Virtual machines
================

Virtual machine images on ZFS should be stored using either zvols or
raw files to avoid unnecessary overhead. The recordsize/volblocksize
and guest filesystem should be configured to match to avoid overhead
from partial record modification. This would typically be 4K. If raw
files are used, a separate dataset should be used to make it easy to
configure recordsize independently of other things stored on ZFS.


QEMU / KVM / Xen
----------------

AIO should be used to maximize IOPS when using files for guest storage.
