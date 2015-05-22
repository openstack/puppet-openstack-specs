..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Pacemaker provider for Openstack services
==========================================

https://blueprints.launchpad.net/puppet-nova/+spec/pacemaker-provider-for-openstack

It is a common practice to make services managed by Pacemaker in Linux HA clusters.
Puppet modules for Openstack should be able to ensure both standard and Pacemaker
providers for Openstack services as well.

In order to achieve this as a transparent solution (w/o modifications to the core
Puppet modules for Openstack) special wrapping classes should be created in
puppet-openstack_extras. These wrappers should contain overrides for Openstack
services as a Pacemaker service provider and ensure creation of corresponding
Pacemaker resources as well.

Problem description
===================

A detailed description of the problem:

Puppet modules for Openstack configure services in a classic way, such as
system service providers.
For HA clusters, though, operators might have wanted to have an option
to delegate Openstack services management for Pacemaker instead.
Pacemaker OCF scripts provide very good options for cluster-wide resources
management and Puppet should deliver this options as well.

Proposed change
===============

A Pacemaker service provider and special wrapper class or classes, like
``openstack_extras::pacemaker::heat`` or ``openstack_extras::pacemaker::nova``
should be created as well as Resource Agents (OCF) scripts for related cluster
resources management. Once a wrapper class is included in the Puppet catalog,
it would override the default service provider of the corresponding Openstack
service configured by its base core module. That will ensure a transparent way
of HA configuration for Openstack services without any modifications to the
base core modules, such as puppet-nova or puppet-heat and so on.

For the first stage of implementation, there should be only a service provider
for Pacemaker created and put into the openstack_extras. That will assume the
deployers should use their own custom classes and Resource Agents (OCF) scripts
for the cluster resources configuration.

The second stage should be a creation of a basic HA wrapper class (or classes)
shipped with a generic OCF scripts for the cluster resources definition which
could benefit every developer. Of course, there should be a way for deployer
to use his custom OCF scripts as well.

Note, that Puppet modules for Openstack will not configure Openstack services
in HA by default. Also, related infra services (such as MySQL or RabbitMQ)
should not be configured by Puppet Openstack core modules, so we do not expect
it for HA provider as well. But HA provider still could be used to do so,
if user wants to override some basic service provider to HA one. By design,
it should work transparently for whichever service in Puppet catalog.

The module for Pacemaker and Corosync to be used with this HA provider is a
puppetlabs/corosync.

Alternatives
------------

The another option is to use puppet-openstacklibs instead of extras.

There is also an option to switch to another module for Pacemaker and Corosync
instead of puppetlabs/corosync. The former one does not support clones,
location constraints, operates via pcs/crmsh CLI which are not good then cluster
is being deoplyed in parallel. So, it could make sence to redesign it
completely, or switch from it to another module, or provide an option to specify
which module deployer wants to use.

A completely another approach could be considered as well.
Pacemaker provider configuration could be ensured as a simple pipe, i.e. we
are passing a some external class name (and its params as a hash) into the
corresponding Openstack classes. This approach resembles the one for
``rabbitmq_class`` which uses ``rabbitmq::server`` as a default value but
could be anything else. But for this case it should be some Pacemaker or
Corosync related class instead.

Each Openstack class providing the service should be as well extended then
with the following parameters:

* ``pacemaker_provider`` (or ``ha-mode``) type parameter. The default value
  should be ``false`` for backward compatibility reasons.

* ``corosync_class`` or ``pacemaker_class`` reference to a some external
  class which should be used for Corosync (Pacemaker) resources definition.
  The default value should be ``false`` for backward compatibility reasons.

* One for ``resource_hash`` parameters being passed into the aforementioned
  class above.

Data model impact
-----------------

None

Module API impact
-----------------

None

End user impact
---------------------

The user will be able to configure Openstack services in a much
more flexible and Highly available way.

Performance Impact
------------------

There would be no performance impact, if default service provider for
Openstack services is used. The performance of Pacemaker provider is
deeply tight with the Pacemaker or Corosync Puppet module chosen by
the user. Anyway, the cluster resources definition for services should
be done only once, hence the performance impact will be the minimal in
overall deployment process and near to none in later operations.

Deployer impact
---------------------

The deployment behavior will not change at all for the case with wrapper
classes in puppet-openstack_extras (openstacklibs).
The only action required would be to include a wrapper classes for Openstack
services in catalog in order to ensure Pacemaker provider for them.
For the approach with an external class reference, the deployer would
have to provide a valid Pacemaker or Corosync class reference for some
external module in catalog and a valid hash of parameters for it.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  bogdando <bdobrelia@mirantis.com>

Other contributors:
  idv1985 <dilyin@mirantis.com>

Work Items
----------

* Create a service provider for Pacemaker (HA provider) as a part of the
  openstack_extras.

* Create a basic HA wrapper classes shipped with a generic OCF scripts for
  the cluster resources definition and an option to specify a custom OCF
  scripts as well.

* Describe in documentation how HA provider could be used with Puppet modules
  for Openstack to configure Openstack services in HA.

* (optional) Describe in documentation how HA provider could be used to
  configure related services, such as RabbitMQ and MySQL in HA.

Dependencies
============

None

Testing
=======

The feature should be tested with the rspecs provided.


Documentation Impact
====================

The feature should be described in the docs for puppet-openstack_extras
(or openstacklibs) module or core Puppet modules for Openstack for the case
with external Corosync (Pacemaker) classes.
New wrapper classes should be described and usage examples provided.

References
==========

None
