..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=======================================
Use OpenstackClient in Module Resources
=======================================

https://blueprints.launchpad.net/puppet-openstacklib/+spec/use-openstackclient-in-module-resources

This spec proposes to alter the course proposed in blueprint:
use-aviator-in-module-resources. The preferred solution to the problem
description is to use the universal OpenStack command-line client.

Problem description
===================

The original problem is framed in the original Aviator blueprint.

The problem that this spec proposes to solve is that using the API
directly adds increased complexity which decreases maintainability. The
providers must manage HTTP sessions themselves, which reinvents the
wheel.

Aviator also does not appear to be actively keeping up with new changes
in OpenStack. It does not yet have support for the Neutron API. It also
only has partial support for the Keystone V3 API, which is an immediate
requirement. The workload to contribute upstream to Aviator to fulfill
these requirements is quite high compared to the workload involved to
incorporate openstackclient.

Proposed change
===============

Work to use Aviator in the base provider in the openstacklib module has
already been done. This work lays out the options that providers can use
for authenticating against the REST APIs. The work to convert the
providers to use the Aviator base provider has not been completed.

The change would simply swap the calls to the Aviator library with calls
to the openstack command. The base provider will no longer have to
manage sessions itself, which means not having to differentiate between
a password-authenticated session and using a token directly.

OpenStackClient is actively keeping up with API changes and is rapidly
developing, so we will monitor its progress and work with the developers
to get features we need and fix bugs.

Alternatives
------------

The alternative is to continue on the path with Aviator.

Data model impact
-----------------

None.

Module API impact
-----------------

The log_file parameter that puppet/util/aviator added to the puppet
types would no longer be necessary since that was a requirement only
for Aviator.

The OpenStack client is bundled with other OpenStack services so it
needs a manifest to install it explicitly.

The version of openstackclient available on Debian, Ubuntu, and RedHat
is 0.4.0. A 1.0.0 release will be available by Kilo. In the meantime,
the 0.4.0 version is sufficient for our needs as it can format its
output in CSV format.

We need to take advantage of OpenStackClient's Keystone API v3
capabilities in a Juno feature release of the keystone module. In order
for the keystone module to maintain backwards compatibility during this
cycle, we will first incorporate the base provider into the keystone
module, since we cannot bump the dependency on openstacklib without a
major release. Once these changes are backported into the Juno branch,
we will extract the base provider into the openstacklib module in
preparation for the Kilo release. Then we can migrate providers from
the other modules on their master branches, targeting Kilo.

End user impact
---------------------

There should be no end user impact.

Performance Impact
------------------

OpenStackClient may not be as fast as using the API directly, but it is
certainly not slower than using the individual command line clients.
OpenStackClient plans to soon provide the ability to cache resources
locally in order to speed up requests, so using that functionality
should increase performance.

Deployer impact
---------------------

None.

Developer impact
----------------

The parameters of the base provider's request() method can change
slightly to better mirror how parameters will be passed to openstack(),
but this is not a hard requirement as long as all the necessary
information is passed to openstack().

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  krinkle
  richm

Work Items
----------

* Rewrite the base provider in openstacklib to reflect these changes. Work
  for this has already been started.
* Rewrite the keystone providers to inherit from the new base provider
  and utilize its methods.
* Rewrite providers for other modules

Dependencies
============

None

Testing
=======

Unit tests will be revised and simplified. Rather than using VCR with
HTTP recorded sessions as fixtures we will simply stub the openstack()
method.

Documentation Impact
====================

None

References
==========

* Proofs of concept, still works in progress:
  - Base provider: https://review.openstack.org/#/c/134843/
  - keystone_tenant rewritten: https://review.openstack.org/#/c/134844/

* Mailing list discussion:
  - https://groups.google.com/a/puppetlabs.com/forum/#!topic/puppet-openstack/GJwDHNAFVYw

* IRC discussion: (starting at 14:36:50)
  - http://eavesdrop.openstack.org/meetings/puppet_openstack/2014/puppet_openstack.2014-11-17-14.01.log.html
