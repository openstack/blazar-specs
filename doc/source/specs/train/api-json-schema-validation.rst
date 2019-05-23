..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

================================
API validation using jsonschema
================================

https://blueprints.launchpad.net/blazar/+spec/json-schema-validation

Currently, Blazar has different implementations for validating request
bodies. The purpose of this blueprint is to validate the request body
in the API layer using json schema validation before it is forwarded to
any of the other Blazar components. Validating the request bodies sent to
the Blazar server, accepting requests that fit the resource schema and
rejecting requests that do not fit the schema. Depending on the content of the
request body, the request should be accepted or rejected.


Problem description
===================

Currently Blazar doesn't have a consistent request validation layer. Some
resources validate input at the resource controller and some fail out in the
backend. Ideally, Blazar would have some validation in place to catch
disallowed parameters and return a validation error to the user.

The end user will benefit from having consistent and helpful feedback,
regardless of which resource they are interacting with.


Use Cases
=========

As a user or developer, I want to observe consistent API validation and values
passed to the Blazar API server.


Proposed change
===============

One possible way to validate the Blazar API is to use jsonschema similar to
Nova, Keystone and Glance (https://pypi.python.org/pypi/jsonschema).
A jsonschema validator object can be used to check each resource against an
appropriate schema for that resource. If the validation passes, the request
can follow the existing flow of control through the resource manager to the
backend. If the request body parameters fails the validation specified by the
resource schema, a validation error wrapped in HTTPBadRequest will be returned
from the server.


Alternatives
------------

Before the API validation framework, we needed to add the validation code into
each API method in ad-hoc. These changes would make the API method code dirty
and we need to create multiple patches due to incomplete validation.

If using JSON Schema definitions instead, acceptable request formats are clear
and we don't need to do ad-hoc works in the future.


Data model impact
-----------------

None


REST API impact
---------------

API Response code changes:

There are some occurrences where API response code will change while adding
schema layer for them. For example, On current master 'computehosts' table has
'hypervisor_hostname' of maximum 255 characters in database table. While creating
host user can pass 'host' of more than 255 characters which
obviously fails with 404 HostNotFound wasting a database call. For this we
can restrict the 'host' of maximum 255 characters only in schema
definition of 'os-hosts'. If user passes more than 255 characters, he/she will
get 400 BadRequest in response.

API Response error messages:

There will be change in the error message returned to user. For example,
On current master if user passes nothing for host name like "name": ""
then below error message is returned to user from blazar-api:

"Host '' not found!".

With schema validation below error message will be returned to user for this
case:

Invalid input for field/attribute name. Value: <value passed by user>.
'<value passed by user>' is too short.


Security impact
---------------

The output from the request validation layer should not compromise data or
expose private data to an external user. Request validation should not
return information upon successful validation. In the event a request
body is not valid, the validation layer should return the invalid values
and/or the values required by the request, of which the end user should know.
The parameters of the resources being validated are public information,
described in the `Blazar API reference document`_ with the exception of
private data. In the event the user's private data fails validation, a check
can be built into the error handling of the validator not to return the actual
value of the private data.

`Jsonschema documentation`_ notes security considerations for both schemas and
instances.

Better up front input validation will reduce the ability for malicious user
input to exploit security bugs.


Notifications impact
--------------------

None

Other end user impact
---------------------

None

Performance Impact
------------------

Blazar will need some performance cost for this comprehensive request
parameters validation, because the checks will be increased for API parameters
which are not validated now.


Other deployer impact
---------------------

None


Developer impact
----------------

This will require developers contributing new extensions to Blazar to have
a proper schema representing the extension's API.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
Asmita Singh : <asmita.singh@nttdata.com>

Work Items
----------

1. Initial validator implementation, which will contain common validator code
   designed to be shared across all resource controllers validating request
   bodies.
2. Introduce validation schemas for existing API resources.
3. Enforce validation on proposed API additions and extensions.
4. Remove duplicated ad-hoc validation code.
5. Add unit and end-to-end tests of related APIs.
6. Add/Update Blazar documentation.

Dependencies
============

None


Testing
=======

Some tests can be added as each resource is validated against its schema.
These tests should walk through invalid request types.

Documentation Impact
====================

1. The Blazar API documentation will need to be updated to reflect the
   REST API changes.
2. The Blazar developer documentation will need to be updated to explain
   how the schema validation will work and how to add json schema for
   new APIs.


References
==========

Some useful links:

* `JSON Schema`_ A Media Type for Describing JSON Documents
* `Validation examples`_ used in Nova project for jsonschema validation
* `Library PyPI jsonschema`_ for implementation of JSON Schema validation for Python
* Explaining the JSON Schema `Core Definitions and Terminology`_
* JSON Schema `specification documentation`_

.. _`Blazar API reference document`: https://developer.openstack.org/api-ref/reservation/
.. _`Jsonschema documentation`: http://json-schema.org/latest/json-schema-core.html#anchor21
.. _`JSON Schema`: http://spacetelescope.github.io/understanding-json-schema/reference/object.html
.. _`Validation examples`: https://opendev.org/openstack/nova/src/branch/master/nova/api/validation
.. _`Library PyPI jsonschema`: https://pypi.python.org/pypi/jsonschema
.. _`Core Definitions and Terminology`: http://tools.ietf.org/html/draft-zyp-json-schema-04
.. _`specification documentation`: http://json-schema.org/specification.html#specification-documents


History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Train
     - Introduced
