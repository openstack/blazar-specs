..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
Network Reservation
===================

https://blueprints.launchpad.net/blazar/+spec/basic-network-plugin

This plugin supports reserving network segments.

Problem description
===================

In OpenStack deployments with switches that support VLAN tagging, the Neutron
service can create isolated network segments by associating VLAN tags with
Neutron networks on a first come first serve basis. Possible VLAN tags are
limited to a 12-bit field to comply with 802.1Q and, in many cases, only a
small subset of the 4,096 possible VLAN tags can be made available to users.
Likewise, if the cloud provider makes external Layer-2 connections (stitching)
available, users can only employ these connections using a finite number of
available network segments.

Use Cases
---------

* A user requires more network isolation for either research or additional
  security.
* A user wants to make use of an external layer 2 connection and requires a
  VLAN tag dedicated to a provider endpoint.
* A user plans to create an isolated VLAN and wants to make sure a VLAN tag
  is available at the reservation start time.
* A cloud admin does not have enough VLAN tags to give to all users.

  * The admin wants to give a user an isolated VLAN with either host or
    instance reservation.

Proposed change
===============

Blazar enables users to reserve network VLAN tags (and, by extension, other
segmentation types) by specifying the physical network type the users want to
reserve. Users can treat the reserved network as usual, except for the create
network operation.

A basic idea for the network reservation and its scenario are as follows:

1. The admin registers some VLAN tags used for network reservation with Blazar.
   The admin calls Blazar's network API with a request body which includes the
   physical network name, the network type, and the segment ID.
2. A user calls the create lease API specifying a network reservation and the
   start and end dates of the lease. Optional parameters would include segment
   ID, network name, and network type. Blazar checks the availability of a
   network segment for the request. If available, Blazar creates an allocation
   between network segment and the reservation, then returns the reservation
   ID. If not, Blazar doesn't return a reservation ID.
3. At the start time, Blazar creates the reserved network in the user's tenant
   (project). The user can then create, configure, or associate subnet and
   router to network as usual.
4. At the end time, Blazar deletes the network and any other associated network
   components such as subnets, router, ports, etc., if the user hasn't deleted
   or disassociated them already.


Alternatives
------------

None

Data model impact
-----------------

The plugin introduces four new tables, "network_reservations",
"networksegment_extra_capabilities", "network_allocations", "network_segments".

The "network_reservations" table keeps user request information for their
network reservation. The role of this table is similar to the role of the
computehost_reservations table in the host reservation plugin. This table has
id, resource_properties, network_properties, network_name, network_description
and network_id columns.

The "networksegment_extra_capabilities" can contain additional informations
about particular network segments that a cloud provider wants to provide.

The "network_allocations" table has the relationship between the
network_reservations table and the network_segments table.

The "network_segments" table stores information about network segments
themselves. The reservable network segments are registered in the table.

The table definitions are as follows:

.. sourcecode:: none

    CREATE TABLE `network_reservations` (
        `created_at` datetime DEFAULT NULL,
        `updated_at` datetime DEFAULT NULL,
        `deleted_at` datetime DEFAULT NULL,
        `deleted` varchar(36) DEFAULT NULL,
        `id` varchar(36) NOT NULL,
        `reservation_id` varchar(36) DEFAULT NULL,
        `resource_properties` mediumtext,
        `network_properties` mediumtext,
        `before_end` varchar(36) DEFAULT NULL,
        `network_name` varchar(255) DEFAULT NULL,
        `network_description` varchar(255) DEFAULT NULL,
        `network_id` varchar(255) DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `reservation_id` (`reservation_id`),
        CONSTRAINT `network_reservations_ibfk_1`
        FOREIGN KEY (`reservation_id`) REFERENCES `reservations` (`id`)
    );

    CREATE TABLE `network_segments` (
        `created_at` datetime DEFAULT NULL,
        `updated_at` datetime DEFAULT NULL,
        `id` varchar(36) NOT NULL,
        `network_type` varchar(255) NOT NULL,
        `physical_network` varchar(255) DEFAULT NULL,
        `segmentation_id` int(11),
        `reservable` boolean NOT NULL,
        PRIMARY KEY (`id`),
        UNIQUE KEY `network_type`
        (`network_type`,`physical_network`,`segmentation_id`)
    );

    CREATE TABLE `networksegment_extra_capabilities` (
        `created_at` datetime DEFAULT NULL,
        `updated_at` datetime DEFAULT NULL,
        `id` varchar(36) NOT NULL,
        `network_id` varchar(36) NOT NULL,
        `capability_name` varchar(64) NOT NULL,
        `capability_value` mediumtext NOT NULL,
        PRIMARY KEY (`id`),
        KEY `network_id` (`network_id`),
        CONSTRAINT `networksegment_extra_capabilities_ibfk_1`
        FOREIGN KEY (`network_id`) REFERENCES `network_segments` (`id`)
    );


    CREATE TABLE `network_allocations` (
        `created_at` datetime DEFAULT NULL,
        `updated_at` datetime DEFAULT NULL,
        `deleted_at` datetime DEFAULT NULL,
        `deleted` varchar(36) DEFAULT NULL,
        `id` varchar(36) NOT NULL,
        `network_id` varchar(36) DEFAULT NULL,
        `reservation_id` varchar(36) DEFAULT NULL,
        PRIMARY KEY (`id`),
        KEY `network_id` (`network_id`),
        KEY `reservation_id` (`reservation_id`),
        CONSTRAINT `network_allocations_ibfk_1`
        FOREIGN KEY (`network_id`) REFERENCES `network_segments` (`id`),
        CONSTRAINT `network_allocations_ibfk_2`
        FOREIGN KEY (`reservation_id`) REFERENCES `reservations` (`id`)
    );


REST API impact
---------------

The network segment reservation introduces a new resource_type to the lease
APIs and five new admin APIs to manage the segments.

Changes in the lease APIs
`````````````````````````

* URL: POST /v1/leases

  * Introduced new resource_type, network, for a reservation.

Request Example:

.. sourcecode:: json

   {
     "name": "network-reservation-1",
     "reservations": [
       {
         "resource_type": "network",
         "network_name": "my-network-1",
         "physical_network": "physnet-name", # optional
         "segmentation_id": 1234,            # optional
         "network_type": "vlan"              # optional
       }
     ],
     "start_date": "2019-05-17 09:07",
     "end_date": "2019-05-17 09:10",
     "events": []
   }

Response Example

.. sourcecode:: json

    {
      "lease": {
        "name": "network-reservation-1"
        "reservations": [
          {
            "id": "reservation-id",
            "status": "pending",
            "lease_id": "lease-id-1",
            "resource_id": "resource_id",
            "resource_type": "network",
            "created_at": "2019-05-17 10:00:00",
            "updated_at": "2017-05-01 11:00:00",
          }],
        "start_date": "2019-05-17 09:07",
        "end_date": "2019-05-17 09:10",
        ..snip..
      }
    }

* URL: GET /v1/leases
* URL: GET /v1/leases/{lease-id}
* URL: PUT /v1/leases/{lease-id}
* URL: DELETE /v1/leases/{lease-id}

* The change is the same as POST /v1/leases

New network APIs
````````````````

The five new APIs are admin APIs by default.

* URL: POST /v1/networks

  * The segmentation_id is a specific VLAN tag the admin wants to add. The tag
    must be out of allocations pools in Neutron. As of now, only networks
    making use of VLAN tagging require a segment id per IEEE 802.1Q. A
    reservable flat network would leave this field empty.
  * The network_type is the type of physical mechanism associated with the
    network segment. Examples include flat, geneve, gre, local, vlan, vxlan.
  * They physical_network is the name of the physical network in which the
    network segment is available. This is required for VLANs.

Request Example:

.. sourcecode:: json

   {
     "network_type": "vlan",
     "physical_network": "physical-network-1",
     "segmentation_id": 1234
   }

* The reservable key is a flag describing if the network segment is reservable
  or not. The flag is always True until the network plugin supports the
  resource healing feature. (Supporting resource healing to network segments
  is out of scope in this spec)

Response Example:

.. sourcecode:: json

  {
    "network": {
        "id": "network-id",
        "network_type": "vlan",
        "physical_network": "physical-network-1",
        "segmentation_id": 1234,
        "reservable": true,
        "created_at": "2020-01-01 10:00:00",
        "updated_at": null
    }
  }

* URL: GET /v1/networks

Response Example:

.. sourcecode:: json

  {
    "networks": [
        {
            "id": "network-id",
            "network_type": "vlan",
            "physical_network": "physical-network-1",
            "segmentation_id": 1234,
            "reservable": true,
            "created_at": "2020-01-01 10:00:00",
            "updated_at": null
        }
    ]
  }

* URL: GET /v1/networks/{network-id}

Response Example:

.. sourcecode:: json

  {
    "network": {
        "id": "network-id",
        "network_type": "vlan",
        "physical_network": "physical-network-1",
        "segmentation_id": "1234",
        "reservable": true,
        "created_at": "2020-01-01 10:00:00",
        "updated_at": null
    }
  }

* URL: DELETE /v1/networks/{network-id}

No Request body or Response body.

* URL: PUT /v1/networks/{network-id}

  Request Example:

.. sourcecode:: json

  {
    "extra_capability_sample": "bar"
  }

  Response Example:

  {
    "network": {
        "id": "network-id",
        "network_type": "vlan",
        "physical_network": "physical-network-1",
        "segmentation_id": "1234",
        "reservable": true,
        "created_at": "2020-01-01 10:00:00",
        "updated_at": null,
        "extra_capability_sample": "bar"
    }
  }

.. sourcecode:: json

Security impact
---------------

None

Notifications impact
--------------------

None

Other end user impact
---------------------

A user can reserve a network segment as well as host, instance or floating IP
in the same lease.

python-blazarclient will support the network segment reservation.

Performance Impact
------------------

None

Other deployer impact
---------------------

None

Developer impact
----------------

None

Upgrade impact
--------------

Some configuration options for the Neutron util class will be introduced to
blazar.conf. If the cloud admin want to activate the network reservation, they
will need to set up the configuration.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  priteau

Other contributors:
  diurnalist
  jakecoll

Work Items
----------

* Create the new DB tables
* Create the network reservation plugin
* Create the network API object and its route in blazar.api.v1
* Add network reservation support in python-blazarclient
* Add scenario tests and API tests in blazar-tempest-plugin
* Update Blazar docs, API reference and user guide

Dependencies
============

None

Testing
=======

API tests and scenario tests need to be implemented.

Documentation Impact
====================

This BP adds new APIs and resource type to the lease APIs. The API reference
and the Blazar documentation need to be updated.

References
==========

1. https://etherpad.openstack.org/p/network-resource-reservation
2. https://etherpad.openstack.org/p/blazar-ptg-stein

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Train
     - Introduced
   * - Ussuri
     - Re-proposed
