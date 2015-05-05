..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

========================
Define our master policy
========================

The blueprint has been created for puppet-nova, but affects all modules:
https://blueprints.launchpad.net/puppet-openstack/+spec/master-policy

Problem description
===================

Here are the problems that lead us to write this blueprint:

* OpenStack projects use to update their configuration files and API every
  release, with all the time backward compatibility support. When using
  deprecated configuration parameters, OpenStack projects uses WARNING level
  in logging files (visible when the program starts), that warn operators to
  update configuration files so they have the latest options.
* Since the beginning of the Puppet modules project, master branch has always been
  the development branch and be used to integrate features from OpenStack master
  or very recent release. Until now, master branch always meant to be used to test
  the last stable OpenStack release.
* Over the time, our community grew and we've got contributions and
  more feedback from operators world that complained master branch was broken
  for a recent stable release.
* When fixing a bug in master, we often miss to backport it to stable branches
  because it requires manual work (cherry-pick). This is problem #4: what can
  we backport to stable branches?

Proposed change
===============

* Accept that master branch is not supposed to work on stable releases of OpenStack.
* Functional testing CI jobs should pass using the latest testing package repositories
  from the Ubuntu and RHEL distributions.
* Submit a feature in master only if it can be tested by functional testing CI jobs.
  We make an announcement on the mailing-list each time we update the repositories for
  the functional tests.
* Help developers to easily backport patches to stable branches when needed.

Alternatives
------------

* Creating a branch called 'future/<release>' for those changes that would get merged
  back into master and then into 'stable/<release>' when that branch is created.

Data model impact
-----------------

Data model needs to be backward compatible at least 2 releases.

Module API impact
-----------------

Module API needs to be backward compatible at least 2 releases.

End user impact
---------------------

None.

Performance Impact
------------------

None.

Deployer impact
---------------------

Deployers will need to be more careful if they used to run master branches.

* use master if they target a deployment with the current development version.
* use stable branches if they plan to deploy a stable version of OpenStack.

Developer impact
----------------

Developers will need to figure if their patch can be tested by the current state of master.
Also, they will need to be more engaged in backport policy and make sure to cherry-pick interesting
bugfixes and features to the right stable branches.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  emilienm

Work Items
----------

None.

Dependencies
============

None.

Testing
=======

Beaker jobs are in place and working on master (which is running Kilo at this time).

Documentation Impact
====================

We need to update the Wiki to explain this policy to our contributors.

References
==========

* Mailing list discussion:
  - http://lists.openstack.org/pipermail/openstack-dev/2015-April/061640.html

* Etherpad:
  - https://etherpad.openstack.org/p/puppet-openstack-master-policy
  - https://etherpad.openstack.org/p/liberty-summit-design-puppet-master-branch
