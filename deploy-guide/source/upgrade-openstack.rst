=================
OpenStack upgrade
=================

This document outlines how to upgrade the OpenStack service components of a
Charmed OpenStack cloud.

.. warning::

   Upgrading an OpenStack cloud is not risk-free. The procedures outlined in
   this guide should first be tested in a pre-production environment.

Please read the :doc:`Upgrades overview <upgrade-overview>` page before
continuing.

.. note::

   The charms only support single-step OpenStack upgrades (N+1). That is, to
   upgrade two releases forward you need to upgrade twice. You cannot skip
   releases when upgrading OpenStack with charms.

It may be worthwhile to read the upstream OpenStack `Upgrades`_ guide.

Software sources
----------------

A key part of an OpenStack upgrade is the stipulation of a unit's software
sources. For an upgrade, the latter will naturally reflect a more recent
combination of Ubuntu release (series) and OpenStack release. This combination
is based on the `Ubuntu Cloud Archive`_ and translates to a "cloud archive
OpenStack release". It takes on the following syntax:

``<ubuntu series>-<openstack-release>``

The value is passed to a charm's ``openstack-origin`` configuration option. For
example, to select the 'focal-victoria' release:

``openstack-origin=cloud:focal-victoria``

In this way the charm is informed on where to find updates for the packages
that it is responsible for.

Notes concerning the value of ``openstack-origin``:

* The default is 'distro'. This denotes an Ubuntu release's default archive
  (e.g. in the case of the focal series it corresponds to OpenStack Ussuri).
  The value of 'distro' is therefore invalid in the context of an OpenStack
  upgrade.

* It should normally be the same across all charms.

* Its series component must be that of the series currently in use (i.e. a
  series upgrade and an OpenStack upgrade are two completely separate
  procedures).

.. note::

   A few charms use option ``source`` instead of ``openstack-origin`` (both
   options support identical values). The ``source`` option is used by charms
   that don't deploy an actual OpenStack service.

Upgradable services
-------------------

Services whose software is not included in the `Ubuntu Cloud Archive`_ do not
get upgraded during a charmed OpenStack upgrade. This software is upgraded by
the administrator (on the units) using other means (e.g. manually via package
utilities, the Landscape management tool, a snap, or as part of a series
upgrade). Common charms where this applies are:

* memcached
* ntp
* percona-cluster
* mysql-innodb-cluster
* mysql-router
* rabbitmq-server
* vault

Services that are associated with subordinate charms are upgradable but only
indirectly. They get upgraded along with their parent principal application.
Subordinate charms do not support the ``openstack-origin`` (or ``source``)
configuration option that is, as will be shown, a pre-requisite for initiating
an OpenStack charm payload upgrade.

.. _openstack_upgrade_prepare:

Prepare for the upgrade
-----------------------

Pay special attention to the below pre-upgrade preparatory and informational
sections.

Release notes
~~~~~~~~~~~~~

The OpenStack Charms `Release notes`_ for the corresponding current and target
versions of OpenStack must be consulted for any special instructions. In
particular, pay attention to services and/or configuration options that may be
retired, deprecated, or changed.

Upgrade path notes
~~~~~~~~~~~~~~~~~~

Any notes relevant to a specific upgrade path are recorded here.

* :doc:`Ussuri to Victoria <upgrade-ussuri-to-victoria>`
* :doc:`Stein to Train <upgrade-stein-to-train>`
* :doc:`Queens to Rocky <upgrade-queens-to-rocky>`
* :doc:`Newton to Ocata <upgrade-newton-to-ocata>`
* :doc:`Mitaka to Newton <upgrade-mitaka-to-newton>`

Manual intervention
~~~~~~~~~~~~~~~~~~~

By design, the latest stable charms will support the software changes related
to the OpenStack services being upgraded. During the upgrade, the charms will
also strive to preserve the existing configuration of their associated
services.

However, there are still times when intervention on the part of the operator
may be needed, such as when an OpenStack service is removed/added/replaced or
when a software bug (in the charms or in upstream OpenStack) affecting the
upgrade is present. The below resources cover these topics:

* :doc:`Special charm procedures <upgrade-special>`
* :doc:`Upgrade issues <upgrade-issues>`
* :doc:`Various issues <various-issues>`

Ensure cloud node software is up to date
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Every machine in the cloud, including containers, should have their software
packages updated to ensure that the latest SRUs have been applied. This is done
in the usual manner:

.. code-block:: none

   sudo apt update
   sudo apt full-upgrade

Verify the current deployment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Confirm that the output for the :command:`juju status` command of the current
deployment is error-free. In addition, if monitoring is in use (e.g. Nagios),
ensure that all alerts have been resolved. You may also consider running a
battery of operational checks on the cloud.

This step is to make certain that any issues that are apparent after the
upgrade are not due to pre-existing problems.

Perform the upgrade
-------------------

Perform the upgrade by following the below sections.

.. _disable_unattended_upgrades:

Disable unattended-upgrades
~~~~~~~~~~~~~~~~~~~~~~~~~~~

When performing a service upgrade on a cloud node that hosts multiple principal
charms (e.g. nova-compute and ceph-osd), ensure that ``unattended-upgrades`` is
disabled on the underlying machine for the duration of the upgrade process.
This is to prevent the other services from being upgraded outside of Juju's
control. On a cloud node run:

.. code-block:: none

   sudo dpkg-reconfigure -plow unattended-upgrades

Perform a backup of the service databases
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Perform a backup of the cloud service databases by applying the ``mysqldump``
action to any unit of the cloud's database application. Be sure to select all
applicable databases; the commands provided are examples only.

The permissions on the remote backup directory will need to be adjusted in
order to access the data. Take note that the transfer method presented here
will capture all existing backups in that directory.

.. important::

   Store the backup archive in a safe place.

The next two sections include the commands to run for the two possible database
applications.

percona-cluster
^^^^^^^^^^^^^^^

The percona-cluster application requires a modification to its "strict mode"
(see `Percona strict mode`_ for an understanding of the implications).

.. code-block:: none

   juju run-action --wait percona-cluster/0 set-pxc-strict-mode mode=MASTER
   juju run-action --wait percona-cluster/0 mysqldump \
      databases=aodh,cinder,designate,glance,gnocchi,horizon,keystone,neutron,nova,nova_api,nova_cell0,placement
   juju run-action --wait percona-cluster/0 set-pxc-strict-mode mode=ENFORCING

   juju run -u percona-cluster/0 -- sudo chmod o+rx /var/backups/mysql
   juju scp -- -r percona-cluster/0:/var/backups/mysql .
   juju run -u percona-cluster/0 -- sudo chmod o-rx /var/backups/mysql

mysql-innodb-cluster
^^^^^^^^^^^^^^^^^^^^

.. code-block:: none

   juju run-action --wait mysql-innodb-cluster/0 mysqldump \
      databases=cinder,designate,glance,gnocchi,horizon,keystone,neutron,nova,nova_api,nova_cell0,placement,vault

   juju run -u mysql-innodb-cluster/0 -- sudo chmod o+rx /var/backups/mysql
   juju scp -- -r mysql-innodb-cluster/0:/var/backups/mysql .
   juju run -u mysql-innodb-cluster/0 -- sudo chmod o-rx /var/backups/mysql

Archive old database data
~~~~~~~~~~~~~~~~~~~~~~~~~

During the upgrade, database migrations will be run. This operation can be
optimised by first archiving any stale data (e.g. deleted instances). Do this
by running the ``archive-data`` action on any nova-cloud-controller unit:

.. code-block:: none

   juju run-action --wait nova-cloud-controller/0 archive-data

This action may need to be run multiple times until the action output reports
'Nothing was archived'.

Purge old compute service entries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Old compute service entries for units which are no longer part of the model
should be purged prior to upgrading. These entries will show as 'down' (and be
hosted on machines no longer in the model) in the current list of compute
services:

.. code-block:: none

   openstack compute service list

To remove a compute service:

.. code-block:: none

   openstack compute service delete <service-id>

.. _openstack_upgrade_order:

List the upgrade order
~~~~~~~~~~~~~~~~~~~~~~

Generally speaking, the upgrade order is determined by the idea of a dependency
tree. Those services that have the most potential impact on other services are
upgraded first and those services that have the least potential impact on other
services are upgraded last.

In the below table, charms are listed in the order in which their corresponding
OpenStack services should be upgraded. Each service represented by a charm will
need to be upgraded individually. Note that since charms merely modify a
machine's apt sources, any co-located service will have their packages updated
along with those of the service being targeted.

.. warning::

   Ceph may require one of its options to be set prior to upgrading, and
   failure to consider this may result in a broken cluster. See the associated
   :ref:`upgrade issue <ceph-require-osd-release>`.

.. note::

   At this time, only stable charms are listed in the upgrade order table.

.. list-table::
   :header-rows: 1
   :widths: auto

   * - Order
     - Charm

   * - 1
     - `ceph-mon`_

   * - 2
     - `keystone`_

   * - 3
     - `aodh`_

   * - 4
     - `barbican`_

   * - 5
     - `ceilometer`_

   * - 6
     - `ceph-fs`_

   * - 7
     - `ceph-radosgw`_

   * - 8
     - `cinder`_

   * - 9
     - `designate`_

   * - 10
     - `designate-bind`_

   * - 11
     - `glance`_

   * - 12
     - `gnocchi`_

   * - 13
     - `heat`_

   * - 14
     - `manila`_

   * - 15
     - `manila-ganesha`_

   * - 16
     - `neutron-api`_

   * - 17
     - `neutron-gateway`_ or `ovn-dedicated-chassis`_

   * - 18
     - `ovn-central`_

   * - 19
     - `placement`_

   * - 20
     - `nova-cloud-controller`_

   * - 21
     - `nova-compute`_

   * - 22
     - `openstack-dashboard`_

   * - 23
     - `ceph-osd`_

   * - 24
     - `swift-proxy`_

   * - 25
     - `swift-storage`_

   * - 26
     - `octavia`_

.. important::

   The OVN control plane will not be available between the commencement of the
   ovn-central upgrade and the completion of the nova-compute upgrade.

Update the charm channel
------------------------

.. warning::

   This step is only performed for charms that follow a channel (see
   :ref:`Charm types <charm_types>`).

A charm's channel needs to be updated according to the target OpenStack
release. This is done as per the following syntax:

.. code-block:: none

   juju refresh --channel=<channel> <application>

For example, if the cloud is being upgraded to OpenStack Yoga then the keystone
charm's channel should be updated to 'yoga/stable':

.. code-block:: none

   juju refresh --channel=yoga/stable keystone

Charms whose services are not technically part of the OpenStack project will
generally use a channel naming scheme that is not based on OpenStack release
names. Here is the ovn-central charm:

.. code-block:: none

   juju refresh --channel=22.03/stable ovn-central

.. _perform_the_upgrade:

Perform the upgrade
-------------------

There are three methods available for performing an OpenStack service upgrade,
two of which have charm requirements in terms of supported actions. Each
method also has advantages and disadvantages with regard to:

* the time required to perform an upgrade
* maintaining service availability during an upgrade

This table summarises the characteristics and requirements of each method:

+--------------------+----------+----------+--------------------------------------------------+
| Method             | Time     | Downtime | Charm requirements (actions)                     |
+====================+==========+==========+==================================================+
| all-in-one         | shortest | most     | *none*                                           |
+--------------------+----------+----------+--------------------------------------------------+
| single-unit        | medium   | medium   | ``openstack-upgrade``                            |
+--------------------+----------+----------+--------------------------------------------------+
| paused-single-unit | longest  | least    | ``openstack-upgrade``, ``pause``, and ``resume`` |
+--------------------+----------+----------+--------------------------------------------------+

For example, although the all-in-one method upgrades a service the fastest, it
also has the greatest potential for service downtime.

.. note::

   A charm's supported actions can be listed with command :command:`juju
   actions <charm-name>`.

All-in-one
~~~~~~~~~~

The all-in-one method upgrades all application units simultaneously. This
method must be used if the application has a sole unit.

Although it is the quickest route, it will also cause a temporary disruption of
the corresponding service.

.. important::

   Exceptionally, the ceph-osd and ceph-mon applications use the all-in-one
   method but their charms are able to maintain service availability during the
   upgrade.

The syntax is:

.. code-block:: none

   juju config <openstack-charm> openstack-origin=cloud:<cloud-archive-release>

For example, to upgrade Cinder across all units (currently running Focal) from
Ussuri to Victoria:

.. code-block:: none

   juju config cinder openstack-origin=cloud:focal-victoria

Charms whose services are not technically part of the OpenStack project will
use the ``source`` charm option instead. The Ceph charms are a classic example:

.. code-block:: none

   juju config ceph-mon source=cloud:focal-victoria

Single-unit
~~~~~~~~~~~

The single-unit method builds upon the all-in-one method by allowing for the
upgrade of individual units in a controlled manner. The charm must support the
``openstack-upgrade`` action, which in turn guarantees the availability of the
``action-managed-upgrade`` option.

This method is slower than the all-in-one method due to the need for each unit
to be upgraded separately. There is a lesser chance of downtime as the unit
being upgraded must be in the process of servicing client requests for downtime
to occur.

As a general rule, whenever there is the possibility of upgrading units
individually, **always upgrade the application leader first**.

.. note::

   The leader is the unit with a ***** next to it in the :command:`juju status`
   output. It can also be discovered via the CLI:

   .. code-block:: none

      juju run -a <application-name> is-leader

For example, to upgrade a three-unit glance application from Ussuri to Victoria
where ``glance/1`` is the leader:

.. code-block:: none

   juju config glance action-managed-upgrade=True
   juju config glance openstack-origin=cloud:focal-victoria

   juju run-action --wait glance/1 openstack-upgrade
   juju run-action --wait glance/0 openstack-upgrade
   juju run-action --wait glance/2 openstack-upgrade

.. _paused_single_unit:

Paused-single-unit
~~~~~~~~~~~~~~~~~~

The paused-single-unit method extends the single-unit method by allowing for
the upgrade of individual units while paused. Additional charm requirements are
the ``pause`` and ``resume`` actions.

This method provides more versatility by allowing a unit to be removed from
service, upgraded, and returned to service. Each of these are distinct events
whose timing is chosen by the operator.

This is the slowest method due to the need for each unit to be upgraded
separately in addition to the required pause/resume management. However, it is
the method that will result in the least downtime as clients will not be able
to solicit a paused service.

For example, to upgrade a three-unit nova-compute application from Ussuri to
Victoria where ``nova-compute/0`` is the leader:

.. code-block:: none

   juju config nova-compute action-managed-upgrade=True
   juju config nova-compute openstack-origin=cloud:focal-victoria

   juju run-action --wait nova-compute/0 pause
   juju run-action --wait nova-compute/0 openstack-upgrade
   juju run-action --wait nova-compute/0 resume

   juju run-action --wait nova-compute/1 pause
   juju run-action --wait nova-compute/1 openstack-upgrade
   juju run-action --wait nova-compute/1 resume

   juju run-action --wait nova-compute/2 pause
   juju run-action --wait nova-compute/2 openstack-upgrade
   juju run-action --wait nova-compute/2 resume

In addition, this method also permits a possible hacluster subordinate unit,
which typically manages a VIP, to be paused so that client requests will never
even be directed to the associated parent unit.

.. attention::

   When there is an hacluster subordinate unit then it is recommended to always
   take advantage of the pause-single-unit method's ability to pause it before
   upgrading the parent unit.

For example, to upgrade a three-unit keystone application from Ussuri to
Victoria where ``keystone/2`` is the leader:

.. code-block:: none

   juju config keystone action-managed-upgrade=True
   juju config keystone openstack-origin=cloud:focal-victoria

   juju run-action --wait keystone-hacluster/1 pause
   juju run-action --wait keystone/2 pause
   juju run-action --wait keystone/2 openstack-upgrade
   juju run-action --wait keystone/2 resume
   juju run-action --wait keystone-hacluster/1 resume

   juju run-action --wait keystone-hacluster/2 pause
   juju run-action --wait keystone/1 pause
   juju run-action --wait keystone/1 openstack-upgrade
   juju run-action --wait keystone/1 resume
   juju run-action --wait keystone-hacluster/2 resume

   juju run-action --wait keystone-hacluster/0 pause
   juju run-action --wait keystone/0 pause
   juju run-action --wait keystone/0 openstack-upgrade
   juju run-action --wait keystone/0 resume
   juju run-action --wait keystone-hacluster/0 resume

.. warning::

   The hacluster subordinate unit number may not necessarily match its parent
   unit number. As in the above example, only for ``keystone/0`` do the unit
   numbers correspond (i.e. ``keystone-hacluster/0`` is its subordinate unit).

Re-enable unattended-upgrades
-----------------------------

In a :ref:`previous step <disable_unattended_upgrades>`, unattended-upgrades
were disabled on those cloud nodes that hosted multiple principal charms. Once
such a node has had all of its services upgraded, unattended-upgrades should be
re-enabled:

.. code-block:: none

   sudo dpkg-reconfigure -plow unattended-upgrades

Verify the new deployment
-------------------------

Check for errors in :command:`juju status` output and any monitoring service.

Example upgrade
---------------

The :doc:`OpenStack upgrade example <upgrade-openstack-example>` page shows the
explicit steps used to upgrade a basic cloud.

.. LINKS
.. _Release Notes: https://docs.openstack.org/charm-guide/latest/release-notes.html
.. _Ubuntu Cloud Archive: https://wiki.ubuntu.com/OpenStack/CloudArchive
.. _Upgrades: https://docs.openstack.org/operations-guide/ops-upgrades.html
.. _Percona strict mode: https://www.percona.com/doc/percona-xtradb-cluster/LATEST/features/pxc-strict-mode.html

.. BUGS
.. _LP #1825999: https://bugs.launchpad.net/charm-nova-compute/+bug/1825999
.. _LP #1809190: https://bugs.launchpad.net/charm-neutron-gateway/+bug/1809190
.. _LP #1853173: https://bugs.launchpad.net/charm-openstack-dashboard/+bug/1853173
.. _LP #1828534: https://bugs.launchpad.net/charm-designate/+bug/1828534

.. _aodh: https://opendev.org/openstack/charm-aodh/
.. _barbican: https://opendev.org/openstack/charm-barbican/
.. _barbican-vault: https://opendev.org/openstack/charm-barbican-vault/
.. _ceilometer: https://opendev.org/openstack/charm-ceilometer/
.. _ceilometer-agent: https://opendev.org/openstack/charm-ceilometer-agent/
.. _cinder: https://opendev.org/openstack/charm-cinder/
.. _cinder-backup: https://opendev.org/openstack/charm-cinder-backup/
.. _cinder-backup-swift-proxy: https://opendev.org/openstack/charm-cinder-backup-swift-proxy/
.. _cinder-ceph: https://opendev.org/openstack/charm-cinder-ceph/
.. _designate: https://opendev.org/openstack/charm-designate/
.. _glance: https://opendev.org/openstack/charm-glance/
.. _heat: https://opendev.org/openstack/charm-heat/
.. _keystone: https://opendev.org/openstack/charm-keystone/
.. _keystone-ldap: https://opendev.org/openstack/charm-keystone-ldap/
.. _keystone-saml-mellon: https://opendev.org/openstack/charm-keystone-saml-mellon/
.. _manila: https://opendev.org/openstack/charm-manila/
.. _manila-ganesha: https://opendev.org/openstack/charm-manila-ganesha/
.. _masakari: https://opendev.org/openstack/charm-masakari/
.. _masakari-monitors: https://opendev.org/openstack/charm-masakari-monitors/
.. _mysql-innodb-cluster: https://opendev.org/openstack/charm-mysql-innodb-cluster
.. _mysql-router: https://opendev.org/openstack/charm-mysql-router
.. _neutron-api: https://opendev.org/openstack/charm-neutron-api/
.. _neutron-api-plugin-arista: https://opendev.org/openstack/charm-neutron-api-plugin-arista
.. _neutron-api-plugin-ovn: https://opendev.org/openstack/charm-neutron-api-plugin-ovn
.. _neutron-dynamic-routing: https://opendev.org/openstack/charm-neutron-dynamic-routing/
.. _neutron-gateway: https://opendev.org/openstack/charm-neutron-gateway/
.. _neutron-openvswitch: https://opendev.org/openstack/charm-neutron-openvswitch/
.. _nova-cell-controller: https://opendev.org/openstack/charm-nova-cell-controller/
.. _nova-cloud-controller: https://opendev.org/openstack/charm-nova-cloud-controller/
.. _nova-compute: https://opendev.org/openstack/charm-nova-compute/
.. _octavia: https://opendev.org/openstack/charm-octavia/
.. _octavia-dashboard: https://opendev.org/openstack/charm-octavia-dashboard/
.. _octavia-diskimage-retrofit: https://opendev.org/openstack/charm-octavia-diskimage-retrofit/
.. _openstack-dashboard: https://opendev.org/openstack/charm-openstack-dashboard/
.. _placement: https://opendev.org/openstack/charm-placement
.. _swift-proxy: https://opendev.org/openstack/charm-swift-proxy/
.. _swift-storage: https://opendev.org/openstack/charm-swift-storage/

.. _ceph-fs: https://opendev.org/openstack/charm-ceph-fs/
.. _ceph-iscsi: https://opendev.org/openstack/charm-ceph-iscsi/
.. _ceph-mon: https://opendev.org/openstack/charm-ceph-mon/
.. _ceph-osd: https://opendev.org/openstack/charm-ceph-osd/
.. _ceph-proxy: https://opendev.org/openstack/charm-ceph-proxy/
.. _ceph-radosgw: https://opendev.org/openstack/charm-ceph-radosgw/
.. _ceph-rbd-mirror: https://opendev.org/openstack/charm-ceph-rbd-mirror/
.. _cinder-purestorage: https://opendev.org/openstack/charm-cinder-purestorage/
.. _designate-bind: https://opendev.org/openstack/charm-designate-bind/
.. _glance-simplestreams-sync: https://opendev.org/openstack/charm-glance-simplestreams-sync/
.. _gnocchi: https://opendev.org/openstack/charm-gnocchi/
.. _hacluster: https://opendev.org/openstack/charm-hacluster/
.. _ovn-central: https://opendev.org/x/charm-ovn-central
.. _ovn-chassis: https://opendev.org/x/charm-ovn-chassis
.. _ovn-dedicated-chassis: https://opendev.org/x/charm-ovn-dedicated-chassis
.. _pacemaker-remote: https://opendev.org/openstack/charm-pacemaker-remote/
.. _percona-cluster: https://opendev.org/openstack/charm-percona-cluster/
.. _rabbitmq-server: https://opendev.org/openstack/charm-rabbitmq-server/
.. _trilio-data-mover: https://opendev.org/openstack/charm-trilio-data-mover/
.. _trilio-dm-api: https://opendev.org/openstack/charm-trilio-dm-api/
.. _trilio-horizon-plugin: https://opendev.org/openstack/charm-trilio-horizon-plugin/
.. _trilio-wlm: https://opendev.org/openstack/charm-trilio-wlm/
.. _vault: https://opendev.org/openstack/charm-vault/
