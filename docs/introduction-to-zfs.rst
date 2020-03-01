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

Storage pools
=============

Datasets
========

file system

volume

snapshot

bookmark


Example
=======

A simple example here creates a new single-device pool "tank" using
the single disk ``sda`` and mounts this pool at ``/mnt/tank``.

.. code-block:: sh

   zpool create -o mountpoint=/mnt/tank tank sda
   zfs mount tank
