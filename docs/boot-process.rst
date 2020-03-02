==============
 Boot process
==============

On a traditional POSIX system, the boot process is as follows:

1. The CPU starts executing the BIOS.
2. The BIOS will do basic hardware initialization and load the
   bootloader.
3. The bootloader will load the kernel and pass information about the
   drive that contains the rootfs.
4. The kernel will do additional initialization, mount the rootfs and
   start /sbin/init.
5. ``init`` will run scripts that start everything else. This includes
   mounting filesystems from fstab and exporting NFS shares from
   exports.

There are some variations on this. For instance, the bootloaders will
also load kernel modules on Illumos (via a boot_archive) and FreeBSD
(individually). Some Linux systems will load an initramfs, which is an
temporary rootfs that contains modules to load and moves some logic of
the boot process into userland, mounts the real rootfs and switches
into it via an operation called pivot_root. Also, EFI systems are
capable of loading operating system kernels directly, which eliminates
the need for the bootloader stage.

ZFS makes the following changes to the boot process:

- When the rootfs is on ZFS, the pool must be imported before the
  kernel can mount it. The bootloader on Illumos and FreeBSD will pass
  the pool informaton to the kernel for it to import the root pool and
  mount the rootfs. On Linux, an initramfs must be used until
  bootloader support for creating the initramfs dynamically is
  written.
- Regardless of whether there is a root pool, imported pools must
  appear. This is done by reading the list of imported pools from the
  zpool.cache file, which is at ``/etc/zfs/zpool.cache`` on most
  platforms. It is at ``/boot/zfs/zpool.cache`` on FreeBSD. This is
  stored as a XDR-encoded nvlist and is readable by executing the
  ``zdb`` command without arguments.
- After the pool(s) are imported, the filesystems must be mounted and
  any filesystem exports or iSCSI LUNs must be made. If the mountpoint
  property is set to legacy on a dataset, fstab can be
  used. Otherwise, the boot scripts will mount the datasets by running
  ``zfs mount -a`` after pool import. Similarly, any datasets being
  shared via NFS or SMB for filesystems and iSCSI for zvols will be
  exported or shared via ``zfs share -a`` after the mounts are
  done. Not all platforms support ``zfs share -a`` on all share
  types. Legacy methods may always be used and must be used on
  platforms that do not support automation via ``zfs share -a``.
