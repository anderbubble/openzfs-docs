============
 Redundancy
============

Two-disk failures are common enough that raidz1 vdevs should not be
used for data storage in production. 2-disk mirrors are also at risk,
but not quite as much as raidz1 vdevs unless only two disks are
used. This is because the failure of all disks in a 2-disk mirror is
statistically less likely than the failure of only 2 disks in a parity
configuration such as raidz1, unless only 2 disks are used, where the
failure risk is the same. The same risks also apply to RAID 1 and
RAID 5.

Single disk vdevs (excluding log and cache devices) should never be
used in production because they are not redundant. However, pools
containing single disk vdevs can be made redundant by attaching disks
via ``zpool attach`` to convert the disks into mirrors. For production
pools, it is important not to create top level vdevs that are not
raidz, mirror, log, cache or spare vdevs.
