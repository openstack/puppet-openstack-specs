Puppet Openstack Specifications
===============================

This git repository is used to hold approved design specifications for additions
to the Puppet modules for Openstack projects.  Reviews of the specs are done
in Gerrit, using a similar workflow to how we review and merge changes
to the code itself.

The layout of this repository is:
```
  specs/<release>/
```

You can find a template spec in `specs/template.rst`.

Specifications are proposed for a given release by adding them to the
`specs/<release>` directory and posting it for review.  The implementation
status of a blueprint for a given release can be found by looking at the
blueprint in launchpad.  Not all approved blueprints will get fully implemented.

Specifications have to be re-proposed for every release. The review may be
quick, but even if something was previously approved, it should be re-reviewed
to make sure it still makes sense as written.

Prior to the Juno development cycle, this repository was not used for spec
reviews. However, backporting proposals to Icehouse may be made here.

Please note, Launchpad blueprints are still used for tracking the
current status of blueprints. For more information, see:
```
  https://wiki.openstack.org/wiki/Blueprints
```

For more information about working with gerrit, see:
```
  https://wiki.openstack.org/wiki/Gerrit_Workflow
```

Attribution
===========
This work and `specs/template.rst`, are derived from the original works at:
```
  https://github.com/openstack/nova-specs
```
