..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Common OpenStack Identity Resource
==================================

https://blueprints.launchpad.net/puppet-openstacklib/+spec/common-openstack-identity-resource

This spec proposes to introduce defined resource types in openstacklib to
provide a common interface for setting up Keystone resources used by the major OpenStack
services. The current modules for these services would use this interface
rather than duplicating code across these modules.

Problem description
===================

Nearly all of the Puppet modules for OpenStack services set up Keystone users, roles,
tenants and endpoints for themselves and most of them do this using nearly identical
code. This causes a great deal of repetition across modules and will lead to
inconsistencies.

Proposed change
===============

Abstract the identity functionality into defined resource types in the
keystone::auth type already done in most of OpenStack modules.
openstacklib module. This resource would implement the same functionality as we had before
with Keystone resources management.
The new resource would use the newest identity modules.

Alternatives
------------

The alternative to implementing this feature in openstacklib is to maintain the
Keystone resources management functionality in each module separately.

Data model impact
-----------------

The OpenStack modules would be updated to depend to use
the new identity resources to configure their respective Keystone resources.

Some modules would have their current code replaced
with a call to the new defined resource type in openstacklib. No new classes
would be needed for these modules.

Module API impact
-----------------

* New defined resource type:

  openstacklib::identity

* Parameters for openstacklib::service_identity:

  password            : Password to create for the service user;
                        string; required
  auth_name           : The name of the service user;
                        string; optional; default to the $title of the resource, i.e. 'nova'
  service_name        : Name of the service;
                        string; required
  public_url          : Public endpoint URL;
                        string; required
  internal_url        : Internal endpoint URL;
                        string; required
  admin_url           : Admin endpoint URL;
                        string; required
  region              : Endpoint region;
                        string; optional: default to 'RegionOne'
  tenant              : Service tenant;
                        string; optional: default to 'services'
  domain              : User domain (keystone v3);
                        string; optional: default to undef
  email               : Service email;
                        string; optional: default to '$auth_name@localhost'
  configure_endpoint  : Whether to create the endpoint.
                        string; optional: default to True
  configure_user      : Whether to create the user.
                        string; optional: default to True
  configure_user_role : Whether to create the user role.
                        string; optional: default to True

* Example use case for openstacklib::service_identity:

  In ::nova::keystone::auth::

    ::openstacklib::service_identity { 'nova':
      password         => 'secrete',
      auth_name        => 'nova',
      domain           => 'domain1',
      service_name     => 'compute',
      public_url       => 'https://my-nova-api.com:8774/v2',
      admin_url        => 'https://my-nova-api.com:8774/v2',
      internal_url     => 'https://my-nova-api.com:8774/v2',
      notify           => Service['nova-api'],
    }

  would create user, tenant, role and endpoint for OpenStack Compute API service and restart nova-api.

In the case of multiple endpoints version (Keystone & Nova v2+v3), we would have to use two times ::openstacklib::service_identity with v3 optional and disabled by default for backward compatibility. In Nova, we have an EC2 endpoint and we will also use the new function exactly how it is in current puppet-nova code.

In the case of multiple regions, it would be suggested to use Hiera and the new nova::keystone::auth with is consumming a define function in openstacklib, to create multiple resources in multiple regions.

In the case of a dedicated Keystone server, it's obvious the nova::keystone::auth classes are only set for Keystone nodes, otherwise it will lead to catalog compilation issues since keystone.conf is not on other nodes.


End user impact
---------------------

None aside from the API.

Performance Impact
------------------

None

Deployer impact
---------------------

The user needs to install the openstacklib module prior to using the
OpenStack modules.

Developer impact
----------------

Changes to keystone setup will happen in the openstacklib module rather than in
the individual OpenStack service modules.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  EmilienM

Other contributors:
  None

Work Items
----------

* Create new defined resource type in openstacklib.

* Update ceilometer, cinder, glance, heat, keystone, neutron, and nova modules
  to depend on openstacklib and use the new resource.

Dependencies
============

None

Testing
=======

Unit test fixtures of all puppet modules would need to be updated to install
openstacklib. Existing tests in these modules would be replicated in
openstacklib.

Documentation Impact
====================

README will be updated for each module consuming this new feature.

References
==========

None
