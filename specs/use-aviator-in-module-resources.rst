..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

===============================
Use Aviator in Module Resources
===============================

https://blueprints.launchpad.net/puppet-openstacklib/+spec/use-aviator-in-module-resources

This spec proposes to incorporate an interface to the OpenStack API in
openstacklib with the purpose of improving maintainability and promoting a more
configurable authentication interface. The providers would avoid the use of
the Python-based command-line OpenStack clients in favor of using Aviator, a
Ruby API binding for OpenStack. The corresponding types would implement an
interface for configuring authentication and endpoint details.

Problem description
===================

The OpenStack puppet modules contain a number of types and providers to access
OpenStack services via their API endpoints. Currently they rely on the
Python-based command-line clients to access these endpoints. The command-line
clients make requests at the respective OpenStack services, parse the response
and restructure it as a human-readable string. This means that the puppet
providers that wrap around these clients are heavily dependent on the format of
the restructured strings to interpret it correctly. Since these clients are
part of fast-paced projects, the output format is subject to change
unpredictably on each new release (which happens fairly frequently;
keystoneclient, for example, releases every few days or weeks
https://github.com/openstack/python-keystoneclient/releases), causing
the puppet modules to have a high maintenance cost. Moreover, the clients often
output warning messages that are not useful to the puppet modules and are
currently explicitly ignored. This adds additional complexity to parsing the
client output.

A related problem is that the providers currently rely on the presence of a
service configuration file on localhost. This means that the puppet
modules must rely on authentication information residing in a configuration
file on the node, whose state may or may not be managed by Puppet and
therefore may be improperly configured or missing (due to misconfiguration or
due to the service residing on another host).

Finally, there is some common functionality, such as configuration of
authentication endpoints and credentials that a number of modules share that
is currently duplicated across the modules. This will lead to inconsistency
across the modules.

Proposed change
===============

The proposed solution is to use the Aviator Ruby library instead of the
command-line clients as an interface to the OpenStack services. The
openstacklib module will add a dependency on the aimonb/aviator puppet
module. This module contains aviator as well as all of its dependencies within
its lib/puppet/features directory, so no gems need to be installed on the
master or the agents. Since Aviator returns pure JSON objects in its
responses, the providers in the puppet modules that need to interact with the
OpenStack REST API will parse the JSON returned directly from the service,
rather than the string output as interpreted by the command-line client. This
means that maintenance of the providers in the OpenStack puppet modules only
needs to keep up with changes to the REST API, and not with the command-line
clients, whose output is subject to change more frequently.

Using a Ruby library also helps promote the value that API wrappers should
reflect the idioms of the language in which they were written.

This change opens the opportunity to update the way the endpoint URI and
authentication parameters are handled in order to provide a greater degree of
configurability to the module user. Much of the logic for configuring endpoints
and credentials is duplicated across the modules and can be abstracted out to
the openstacklib module.

Alternatives
------------

Without implementing these changes in the providers, the module maintainers
will have to continue to watch for changes in the command-line clients and
update the providers to handle these changes appropriately.

Fog is another Ruby binding for OpenStack that could be uses as an alternative
to Aviator, but it is too large to incorporate into the module and is too
overbearing for our needs.

The providers could implement the REST client functionality themselves, but
this adds significant complexity to the providers when a library can accomplish
this already.

Data model impact
-----------------

None

Module API impact
-----------------

Aviator provides a variety of ways to authenticate to OpenStack services,
among them providing credentials at run time. The existing types need to have
parameters added to accept these credentials as well as endpoint URIs. The
openstacklib module will implement a class Puppet::Provider::Aviator that
inherits from Puppet::Provider. This class will implement an authenticate
method and an endpoint method, both of which will accept parameters that can
be passed from a type's parameters as well as the service being utilized. The
providers in the openstack modules will use this class as a :parent, e.g.::

Puppet::Type.type(:keystone_service).provide(
  :aviator,
  :parent => Puppet::Provider::Aviator
) do

The authenticate method will replace methods like auth_keystone and
auth_neutron. The endpoint method will replace methods like get_admin_endpoint
(in keystone) and get_auth_endpoint (in glance and neutron). These methods
currently rely on parsing the configuration file with methods such as
neutron_conf or keystone_file. Those methods will remain, but other methods
need to be written to accept credential and endpoint for the authenticate and
endpoint methods. They will fall back to reading the configuration file. This
will offer the modules a more robust and consistent way of authenticating
without relying on configuration files.

The authenticate method needs to ensure that a connection is retried
appropriately if needed.

The list_* methods will need to be rewritten to use the new authenticate and
endpoint methods.

The parse_* methods will need to be rewritten to parse JSON objects rather
than strings.

Every provider in the various OpenStack modules that currently uses a
command-line tool in order to function will be rewritten to use calls to
aviator. These can be identified by providers that contain a "commands" or
"optional_commands" statement to import a command-line tool as a Ruby
function.

The remove_warnings methods in puppet-keystone and puppet-glance will be
removed.

End user impact
---------------------

An end user of the OpenStack modules will need to pull in the openstacklib
module. They will have the option to supply authentication information to the
reworked types, but this will be optional and the providers will default to
using data from configuration files if these parameters are not defined.

Performance Impact
------------------

None

Deployer impact
---------------------

The user needs to install the openstacklib module prior to using the types that
interact with the REST endpoints in the OpenStack modules.

Developer impact
----------------

Developers of the module plugins will have to learn the Aviator API and will
no longer have to use the Python-based command-line clients.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  krinkle (crinkle on freenode)

Work Items
----------

* Update the Modulefile or metadata.json and .fixtures.yml in openstacklib to
  reflect the new dependency on aimonb/aviator

* Add providers to openstacklib to wrap some of the common procedures that will
  be done with Aviator, such as authentication

* Update the providers in the OpenStack modules, starting with puppet-
  keystone, to use Aviator and the new functions provided by openstacklib

* Add new authentication and endpoint configuration parameters to the module
  types, with options to configure from parameters specified in the manifest, a
  .conf file, or openrc.

Dependencies
============

None

Testing
=======

Unit test fixtures of all the OpenStack puppet modules will need to be updated
to install openstacklib.

Documentation Impact
====================

None

References
==========

Relevant research:

* Error messages changing often

  Bug: https://bugs.launchpad.net/puppet-keystone/+bug/1340447

  The keystone command-line client is subject to change its output
  unexpectedly, causing the puppet modules to fail to parse it properly.

* Retrying neutron connections

  Bug: https://bugs.launchpad.net/fuel/+bug/1246795
  Discussion: http://irclog.perlgeek.de/puppet-openstack/2014-07-21#i_9056413

  Numerous error messages from the neutron command-line client indicate that
  a request should be retried. Using HTTP responses via Aviator rather than
  error strings to detect when a retry is necessary makes the regexes easier
  to write and interpret.

* Caching query results

  Bug: https://bugs.launchpad.net/puppet-neutron/+bug/1344293
  Discussion: http://irclog.perlgeek.de/puppet-openstack/2014-07-16#i_9036216

  Caching values during authentication can cause problems when setting up
  multiple services within one puppet run. The rewritten providers will need to
  be careful to fetch fresh values when authenticating to various services with
  Aviator.
