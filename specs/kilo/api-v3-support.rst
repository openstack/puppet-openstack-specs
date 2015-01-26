..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===================================================
Support Keystone v3 API in OpenStack puppet modules
===================================================

Launchpad blueprint:

https://blueprints.launchpad.net/puppet-keystone/+spec/api-v3-support

Beginning with Icehouse, Keystone has declared the v3 API (URLs with /v3) fully
supported, and beginning with Kilo, Keystone has deprecated the v2 API (URLs
with /v2.0).  The OpenStack puppet modules should use the v3 API wherever
possible.  Keystone will continue to fully support v2 and v3 at the same time
for the next couple of cycles.

.. _project-is-tenant-note:
.. note:: Project is used instead of tenant

  Keystone has renamed "tenant" to "project".  Everywhere in the v3 API,
  `openstack` command line tool, environment variables, Keystone documentation,
  and elsewhere, project is used instead of tenant. However, in
  puppet-keystone, we still use the keystone_tenant resource.

Problem description
===================

There are a number of features that are only available with the v3 API:

* The ability to for one user, to delegate to another user, the authority to
  perform certain tasks, using the Keystone v3 Trusts extension.  For example:

  * Have a service launch an instance on behalf of a user without that user
    having to first authenticate (e.g. for auto-scaling at 3 AM).
  * Delegate restricted nova API access to (implicitly) untrusted in-instance
    processes.
  * Provide access to accounts and containers on swift without using ACLs.
  * http://techs.enovance.com/5858/role-delegation-in-keystone-trusts

* The ability to have multiple domains, each with its own separate identity
  backend. One of the most important use cases is to put the OpenStack service
  accounts in a "service" domain, backed by an SQL identity backend, and have
  the default domain (that is, the domain that is used if not explicitly
  specified in the request) point to the enterprise LDAP server, both in
  read-write and read-only mode.  In order to do this, the services will need
  to use v3 for authentication and specify the domain for the user and for the
  project.

Proposed change
===============

In Keystone, the *domain* is the container for users, groups, and projects.
Other identity resources, such as roles, services, and endpoints, are not
contained within a domain.  In order to uniquely identify a Keystone resource
by name, it must be qualified with the domain name.  This is a problem with
naming puppet resources, which must have a unique name.  The string "::" can be
used to construct a unique Puppet resource name from a name part and a domain
name part.  However, existing manifests expect to be able to name resources
without the domain part, and it may be difficult for operators to change
manifests to make all Keystone resource names have the domain in them.
Therefore, the puppet-keystone v3 support implementation must support current
manifests as much as possible.  Consider this case::

    # Composition layer
    class { '::glance::keystone::auth':
      region           => $region,
      password         => $glance_user_password,
      public_address   => $public_url,
      admin_address    => $admin_url,
      internal_address => $internal_url,
    }

    # puppet-glance
    class glance::keystone::auth(
      $password,
      $email                   = 'glance@localhost',
      $auth_name               = 'glance',
      $configure_user          = true,
      $configure_user_role     = true,
      ...
      $tenant                  = 'services',
      ...
    ) {
      ...
      keystone::resource::service_identity { $auth_name:
        configure_user      => $configure_user,
        configure_user_role => $configure_user_role,
        ...
        tenant              => $tenant,
        ...
      }
      ...
    }

    # puppet-keystone
    define keystone::resource::service_identity(
      ...
      $auth_name             = $name,
      ...
      $tenant                = 'services',
      ...
    ) {
      ...
      keystone_user { $auth_name:
        ensure                => 'present',
        enabled               => true,
        password              => $password,
        email                 => $email,
        tenant                => $tenant,
        ignore_default_tenant => $ignore_default_tenant,
        domain                => $user_domain_real,
      }
    }

    if $configure_user_role {
      keystone_user_role { "${auth_name}@${tenant}":
        ensure => 'present',
        roles  => $roles,
      }
    }

The puppet-keystone code must be able to use just `glance` as the name without
it having to be qualified by a domain.  The puppet-keystone code must be smart
enough to figure out that there is only one user named `glance` among all of
the domains, figure out which domain it is in, and use it.  Same with the
`service` project, and with the `glance@service` user role.  As long as there
is only one resource with the given name, the code should work.  However, if
there is a `glance` user in `domain1`, and a `glance` user in `domain2`, it is
the responsibility of the manifest writer to fully qualify both user names.
There is no way for puppet-keystone, when given a username of `glance`, to know
if it is referring to `glance::domain1` or `glance::domain2`.  We will need to
warn puppet users that the use of "::" in Keystone resource names (such as a
user named "foo::bar") is not supported.

.. _back-compat-note:
.. note:: Resource names with "::domainname" are not required if the resource names are unique

  One of the goals of this effort is to preserve backwards compatibility with
  existing manifests and other related Puppet code.  You do not have to declare
  a resource name with "::domainname" if the base resource name is unique among
  all domains.  For example, if you have only one user named `glance` among all
  domains, you can refer to this using only 'glance' in keystone_user and
  keystone_user_role resources.  The puppet-keystone code will determine if
  there is only one user named glance among all domains, and will know that if
  you just specify `glance` it means the unique user named glance in whatever
  domain it happens to be in.  However, if the resource name is **not** unique,
  you **must** specify the '::domainname' part in the resource name.

.. _finding-unique-resources-note:
.. note:: How to find unique resources

  You can use `openstack user list --long` and `openstack project list --long`
  to search through all domains (with admin credentials - that is, use a user
  that has the role `admin`), then grep for users that you want to ensure are
  unique.   If the users are coming from your enterprise identity provider
  (e.g. an LDAP server, an SQL database), use the tools provided to search for
  users.  The `keystone_user` and `keystone_tenant` resource code will both
  search through all domains looking for user and project resources, so you can
  also use `puppet resource keystone_user` and `puppet resource
  keystone_tenant` to see what Puppet's "view" is of the deployment.

Puppet resource declarations that have a name with "::" and a domain part will
look like this::

  keystone_user { 'admin::services': ...}
  keystone_user { 'admin::users': ...}
  keystone_tenant { 'admin::services': ...}

These would declare two different admin users - one for the "services" domain,
and one for the "users" domain, and create a project called "admin" in
the services domain.

The current keystone_user_role resource looks like this::

  keystone_user_role { 'glance@services': roles => ['admin'] }

This assigns the user "glance" to the role of "admin" in the project
"services".  With Keystone v3, if you need to specify a domain name, the
declarations will look like this::

  keystone_user_role { 'sysadmin::admin_domain@administrators::services': roles => ['admin']}
  keystone_user_role { 'sysadmin::admin_domain@::users': roles => ['admin']}

This would assign the user "sysadmin" in the domain "admin_domain" to the role
"admin" in the project "administrators" in the domain "services".  The last one
would assign the user "sysadmin" in the domain "admin_domain" the role of
"admin" in the domain "users".  If the project name (the string after the "@"
character) starts with "::", `keystone_user_role`, will assume it is a domain
name and this role is a domain scoped role instead of a project scoped role.

When you name a resource with '::domainname', you **must** consistently use
that name everywhere.  For example, if you need to have multiple projects in
different domains with the name `service`, you will need to declare the
resource like this: `keystone_tenant { 'service::domain1': ...}`.  If you
use that project in a keystone_user resource, you **must** use it like this:
`keystone_user { 'username': tenant => 'service::domain1', ... }`.  If you use
that project in a keystone_user_role resource, you **must** use it like this:
`keystone_user_role { 'username@service::domain1': ... }`.

The resources that are part of a domain, namely keystone_user, keystone_tenant,
keystone_group, will have a **domain** parameter.  The domain parameter will
override any domain specified in the name.  For example, if you have::

  keystone_user { 'user::foodomain':
    domain => 'bardomain',
  }

The domain parameter 'bardomain' would override the domain specified in the
resource title 'foodomain'.  In this case, the domain in the title would just
serve to make the resource unique (as opposed to "user" or
"user::anotherdomain").

Why have both a domain parameter and a domain in the resource name?  There may
be cases where you want to override the domain.  Using a parameter makes it
easier to use some of the puppet facilities for doing overrides.  In most
cases, the domain parameter would not be used.  Instead, the domain would be
specified as part of the resource name, in order to unique-ify the resource
name.

.. note:: About the "@" character in keystone_user_role

  The "@" character is a valid character in Keystone user names.  The way
  keystone_user_role works is that everything before the **last** "@" is part
  of the username, and everything after the **last** "@" is part of the project
  name.  So "user@domain.com@project" has a username of "user@domain.com" and a
  project name of "project".  This also means that the "@" character is **NOT**
  a valid character in project names or in domain names.  Keystone v3 doesn't
  change this, this is the way it has always been, as a side effect of choosing
  "@" as the delimiter character for keystone_user_role.

For legacy applications, there are two cases:

* The v2.0 api is being used - using a URL ending in "/v2.0" and using
  OpenStack with `OS_IDENTITY_API_VERSION=2`, or omitting the api version
  altogether.  In this case, Keystone, not puppet-keystone, will implicitly add
  a 'default domain' to the request.  The default domain is specified in the
  Keystone config with the **identity/default_domain_id** configuration
  parameter.

.. _domain-search-note:

* The v3 api is being used - using a URL ending in "/v3" and using OpenStack
  with `OS_IDENTITY_API_VERSION=3`.  In this case, the Keystone server side
  will **not** implicitly add a default domain - the domain must be explicitly
  specified on the client side, either in the resource title (`keystone_user {
  'admin::domain' }`) or in a parameter (`keystone_user {'admin': domain =>
  'domain'}`).  In order to accommodate legacy puppet code that does not
  specify a domain in either of these two ways, the keystone provider will
  attempt to determine a domain to use by the following methods in order:

  * default_domain_id from keystone.conf
  * 'Default' - the Keystone "default" default domain, if none is specified in
    identity/default_domain_id in keystone.conf

.. note:: Using default_domain_id and other settings from keystone.conf

  Using the default_domain_id from keystone.conf is considered *deprecated*.
  Developers *must* allow the domain to be explicitly provided everywhere, and
  users should specify the domain wherever possible.

Keystone v3 provides two new API objects - *domain* and *group*.  There will
need to be puppet-keystone resources for each of these - **keystone_domain**
and **keystone_group**.  In order to create roles with groups, there will need
to be a **keystone_group_role** resource.  This resource will work exactly like
keystone_user_role, except for groups.

The **openstack** command line tool provided by the python-openstackclient
package has full Keystone v3 support.  This command will be used by puppet to
access the Keystone v3 API and features.  Version 1.0.2 or later of this
command is required for all features.  The `puppet-openstacklib` `openstack`
provider will support using Keystone v2 or v3 credentials.

All of the domain-aware Keystone resources (such as keystone_user) will always
add a *domain* argument for those operations which require a domain (such as
creating a user), or use the resource *id* instead of the resource *name*
wherever possible to avoid having to use the domain.  In Keystone, the *id* is
guaranteed to be unique among all domains.

For trust support, a **keystone_trust** resource will be added.  This resource
will be used to manage trusts.

**Puppet Modules Other Than puppet-keystone**

There are a few changes that will need to be made to every OpenStack Puppet
module to allow Keystone v3 authentication:

* Allow specifying the domain for users and projects

Any place where a manifest specifies a user or project used for Keystone
authentication will need to be changed to add parameters for the domain of the
user and the domain of the project (and renaming tenant to project is
probably a good idea since Keystone uses *project* in Icehouse and later
:ref:`Use Project <project-is-tenant-note>`).

In addition, the resource name may be specified as `name::domainname`.  The
puppet-keystone code will handle this case.  It should not be necessary for
other puppet modules to split resource names by "::" to get the base name part
and the domain part.  It should just pass these names down to the
puppet-keystone code which should handle it.

For ease of use, an additional *default domain* parameter can be added which
will be used for both the user and project.  For example, from glance::

    class glance::api(
      ...
      $keystone_tenant         = 'services',
      $keystone_user           = 'glance',
      ...
      $keystone_user_domain    = undef,
      $keystone_project_domain = undef,
      $keystone_default_domain = undef,
      ...
    )

`$keystone_user_domain` is used to specify the domain of the `$keystone_user`,
and `$keystone_project_domain` is used to specify the domain of the
`$keystone_tenant`.  If the user or the project domain is omitted, and
`$keystone_default_domain` is specified, then that value will be used for the
missing user or project domain.  Puppet modules that pass authentication
parameters will need to be able to pass domain arguments.  For example, from
`puppet-glance/lib/puppet/providers/glance.rb`::

    def self.request(service, action, properties=nil)
      super
    rescue Puppet::Error::OpenstackAuthInputError => error
      glance_request(service, action, error, properties)
    end

That is, try the `openstack` request with the provided credentials, and if that
fails, fall back to try the request again with the credentials from the glance
config file.  The module local request method (e.g. `glance_request`) will need
to be able to pass in the user domain, project domain, and other v3
authentication parameters from its config file as authentication arguments.

* Use `keystone::resource::authconfig` and the new `keystone_authtoken` parameters in config files

The application config files are usually managed with a config resource.  For
example, the file `/etc/glance/glance-api.conf` is managed with a
`glance_api_config` resource, `/etc/glance/glance-registry.conf` is managed
with a `glance_registry_config` resource, etc.  The config section that
contains the Keystone authentication parameters is `keystone_authtoken`.  For
v3, there are some name changes (`admin_user => username`) and several new
parameters for domains and other v3 resources.  To make it easier to manage
this section, a new Keystone resource `keystone::resource::authconfig` has been
added.  For example, instead of doing this::

    glance_api_config {
      'keystone_authtoken/admin_tenant_name': value => $keystone_tenant;
      'keystone_authtoken/admin_user'       : value => $keystone_user;
      'keystone_authtoken/admin_password'   : value => $keystone_password,
      secret => true;
      ...
    }

manifests should do this instead::

    keystone::resource::authconfig { 'glance_api_config':
      username            => $keystone_user,
      password            => $keystone_password,
      auth_url            => $real_identity_uri,
      project_name        => $keystone_tenant,
      user_domain_name    => $keystone_user_domain,
      project_domain_name => $keystone_project_domain,
      default_domain_name => $keystone_default_domain,
      cacert              => $ca_file,
      ...
    }

The use of `keystone::resource::authconfig` makes it easy to avoid mistakes,
and makes it easier to support some of the newer authentication types coming
with Keystone Kilo and later, such as Kerberos, Federation, etc.
`keystone::resource::authconfig` knows how to handle the case where the
`username` is specified as `user::domainname` and will use the `domainname` part
as the `user_domain_name` if the `user_domain_name` is not provided.  Same with
`project_name`.

* Do not use a version suffix in Keystone authentication URLs

For example, for any URL used to perform authentication to Keystone, such as
provided by the `OS_AUTH_URL`, `--os-auth-url`, or similar configuration
parameters, do not add a Keystone API version suffix to the URL.  For example,
use `http://keystone.host:35357/` instead of `http://keystone.host:35357/v2.0/`
or `http://keystone.host:35357/v3/`.  Both the `openstack` resource provider,
and the `keystonemiddleware` used for service to Keystone authentication, will
determine if v3 can be used based on the parameters given.

Alternatives
------------

There aren't really any alternatives to support for domains, groups, and
trusts.

Data model impact
-----------------

* The addition of the keystone_trust, keystone_domain, keystone_group,
  keystone_group_role resources

This will mostly be using the existing resources as a template to create the
new resources.  This should be very straightforward.

* The addition of domain and group to the other puppet-keystone resources

For example, being able to create a user in a specific domain will require the
addition of domain property to keystone_user.  Likewise with projects and
keystone_tenant.  For service accounts, the resource
keystone::resource::service_identity hides most of the actual implementation,
so it should be easy to assign domains to service accounts without having to
change other puppet modules.


Module API impact
-----------------

Each API method which is either added or changed should have the following,
depending upon if it is a new class or an addition to an existing class.

* New defined resource types:

  * Name

    keystone_domain

  * Description

    This resource represents a Keystone domain.  It can optionally be used to
    ensure that the default_domain_id is set.

  * Parameters for keystone_domain:

    name        : Domain name;
                  string; required; namevar
    enabled     : Domain is enabled for use;
                  boolean; optional: default to True
    description : Domain description;
                  string; optional; default to undef
    id          : Domain id assigned by Keystone;
                  string; required; read-only
    is_default  : If this is true, the specified domain is the default domain,
                  and the provider will ensure that this is the [identity]
                  default_domain_id value in the keystone.conf file;
                  boolean; optional: default to False

  * Example use::

      keystone_domain { 'services':
        ensure      => present,
        description => 'Domain to use for service accounts',
        enabled     => true,
      }

  * Name

    keystone::resource::authconfig

  * Description

    This resource provides a convenient, safe way to update the Keystone
    authentication parameters in application config files which use a
    `*_config` resource.
    The username and project_name parameters may be given in the form
    "name::domainname".  The authconfig resource will use the domains in
    the following order:
    1) The given domain parameter (user_domain_name or project_domain_name)
    2) The domain given as the "::domainname" part of username or project_name
    3) The default_domain_name

  * Parameters for keystone::resource::authconfig

    [*name*]
    The name of the resource corresponding to the config file.  For example,
    keystone::authconfig { 'glance_api_config': ... }
    Where 'glance_api_config' is the name of the resource used to manage
    the glance api configuration.
    string; required

    [*username*]
    The name of the service user;
    string; required

    [*password*]
    Password to create for the service user;
    string; required

    [*auth_url*]
    The URL to use for authentication.
    string; required

    [*auth_plugin*]
    The plugin to use for authentication.
    string; optional: default to 'password'

    [*user_id*]
    The ID of the service user;
    string; optional: default to undef

    [*user_domain_name*]
    (Optional) Name of domain for $username
    Defaults to undef

    [*user_domain_id*]
    (Optional) ID of domain for $username
    Defaults to undef

    [*project_name*]
    Service project name;
    string; optional: default to undef

    [*project_id*]
    Service project ID;
    string; optional: default to undef

    [*project_domain_name*]
    (Optional) Name of domain for $project_name
    Defaults to undef

    [*project_domain_id*]
    (Optional) ID of domain for $project_name
    Defaults to undef

    [*domain_name*]
    (Optional) Use this for auth to obtain a domain-scoped token.
    If using this option, do not specify $project_name or $project_id.
    Defaults to undef

    [*domain_id*]
    (Optional) Use this for auth to obtain a domain-scoped token.
    If using this option, do not specify $project_name or $project_id.
    Defaults to undef

    [*default_domain_name*]
    (Optional) Name of domain for $username and $project_name
    If user_domain_name is not specified, use $default_domain_name
    If project_domain_name is not specified, use $default_domain_name
    Defaults to undef

    [*default_domain_id*]
    (Optional) ID of domain for $user_id and $project_id
    If user_domain_id is not specified, use $default_domain_id
    If project_domain_id is not specified, use $default_domain_id
    Defaults to undef

    [*trust_id*]
    (Optional) Trust ID
    Defaults to undef

    [*cacert*]
    (Optional) CA certificate file for TLS (https)
    Defaults to undef

    [*cert*]
    (Optional) Certificate file for TLS (https)
    Defaults to undef

    [*key*]
    (Optional) Key file for TLS (https)
    Defaults to undef

    [*insecure*]
    If true, explicitly allow TLS without checking server cert against any
    certificate authorities.  WARNING: not recommended.  Use with caution.
    boolean; Defaults to false (which means be secure)

  * Example use::

      keystone::resource::authconfig { 'glance_api_config':
        username            => $keystone_user,
        password            => $keystone_password,
        auth_url            => $real_identity_uri,
        project_name        => $keystone_tenant,
        user_domain_name    => $keystone_user_domain,
        project_domain_name => $keystone_project_domain,
        default_domain_name => $keystone_default_domain,
        cacert              => $ca_file,
      }

* New parameters for keystone_user, keystone_tenant:

  * Name

    domain : Domain name;
             string; optional: default see :ref:`Domain Search Note <domain-search-note>`

  * Description

    Name of the domain to which the resource belongs.

  * Resources affected:

    keystone_user
    keystone_tenant

  * Reason for addition:

    With Keystone v3, you can have a resource with the same name in multiple
    domains.  It is the combination of name and domain that uniquely identifies
    a resource.  With Keystone v3, users, groups, and projects exist inside
    domains, so the domain must be specified when creating these resources.

* Changed parameters:

  * Name:

    title : The Puppet resource title;
            string; required;

  * Resources affected:

    keystone_user
    keystone_tenant
    keystone_user_role
    keystone_group
    keystone_group_role
    keystone::roles::admin
    keystone::resource::service_identity

  * Reason for change:

    With Keystone v3, you can have a resource with the same name in multiple
    domains.  In Puppet, you cannot have two resources with the same title.
    With Keystone resources, the title is usually also the name of the
    resource.  By using "name::domain" in the resource title, you can uniquely
    identify the resource to Puppet, as well as specify both the resource name
    and domain.  The affected resources have been changed to look for a title
    in the form "name::domain".

  * Example use::

      keystone_user { 'admin::admin_domain':
        ensure      => present,
        enabled     => true,
        tenant      => 'admin',
        email       => 'admin@localhost',
        password    => 'itsasecret',
      }


End user impact
---------------

There should be no mandatory end user impact.  Existing manifests should
continue to work exactly as before.  See also :ref:`Backwards Compatibility Note <back-compat-note>`.

Users wanting to create users and projects in domains other than the default
domain will need to change their manifests in order to pass in the domain or
specify the domain in the resource title.  Note that there may be several
layers of resources/classes in manifests before the actual declaration of the
keystone resource, so intermediate resources/classes may need to be changed so
that the domain can be passed all the way down to the keystone resource.

Performance Impact
------------------

There will be additional domain lookups, in order to map domain ids to domain
names in certain calls.  For example, when creating a user, the create call
will return the domain id, not the domain name, but the domain name is needed
for resource name/title/domain comparisons.  The keystone provider provides
utility methods for this, and will cache the results of domain lookups.

Deployer impact
---------------

None other than what has been already mentioned.

Developer impact
----------------

Developers will have to make themselves aware of the new Keystone v3
authentication parameters mentioned in the links and elsewhere in this
documentation, and will need to make sure composition layers, config files,
etc. allow the specification of those parameters such as user domains, project
domains, etc.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  rmeggins (IRC nick richm)

Other contributors:
  gilles@redhat.com (IRC nick gildub)
  ichavero (IRC nick imcsk8)

Work Items
----------

puppet-keystone

* Create keystone_domain resource
* Create domain splitting utility code
* Create domain id to name mapping code
* Add domain support to keystone_user
* Add domain support to keystone_tenant
* Add domain support to keystone_user_role
* Add default domain support to class keystone
* Add admin user domain, admin tenant domain, and service tenant domain to
  class keystone::roles::admin
* Create keystone::resource::authconfig
* Convert keystone_service and keystone_endpoint to use the v3 api
* Convert keystone_role to use the v3 api

Other puppet-modules

* Allow specifying the domain for users and projects
* Use `keystone::resource::authconfig` and the new `keystone_authtoken`
  parameters in config files
* Do not use a version suffix in Keystone authentication URLs

Dependencies
============

* https://blueprints.launchpad.net/puppet-openstacklib/+spec/use-openstackclient-in-module-resources

  * Use OpenstackClient in Module Resources
  * puppet-keystone has already been converted to use the openstack provider.
    All other puppet modules will need to be converted to use the openstack
    provider from puppet-openstacklib.
* https://blueprints.launchpad.net/puppet-openstacklib/+spec/auth-consolidation

  * Restructures authentication for resource providers
  * puppet-keystone has already been converted to do this.
    All other puppet modules will need to be converted.
* python-openstackclient version 1.0.2 or later is required for full
  functionality.  1.0.1 may be used for testing purposes, but will not provide
  the full functionality required.
* python-keystonemiddleware version 1.3 or later is required for OpenStack
  services to perform service to Keystone v3 authentication.

Testing
=======

If and when tempest tests are available for puppet-keystone, tests of the
functionality in this blueprint should be added.


Documentation Impact
====================

The README.md and the examples in the examples directory will be updated.

References
==========

Openstack client: http://docs.openstack.org/developer/python-openstackclient/

Keystone v3 REST API: http://developer.openstack.org/api-ref-identity-v3.html

Trust extension:
https://github.com/openstack/identity-api/blob/master/v3/src/markdown/identity-api-v3-os-trust-ext.md

and: https://wiki.openstack.org/wiki/Keystone/Trusts

Service to Keystone v3 authentication:
http://www.jamielennox.net/blog/2015/02/23/v3-authentication-with-auth-token-middleware/

New Keystone v3 authentication parameters in config files:
http://www.jamielennox.net/blog/2015/02/17/loading-authentication-plugins/
