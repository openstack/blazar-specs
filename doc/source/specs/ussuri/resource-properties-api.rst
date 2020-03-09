..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Resource Properties Discovery API
=================================

https://blueprints.launchpad.net/blazar/+spec/resource-properties-discovery-api

We propose to add an API that allows users to retrieve a list of all possible
resource properties: which keys are supported for each resource type, and what
range of values are valid for those keys. This will allow users to craft
resource requests with more precision, and allows more advanced workflows in
tooling and the Horizon dashboard.

Problem description
===================

Currently, only operators are given permission to enumerate resources
registered in Blazar. Users are allowed to make lease requests and can provide
resource constraints, yet have no way of identifying what constraints are
valid/possible. Providing some mechanism that allows users to understand which
resource keys and values are allowed can improve the user experience around
reservations.

Use Cases
---------

* As a user, I know I can reserve resources based on resource properties. I
  want to know what resource properties exist for my target pool of resources
  so I can utilize this functionality effectively.
* As an operator, I want users to be able to automatically discover changes I
  make to Blazar resources without announcing them or otherwise updating a
  separate set of documentation.
* As a CLI user, I want to be able to retrieve a list of resource properties
  so I can make future valid CLI requests for reservations.

Proposed change
===============

We propose adding a new REST API endpoint that, for each resource type, returns
a list of resource properties and their possible values. This API can be
exposed to users to query directly.

Alternatives
------------

Instead of exposing a new API to users, a better user interface can be
implemented in the blazar-dashboard plugin. An admin token can fetch a list of
all resources and the properties can be read from there and presented to the
user in a number of ways. However, the general pattern with Horizon is that the
user's token is used for all requests, and adding admin tokens would require
configuring Horizon with the Blazar service user credentials.

Another alternative is to allow users to fetch the Blazar resources themselves
and do their own inspection. By default, only admins are allowed to list and
view resources. Operators are able to loosen this restriction, though this
risks exposing sensitive data about the resource accidentally. Additionally,
this workflow is less intuitive and doesn't scale well when comparing a large
amount of resources.

Data model impact
-----------------

Extra capabilities will gain an additional property, ``private``, which marks
them as non-enumerable. This allows operators to add sensitive metadata to
resources without worrying about exposing them to users.

* The notion of 'private' capabilities has not existed prior, and assumes that
  a given property will be either private for all resources, or public for all.
  Setting some resources to have a private value and others a public value will
  not be supported by the API. However, the existing data model theoretically
  supports it, as capabilities are not stored in a normal form (they are simply
  rows with a given key, value, and attached resource ID).

* To ensure that the 'private' constraint holds, a new table will be created:

  .. sourcecode:: sql

    CREATE TABLE extra_capabilities (
      id VARCHAR(36) NOT NULL,
      resource_type VARCHAR(255) NOT NULL,
      capability_name VARCHAR(255) NOT NULL,
      private BOOLEAN NOT NULL,

      PRIMARY key (id),
      UNIQUE INDEX (resource_type, capability_name),
    );

* A migration will be included that creates ``extra_capability`` entries for
  all existing capabilities, and then updates the extra_capability tables for
  any resources that implement this feature, removing the capability_name
  column and replacing it with capability_id.

* Normalizing the extra capabilities means the constraint lookups for existing
  resources will require joining against the extra_capabilities table.

REST API impact
---------------

Because the space of properties and their valuesets for a given resource type
may be quite large, this functionality will be broken into two endpoints: one
to list all possible keys, and another to list possible values for each key.

* ``GET /v1/<resource_type>/properties``

  * Returns a list of public properties supported by the resource type.

  * Normal response code: ``200 OK``

  * Request query parameters:

    * ``detail`` (optional boolean): whether to return the list of possible
      values for each property in addition to the property name. If present, an
      additional ``values`` key will be returned with each property item, with
      a list of all the values, similar to how they are returned in the endpoint
      that returns values for a given property name.

  * Error response code(s):

    * ``404 Not Found``: if the resource type is not valid.

  * Example response (for ``physical:host`` resource type):

  .. sourcecode:: json

    [
      {
        "property": "local_gb"
      },
      {
        "property": "memory_mb"
      },
      {
        "property": "custom_capabilities.first"
      },
      {
        "property": "custom_capabilities.second"
      }
    ]

* ``GET /v1/<resource_type>/properties/<property_name>``

  * Returns a list of valid values for the property name.

  * Normal response code: ``200 OK``

  * Error response code(s):

    * ``403 Forbidden``: if the property is marked private, but is requested
      with a non-admin token.

    * ``404 Not Found``: if the property name is not valid for the resource
      type. A message indicating this will be returned in the response body.

  * Example response (for an ``arch`` property of ``physical:host`` resources):

  .. sourcecode:: json

    {
      "private": false,
      "values": [
        {
          "value": "x86"
        },
        {
          "value": "arm"
        }
      ]
    }

* ``PATCH /v1/<resource_type>/properties/<property_name>``

  * Updates a given property with some metadata. Currently the ``private``
    option is the only supported.

  * Normal response code: ``204 No Content``

  * Error response code(s):

    * ``403 Forbidden``: if the policy disallows ``put:extra_capability`` for
      the requesting user.

    * ``404 Not Found``: if the property name is not valid for the resource
      type. A message indicating this will be returned in the response body.

  * Example request:

  .. sourcecode:: json

    {
      "private": true
    }

Security impact
---------------

All capabilities will be private by default, which should reduce the likelihood
of exposure. By default, future capabilities will also be private. Operators
will be able to override this behavior via the ``something`` flag.

We could improve security at the expense of operator experience if we require
that capabilities are explicitly registered before being attached to a
resource, instead of creating new capabilities on-demand when a resource is
updated to have a property that has not yet been seen for the resource type.

Notifications impact
--------------------

None

Other end user impact
---------------------

The Blazar CLI client will be updated to support the following functions:

* ``host-capability-list``: lists all properties for physical:host resources
* ``host-capability-get <name>``: gets all values for a given host capability
* ``host-capability-set <name> [--private|--public]``: updates the visibility
  for a given host capability
* If we want to enforce creating capabilities explicitly:
  ``host-capability-create <name> [--private|--public]``: creates a new host
  capability with given visibility settings.

Performance Impact
------------------

The normalization of the capabilities into a new table introduces an additional
join when looking up resources for allocation candidates, which could add time
to that operation. In practice this overhead should be low, as the space of the
capability table will be quite small, and an index will be used over the
resource type and capability name, which would be the two columns needed by
the allocation selection.

Other deployer impact
---------------------

This new API will default to only be accessible to operators to mitigate risk
of accidentally exposing properties to users. Operators will have to opt-in by
allowing all users to access the API, after making sure that any sensitive
capabilities they have attached to resources are made private. We may elect
to change the default visibility for the API in a future release.

* ``[DEFAULT] capability_default_visibility``: Defines how new capabilities
  added to resources are exposed for end-users. Defaults to ``private``, meaning
  users will not be able to enumerate the property. Operators can set this to
  ``public`` to instead default capabilities to public visibility.

Developer impact
----------------

None

Upgrade impact
--------------

A database migration will be required to add the new column to the extra
capability tables.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jasonandersonatuchicago

Work Items
----------

* Create REST API endpoints for retrieving keys/value sets
* Create database migration for existing extra capabilities
* Update Blazar CLI to support discovering resource properties
* Update Blazar CLI to support updating resource properties as private

Dependencies
============

None

Testing
=======

Unit tests will be written, and a scenario test for the new API endpoints, if
the supporting infrastructure for these tests is in place.

Documentation Impact
====================

REST API documentation will need to be updated to include these new endpoints.
A release note will indicate the new API and the ability it grants users.

References
==========

* IRC discussion: http://eavesdrop.openstack.org/meetings/blazar/2020/blazar.2020-02-27-16.00.log.html

History
=======

Optional section intended to be used each time the spec is updated to describe
new design, API or any database schema updated. Useful to let reader understand
what's happened along the time.

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Ussuri
     - Introduced
