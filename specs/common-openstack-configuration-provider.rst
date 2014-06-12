..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Common OpenStack Configuration Provider
=======================================

https://blueprints.launchpad.net/puppet-openstacklib/+spec/common-openstack-configuration-provider

This spec is about introducing a common configuration provider based on
the ini_setting provider from puppetlabs-inifile. Other configuration
providers would now inherit from this provider instead.


Problem description
===================

All puppet modules for OpenStack uses the ini_setting provider as their
grounds for their specialized version.

Recent work has been started to implement a secret parameter where changes
for a specific configuration value would be hidden from Puppet logs.
This work is and will be duplicated across all the specialized providers,
violating the DRY concept.


Proposed change
===============

Introduce a common configuration provider which could be used by other
specialized configuration providers. This common provider would provide
a basic set of features. Example of such features are the secret parameter
and the capitalization of boolean values.


Alternatives
------------

Without this proposition, we should have to continue using the ini_setting
provider as the base of our specialized configuration providers and
implement/copy features in each of them.

We could also try proposing our features and changes to upstream so
we don't have to maintain them anymore. Those changes might however not
fit the vision of upstream, like the capitalization of boolean values.


Module API impact
-----------------

This proposition does not include any change to any module API.


Deployer impact
---------------

This proposition introduces a new mandatory dependency on openstacklib.

Those deploying from the master branch and/or using Puppetfile would need to
install the openstacklib puppet module.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  mgagne

Other contributors:
  None


Work Items
----------

* Create openstack_ini_setting provider
* Refactor the existing configuration providers to use openstack_ini_setting.


Dependencies
============

None


Testing
=======

* Unit test fixtures of all puppet modules would need to be updated
  to install openstacklib.


References
==========

None
