..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

============================
Support domain configuration
============================
:tags: keystone,domain

`Launchpad blueprint <https://blueprints.launchpad.net/puppet-keystone/+spec/keystone-domain-configuration>`_

Offer a way to configure multiple domains for Openstack with keystone
API v3.

Problem description
===================

As a deployer, I would like to install OpenStack keystone running API
v3 and be able to configure multiple domains.

This requires multiple keystone configuration files. Openstack expects
such configuration in a file named ``keystone.$domain.conf`` in a
directory defined by ``identity/domain_config_dir`` in
``keystone.conf``.  It's ``/etc/keystone/domains`` by default.

Proposed change
===============

Make a new provider ``keystone_domain_config``. The syntax would be to
allow the use of the domain in the resource name::

    keystone_domain_config {
      "services::ldap/url":  value => $url;
    }

This will set the ``[ldap]`` section ``url`` to the value of ``$url``
in the configuration file
``/etc/keystone/domains/keystone.services.conf``

I'm proposing ``::`` as the delimiter since that's what the proposed
keystone v3 patch uses. Note that in this case, the domain comes
first, before the ``::``

This implementation will be a subclass keystone_config.

Alternatives
------------

It should be noted that the rest API offer a way to do it directly
without the configuration file, but it's currently unavailable to the
openstack cli see this `openstackclient bug
<https://bugs.launchpad.net/python-openstackclient/+bug/1433307>`_.
When this becomes available, the file creation can be removed in favor
of the cli.

Another way to do it would be to add the missing name parsing in
"keystone_config".  The lesser encapsulation means that when the
openstack cli finally supports the direct modification of the
configuration, we won't be able to adjust the provider easily.

Data model impact
-----------------

None

Module API impact
-----------------

Everything has been already mentioned in Proposed change.

End user impact
---------------------

None

Performance Impact
------------------

None

Deployer impact
---------------------

If a deployer use a parameter with ``::`` in it, then the left side of
the string will be interpreted as a domain and put in the domain file,
not in the keystone.conf file.  There is no such parameter for the
moment in the whole configuration and it's unlikely that they ever
will be.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  sofer-athlan-guyot

Other contributors:
  rmeggins

Work Items
----------

* Create a unit rspec;
* Create a functional test;
* Add the name parsing and file path logic in the provider;

Dependencies
============

* The keystone version must be at least stable/kilo.

Testing
=======

For the moment the functional test are covered by beaker.  Future
change in the puppet gate can make tempest tests useful for this
feature.

Documentation Impact
====================

Add a examples in the puppet-keystone repository for this feature.

References
==========

1. `Official documentation; <http://docs.openstack.org/kilo/config-reference/content/section_keystone-domain-configs.html>`_

2. `Discussion on trello; <https://trello.com/c/xDDgtctf/22-extend-keystone-config-to-support-multiple-domains-with-a-config-file-per-domain>`_

3. `Domain configuration management API <http://specs.openstack.org/openstack/keystone-specs/api/v3/identity-api-v3.html#domain-configuration-management>`_
