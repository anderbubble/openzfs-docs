=================
ZFS storage pools
=================

The zpool command
=================

Virtual devices
===============

disk

file

mirror

raidz, raidz1, raidz2, raidz3

spare

log

dedup

special

cache

.. code-block:: sh

   zpool create tank mirror sda sdb mirror sdc sdd

Device sanity checks
--------------------


Creating storage pools
======================


Adding devices to an existing pool
==================================


Display options
---------------


Attaching a mirror device
=========================


Importing and exporting
=======================

Pool properties
===============

allocated
   *read-only*

altroot
   *set at creation time and import time only*

ashift=ashift

autoexpand=on|off

autoreplace=on|off

autotrim=on|off

bootfs=(unset)|pool/dataset

cachefile=path|none

capacity
   *read-only*

comment=text

dedupditto=number
   *deprecated.*

delegation=on|off

expandsize
   *read-only*

feature\@feature_name=enabled

fragmentation
   *read-only*


free
   *read-only*

freeing
   *read-only*

health
   *read-only*

guid
   *read-only*

listsnapshots=on|off

load_guid
   *read-only*

multihost=on|off

readonly=on|off
   *set only at import time*

size
   *read-only*

unsupported\@feature_guid
   *read-only*

version=version


Device failure and recovery
===========================

DEGRADED

FAULTED

OFFLINE

ONLINE

REMOVED

UNAVAIL

Hot spares
----------

.. code-block:: sh

   zpool create tank mirror sda sdb spare sdc sdd

Clearing errors
---------------

.. code-block:: sh

   zpool clear pool [device]

Scrubs
======

The intent log
==============

.. code-block:: sh

   zpool create tank sda sdb log sdc

Cache devices
=============

.. code-block:: sh

   zpool create tank sda sdb cache sdc sdd

Checkpoints
===========

.. code-block:: sh

   zpool checkpoint [-d, --discard] pool

.. code-block:: sh

   zpool checkpoint pool

.. code-block:: sh

   zpool export pool
   zpool import --rewind-to-checkpoint pool

.. code-block:: sh

   zpool checkpoint --discard pool


The special allocation class
============================


Pool features
=============
