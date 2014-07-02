..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================
Common OpenStack Database Resource
==================================

https://blueprints.launchpad.net/puppet-openstacklib/+spec/commmon-openstack-database-resource

This spec proposes to introduce defined resource types in openstacklib to
provide a common interface for setting up databases used by the major OpenStack
services. The current modules for these services would use this interface
rather than duplicating code across these modules.

Problem description
===================

Nearly all of the Puppet modules for OpenStack services set up a database for
themselves and most of them do this using nearly identical code. This causes a
great deal of repetition across modules and will lead to inconsistencies.

Proposed change
===============

Abstract the database functionality into defined resource types in the
openstacklib module. This resource would implement the same functionality as
the db::mysql class and db::mysql::host_access type already do in ceilometer,
cinder, glance, heat, keystone, neutron, and nova and will make the ability to
configure postgresql databases consistent. The new resource would use the
newest mysql and postgresql modules.

Alternatives
------------

The alternative to implementing this feature in openstacklib is to maintain the
database setup functionality in each module separately. This would necessitate
updating the postgresql classes in each module to work with the current version
of the postgresql module as well as adding postgresql support to the heat module.
If additional backends are desired they would have to be implemented in each
module individually.

Data model impact
-----------------

The ceilometer, cinder, glance, heat, keystone, neutron, and nova modules
would be updated to depend on the openstacklib module and use the new database
resources to configure their respective databases.

The mysql and postgresql subclasses of ceilometer, cinder, glance, heat,
keystone, neutron, and nova modules would have their current code replaced
with a call to the new defined resource type in openstacklib. No new classes
would be needed for these modules, except for heat which would need to have a
postgresql subclass added.

Module API impact
-----------------

* New defined resource types:

  openstacklib::db::mysql
  openstacklib::db::postgresql

  A later spec may introduce an openstacklib::db::mongodb resource.

* Parameters for openstacklib::db::mysql:

  dbname         : The name of the database;
                   string; optional; default to the $title of the resource, i.e. 'nova'
  user           : The database user to create;
                   string; optional; default to the $title of the resource, i.e. 'nova'
  password_hash  : Password hash to use for the database user for this service;
                   string; required
  host           : The IP address or hostname of the user in mysql_grant;
                   string; optional; default to '127.0.0.1'
  charset        : The charset to use for the database;
                   string; optional; default to 'utf8'
  collate        : The collate to use for the database;
                   string; optional; default to 'utf8_unicode_ci'
  allowed_hosts  : Additional hosts that are allowed to access this database;
                   array or string; optional; default to undef
  privileges     : Privileges given to the database user;
                   string or array of strings; optional; default to 'ALL'

* Example use case for openstacklib::db::mysql:

  In ::nova::db::mysql:

    ::openstacklib::db::mysql { 'nova':
      password_hash => '2470C0C06DEE42FD1618BB99005ADCA2EC9D1E19',
      notify        => Exec['nova-db-sync'],
    }

  would create a mysql database called 'nova' with user 'nova@127.0.0.1'. It
  would not allow other hosts access to the database. It will re-execute the
  'nova-db-sync' exec when the database refreshes.

  Another example in ::keystone::db::mysql:

    ::openstacklib::db::mysql { 'keystone':
      password_hash  => '2470C0C06DEE42FD1618BB99005ADCA2EC9D1E19',
      host           => '1.2.3.4',
      allowed_hosts  => ['1.2.3.4', '5.6.7.8'],
      notify         => Exec['keystone-manage db_sync'],
      before         => Service['keystone'],
    }

  would create a mysql database called 'keystone' with user 'keystone@1.2.3.4'
  and grants to 'keystone' on hosts '1.2.3.4' and '5.6.7.8'. The database will
  be set up before the keystone service is started and the 'keystone-manage
  db_sync' exec will be re-executed when it refreshes.

* Parameters for openstacklib::db::postgresql:

  dbname         : The name of the database;
                   string; optional; default to the $title of the resource, i.e. 'nova'
  user           : The database user to create;
                   string; optional; default to the $title of the resource, i.e. 'nova'
  password_hash  : Password hash to use for the database user for this service;
                   string; required
  encoding       : The charset or encoding to use for the database;
                   string; optional; default to undef

* Example use case for openstacklib::db::postgresql

  In ::nova::db::postgresql:

    ::openstacklib::db::postgresql { 'nova':
      password_hash => '2470C0C06DEE42FD1618BB99005ADCA2EC9D1E19',
      notify        => Exec['nova-db-sync'],
    }

  would create a postgresql database called 'nova' with user 'nova'. It will
  re-execute the 'nova-db-sync' exec when the database refreshes.

  Another example in ::keystone::db::postgresql:

    ::openstacklib::db::postgresql { 'keystone':
      password_hash => '2470C0C06DEE42FD1618BB99005ADCA2EC9D1E19',
      notify        => Exec['nova-db-sync'],
      before        => Service['keystone'],
    }

  would create a postgresql database called 'keystone' with user 'keystone'.
  The database will be set up before the keystone service is started and the
  'keystone-manage db_sync' exec will be re-executed when it refreshes.


End user impact
---------------------

None aside from the API.

Performance Impact
------------------

None

Deployer impact
---------------------

The user needs to install the openstacklib module prior to using the
ceilometer, cinder, glance, heat, keystone, neutron, or nova modules.

Developer impact
----------------

Changes to database setup will happen in the openstacklib module rather than in
the individual OpenStack service modules.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  krinkle

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

None

References
==========

None
