==========================
Code Review Guide for Nova
==========================

This is a very terse set of points for reviewers to consider when
looking at nova code. These are things that are important for the
continued smooth operation of Nova, but that tend to be carried as
"tribal knowledge" instead of being written down. It is an attempt to
boil down some of those things into nearly checklist format. Further
explanation about why some of these things are important belongs
elsewhere and should be linked from here.

Upgrade-Related Concerns
========================

RPC API Versions
----------------

* If an RPC method is modified, the following needs to happen:

 * The manager-side (example: compute/manager) needs a version bump
 * The manager-side method needs to tolerate older calls as well as
   newer calls
 * Arguments can be added as long as they are optional. Arguments
   cannot be removed or changed in an incompatible way.
 * The RPC client code (example: compute/rpcapi.py) needs to be able
   to honor a pin for the older version (see
   self.client.can_send_version() calls). If we are pinned at 1.5, but
   the version requirement for a method is 1.7, we need to be able to
   formulate the call at version 1.5.
 * Methods can drop compatibility with older versions when we bump a
   major version.

* RPC methods can be deprecated by removing the client (example:
  compute/rpcapi.py) implementation. However, the manager method must
  continue to exist until the major version of the API is bumped.

Object Versions
---------------

* If a tracked attribute (i.e. listed in fields) or remotable method
  is added, or a method is changed, the object version must be
  bumped. Changes for methods follow the same rules as above for
  regular RPC methods. We have tests to try to catch these changes,
  which remind you to bump the version and then correct the
  version-hash in the tests.
* Field types cannot be changed. If absolutely required, create a
  new attribute and deprecate the old one. Ideally, support converting
  the old attribute to the new one with an obj_load_attr()
  handler. There are some exceptional cases where changing the type
  can be allowed, but care must be taken to ensure it does not affect
  the wireline API.
* New attributes should be removed from the primitive in
  obj_make_compatible() if the attribute was added after the target
  version.
* Remotable methods should not return unversioned structures wherever
  possible. They should return objects or simple values as the return
  types are not (and cannot) be checked by the hash tests.
* Remotable methods should not take complex structures as
  arguments. These cannot be verified by the hash tests, and thus are
  subject to drift. Either construct an object and pass that, or pass
  all the simple values required to make the call.
* Changes to an object as described above will cause a hash to change
  in TestObjectVersions. This is a reminder to the developer and the
  reviewer that the version needs to be bumped. There are times when
  we need to make a change to an object without bumping its version,
  but those cases are only where the hash logic detects a change that
  is not actually a compatibility issue and must be handled carefully.

Database Schema
---------------

* Changes to the database schema must generally be additive-only. This
  means you can add columns, but you can't drop or alter a column. We
  have some hacky tests to try to catch these things, but they are
  fragile. Extreme reviewer attention to non-online alterations to the
  DB schema will help us avoid disaster.
* Dropping things from the schema is a thing we need to be extremely
  careful about, making sure that the column has not been used (even
  present in one of our models) for at least a release.
* Data migrations must not be present in schema migrations. If data
  needs to be converted to another format, or moved from one place to
  another, then that must be done while the database server remains
  online. Generally, this can and should be hidden within the object
  layer so that an object can load from either the old or new
  location, and save to the new one.

Config Options
==============

A config option should be checked for:

* A short description which explains what it does. If it is a unit
  (e.g. timeouts or so) describe the unit which is used (seconds, megabyte,
  mebibyte, ...).

* A long description which shows the impact and scope. The operators should
  know the expected change in the behavior of Nova if they tweak this.

* Hints which services will consume this config option. Operators/Deployers
  should not be forced to read the code to know which one of the services will
  change its behavior nor should they set this in every ``nova.conf`` file to
  be sure.

* Descriptions for the possible values.

    * If this is an option with numeric values (int, float), describe the
      edge cases (like the min value, max value, 0, -1).
    * If this is a DictOpt, describe the allowed keys.
    * If this is a StrOpt, lookout for regex validations.

* Interdependencies to other options. If other config options have to be
  considered when this config option gets changed, is this described?

Release Notes
=============

What is reno ?
--------------

Nova uses `reno <http://docs.openstack.org/developer/reno/usage.html>`_ for
providing release notes in-tree. That means that a patch can include a *reno
file* or a series can have a follow-on change containing that file explaining
what the impact is.

A *reno file* is a YAML file written in the releasenotes/notes tree which is
generated using the reno tool this way:

.. code-block:: bash

  $ tox -e venv -- reno new <name-your-file>

where usually ``<name-your-file>`` can be ``bp-<blueprint_name>`` for a
blueprint or ``bug-XXXXXX`` for a bugfix.

Refer to the `reno documentation <http://docs.openstack.org/developer/reno/usage.html#editing-a-release-note>`_
for the full list of sections.


When a release note is needed
-----------------------------

A release note is required anytime a reno section is needed. Below are some
examples for each section. Any sections that would be blank should be left out
of the note file entirely. If no section is needed, then you know you don't
need to provide a release note :-)

* ``upgrade``
    * The patch has an `UpgradeImpact <http://docs.openstack.org/infra/manual/developers.html#peer-review>`_ tag
    * A DB change needs some deployer modification (like a migration)
    * A configuration option change (deprecation, removal or modified default)
    * some specific changes that have a `DocImpact <http://docs.openstack.org/infra/manual/developers.html#peer-review>`_ tag
      but require further action from an deployer perspective
    * any patch that requires an action from the deployer in general

* ``security``
    * If the patch fixes a known vulnerability

* ``features``
    * If the patch has an `APIImpact <http://docs.openstack.org/infra/manual/developers.html#peer-review>`_ tag
    * For nova-manage and python-novaclient changes, if it adds or changes a
      new command, including adding new options to existing commands
    * not all blueprints in general, just the ones impacting a `contractual API <http://docs.openstack.org/developer/nova/policies.html#public-contractual-apis>`_
    * a new virt driver is provided or an existing driver impacts the `HypervisorSupportMatrix <http://docs.openstack.org/developer/nova/support-matrix.html>`_

* ``critical``
    * Bugfixes categorized as Critical in Launchpad *impacting users*

* ``fixes``
    * No clear definition of such bugfixes. Hairy long-standing bugs with high
      importance that have been fixed are good candidates though.


Three sections are left intentionally unexplained (``prelude``, ``issues`` and
``other``). Those are targeted to be filled in close to the release time for
providing details about the soon-ish release. Don't use them unless you know
exactly what you are doing.
