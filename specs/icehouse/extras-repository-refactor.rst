..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Repository Management in Extras
==========================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/puppet-openstack/+spec/extras-repository-refactor

Problem description
===================

Currently it is necessary to maintain a separate module to handle base
repositories and any other repos that are needed for operation. This patch seeks
to allow arbitrary definitions of repositories via hash parameters and
create_resources integrated with and alongside the rdo and uca options already
available.

This patch will also define a sensible directory structure to use, so that common
code between similar systems can be shared.

Proposed change
===============

Create classes that manage arbitrary repositories but also RDO and UCA.
Share code for similar systems where possible.
Define directory structure for repo management

Alternatives
------------

Port the current repo management code across

This results in an implicit requirement to use either data-bindings or another class
with collectors to override resources in the case where the user wants to provide
their own baseurl for epel, or provide a proxy, since the parameters for those
resources are not exposed to the class.

If every parameter for each resource is exposed, we end up with a bloated interface
for the classes.

A suggestion was made to use stahnma's epel class, but this is an even greater
exaggeration of this problem since that class has so many parameters. Instead,
we should offer to not manage epel, and allow the user to include the epel module
if they wish, while continuing to offer a convenience parameter with the current
defaults.

Data model impact
-----------------

None

Module API impact
-----------------

New classes for debian:

openstack_extras::repo::debian::ubuntu
openstack_extras::repo::debian::debian
openstack_extras::repo::debian::params

Parameters for debian osfamily:
 - release          : The openstack release name
 - manage_[uca|whz] : whether to add the default uca/wheezy repo
 - source_hash      : a hash of apt::source resources
 - source_defaults  : a hash of apt::source parameters for defaults
 - package_require  : whether to use a collector for all packages to require apt-get update

New classes for redhat:

openstack_extras::repo::redhat::redhat
openstack_extras::repo::redhat::params

Parameters for redhat osfamily:
 - release           : The openstack release name
 - manage_rdo        : whether to add the default rdo repo
 - repo_hash         : a hash of yumrepo resources
 - repo_defaults     : a hash of yumrepo parameters for defaults
 - gpgkey_hash       : a hash of file resources to create gpg keys
 - gpgkey_defaults   : a hash of file parameters for defaults
 - purge_unmanaged   : whether to purge unmanaged yum repos from yum.repos.d
 - package_require   : whether to use a collector for all packages to require all yum repos

Directory structure should follow:

openstack_extras::repo::${downcase(::osfamily)}::${downcase(::operatingsystem)}

currently redhat, centos, fedora all handled using RDO, but leave the option to diverge later

New functions:

We need some functions that will perform validation on the hashes passed to file and yumrepo
to catch typos as early as possible.

End user impact
---------------------

None

Performance Impact
------------------

None

Deployer impact
---------------------

Allows deployers to manage all repos on their system without
having to make their own class.

Deployers will need to understand how to pass resource hashes
around in order to take advantage of this. Hiera example will
be provided. All users of the current repo mgmt system should
migrate to the new format at their leisure - the old classes
will not be changed.

Developer impact
----------------

None

Implementation
==============

Assignee(s)
-----------

michchap

Work Items
----------

debian/ubuntu support
redhat/centos/fedora support

Dependencies
============

Adds apt dependency to openstack_extras

Testing
=======

rspec for unit tests
will be integrated into aptira's product during testing for integration tests

Documentation Impact
====================

Complete change of repo management API

References
==========

None
