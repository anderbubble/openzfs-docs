===================
Introduction to ZFS
===================

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
