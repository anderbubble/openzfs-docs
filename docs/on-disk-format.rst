========================
 The ZFS on-disk format
========================

The last numbered version of the ZFS on-disk format is v28. Use of
this version in OpenZFS guarantees compatibility between Solaris ZFS
and OpenZFS. As Oracle's code is no longer open source, OpenZFS is not
compatible with Solaris ZFS pools beyond v28.

Features
========

Version numbering was unsuitable for the distributed development of
OpenZFS: any change to the number would have required agreement,
across all implementations, of each change to the on-disk format.

OpenZFS feature flags–-an alternative to traditional version
numbering-–allow a uniquely named pool property for each change to the
on-disk format. This approach supports both independent and
inter-dependent format changes.

Feature flags are implemented by artificially rolling the pool version
to v5000 to avoid any conflicts with Oracle's version.

The `zgrep.org`_ project tracks and documents the availability of each
known OpenZFS feature by parsing each project's man pages.

.. _zgrep.org: https://zgrep.org/zfs.html
