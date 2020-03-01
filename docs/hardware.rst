==========
 Hardware
==========

Storage before ZFS involved rather expensive hardware that was unable
to protect against silent corruption and did not scale very well. The
introduction of ZFS has enabled people to use far less expensive
hardware than previously used in the industry with superior
scaling. This page attempts to provide some basic guidance to people
buying hardware for use in ZFS-based servers and workstations.

Hardware that adheres to this guidance will enable ZFS to reach its
full potential for performance and reliability. Hardware that does not
adhere to it will serve as a handicap. Unless otherwise stated, such
handicaps apply to all storage stacks and are by no means specific to
ZFS. Systems built using competing storage stacks will also benefit
from these suggestions.


BIOS / CPU microcode updates
============================

Running the latest BIOS and CPU microcode is highly recommended.


Background
----------

Computer microprocessors are very complex designs that often have
bugs, which are called errata. Modern microprocessors are designed to
utilize microcode. This puts part of the hardware design into
quasi-software that can be patched without replacing the entire
chip. Errata are often resolved through CPU microcode updates. These
are often bundled in BIOS updates. In some cases, the BIOS
interactions with the CPU through machine registers can be modified to
fix things with the same microcode. If a newer microcode is not
bundled as part of a BIOS update, it can often be loaded by the
operating system bootloader or the operating system itself.


ECC Memory
==========

Bit flips can have fairly dramatic consequences for all computer
filesystems, and ZFS is no exception. No technique used in ZFS (or any
other filesystem) is capable of protecting against bit flips in
memory. Consequently, ECC memory is highly recommended.

Contrary to popular misconception, however, ZFS is not *more*
susceptible to problems that result from in-memory bit flips. As such,
a lack of ECC memory is not an argument against using ZFS.


Background
----------

Ordinary background radiation will randomly flip bits in computer
memory, which causes undefined behavior. These are known as "bit
flips". Each bit flip can have multiple possible consequences
depending on which bit is flipped:

- Bit flips can have no effect.

  - Bit flips that have no effect occur in unused memory.

- Bit flips can cause runtime failures.

  - This is the case when a bit flip occurs in something read from
    disk.
  - Failures are typically observed when program code is altered.

  - If the bit flip is in a routine within the system's kernel or
    /sbin/init, the system will likely crash. Otherwise, reloading the
    affected data can clear it. This is typically achieved by a
    reboot.

- It can cause data corruption.

  - This is the case when the bit is in use by data being written to
    disk.

  - If the bit flip occurs before ZFS' checksum calculation, ZFS will
    not realize that the data is corrupt.

  - If the bit flip occurs after ZFS' checksum calculation, but before
    write-out, ZFS will detect it, but it might not be able to correct
    it.

- It can cause metadata corruption.

  - This is the case when a bit flips in an on-disk structure being
    written to disk.

  - If the bit flip occurs before ZFS' checksum calculation, ZFS will
    not realize that the metadata is corrupt.

  - If the bit flip occurs after ZFS' checksum calculation, but before
    write-out, ZFS will detect it, but it might not be able to correct
    it.

  - Recovery from such an event will depend on what was corrupted. In
    the worst, case, a pool could be rendered unimportable.

    - All filesystems have poor reliability in their absolute worst
      case bit-flip failure scenarios. Such scenarios should be
      considered extraordinarily rare.


Drive Interfaces
================


SAS vs. SATA
------------

ZFS depends on the block device layer for storage. Consequently, ZFS
is affected by the same things that affect other filesystems, such as
driver support and non-working hardware. Consequently, there are a few
things to note:

- Never place SATA disks into a SAS expander without a SAS interposer.

  - If you do this and it does work, it is the exception, rather than
    the rule.
- Do not expect SAS controllers to be compatible with SATA port
  multipliers.

  - This configuration is typically not tested.

  - The disks could be unrecognized.
- Support for SATA port multipliers is inconsistent across Open ZFS
  platforms

  - Linux drivers generally support them.

  - Illumos drivers generally do not support them.

  - FreeBSD drivers are somewhere between Linux and Illumos in terms
    of support.

    
USB hard drives and/or adapters
-------------------------------

These have problems involving sector size reporting, SMART
passthrough, the ability to set ERC and other areas. ZFS will perform
as well on such devices as they are capable of allowing, but try to
avoid them. They should not be expected to have the same up-time as
SAS and SATA drives and should be considered unreliable.


Controllers
===========

The ideal storage controller for ZFS has the following attributes:

- Driver support on major Open ZFS platforms

  - Stability is important.

- High per-port bandwidth

  - PCI Express interface bandwidth divided by the number of ports

- Low cost

  - Support for RAID, Battery Backup Units and hardware write caches
    is unnecessary.


Hardware RAID controllers
=========================

Hardware RAID controllers should not be used with ZFS. While ZFS will
likely be more reliable than other filesystems on Hardware RAID, it
will not be as reliable as it would be on its own.

- Hardware RAID will limit opportunities for ZFS to perform self
  healing on checksum failures. When ZFS does RAID-Z or mirroring, a
  checksum failure on one disk can be corrected by treating the disk
  containing the sector as bad for the purpose of reconstructing the
  original information. This cannot be done when a RAID controller
  handles the redundancy unless a duplicate copy is stored by ZFS in
  the case that the corruption involving as metadata, the copies flag
  is set or the RAID array is part of a mirror/raid-z vdev within ZFS.

- Sector size information is not necessarily passed correctly by
  hardware RAID on RAID 1 and cannot be passed correctly on RAID
  5/6. Hardware RAID 1 is more likely to experience read-modify-write
  overhead from partial sector writes and Hardware RAID 5/6 will
  almost certainty suffer from partial stripe writes (i.e. the RAID
  write hole). Using ZFS with the disks directly will allow it to
  obtain the sector size information reported by the disks to avoid
  read-modify-write on sectors while ZFS avoids partial stripe writes
  on RAID-Z by desing from using copy-on-write.

  - There can be sector alignment problems on ZFS when a drive
    misreports its sector size. Such drives are typically NAND-flash
    based solid state drives and older SATA drives from the advanced
    format (4K sector size) transition before Windows XP EoL
    occurred. This can be manually corrected at vdev creation.

  - It is possible for the RAID header to cause misalignment of sector
    writes on RAID 1 by starting the array within a sector on an
    actual drive, such that manual correction of sector alignment at
    vdev creation does not solve the problem.

- Controller failures can require that the controller be replaced with
  the same model, or in less extreme cases, a model from the same
  manufacturer. Using ZFS by itself allows any controller to be used.
  If a hardware RAID controller's write cache is used, an additional
  failure point is introduced that can only be partially mitigated by
  additional complexity from adding flash to save data in power loss
  events. The data can still be lost if the battery fails when it is
  required to survive a power loss event or there is no flash and
  power is not restored in a timely manner. The loss of the data in
  the write cache can severely damage anything stored on a RAID array
  when many outstanding writes are cached. In addition, all writes are
  stored in the cache rather than just synchronous writes that require
  a write cache, which is inefficient, and the write cache is
  relatively small. ZFS allows synchronous writes to be written
  directly to flash, which should provide similar acceleration to
  hardware RAID and the ability to accelerate many more in-flight
  operations.

- Behavior during RAID reconstruction when silent corruption damages
  data is undefined. There are reports of RAID 5 and 6 arrays being
  lost during reconstruction when the controller encounters silent
  corruption. ZFS' checksums allow it to avoid this situation by
  determining if not enough information exists to reconstruct data. In
  which case, the file is listed as damaged in zpool status and the
  system administrator has the opportunity to restore it from a
  backup.

- IO response times will be reduced whenever the OS blocks on IO
  operations because the system CPU blocks on a much weaker embedded
  CPU used in the RAID controller. This lowers IOPS relative to what
  ZFS could have achieved.

- The controller's firmware is an additional layer of complexity that
  cannot be inspected by arbitrary third parties. The ZFS source code
  is open source and can be inspected by anyone.

- If multiple RAID arrays are formed by the same controller and one
  fails, the identifiers provided by the arrays exposed to the OS
  might become inconsistent. Giving the drives directly to the OS
  allows this to be avoided via naming that maps to a unique port or
  unique drive identifier.

  - e.g. If you have arrays A, B, C and D; array B dies, the
    interaction between the hardware RAID controller and the OS might
    rename arrays C and D to look like arrays B and C
    respectively. This can fault pools verbatim imported from the
    cachefile.

  - Not all RAID controllers behave this way. However, this issue has
    been observed on both Linux and FreeBSD when system administrators
    used single drive RAID 0 arrays. It has also been observed with
    controllers from different vendors.

One might be inclined to try using single-drive RAID 0 arrays to try
to use a RAID controller like a HBA, but this is not recommended for
many of the reasons listed for other hardware RAID types. It is best
to use a HBA instead of a RAID controller, for both performance and
reliability.


Hard drives
===========


Sector size
-----------

Historically, all hard drives had 512-byte sectors, with the exception
of some SCSI drives that could be modified to support slightly larger
sectors. In 2009, the industry migrated from 512-byte sectors to
4096-byte "Advanced Format" sectors. Since Windows XP is not
compatible with 4096-byte sectors or drives larger than 2TB, some of
the first advanced format drives implemented hacks to maintain Windows
XP compatibility.

- The first advanced format drives on the market misreported their
  sector size as 512-bytes for Windows XP compatibility. As of 2013,
  it is believed that such hard drives are no longer in
  production. Advanced format hard drives made during or after this
  time should report their true physical sector size.
- Drives storing 2TB and smaller might have a jumper that can be set
  to map all sectors off by 1. This to provide proper alignment for
  Windows XP, which started its first partition at sector 63. This
  jumper setting should be off when using such drives with ZFS.

As of 2014, there are still 512-byte and 4096-byte drives on the
market, but they are known to properly identify themselves unless
behind a USB to SATA controller. Replacing a 512-byte sector drive
with a 4096-byte sector drives in a vdev created with 512-byte sector
drives will adversely affect performance. Replacing a 4096-byte sector
drive with a 512-byte sector drive will have no negative effect on
performance.

Error recovery control
----------------------

ZFS is said to be able to use cheap drives. This was true when it was
introduced and hard drives supported error recovery control. Since
ZFS' introduction, error recovery control has been removed from
low-end drives from certain manufacturers, most notably Western
Digital. Consistent performance requires hard drives that support
error recovery control.

Background
~~~~~~~~~~

Hard drives store data using small polarized regions a magnetic
surface. Reading from and/or writing to this surface poses a few
reliability problems. One is that imperfections in the surface can
corrupt bits. Another is that vibrations can cause drive heads to miss
their targets. Consequently, hard drive sectors are composed of three
regions:

- A sector number

- The actual data

- ECC

The sector number and ECC enables hard drives to detect and respond to
such events. When either event occurs during a read, hard drives will
retry the read many times until they either succeed or conclude that
the data cannot be read. The latter case can take a substantial amount
of time and consequently, IO to the drive will stall.

Enterprise hard drives and some consumer hard drives implement a
feature called Time-Limited Error Recovery (TLER) by Western Digital,
Error Recovery Control (ERC) by Seagate and Command Completion Time
Limit by Hitachi and Samsung, which permits the time drives are
willing to spend on such events to be limited by the system
administrator.

Drives that lack such functionality can be expected to have
arbitrarily high limits. Several minutes is not impossible. Drives
with this functionality typically default to 7 seconds. ZFS does not
currently adjust this setting on drives. However, it is advisable to
write a script to set the error recovery time to a low value, such as
0.1 seconds until ZFS is modified to control it. This must be done on
every boot.


RPM speeds
----------

High RPM drives have lower seek times, which is historically regarded
as being desirable. They increase cost and sacrifice storage density
in order to achieve what is typically no more than a factor of 6
improvement over their lower RPM counterparts.

To provide some numbers, a 15k RPM drive from a major manufacturer is
rated for 3.4 millisecond average read and 3.9 millisecond average
write. Presumably, this number assumes that the target sector is at
most half the number of drive tracks away from the head and half the
disk away. Being even further away is worst-case 2 times
slower. Manufacturer numbers for 7200 RPM drives are not available,
but they average 13 to 16 milliseconds in empirical measurements. 5400
RPM drives can be expected to be slower.

ARC and ZIL are able to mitigate much of the benefit of lower seek
times. Far larger increases in IOPS performance can be obtained by
adding additional RAM for ARC, L2ARC devices and SLOG devices. Even
higher increases in performance can be obtained by replacing hard
drives with solid state storage entirely. Such things are typically
more cost effective than high RPM drives when considering IOPS.


Command queuing
---------------

Drives with command queues are able to reorder IO operations to
increase IOPS. This is called Native Command Queuing on SATA and
Tagged Command Queuing on PATA/SCSI/SAS. ZFS stores objects in
metaslabs and it can use several metastabs at any given
time. Consequently, ZFS is not only designed to take advantage of
command queuing, but good ZFS performance requires command
queuing. Almost all drives manufactured within the past 10 years can
be expected to support command queuing. The exceptions are:

- Consumer PATA/IDE drives
- First generation SATA drives, which used IDE to SATA translation
  chips, from 2003 to 2004.
- SATA drives operating under IDE emulation that was configured in the
  system BIOS.

Each Open ZFS system has different methods for checking whether
command queuing is supported. On Linux, ``hdparm -I /path/to/device |
grep Queue`` is used. On FreeBSD, ``camcontrol identify $DEVICE`` is
used.


NAND Flash SSDs
===============

As of 2014, Solid state storage is dominated by NAND-flash and most
articles on solid state storage focus on it exclusively. As of 2014,
the most popular form of flash storage used with ZFS involve drives
with SATA interfaces. Enterprise models with SAS interfaces are
beginning to become available.

As of 2017, Solid state storage using NAND-flash with PCI-E interfaces
are widely available on the market. They are predominantly enterprise
drives that utilize a NVMe interface that has lower overhead than the
ATA used in SATA or SCSI used in SAS. There is also an interface known
as M.2 that is primarily used by consumer SSDs, although not
necessarily limited to them. It can provide electrical connectivity
for multiple buses, such as SATA, PCI-E and USB. M.2 SSDs can use
either SATA or NVME.


Power failure protection
------------------------


Background
~~~~~~~~~~

On-flash data structures are highly complex and consequently,
vulnerable to corruption. Such corruption can result in the loss of
*all* drive data and an event such as a PSU failure can result in
multiple drives simultaneously failing. Since the drive firmware is
not available for review, the only reasonable conclusion is that all
drives that lack hardware features to avoid power failure events
cannot be trusted. Therefore, such drives are only suitable for use as
L2ARC.

Flash drives used for top-level vdevs or SLOG devices should have
power failure protection to protect both their own metadata and
flushed data. Protection of unflushed data does not occur on
mechanical drives and therefore is not a requirement of filesystems in
general, which include ZFS.


Flash pages
-----------

The smallest unit on a NAND chip that can be written is a flash
page. The first NAND-flash SSDs on the market had 4096-byte
pages. Further complicating matters is that the the page size has been
doubled twice since then. NAND flash SSDs *should* report these pages
as being sectors, but so far, all of them incorrectly report 512-byte
sectors for Windows XP compatibility. The consequence is that we have
a similar situation to what we had with early advanced format hard
drives.

As of 2014, most NAND-flash SSDs on the market have 8192-byte page
sizes. However, models using 128-Gbit NAND from certain manufacturers
have a 16384-byte page size. Maximum performance requires that vdevs
be created with correct ashift values (13 for 8192-byte and 14 for
16384-byte). However, not all Open ZFS platforms support this. The
Linux port supports ashift=13, while others are limited to ashift=12
(4096-byte).

As of 2017, NAND-flash SSDs are tuned for 4096-byte IOs. Matching the
flash page size is unnecessary and ashift=12 is usually the correct
choice. Public documentation on flash page size is also nearly
non-existent.


ATA TRIM / SCSI UNMAP
=====================

Support for sending block discard commands to vdevs to generate
appropriate ATA TRIM and/or SCSI UNMAP commands varries by
platform. It should be noted that this is a separate case from discard
on zvols or hole punching on filesystems. Those work regardless of
whether ATA TRIM / SCSI UNMAP is sent to the actual block devices.


ATA TRIM performance issues
---------------------------

The ATA TRIM command in SATA 3.0 and earlier is a non-queued
command. Issuing a TRIM command on a SATA drive conforming to SATA 3.0
or earlier will cause the drive to drain its IO queue and stop
servicing requests until it finishes, which hurts performance. SATA
3.1 removed this limitation, but very few SATA drives on the market
are conformant to SATA 3.1 and it is difficult to distinguish them
from SATA 3.0 drives. At the same time, SCSI UNMAP has no such
problems.


Power
=====

Ensuring that computers are properly grounded is highly
recommended. There have been cases in user homes where machines
experienced random failures when plugged into power receptacles that
had open grounds (i.e. no ground wire at all). This can cause random
failures on any computer system, whether it uses ZFS or not.

Power should also be relatively stable. Large dips in voltages from
brownouts are preferably avoided through the use of UPS units or line
conditioners. Systems subject to unstable power that do not outright
shutdown can exhibit undefined behavior.
