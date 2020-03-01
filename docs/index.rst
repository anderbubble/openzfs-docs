===========================
 a civilized OpenZFS guide
===========================

This guide is intended to replace the various bits of ZFS
documentation found online with something more comprehensive for
OpenZFS. The closest thing I've been able to find for a comprehensive
guide to ZFS is targeted specifically at Solaris ZFS, which always
comes with caveats of not being quite accurate for OpenZFS.

My experience is definitely with ZFS on Linux, and that's where the
first draft of this guide will focus; but contributions specific to
BSD, illumos, or other OpenZFS distributions are welcome and
encouraged. (Where behavior or recommendations differ between
platforms, these differences should be called out with no preference
for any specific platform.)


----

.. toctree::
   :maxdepth: 1
   :caption: OpenZFS

   introduction-to-zfs
   zfs-storage-pools
   on-disk-format
