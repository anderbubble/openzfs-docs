========================
 The ZFS on-disk format
========================

Features
========

ZFS on-disk formats were originally versioned with a single number,
which increased whenever the format changed. The numbered approach was
suitable when development of ZFS was driven by a single organisation.

Version numbering was unsuitable for the distributed development of
OpenZFS: any change to the number would have required agreement,
across all implementations, of each change to the on-disk format.

OpenZFS feature flags–-an alternative to traditional version
numbering-–allow a uniquely named pool property for each change to the
on-disk format. This approach supports both independent and
inter-dependent format changes.

The `zgrep.org`_ project tracks and documents the availability of each
known OpenZFS feature by parsing each project's man pages.

.. _zgrep.org: https://zgrep.org/zfs.html
