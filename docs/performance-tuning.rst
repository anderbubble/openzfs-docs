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
into a buffer before it can be written. ZFS makes the implicit
assumption that the sector size reported by drives is correct and
calculates ashift based on that.

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
