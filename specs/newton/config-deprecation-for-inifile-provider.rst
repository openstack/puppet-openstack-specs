..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Configuration File Deprecation Support
==========================================

`Launchpad blueprint <https://blueprints.launchpad.net/puppet-openstacklib/+spec/config-file-deprecation-support>`_

Allow the inifile (config file) providers to handle deprecated options.  This
should include cleaning up old deprecated names from config files and also
allow using the deprecated name instead of the new name.

Problem description
===================

When moving to new releases of OpenStack, config file options are frequently
deprecated because they're no longer needed, or they've been renamed.

After an upgrade this means that operators will frequently have the old and new
config option in place, but the old option is no longer used or maintained.

Additionally, when upgrading it can be useful to update the Puppet modules
before actually upgrading the service.  A frequent problem when doing this is
that the newer module only supports the new names for options instead of the
older deprecated names.

Proposed change
===============

Add additional parameters to the openstacklib inifile provider help manage
cleanup removed and renamed configuration options.  All inifile providers in
other OpenStack Puppet modules that derive from this provider would inherit the
new behavior.  The syntax would allow specifying the old name for options which
could be used to purge the old names or to use them in preference to the new
names.  For example::

    cinder_config { 'oslo_logging/new_option':
      deprecated_name => 'DEFAULT/old_option_name',
      value           => $value,
    }

In this case, it would populate the ``oslo_logging/new_option`` key in the
Cinder config file with the contents of ``$value`` as normally expected.
However, it would also purge the old name ``DEFAULT/old_option_name`` as given
by ``deprecated_name``.

The following new parameters would be supported:

``deprecated_name``
  This is a string or array of old names that have been deprecated.  Array
  should be supported because there are sometimes multiple names that have been
  deprecated in a single release.  Defaults to an empty array.

``purge_deprecated``
  If true, and if ``use_deprecated`` is false, then any ``deprecated_name``
  values provided will be treated as if they had been specified separately with
  an ensure value of ``absent``.  Defaults to true.

``use_deprecated``
  If true, then the inifile provider will act as if additional resources of the
  same type were specified for each ``deprecated_name``  with the same
  ``value`` and ``ensure`` parameters.  Defaults to false.

It's expected that both ``use_deprecated`` and ``purge_deprecated`` would be
set globally for each config provider as needed, either using resource defaults
or resource collectors.

Alternatives
------------

Currently the old deprecated options left behind cause no harm.  We could
ignore this issue.  This has the downside that it is not clear which value is
currently used.

The inifile provider could be set to purge unmanaged options.  Currently we
depend on distributions to provide a base configuration file.  In the past
there have been issues with the OpenStack Puppet modules not managing all
required options and depending on values provided in the base configuration
files.  This approach would require testing all of the existing modules to
ensure they manage all needed options and also changing acceptance tests to use
the purge functionality to prevent regressions.

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

Potential implementations might create additional resources to ensure the
deprecated config value state.  This could increase the number of resources
being managed and the Puppet catalog size.

A small amount of additional overhead may be incurred to find and remove the
deprecated names.


Deployer impact
---------------------

Existing configuration files with deprecated options would be removed as
described.  When this occurs, services will be restarted.  Impact of this is
expected to be minimal since most of these changes would occur between major
releases of the modules.  In that case, the deployer is expected to be
upgrading services and restarting them already.

It is expected that deployers may set ``use_deprecated`` to true in
preparation to for upgrades, allowing upgrading the Puppet modules in some
cases before upgrading the services they manage.

Deployers that wish to disable the new behavior could set the
``purge_deprecated`` parameter to false.


Developer impact
----------------

Developers would need to populate the ``deprecated_names`` parameter when
renaming configuration options.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  clayton-oneill (IRC nick clayton)

Work Items
----------

* Identify existing option with deprecation as test candidate
* Create a unit rspec in openstacklib and child module
* Create a functional test in openstacklib and child module
* Add implementation to openstacklib inifile provider

Dependencies
============

None

Testing
=======

Unit and functional should be added to ensure base functionality and prevent
regressions.

Documentation Impact
====================

Currently documentation of inifile providers is spotty.  This may be an
opportunity to move that documentation into the ``openstacklib`` module.

References
==========
None
