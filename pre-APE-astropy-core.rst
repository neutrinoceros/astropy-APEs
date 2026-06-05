Re-packaging Astropy as a pure Python library
=============================================

author: Clément Robert, Thomas Robitaille

date-created: 2025 September 03

date-last-revised: <TODO>

date-accepted: <TODO>

type: Standard Track

status: Discussion

revised-by:

* ...


Abstract
========

This APE proposes that the ``astropy`` package be revamped as a pure Python
library, pushing all low-level extension modules (primarily C and Cython code)
into one or several core package(s) that ``astropy`` itself would depend on.


Detailed description
====================

The contents of the ``astropy`` package has historically been primarily pure
Python. In particular, all user-exposed APIs are found in Python modules; any
low level code included in ``astropy`` is regarded as an implementation detail
and is not intended as user-facing. In practice, only a fraction of the entire
package is implemented in low-level, assembly-compilable languages, and these
modules are rarely updated. Nevertheless, their inclusion in the package creates
a least three sailent points of friction for maintainers, as well as infrequent
contributors:

* complex build systems are required to build extension modules
* all extensions, albeit essentially invariant, must be compiled over and over
  in all CI jobs, essentially wasting CPU as well as developer time, when
  running the actual test suite takes a comparable amount of time.
  The same is true for any developer running the test suite locally.
* releasing astropy requires complex infrastructure and non-neglible storage,
  creating an incentive for cutting bugfix releases *infrequently*.

Costs are also incurred on the user side:

* release artifacts are much larger than they need to be, for the majority of
  times that users upgrade astropy but extension modules didn't change at all in
  between releases. This is a problem in resource-constrained environments
  (bandwidth and storage, mostly).
* as hinted to earlier, bugfix releases may be delayed, pushing more users to
  install from source, manually patching their installations, or working around
  bugs with known solutions in hardly maintainable, expeditive ways.

We can address all these issues, improving maintainer, contributor, and user
experiences by pushing all extension modules up the stack, and into a new
``astropy_core`` package. The exact name and shape of this package are
implementation details, subject to debate and, crucially, *not intended for
end-user consumption*, but we'll use this name in the following for simplicity.


Branches and pull requests
==========================

<TODO>
insert Tom's proof-of-concept
</TODO>

note: we could go as far as implementing the PR from the Implementation section,
using ``[tool.uv.source]`` to temporarily work around the fact that the core
package isn't published yet.


Implementation
==============

The implementation can be performed in 4 consecutive stages:

1. copy all extension code to a separate repository, making sure contribution
   history is retained.
2. re-organize this new repository into one, or optionally multiple new
   packages. See :ref:`monorepo-layout` for details.
3. publish new package(s) to our usual, community controled distribution outlets
   (PyPI and conda-forge)
4. in a single PR to ``astropy``, add the new package(s) as dependency(ies), remove
   duplicate extension code and associated test code, drastically simplify build
   system. <TODO: link POC PR ?>

As laid out, the whole process is relatively straightforward. However, we
anticipate that many extension modules may not be directly covered by astropy's
test suite, but instead rely on high level API tests, which we won't be able to
simply relocate. Thus, stage 2 will be the most time consuming, requiring a 
dedicated validation strategy to ensure all relocated modules are stable in isolation.

Achieving this requires establishing a low-level testing layer that entirely bypasses 
the high-level Python API (e.g., ``astropy.table.Column``). The validation strategy involves:

* **Direct C-Slot Testing:** Utilizing minimal shim classes and ``.view()`` casting 
  to map directly onto raw NumPy arrays. This ensures the C-level routing 
  (such as ``tp_as_mapping->mp_subscript``) is exercised in pure isolation without 
  triggering fallback sequence slots.
* **Strict Memory & Type Validation:** Executing direct C-engine tests (e.g., within 
  ``_np_utils.pyx``) to verify memory allocation, index mapping, and boolean masking 
  without requiring bulky ``Table`` object instantiation.
* **Breaking Dependency Cycles:** Refactoring C-extensions that possess runtime 
  dependencies on high-level Python objects (e.g., ``astropy.units.UnitBase`` within 
  ``unit_list_proxy.c``) to allow for compilation and validation in a vacuum.

.. _monorepo-layout:

Single repository, multiple packages (optional)
-----------------------------------------------

While the authors recommend that, in order to avoid infrastructure fragmentation
and duplications, all moved code be hosted in a single new repository, we also
want to highlight that this goal doesn't preclude the new code be re-organized
as *multiple* smaller packages, which could be useful to further refine the
granulosity of (partial) distributions. In the following, we provide an overview
of how this secondary goal could be achieved.

workspaces
^^^^^^^^^^

Modern Python packaging tools, such as ``uv`` and ``pixi``, allow for multiple
packages to be developed seamlessly within a single repository, forming a
*workspace*. Importantly, this notion does not, at the time of writing,
have a standard definition, which means that opting into this structure
currently incurs a cost in flexibility; workspaces are still a tool-specific
notion, so migration from one tool to another isn't necessarily supported or
trivial.

This repository structure allows to seamlessly work on multiple packages while
enforcing a strong form of separation of concerns and low coupling between
subpackages.


example repository layout
^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: shell

  .
  ├── .github
  │   └── workflows
  │       └── ... # shared CI infrastructure
  ├── packages
  │   ├── astropy_core.table
  │   │   ├── pyproject.toml
  │   │   └── src
  │   │       └── astropy_core_table
  │   │           ├── __init__.py
  │   │           └── ...
  │   └── astropy_core.wcs
  │       ├── pyproject.toml
  │       └── src
  │           └── astropy_core_wcs
  │               ├── __init__.py
  │               └── ...
  └── pyproject.toml # root project

The root ``pyproject.toml`` defines a package, but more importantly, it provides
overarching structure and shared tool configurations, including workspace
definition. CI and release infrastructure can be shared using local, reusable
GitHub Actions workflows, allowing for both targetted test jobs as well as
partial publications, where one or more packages can be published
independently.

<Q to TR: does this part need to be illustrated in the POC repo ?>


a PEP 420 namespace package
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Should we agree to publish subpackages separately, it might still be convenient
for regular development in the main ``astropy`` library to still *treat* core
packages as a unified namespace.

This is feasible using a `PEP 420 <https://peps.python.org/pep-0420/>`__
implicit namespace package, where we could defined ``astropy_core`` as a
namespace for publishing other subpackages under, e.g., ``astropy_core.table``,
``astropy_core.wcs`` ...

.. note:: Security concerns
  
  At the time of writing, there is no way for the astropy
  organization to *reserve* a namespace on PyPI, meaning the namespace would
  theoritically be subject to typo-squating attacks.
  However, this would only affect the development of ``astropy`` itself since,
  as previously stated, core package(s) would be advertised as not intended for
  direct consumption. Also note the missing feature has been proposed in
  `PEP 755 <https://peps.python.org/pep-0755/>`__


Runtime dependency management
-----------------------------

``astropy_core`` packages are expected to have little to no runtime dependencies,
other than the standard library, NumPy, and, possibly, each other. ``astropy``
itself could pin every ``astropy_core`` packages to an exact version, which would
result in a similar user experience we see today: in order to upgrade *any*
extension code from astropy, users would need to upgrade astropy itself.
However, the cost of *making* an astropy release, albeit only for the sake of
upgrading selected dependencies, would be far less than today.

Another possible strategy is to pin ``astropy_core`` packages *loosely*, allowing
for partial upgrades. For instance, say some version of ``astropy`` depends on
``astropy_core.table``, specified as

.. code-block:: toml

  [project]
  # ...
  dependencies =  [
    "astropy_core.table >=0.1.1, <0.2.0",
    # ...
  ]

It would then be possible for users to upgrade ``astropy_core.table`` to a newer
version, fixing bugs and security issues, without requiring a fresh ``astropy``
release, while still preventing any breaking changes in ``astropy_core.table``
does not unexpectedly percolate into existing astropy versions. This of course
requires that ``astropy_core`` packages adopt a clear semantic versionning
policy.


Public library Development
--------------------------

A cost of the proposed split is that, in the rare event that important or
potentially backward-incompatible changes need be developed in the core
package(s), developing astropy with development versions of extension modules
seemingly becomes less trivial: if they do not exist as part of the same
package, then error-prone environment tinkering might seem unavoidable.
Fortunately this isn't so: ``uv`` supports custom packages sources, which might
point to a local directory or a GitHub repository, through the
``[tool.uv.source]`` table in ``pyproject.toml``. For instance, say you're
developing a patch for ``astropy_core.table``` locally add need to check
``astropy``'s test suite's response. You can then add

.. code-block::

  [tool.uv.sources]
  astropy_core_table = { path = '../astropy_core/packages/table', editable = true }

to ``astropy``'s ``pyproject.toml`` and run the test suite either with ``uv``,
or ``tox`` (which is ``uv``-aware). Note that you can obtain the same patch by
running ``uv add ../astropy_core --editable`` from the command line. This allows
interested developers to simulate our current development workflow locally when
needed.

One can also temporarily set some package's source to be a GitHub repo:
``uv add https://github.com/usr-name/astropy_core/``, which is useful to
CI-proof a patch against the core package(s).


Additional benefits
===================

In addition to solving all aforementioned quality-of life issues, the proposed
separation between the main ``astropy`` package and its underlying extension
modules would create an opportunity to experiment with modern build backend
(included, but not restricted to ``meson-python`` or ``scikit-build-core``),
with little to no maintainance overhead for the main package itself.
Essentially, this creates a greenfield for interested parties to revamp exisiting
extensions, or experiment with new ones, away from the very active main package.

It would also provide an avenue for distributors to create partial distributions
for astropy as they see fit, meaning we could start introducing new subpackages
with lower levels of support for exotic platform, without preventing
distributors from packaging *working* software.


Backward compatibility
======================

The proposed implementation doesn't create backward incompatibilities in that it
only affects packaging of private APIs. However, the authors recognize that not
all affected APIs may already be clearly marked as private (see APE 22), which
might result in breaking changes for any existing consumer code relying on
private imports. In order to mitigate this issue, we recommend diligently
re-exporting any such APIs in the backward-compatible location within the
``astropy`` package and, crucially, to wrap the re-exports so deprecation
warnings are emitted at import time. This can be achieved with
`PEP 562 <https://peps.python.org/pep-0562/`>_ module level ``__getattr__``
functions.

Alternatives
============

If there were any alternative solutions to solving the same problem, they should
be discussed here, along with a justification for the chosen approach.


Decision rationale
==================

<To be filled in by the coordinating committee when the APE is accepted or rejected>
