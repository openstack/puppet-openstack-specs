..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================================
Restructure authentication for OpenStack Puppet resource providers
==================================================================

Launchpad blueprint:

https://blueprints.launchpad.net/puppet-openstacklib/+spec/auth-restructure

Since OpenStack Puppet modules providers are managing their resources using
python-openstackclient (OSC), the 'openstack' CLI, to interface to OpenStack,
the mechanism which is used for holding authentication data is causing
unnecessary complexity and an inconsistent behavior for providers depending on
their context usage.


Problem description
====================

The authentication data is currently stored as a type parameter ':auth'.
Having the authentication information as type parameter makes the credentials
available in an instance context but not in class context such as when a
resource is used in general context, for example in self.instances method.

This has several consequences:

1. It creates code duplication down the the road to work around the fact there
isn't authentication data provided in a general context, for example in keystone
provider:

* lib/puppet/provider/keystone_tenant/openstack.rb#L60

* lib/puppet/provider/keystone_tenant/openstack.rb#L73

When dealing with authentication, each provider has to duplicate every method
in regards of instance or class context.

2. Consequently, it appears that even in a general context, authentication
information has to be obtained from somewhere. Which means the authentication
mechanism needs to be revisited to have a consistent way of setting the
credentials in a consistent way no matter the context.

Also, independently of this problem but related to the authentication data, a
security issue came up when using command line options to pass credentials to
openstack command, for instance '--os_password=blah'. The risk is for a spawned
process to have the information displayed, i.e. with a 'PS' command.

And finally, a change in the way the credentials are obtained changes with step
2. which formalizes the default RC file to be used if needed:

1. Environment variables OS_* are used by default and will be overridden only if
   defined below

2. If not enough credentials then OS_* variables defined in RC file to be used.
   The default is located current user (per execution of Puppet) home directory:
   $HOME/openrc

3. If authentication fails it's up to the provider to implement a failsafe to
   use configuration file to find the credentials.

Proposed change
===============

Therefore the need to review the architecture of handling the authentication
data while offering the same features to the OpenStack providers.

The fist change removes the :auth parameter from the types.
This is replaced with a combination of a auth module, and a credentials object.

Replacing the polymorphism approach used by providers inheriting from
Puppet::Provider::Openstack class by a module.
The authentication related methods are moved to the module.

The module creates an interface to the provider classes.
When a provider 'extend' the module, all the methods defined in the module
are available to the 'inheriting' class as class methods.

The authentication methods are acting at a sublevel, leaving the superclass
Puppet::Provider::Openstack to deal purely interfacing with openstack client.

The openstacklib structure is used as follow, using keystone provider cases:

class Puppet::Provider::Openstack
end

class Puppet::Provider::Keystone < Puppet::Provider::Openstack
end

class Puppet::Provider::Keystone_tenant < Puppet::Provider::Keystone
end

The module is extended by the provider intermediate class, for instance
class Puppet::Provider::Keystone < Puppet::Provider::Openstack
extend Puppet::Provider::Openstack::Auth
end

Secondly a credentials class Puppet::Provider::Openstack::Credentials
makes manipulating the authentication data not only easier but allows a class
instance variable @credentials to hold the information into a dedicated object.

To address the security issue point:
Credentials passed on a CLI must be forbidden in order to not be visible via PS
commands.

This is enforced by using of `withenv` method from Puppet Util library could help:
https://github.com/puppetlabs/puppet/blob/3.0.0/lib/puppet/util.rb#L39-L51

Note:
  Although using ENV instead of CLI parameters doesn't remove issues such as:
  "puppet resource keystone_user foo email=foo@example.com password=test"
  In order to avoid the above, the related provider type API would have to be
  changed as well.

Alternatives
------------

This is an alternative way to the existing solution, other was have not been
envisaged so far.

Data model impact
-----------------

The goal of this effort is: no data model impact.  Existing manifests should
continue to work as before.


Module API impact
-----------------

None other than what has been already mentioned.

End user impact
---------------------

There should be no mandatory end user impact.  Existing manifests should
continue to work exactly as before.

Operations wanting use different authentication should be able to provide it.

Performance Impact
------------------

The `request` method and helper methods used by it should cache the contents of
files instead of opening/reading/parsing/closing every time.


Deployer impact
---------------------

None other than what has been already mentioned.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gilles@redhat.com (IRC nick gildub)

Other contributors:
  rmeggins (IRC nick richm)

Work Items
----------

* Implement the code described in the "Module API impact" section.

Dependencies
============

None

Testing
=======

Write tests for the new beaker CI test framework.


Documentation Impact
====================

The README.md and the examples in the examples directory will be updated.

References
==========

Openstack client: http://docs.openstack.org/developer/python-openstackclient/
Openstack client config file:  http://docs.openstack.org/developer/python-openstackclient/configuration.html#configuration-files
