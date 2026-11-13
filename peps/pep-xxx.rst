PEP: ???
Title: Extending PEP 517 Build Backends to Handle Cross Builds
Author: Hood Chatham <roberthoodchatham at gmail.com>, Russell Keith-Magee <russell@keith-magee.com>
Sponsor: Russell Keith-Magee
Discussions-To: https://discuss.python.org/t/86862
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 22-Oct-2026

Abstract
========



Motivation
==========

New platforms such as Android, Emscripten, and iOS are not self-hosted: It is
impossible to run their compilers on the system itself. Instead they require
cross compilation.

Rationale
=========


Specification
=============

Changes to the `pyproject.toml` format
--------------------------------------

The ``[build-system]`` section gains a new `host-requires` field.

Changes to the ``packaging`` package
------------------------------------

The ``packaging.tags`` module gains two new functions:

``host_platform_tags(build_details)``
    Returns an iterator over the platform tags that are compatible with the
    host platform, in order of preference. This is the host counterpart of
    ``packaging.tags.platform_tags()``.

``host_tags(build_details)``
    Returns an iterator over the ``(interpreter, abi, platform)`` tag triples
    supported by the host interpreter, in order of preference. This is the
    host counterpart of ``packaging.tags.sys_tags()``: the interpreter and ABI
    tags are derived from the ``language``, ``implementation`` and ``abi``
    sections of ``build_details`` and the platform tags from
    ``host_platform_tags(build_details)``. Build frontends should use this
    function to select wheels for host requirements, for example with
    ``packaging.tags.create_compatible_tags_selector(host_tags(build_details))``.

Neither function may consult the running interpreter or operating system; the
result must depend only on ``build_details`` and the keyword arguments.

The ``packaging.markers`` modules gains a new function called
``host_environment(build_details)`` which computes an appropriate
``packaging.markers.Environment`` for the host ``build_details``. The
``platform_release`` and ``platform_version`` entries describe the running
kernel and cannot be determined statically, so they are set to the empty
string.


Changes to the PEP 517 backend Hooks
------------------------------------

We add three new hooks for cross-build-aware backends,
``get_build_requires_for_cross_build_wheel()``,
``get_host_requires_for_cross_build_wheel()`` and ``cross_build_wheel()``.
Presence of the ``cross_build_wheel()`` hook should be used by the build
frontend to determine whether the build backend is cross-build aware. It is
optional whether a cross-build-aware build backend defines
``get_build_requires_for_cross_build_wheel()`` or
``get_host_requires_for_cross_build_wheel()``.

The new hooks take a common ``build_details`` argument which is created by
parsing the ``build_details.json`` file for the host platform.


``get_build_requires_for_cross_build_wheel``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    def get_build_requires_for_cross_build_wheel(config_settings, build_details):
        ...

This hook is the cross-build counterpart of ``get_requires_for_build_wheel()``.
If present, it MUST return an additional list of strings containing :pep:`508`
dependency specifications, above and beyond those specified in the
``pyproject.toml`` file, to be installed into the build environment before
calling ``cross_build_wheel()``. The implicit default hook always returns an
empty list.


``get_host_requires_for_cross_build_wheel``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: python

    def get_host_requires_for_cross_build_wheel(config_settings, build_details):
        ...

This hook, if present, MUST return an additional list of strings containing
:pep:`508` dependency specifications, above and beyond those specified in the
``pyproject.toml`` file, to be installed into the ``host_requirements``
directory when calling the ``cross_build_wheel``. The implicit default hook
always returns an empty list.


``cross_build_wheel``
~~~~~~~~~~~~~~~~~~~~~

Presence of this hook signals that the backend supports cross builds. It takes
the same arguments as the ``build_wheel`` hook, as well as two additional
arguments: ``build_details`` and ``host_requires_directory``. The
``host_requires_directory`` will be prepared by the build frontend containing
installed copies of all the host requirements.


.. code:: python

    def cross_build_wheel(wheel_directory, metadata_directory=None, config_settings=None, build_details=None, host_requires_directory=None, toolchain=None):
        ...

The ``toolchain`` argument consists of a config object with the values of
``CC``, ``CXX``, ``RANLIB``, ``AR``, ``AS``, ``STRIP``. It also carries the
conventional variables describing how to locate host libraries: ``PKG_CONFIG``,
``PKG_CONFIG_PATH``, ``PKG_CONFIG_LIBDIR`` and ``PKG_CONFIG_SYSROOT_DIR``. The
backend MUST take this information from the ``toolchain`` argument rather than
from its environment.


What a Build Frontend should Do
-------------------------------

Suppose given a host platform tag, a ``build_details.json`` file, and a project
directory. To cross build the project, the build frontend should do the
following steps:

1. Check that the build backend implements a ``cross_build_wheel`` hook. If it
   does not, error out.
2. Install build requirements.
   a. Install the `build.requirements` from ``pyproject.toml``.
   b. If it is present, invoke the
      ``get_build_requires_for_cross_build_wheel()`` build backend hook  and
      install the requirements returned.
3. Install host requirements:
   a. Make a directory to use as `host_requires_directory`.
   b. Look up the `build.host-requirements` from ``pyproject.toml``
   c. If it is present, invoke the
      ``get_host_requires_for_cross_build_wheel()`` hook and extend the
      ``host-requirements`` list with the result.
   d. Call ``packaging.markers.host_environment(build_details)`` to get the
      host environment for evaluating requirement markers.
   e. For each host requirement, call
      ``requirement.marker.evaluate(host_environment, "requirement")``. Filter
      the ``host-requirements`` list to only include the requirements that return
      ``True``.
   f. Call ``packaging.tags.host_tags(build_details)`` to get an iterator
      over the tags supported by the host interpreter.
   g. For each of the host requirements, find a wheel whose tags include one
      of the host tags, preferring wheels matching earlier tags, and install
      it into the `host_requires_directory`.
4. Invoke the `cross_build_wheel()` build backend hook.

The `host_requires_directory` is a directory laid out like a ``site-packages``
directory. If the build frontend prefers it can be the `site-packages` inside of
some cross virtual environment, but we do not need to execute the host packages
as part of the build and there is not yet a well-specified notion of a cross
virtual environment.


Resolving Host Dependencies in the Build Backend
------------------------------------------------

The contents of the ``host_requires_directory`` are built for the host platform
and MUST NOT be executed by the backend.

When a backend resolves dependencies for the host machine using tools whose
default search paths are those of the build machine, such as ``pkg-config``, it
should restrict those search paths to the ``host_requires_directory``, the host
interpreter's own files as described by ``build_details``, and whatever the
``toolchain`` provides (``PKG_CONFIG_LIBDIR``, ``PKG_CONFIG_SYSROOT_DIR``).
