..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

======================================
Flexible Reservation Usage Enforcement
======================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/blazar/+spec/flexible-reservation-usage-enforcement

Blazar reservations provide temporary ownership of resources. In the absence of
quota imposed on reservations, users may be able to reserve all cloud resources
for long periods of time. To avoid resource shortage, operators may want to
restrict reservation usage from users. Because clouds have different usage
patterns, enforcement policies should be flexible. However, there are some
basic rules and policies that would apply to many clouds, such as restrictions
on the length of leases.

This specification proposes to enable policy restrictions via a chain of
filters to apply to a particular reservation request. By default, Blazar will
provide a set of simple default filters. Operators or developers can choose to
implement their own custom filters as Python modules. Additionally, Blazar will
provide one built-in filter that delegates to an external HTTP service that
provides a compatible interface. This specification includes a proposal for
that interface. The overall approach is analogous to Nova's scheduling filters.

Problem description
===================

Blazar reservations provide temporary ownership of resources. In the absence of
quota imposed on reservations, users may be able to reserve all cloud resources
for long periods of time. To avoid resource shortage, operators may want to
restrict reservation usage from users. Because clouds have different usage
patterns, enforcement policies should be flexible. However, there are some
basic rules and policies that would apply to many clouds, such as restrictions
on the length of leases; these types of policies ideally should not require
much setup from the operator.

Use Cases
---------

To ensure good resource sharing between users, an operator may want to apply
policies such as:

* limiting the duration of reservations
* limiting the size (number of hosts, instances, floating IPs) of reservations
* limiting the number of pending or active leases
* more complex policies taking into account compute node type, usage
  allocations per project, etc.

From the user's point of view, the Blazar interface would remain the same. The
only difference may be that an error message describes why a reservation
operation was denied.

Proposed change
===============

We propose to add a chain of *enforcement filters* to Blazar. The filters
will execute in series, with each filter taking as its input information about
the user's request (e.g., the lease length, its desired reservations, and the
user's ID and their project ID, etc.) The filter will either return no value,
or it will raise an error with a specific message about why the request failed.
If a filter returns no value, the next filter in the chain will have the chance
to respond to the input. If no filters raise an error, the user's request is
allowed.

We will implement two initial enforcement filters: the first will reject any
leases with length exceeding a threshold, and the second will delegate to
an external HTTP service. Filters that operate against Nova quotas, for
example, disallowing more reservations for physical hosts than a project's
instance quota allows for, would be future extensions of this pattern.

The enforcement filters will grant or deny reservation operations, specifically
lease create and lease update. Lease delete should generally not require an
approval, however, the enforcement filters should be informed when a lease ends
so that they can respond to early lease termination if desired.

The enforcement filters would thus be called at three specific points in the
reservation lifecycle:

1. During lease create, after allocation candidates have been selected, the
   lease and its reservations and candidate resources will be passed to the
   set of enforcement filters. If all of the filters pass the request, Blazar
   will create the allocations and transition the lease and reservations to the
   PENDING state as usual. If any filter rejects the request, then the lease
   will transition to the ERROR state, with a message from the failed filter
   included in the error raised to the user.

2. During lease update, after allocation candidates have been selected, both
   the prior lease/reservations/allocations and the new proposed values will
   be passed to the enforcement filters. If all of the filters pass the
   request, Blazar will perform the update to the lease as usual. If any filter
   rejects the request, then an error with a message from the failed filter
   will be raised to the user. The lease will not transition to an ERROR state;
   it will remain in its prior state.

3. On lease end, pass the lease and its reservations and current allocated
   resources to the enforcement filters. The filter result in this case does
   not affect the rest of the lease lifecycle and the lease should continue its
   on_end actions. This integration point acts more like a notification, and
   is intended to support budget-based policies that may be affected by a
   reservation's early termination (e.g., by refunding the user/project the
   unused portion of the lease in a "charge up front" model.)

Filter API
----------

Lease, reservation, and allocation information will be passed to each filter.
For leases and reservations, the subset of user-controllable attributes (e.g.,
``start_date``, ``end_date``, ``resource_type`` and plugin-specific reservation
attributes) will be passed to the filter; UUIDs and other internal bookkeeping
attributes such as ``created_at`` will not be sent. Each allocation object will
similarly contain the relevant operator-defined attributes, e.g.,
``hypervisor_properties`` for ``physical:host`` reservations and
``floating_network_id`` for ``virtual:floatingip`` reservations. If extra
capabilities are defined for the resource, they will additionally be sent. This
allows for operator-defined custom metadata to be tagged to resources and later
utilized when determining policy decisions.

Each filter must support the following operations:

* ``check_create``: Check that a new lease request can be satisfied. Receives
  a ``context`` object and a ``lease`` object as arguments.

  * The context contains information about the request:

  .. sourcecode:: json

    {
      "user_id": "c631173e-dec0-4bb7-a0c3-f7711153c06c",
      "project_id": "a0b86a98-b0d3-43cb-948e-00689182efd4",
      "auth_url": "https://api.example.com:5000/v3",
      "region_name": "RegionOne"
    }

  * The lease object contains the user-definable aspects of the lease and its
    reservations, and includes the allocations already chosen by Blazar:

  .. sourcecode:: json

    {
      "start_date": "2020-05-13 00:00",
      "end_time": "2020-05-14 23:59",
      "reservations": [
        {
          "resource_type": "virtual:floatingip",
          "network_id": "external-network-id",
          "amount": 1,
          "allocations": [
            {
              "floating_network_id": "external-network-id",
              "floating_ip_address": "192.168.1.100",
              "id": "7375755f-717e-4883-91c1-06ceba2da96e"
            }
          ]
        },
        {
          "resource_type": "physical:host",
          "min": 1,
          "max": 2,
          "hypervisor_properties": "[]",
          "resource_properties": "[\"==\", \"$availability_zone\", \"az1\"]",
          "allocations": [
            {
              "id": "1",
              "hypervisor_hostname": "32af5a7a-e7a3-4883-a643-828e3f63bf54",
              "extra": {
                "availability_zone": "az1"
              }
            }
          ]
        }
      ]
    }

* ``check_update``: Check that a lease update request can be satisfied.
  Receives a ``context`` object and both the lease's current state and
  desired state as arguments. Sending both sets of state relieve the filter
  from having to look up the information itself.

  **Note**: a project lease can be created by one user and updated
  by another. In the update request, the ``user_id`` is the ID of the user
  performing the update.

  * The context contains information about the request (see above).

  * The lease (current and requested) states contain the user-definable aspects
    of the lease and include the allocations chosen by Blazar.

* ``on_end``: Notify the filter that a lease is terminating. Receives a
  ``context`` object and the lease's current state as arguments.

  * The context contains information about the request (see above).

  * The lease contains the user-definable aspects of the lease and include the
    allocations chosen by Blazar.

MaximumReservationLengthFilter
------------------------------

This filter simply examines the lease's ``end_time`` and ``start_time`` and
rejects the lease if its length exceeds a threshold.

ExternalServiceFilter
---------------------

This filter delegates the decision for each API to an external HTTP service.
The service should adhere to the `API-WG HTTP Response Codes`_ guidelines
and return the appropriate HTTP status for success versus error responses
(as described in the following endpoint specifications.)

A Keystone token for the ``blazar`` service user will be sent to the usage
service via the ``X-Auth-Token`` header for all endpoints and can be used to
verify a request's authenticity (to prevent spoofing or abuse).

External service API
^^^^^^^^^^^^^^^^^^^^

To be compatible with Blazar's filter, the HTTP service must provide the
following interface:

* ``POST /v1/check-create``

  * Check that a new lease request can be satisfied with the usage policies.

  * Normal response code: ``204 No Content``

  * Error response codes:

    * ``403 Forbidden``: if the lease is determined to not be allowed
      according to the implemented usage policies. An error message can be
      returned as JSON in the response body.

  * Request example:

  .. sourcecode:: json

    {
      "context": {
        "user_id": "c631173e-dec0-4bb7-a0c3-f7711153c06c",
        "project_id": "a0b86a98-b0d3-43cb-948e-00689182efd4",
        "auth_url": "https://api.example.com:5000/v3",
        "region_name": "RegionOne"
      },
      "lease": {
        "start_date": "2020-05-13 00:00",
        "end_time": "2020-05-14 23:59",
        "reservations": [
          {
            "resource_type": "virtual:floatingip",
            "network_id": "external-network-id",
            "amount": 1,
            "allocations": [
              {
                "floating_network_id": "external-network-id",
                "floating_ip_address": "192.168.1.100",
                "id": "7375755f-717e-4883-91c1-06ceba2da96e"
              }
            ]
          },
          {
            "resource_type": "physical:host",
            "min": 1,
            "max": 2,
            "hypervisor_properties": "[]",
            "resource_properties": "[\"==\", \"$availability_zone\", \"az1\"]",
            "allocations": [
              {
                "id": "1",
                "hypervisor_hostname": "32af5a7a-e7a3-4883-a643-828e3f63bf54",
                "extra": {
                  "availability_zone": "az1"
                }
              }
            ]
          }
        ]
      }
    }

  * Error response body example:

  .. sourcecode:: json

    {
      "message": "Your lease exceeds the maximum length of 24 hours."
    }

* ``POST /v1/check-update``

  * Check that a lease update request can be satisfied with the usage policies.

  * Normal response code: ``204 No Content``

  * Error response codes:

    * ``403 Forbidden``: if the lease's updated state is determined not to be
      allowed according to the usage policies. An error message can be
      returned as JSON in the response body.

  * Request example:

  .. sourcecode:: json

    {
      "context": {
        "user_id": "c631173e-dec0-4bb7-a0c3-f7711153c06c",
        "project_id": "a0b86a98-b0d3-43cb-948e-00689182efd4",
        "auth_url": "https://api.example.com:5000/v3",
        "region_name": "RegionOne"
      },
      "current_lease": {
        "start_date": "2020-05-13 00:00",
        "end_time": "2020-05-14 23:59",
        "reservations": [
          {
            "resource_type": "physical:host",
            "min": 1,
            "max": 2,
            "hypervisor_properties": "[]",
            "resource_properties": "[\"==\", \"$availability_zone\", \"az1\"]",
            "allocations": [
              {
                "id": "1",
                "hypervisor_hostname": "32af5a7a-e7a3-4883-a643-828e3f63bf54",
                "extra": {
                  "availability_zone": "az1"
                }
              }
            ]
          }
        ]
      },
      "lease": {
        "start_date": "2020-05-13 00:00",
        "end_time": "2020-05-14 23:59",
        "reservations": [
          {
            "resource_type": "physical:host",
            "min": 2,
            "max": 3,
            "hypervisor_properties": "[]",
            "resource_properties": "[\"==\", \"$availability_zone\", \"az1\"]",
            "allocations": [
              {
                "id": "1",
                "hypervisor_hostname": "32af5a7a-e7a3-4883-a643-828e3f63bf54",
                "extra": {
                  "availability_zone": "az1"
                }
              },
              {
                "id": "2",
                "hypervisor_hostname": "af69aabd-8386-4053-a6dd-1a983787bd7f",
                "extra": {
                  "availability_zone": "az1"
                }
              }
            ]
          }
        ]
      }
    }

  * Error response body example:

  .. sourcecode:: json

    {
      "message": "Your project is limited to reserving 1 physical host."
    }

* ``POST /v1/on-end``

  * Notify the usage service that a lease is terminating.

  * Normal response code: ``204 No Content``

  * Error response codes: None

  * Request example:

  .. sourcecode:: json

    {
      "context": {
        "user_id": "c631173e-dec0-4bb7-a0c3-f7711153c06c",
        "project_id": "a0b86a98-b0d3-43cb-948e-00689182efd4",
        "auth_url": "https://api.example.com:5000/v3",
        "region_name": "RegionOne"
      },
      "lease": {
        "start_date": "2020-05-13 00:00",
        "end_time": "2020-05-14 23:59",
        "reservations": [
          {
            "resource_type": "physical:host",
            "min": 1,
            "max": 2,
            "hypervisor_properties": "[]",
            "resource_properties": "[\"==\", \"$availability_zone\", \"az1\"]",
            "allocations": [
              {
                "id": "1",
                "hypervisor_hostname": "32af5a7a-e7a3-4883-a643-828e3f63bf54",
                "extra": {
                  "availability_zone": "az1"
                }
              }
            ]
          }
        ]
      }
    }

.. _API-WG HTTP Response Codes: https://specs.openstack.org/openstack/api-wg/guidelines/http/response-codes.html

Alternatives
------------

This approach encompasses several competing designs, notably including standard
policies within Blazar (addressed by the inclusion of default filters that
address common use-cases) and Python module plugins that provide filters
(addressed by the ability to provide custom plugins similarly to the mechanism
used for Nova scheduler filters).

An alternative is to simplify the approach to only allow one method of
use/extension. However, this limits the functionality's future utility due
to additional constraints placed on the operator/developer.

Data model impact
-----------------

None

REST API impact
---------------

None

Security impact
---------------

Information about users, projects, and their usage will be sent to an external
web service. If the communication with this service isn't secure, an attacker
may be able to capture this information, approve reservation operations which
should have been denied, and deny operations which should have been approved.
An attacker may also be able to create bogus requests that nonetheless cause
real usage charges to accrue on the target account; for this reason, the
service should be exposed internally on the same service as Blazar, or should
at minimum check the ``X-Auth-Token`` header contains a valid token for the
``blazar`` service account.

A high level of reservation operations to Blazar would create a similar level
of calls to the policy web service, which may cause a denial of service.

Notifications impact
--------------------

None

Other end user impact
---------------------

From an end user point of view, the Blazar interface would remain the same. The
only difference may be that an error message describes why a reservation
operation was denied.

Performance Impact
------------------

Reservation operations can trigger synchronous calls to other services when
evaluating enforcement filters, which may cause a slowdown of blazar-manager
RPC replies, and thus a slowdown of API responses. Monitoring the performance
of the external enforcement service is important, when one is used.

Other deployer impact
---------------------

A new configuration section ``[enforcement]`` would be added, with one initial
property:

* ``available_filters``: a list of modules that provide enforcement filters.
  Can be specified multiple times. Defaults to
  ``blazar.enforcement.filters.all_filters``
* ``enabled_filters``: a comma-separated list of filter names, in the order
  that they will execute. Defaults to
  ``MaximumReservationLengthFilter,ExternalServiceFilter``
* ``reservation_max_length``: the maximum length, in seconds, of a reservation.
  This is used by the ``MaximumReservationLengthFilter``. Defaults to 0,
  meaning there is no limit, effectively disabling this filter.
* ``exempted_projects``: a list of project names or IDs that are not subject
  to any enforcement filter. If project names are used, the domain name must
  be provided, e.g. ``my-project@Default`` in the case of the "Default" domain.

Additionally, a new configuration section ``[enforcement_external]`` would be
added for the ``ExternalServiceFilter``, with the following properties:

* ``endpoint_url``: if set to a URL, enables enforcement of reservation usage
  policies. Defaults to None, which effectively disables this filter.
* ``allow_on_error``: if the external enforcement service responds with an HTTP
  error code or an error occurs elsewhere in the transport layer, allow the
  reservation request. Defaults to False.

Developer impact
----------------

Developer wanting to test reservation usage enforcement will need to deploy an
extra web service. We may want to package an example one with Blazar.

Upgrade impact
--------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  diurnalist

Work Items
----------

* Implement enforcement filter mechanism (default pass all requests)
* Implement filter that checks for maximum reservation length
* Implement filter for external enforcement service
* Add scenario test to blazar-tempest-plugin
* Update Blazar documentation

Dependencies
============

None

Testing
=======

Unit tests and, if possible, a scenario test should be implemented.

Documentation Impact
====================

The configuration documentation will need to be updated with the new
configuration option. Calls to the policy web service should also be
documented so that operators can write their own services.

References
==========

1. IRC meeting discussion: http://eavesdrop.openstack.org/meetings/blazar/2019/blazar.2019-05-23-16.00.log.html
2. Nova scheduler filters: https://docs.openstack.org/nova/latest/user/filter-scheduler.html

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Ussuri
     - Introduced
