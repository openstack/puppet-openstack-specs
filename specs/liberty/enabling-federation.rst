..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================
Enabling Federation
===================
:tags: federation

`bp enabling-federation <https://blueprints.launchpad.net/puppet-keystone/+spec
/enabling-federation>`_

The purpose of this spec is to provide an efficient way for cloud
administrators to configure their Keystone as a Service Provider or Identity
Provider in order to use Identity Federation.

Problem description
===================

As a cloud administrator, I want a fast and convenient way to configure my
Keystone (Kilo version or newer) as a Service Provider or as an Identity
Provider. Nowadays, the manual configuration of such features can be
cumbersome and error prone.


Proposed change
===============

Introduce two new classes, one to configure Keystone as a Service Provider
and the other to configure Keystone as an Identity Provider.

* Service Provider

 There will be a class service_provider to configure Keystone as a Service
 Provider. There are two major protocols (OpenID Connect and SAML) and three
 different ways to configure: OpenID Connect, Shibboleth and Mellon.
 Packages and extra configuration will be applied according to the choice
 made, the three possible choices are: OpenID Connect, Mellon and Shibboleth.

 As attributes for this class we can have:

  - entityID:
    Is the globally unique name for a SAML entity, must be a URI and should be
    an absolute URL.
  - idps_urls:
    A list containing all the Identity Providers urls. All urls are in format
    <address>:<port>. This will be used to set the configurations in Keystone
    and Apache. This list should contain one or more urls to be valid.
  - metadata_urls:
    A list containing one metadata_url for each Identity Provider in idps_urls.
  - protocol:
    The protocol used to provide Federation. There is support for two
    protocols: OpenID Connect and SAML
  - module:
    The name of the Apache module to install. This will be used to guide the
    installation and configuration of a Service Provider. The value of this
    attribute should be the name of one of the modules available.
    Available modules name are: shibboleth and mellon.

* Identity Provider

 There will be a class identity_provider to configure Keystone as an Identity
 Provider, by installing the necessary packages and adding the necessary
 configuration to the keystone.conf file. Currently Keystone can only provide
 SAML assertions.


 As attributes for this class we can have:

 * Required

  - certfile:
    Path of the certfile for SAML signing. The attribute ssl_ca_certs from
    Keystone can be used.
  - keyfile:
    Path of the keyfile for SAML signing. The attribute ssl_ca_key from
    Keystone can be used.
  - idp_entity_id:
    Entity ID value for unique Identity Provider identification.
  - idp_sso_endpoint:
    Identity Provider Single-Sign-On service value, required in the Identity
    Provider's metadata.
  - idp_metadata_path:
    Path to the Identity Provider Metadata file.

 * Optional - this attributes will be undef.

  - idp_organization_name:
    Organization name the installation belongs to.
  - idp_organization_display_name:
    Organization name to be displayed.
  - idp_organization_url:
    URL of the organization.
  - idp_contact_company:
    Company of contact person.
  - idp_contact_name:
    Given name of contact person
  - idp_contact_surname:
    Surname of contact person.
  - idp_contact_email:
    Email address of contact person.
  - idp_contact_telephone:
    Telephone number of contact person.
  - idp_contact_type:
    Contact type.

 To know more about this attributes see `Table 7.28 <http://docs.openstack.org
 /kilo/config-reference/content/keystone-configuration-file.html>`_.


.. note::
  For the Federated Identity feature, any web server is allowed since it
  supports SAML or OpenID Connector. In this spec we are only considering the
  use of Apache, since it supports both and have more documentation. To know
  more about some values for the attributes see reference number two.

.. note::
  For the Service Provider, there are some packages that Keystone needs.
  For example, in a Debian based distribution: the packages
  ``libapache2-mod-shib2`` or ``libapache-mod-auth-mellon`` for SAML2 support
  and ``libapache2-auth-openidc`` for OpenID Connect support.

.. note::
  For the Identity Provider, there are some packages that Keystone needs.
  For example, in a Debian/Ubuntu distribution: the packages ``pysaml2`` and
  ``xmlsec1`` are necessary.

.. note::
   The apache packages required for the Service Provider will be installed
   using the defined type ``apache::mod`` from puppetlabs-apache, according
   to the selected module.

.. note::
   Only the configurations that belongs to Keystone's configuration files will
   be applied by puppet.


Alternatives
------------
Keep these setups as they are right now: manually install packages and add
necessary changes to configuration files.


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

To use this the modules from this feature the cloud operator/admin will need
to apply during or after Keystone is installed and running over apache.

``site.pp`` example - Keystone as a Service Provider using Mellon::

 class { 'keystone::federation::service_provider':
  idps_urls        => 'https://ipa.rdodom.test/idp',
  saml_dir         => '/etc/httpd/saml2/test',
  http_conf        => '/etc/httpd/conf.d/keystone-mellon.conf',
  service          => 'keystone',
  protocol         => 'SAML',
  module           => 'mellon',
  saml_base        => '/v3',
  saml_auth        => 'OS-FEDERATION/identity_providers/ipsilon/protocols/saml2/auth',
  saml_sp          => 'mellon',
  saml_sp_logout   => 'logout',
  saml_sp_postresp => 'postResponse',
  enable_ssl       => false,
  sp_port          => 5000,
 }

``site.pp`` example - Keystone as an Identity Provider::

 class { 'keystone::federation::identity_provider':
  idp_entity_id     => 'https://keystone.example.com/v3/OS-FEDERATION/saml2/idp',
  idp_sso_endpoint  => 'https://keystone.example.com/v3/OS-FEDERATION/saml2/sso',
  idp_metadata_path => '/etc/keystone/saml2_idp_metadata.xml',
 }



Examples of the configurations added to Keystone and Apache can be found below:

**For Identity Provider:**

See topic `Keystone as an Identity Provider (IdP) <http://docs.openstack.org/de
veloper/keystone/configure_federation.html>`_.

**For Service Provider**

See topic `Keystone as a Service Provider (SP) <http://docs.openstack.org/devel
oper/keystone/configure_federation.html>`_.

* For Shibboleth configuration see `Setup Shibboleth
  <http://docs.openstack.org/developer/keystone/federation/shibboleth.html>`_.

* For OpenID configuration see `Setup OpenID Connect
  <http://docs.openstack.org/developer/keystone/federation/openidc.html>`_.

* For mod_auth_mellon, see `Setup Mellon
  <http://docs.openstack.org/developer/keystone/federation/mellon.html>`_.



Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  iurygregory


Work Items
----------

* Create a class for the Identity Provider configuration, which will apply
  the extra configurations to the Keystone configuration file and install all
  necessary packages to make Keystone work as Identity Provider.
* Write tests to ensure that the Identity Provider configuration is valid.
* Create a class for a Service Provider and acording to the protocol apply the
  necessary configuration.
* Write tests to ensure that the Service Provider configuration is valid for
  both modules (Shibboleth, Mellon and OpenID)


Dependencies
============

* The Keystone version should be at least stable/kilo.
* This feature will be only supported if Keystone is running over an Apache.

Testing
=======

* Create spec tests files for each configurable parameter used by the new
  Identity/Service Provider classes, to ensure that all applied settings
  are valid.
* Create functional tests with acceptance.

Documentation Impact
====================

Add examples in the puppet-keystone repository for both classes.


References
==========

1. Summit video: https://www.youtube.com/watch?v=PxNM8tBdCs4
2. http://docs.openstack.org/kilo/config-reference/content/config_overview.html
   Chapter 7. Identity service - Identity service configuration
   - for identity provider take a look at [saml]
   - for service provider take a look at [auth]
3. http://rodrigods.com/
4. http://irclog.perlgeek.de/puppet-openstack/2015-05-19
   start: 19:26
5. http://docs.openstack.org/developer/keystone/configure_federation.html

