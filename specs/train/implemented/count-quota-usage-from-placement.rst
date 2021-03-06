..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
Count quota usage from placement
================================

https://blueprints.launchpad.net/nova/+spec/count-quota-usage-from-placement

In Pike, we re-architected the quota system to count actual resource usage
instead of using reservations and tracking quota usages in a separate database
table. We're counting resources like instances, CPU, and RAM by querying each
cell database and aggregating the results per project and per user. This
approach is problematic in the context of "handling of a down cell. If a cell
becomes unavailable, resources in its database cannot be counted and will not
be included in resource usage until the cell returns. Cells could become
unavailable if an operator is performing maintenance on a cell or if a cell
database is experiencing problems and we cannot connect to it.

We can make resource usage counting for quotas resilient to temporary cell
outages by querying placement and the API database for resource usage instead
of reading separate cell databases.


Problem description
===================

When we count quota resource usage for CPU and RAM, we do so by reading
separate cell databases and aggregating the results. CPU and RAM amounts per
instance are derived from the flavor and are stored in the database as columns
in the instances table. So, each time we check quota usage against limits, we
query a count of instance ID and a sum of CPU and RAM per cell database and
aggregate them to calculate the resource usage.

This approach is sensitive to temporary cell outages which may occur during
operator maintenance or if a cell database is experiencing issues and we cannot
connect to it. While a cell is unavailable, we cannot count resource usage
residing in that cell database and things would behave as though more quota is
available than should be. That is, if someone has used all of their quota and
part of it is in cell A and cell A goes offline temporarily, that person will
suddenly be able to allocate more resources than their limit (assuming cell A
returns, the person will have more resources allocated than their allowed
quota).

We could take a different approach by querying the placement API and the API
database to get resource usage counts. Since placement is managing resource
allocations, it has the information we need to count resource usage for CPU and
RAM quotas. By querying placement and the API database, we can avoid reading
separate cell databases for resource usage.

Use Cases
---------

Counting quota resource usage from placement would make quota behavior
consistent in the event of temporary cell database disruptions. It would be
easier for Operators to take cells offline if needed for maintenance without
concern about the possibility of quota limits being exceeded during the
maintenance. It could spare Operators the trouble of potentially having to fix
cases where quota has been exceeded during maintenance or if a cell database
connection could not be established.

Proposed change
===============

We will add a new method for counting instances that queries the
``instance_mappings`` table in the API database and make a separate limit check
for number of instances.

The new method will contain:

* One query to the API database to get resource usage for instances. We can get
  the number of instances for a project and user if we add a new column
  ``user_id`` to the ``nova_api.instance_mappings`` table. We already have a
  ``project_id`` column on the table. This will allow us to count instance
  mappings for a project and a user to represent the instance count.

We will rename the ``_instances_cores_ram_count`` method to
``_instances_cores_ram_count_legacy`` that counts cores and ram from the cell
databases and is only used if ``[quota]count_usage_from_placement`` is False or
if the data migration has not yet completed.

Because there is not yet an ability to partition resource providers in
placement, in order to support deployments where multiple Nova deployments
share the same placement service, like possibly in an Edge scenario, we will
add a ``[quota]count_usage_from_placement`` config option which defaults to
False. If False, we use the legacy quota counting method for instances, cores,
and ram. If True, we use a quota counting method that calls placement. This is
a way to keep "legacy" quota counting available for the scenario of multiple
Nova deployments sharing one placement service. The config option will simply
control which counting method will be called by the pluggable quota system.
For example (pseudo-code):

::

    if CONF.quota.count_usage_from_placement:
        return _instances_cores_ram_count_api_db_placement(...)
    else:
        return _instances_cores_ram_count_legacy(...)

We will add a new method for counting cores and ram from placement that is used
when ``[quota]count_usage_from_placement`` is True. This method could be called
``_cores_ram_count_placement``.

The new method will contain:

* Up to two calls to placement to get resource usage for CPU and RAM. One call
  will count usage across a project. Then, if user-scoped quota limits are
  found for a resource, a second call will count usage across a project and a
  user.
  We can get CPU and RAM usage for a project and user by querying the
  ``/usages`` resource::

    GET /usages?project_id=<project id>
    GET /usages?project_id=<project id>&user_id=<user id>

Alternatives
------------

One alternative is to hold off on counting any quota usage from placement
until placement has allocation partitioning support. The problem with that is
in the meantime, the only solution we have for handling of down cells is to
implement the `policy-driven behavior`_ where an operator has to choose between
failing server create requests when a project has instances in a down cell or
allowing server create requests to potentionally exceed quota limits.

Another alternative which has been discussed is, to use placement aggregates to
surround each entire Nova deployment and use that as a means to partition
placement usages. We would need to add a ``aggregate=`` query parameter to the
placement ``/usages`` API in this case. This approach would also require some
work by either Nova or the operator to keep the placement aggregate
synchronized.

.. _policy-driven behavior: https://review.openstack.org/614783

Data model impact
-----------------

A nova_api database schema change will be required for adding the ``user_id``
column of type String to the ``nova_api.instance_mappings`` table.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

End users will see consistent quota behavior even when cell databases are
unavailable.

Performance Impact
------------------

There will be a performance impact for checking if data needs to be migrated at
the time of the quota check. The impact can be reduced by caching the results
of checks that indicate data migration has been completed for a project and
avoid a useless check per project in that case.

The change involves making external REST API calls to placement instead of
doing a parallel scatter-gather to all cells. It might be slower to make the
external REST API calls if all cells are fast responding. It might be faster to
make external REST API calls if any cells are slower responding.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Upgrade impact
--------------

The addition of the ``user_id`` column to the ``nova_api.instance_mappings``
table will require a data migration of all existing instance mappings to
populate the ``user_id`` field. The migration routine would look for mappings
where ``user_id`` is None and query cells by corresponding ``cell_id`` in
the mapping. The query could filter on instance UUIDs, finding the ``user_id``
values to populate in the mappings. This would implement the batched
``nova-manage db online_data_migrations`` way of doing the migration.

We will also heal/populate an instance mapping on-the-fly when it is accessed
during a server GET request. This would provide some data migration in the
situation where an upgrade has not run
``nova-manage db online_data_migrations`` yet.

In order to handle a live in-progress upgrade, we will need to be able to fall
back on the legacy counting method for instances, cores, and ram if
``nova_api.instance_mappings`` do not yet have ``user_id`` populated (if the
operator has not yet run the data migration). We will need a way to detect that
the migration has not yet been run in order to fall back on the legacy counting
method. We could have a check such as ``if exists(InstanceMapping.id) where
project_id=<project id> and user_id=None``, then fall back on the legacy
counting method to query cell databases. We should cache the results of the
each migration completeness check per ``project_id`` so we avoid needlessly
checking a ``project_id`` that has already been migrated every time quota is
checked.

We will populate the ``user_id`` field even for instance mappings that are
``queued_for_delete=True`` because such instance mappings include instances
that are ``SOFT_DELETED`` and these can be restored at any time in the future.
If we do not migrate ``SOFT_DELETED`` instances with ``queued_for_delete=True``
and they are restored in the future, their instance mappings would be
unmigrated and would prevent us being able to eventually drop the related data
migration code.

The data migrations and fallback to the legacy counting method will be
temporary for Train, to be dropped in U or V with a blocker migration. That is,
you cannot pass ``nova-manage api_db sync`` if there are any instance mappings
with ``user_id=None`` to force the batched migration using ``nova-manage``.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  melwitt

Other contributors:
  None

Work Items
----------

* Add a new column ``user_id`` to the ``nova_api.instance_mappings`` table.
* Implement an online data migration to populate the ``user_id`` field.
* Update the ``_server_group_count_members_by_user`` quota counting method to
  use only the ``nova_api.instance_mappings`` table instead of querying cell
  databases.
* Add a config option ``[quota]count_usage_from_placement`` that
  defaults to False. This will be able to be deprecated when partitioning of
  resource providers is available in placement and other quirks around
  placement resource allocations in Nova are resolved in the future (example:
  "doubling" of allocations during resizes).
* Add a new method to count instances with a count of
  ``nova_api.instance_mappings`` filtering by ``project_id=<project_id>`` and
  ``user_id=<user_id>`` and ``queued_for_delete=False``.
* Add a new count method that queries the placement API for CPU and RAM usage.
  In the new count method, add a check for whether the online data migration
  has been run yet and if not, fall back on the legacy count method.
* Rename the ``_instances_cores_ram_count`` method to
  ``_instances_cores_ram_count_legacy`` and let it count only cores and ram in
  the legacy way, for use if ``[quota]count_usage_from_placement`` is False or
  the data migration is not yet completed.
* Adjust the nova-next CI job to run with
  ``[quota]count_usage_from_placement=True``.

Dependencies
============

None.

Testing
=======

Unit tests and functional tests will be included to test the new functionality.
We will also adjust one CI job (nova-next) to run with
``[quota]count_usage_from_placement=True`` to make sure we have integration
test coverage of that path.

Documentation Impact
====================

The documentation_ of Cells v2 caveats will be updated to update the paragraph
about the inability to correctly calculate quota usage when one or more cells
are unreachable. We will document that beginning in Train, there are new
deployment options.

.. _documentation: https://docs.openstack.org/nova/latest/user/cellsv2-layout.html#quota-related-quirks

References
==========

This builds upon the work done in Pike to re-architect quotas to count
resources.

* http://specs.openstack.org/openstack/nova-specs/specs/pike/approved/cells-count-resources-to-check-quota-in-api.html

This may also inadvertantly fix a bug we have where if the "recheck" quota
check fails during the conductor check and the request is a multi-create, we
will have all servers fall into ERROR state for the user to clean up. Because
this change will count instance mappings for the instance count and instance
mappings have almost [*]_ the same lifetime as build requests, we should not
see the behavior of multi-create servers in ERROR state if they fail the quota
"recheck" in conductor.

* https://bugs.launchpad.net/nova/+bug/1716706

.. [*] We create build request and instance mapping in separate database
       transactions, so there is a tiny window where build request can exist
       without a corrensponding instance mapping.

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Stein
     - Introduced
   * - Train
     - Re-proposed
