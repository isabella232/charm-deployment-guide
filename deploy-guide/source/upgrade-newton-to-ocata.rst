:orphan:

========================
Upgrade: Newton to Ocata
========================

This page contains notes specific to the Newton to Ocata upgrade path. See the
main :doc:`upgrade-openstack` page for full coverage.

cinder/ceph topology change
---------------------------

If cinder is directly related to ceph-mon rather than via cinder-ceph then
upgrading from Newton to Ocata will result in the loss of some block storage
functionality, specifically live migration and snapshotting. To remedy this
situation the deployment should migrate to using the cinder-ceph charm. This
can be done after the upgrade to Ocata.

.. warning::

   Do not attempt to migrate a deployment with existing volumes to use the
   cinder-ceph charm prior to Ocata.

The intervention is detailed in the below three steps.

Step 0: Check existing configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Confirm existing volumes are in an RBD pool called 'cinder':

.. code-block:: none

   juju run --unit cinder/0 "rbd --name client.cinder -p cinder ls"

Sample output:

.. code-block:: none

   volume-b45066d3-931d-406e-a43e-ad4eca12cf34
   volume-dd733b26-2c56-4355-a8fc-347a964d5d55

Step 1: Deploy new topology
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Deploy the ``cinder-ceph`` charm and set the 'rbd-pool-name' to match the pool
that any existing volumes are in (see above):

.. code-block:: none

   juju deploy --config rbd-pool-name=cinder cinder-ceph
   juju add-relation cinder cinder-ceph
   juju add-relation cinder-ceph ceph-mon
   juju remove-relation cinder ceph-mon
   juju add-relation cinder-ceph nova-compute

Step 2: Update volume configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The existing volumes now need to be updated to associate them with the newly
defined cinder-ceph backend:

.. code-block:: none

   juju run-action cinder/0 rename-volume-host currenthost='cinder' \
       newhost='cinder@cinder-ceph#cinder.volume.drivers.rbd.RBDDriver'
