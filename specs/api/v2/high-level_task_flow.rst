====================
High-level task flow
====================

**Procedure 1.1. To create and attach a volume**

#. You create a volume.

   For example, you might create a 30 GB volume called ``vol1``, as
   follows:

   .. code::

       $ cinder create --display-name vol1 30

   The command returns the ``521752a6-acf6-4b2d-bc7a-119f9148cd8c``
   volume ID.

#. You attach that volume to a virtual machine (VM) with the
   ``616fb98f-46ca-475e-917e-2563e5a8cd19`` ID, as follows:

   For example:

   .. code::

       $ nova volume-attach 616fb98f-46ca-475e-917e-2563e5a8cd19 521752a6-acf6-4b2d-bc7a-119f9148cd8c /dev/vdb

