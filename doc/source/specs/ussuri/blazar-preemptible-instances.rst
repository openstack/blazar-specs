..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Support preemptible instances with Blazar
=========================================

https://blueprints.launchpad.net/blazar/+spec/blazar-preemptible-instances

This spec proposes to add support for preemptible instances with Blazar. It
would allow to increase utilization by filling gaps between reservations, as
well as provide an implementation of preemptible instances that does not
require modifications to the Nova compute service.

Problem description
===================

To ensure resources allocated for reservations are not in use when reservations
reach their start time, Blazar relies on exclusive management of dedicated
pools of resources. In the case of compute resources, compute hosts managed by
Blazar are initially moved to a host aggregate named *freepool*. Regular
instances which are not launched inside a reservation are not allowed to run on
nodes in the *freepool*. This can lead to low utilization if reservations are
not making full use of the resources managed by Blazar. This spec proposes to
add support for preemptible instances, i.e. instances that can be terminated at
any time when required by the infrastructure, to increase utilization by
filling gaps between reservations.

This feature would also provide a minimally intrusive implementation of
preemptible instances for OpenStack: an alternative implementation based on
adding a `pending state`_ to Nova has proved difficult to be accepted.

Use Cases
---------

This feature primarily targets deployers who operate Blazar and want to
increase the utilization of their infrastructure by filling gaps between user
reservations. It would also benefit end users who can run preemptible workloads
by providing them with extra resources.

Proposed change
===============

Due to Blazar's design, preemptible instances can be supported with few
modifications to the code. The following mechanisms are required:

#. A way to identify different types of instance creation requests and of
   running instances: regular (non-reserved) instances, reserved instances, and
   preemptible instances.
#. A modification of the custom Nova scheduler filter, BlazarFilter, to allow
   instances considered as preemptibles to be launched on hosts located in the
   freepool and only in the freepool: we do not want preemptibles running on
   hosts not managed by Blazar or on hosts part of a reservation.
#. Before a lease becomes active, the Blazar compute plugins implementing
   instance reservation and host reservation would terminate any preemptible
   instance running on allocated hosts, identifying them using the method
   previously mentioned.

These mechanisms are described in more details below.

Identification of preemptible instance creation requests
--------------------------------------------------------

To identify an instance creation request as one for a preemptible instance,
some possible methods are:

#. Operators would configure via Blazar static configuration or API call a list
   of project IDs describes projects for which any instance creation request is
   treated as one for preemptibles.

   - Pros: can restrict preemptibles to specific projects
   - Cons: cannot allow preemptibles for all projects, unless a special globing
     pattern is used; may also need a method for projects to requests
     non-preemptible instances; may be difficult to identify running
     preemptibles.

#. Via scheduler hints: an instance creation request is identified as being for
   a preemptible instance if the scheduler hint ``preemptible=true`` is passed.

   - Pros: can be used by any project without configuration by operator
   - Cons: can be used by any project without configuration by operator; may be
     difficult to identify running preemptibles.

#. Via flavor properties: a flavor property is used to identify preemptible
   instances. For example, an existing flavor could be configured to launch
   preemptible instances by using: ``openstack flavor set FLAVOR-NAME
   --property blazar:preemptible=true``

   - Pros: flavor can be made public or shared only with specific projects;
     easy to identify running preemptibles.
   - Cons: duplicates the number of flavors if all flavors can potentially be
     preemptible

#. Via image properties: an image property is used to identify preemptible
   instances. For example, an existing image could be configured to launch
   preemptible instances by using: ``openstack image set --property
   blazar:preemptible=true IMAGE_UUID``

   - Pros: easy to identify running preemptibles
   - Cons: any project could use preemptibles, unless policy prevents setting
     image properties; would need to cover volumes as well; could lead to image
     duplication.

Identification of running preemptible instances
-----------------------------------------------

For compute hosts allocated to host reservations, running preemptible instances
are simply instances that are executing on allocated hosts outside of any
reservation.

For compute hosts allocated to instance reservation, instances launched as part
of a reservation are identified by the flavor they use, which is encoded as
``flavor_id`` in the reservation information. Blazar flavors also follow a
specific naming scheme: ``reservation:<reservation_id>``. Instances running on
allocated hosts that don't use such flavors would be identified as preemptibles.

Termination of preemptible instances
------------------------------------

Preemtible instances need to be terminated before a lease becomes active. The
Blazar compute plugins implementing instance reservation and host reservation
would terminate any preemptible instance running on allocated hosts,
identifying them using the previously mentioned method.

To allow preemptible instances to terminate their workload cleanly, a soft
shutdown operation could first be triggered, followed by a hard shutdown
(instance deletion). In order to keep the lease start operation quick, we may
want to implement a ``before_start_lease`` event, similar to the
``before_end_lease`` event, which would terminate any preemptible instance
prior to triggering the ``start_lease`` event.

However, we would also want to prevent any new preemptible from launching after
having sent instance termination requests. If using the ``before_start_lease``
approach, this could be difficult to implement from BlazarFilter which does not
have access to the Blazar database, and thus cannot efficiently check the
status of leases and events. API calls to Blazar may be too slow to process
large number of hosts going through the filter.

Interaction with quotas
-----------------------

Preemptible instances would count as regular instances for Nova quotas of
number of instances, vCPUs, memory, and disk. This means that projects having
reached their quota by running preemptible instances would not be able to
launch any new regular instance: they would first need to terminate preemptible
instances to reduce their usage below the maximum allowed by their quota.

This could be prevented by the operator by configuring dedicated projects to
exclusively run preemptible instances, i.e. not mix preemptible and regular
instances within the same project.

Alternatives
------------

A native implementation within Nova would be more flexible, as it would allow
preemptible instances to run on reserved hosts until they become actively used
by regular instances. However, a proposed implementation based on introducing a
`pending state`_ has proved difficult to be accepted by the community.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Race conditions between the Nova scheduler and the Blazar manager would have to
be handled correctly to prevent preemptible instances from launching while a
reservation is started. Failure to terminate all preemptible instances would
result in reservation owners not being able to fully use their reservation.

Notifications impact
--------------------

None

Other end user impact
---------------------

Nova end users would use one of the mechanisms described earlier to request
preemptible instances.

Performance Impact
------------------

Terminating preemptible instances during ``on_start()`` would make leases
longer to start, particularly if a soft shutdown signal is sent to instances.
A solution based on introducing a new event is described earlier in this spec,
but it has shortcomings.

Other deployer impact
---------------------

The following configuration options would be added to blazar.conf:

* [preemptible]/graceful_shutdown_time: time to wait between a soft shutdown
  request and a terminate request

Developer impact
----------------

None

Upgrade impact
--------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  priteau

Work Items
----------

1. Proof of concept with host reservation plugin only
2. Support for instance reservation
3. General improvements (e.g. soft shutdown signal)
4. Documentation and Tempest scenario testing

Dependencies
============

None

Testing
=======

A Tempest scenario verifying that preemptible instances can be launched and
terminated when a reservation starts must be implemented. This requires a full
OpenStack environment. The current DevStack can be used.

Documentation Impact
====================

Operators and users launching preemptible instances are most affected by this
change. The admin and user guides should be updated.

References
==========

.. _`pending state`: https://review.opendev.org/#/q/project:openstack/nova+topic:bp/introduce-pending-vm-state

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Ussuri
     - Introduced
