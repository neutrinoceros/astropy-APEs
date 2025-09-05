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
into one or several core package(s) that astropy itself would depend on.


Detailed description
====================

The contents of the core ``astropy`` package has historically be primarily pure
Python. In particular, all user-exposed APIs are found in Python modules; any
low level code developed in astropy is regarded as an implementation detail and
aims at improving performance in critical sections, but is not intended as
user-facing. In practice, only a fraction of the entire package is implemented
in low-level, assembly-compilable languages, which are seldom updated.
Nevertheless, their inclusion in the core package creates a least three sailent
points of friction for developers and maintainers:

* complex build systems are required to build extension modules
* all extensions, albeit essentially invariant, must be compiled over and over
  again in all CI jobs, essentially wasting CPU as well as developer time, when
  running the actual test suite takes a comparable amount of time.
* releasing astropy requires complex infrastructure and non-neglible storage,
  creating an incentive for cutting bugfix releases *infrequently*

Costs are also incurred on the user side:

* release artifacts are much larger than they need to be, for the majority of
  times that users upgrade astropy but extension modules didn't change at all in
  between releases.
* as hinted to earlier, bugfix releases may be delayed, pushing more users to
  install from source, manually patching their installations, or working around
  bugs with known solutions in less-than-maintainable, expeditive ways.

We can address all these issues, improving developer, contributor, and user
experiences by pushing all extension modules up the stack, and into a new
``astropy_core`` package. The exact name and shape of this package are
implementation details, subject to change, and crucially *not intended for
end-user consumption*, but we'll use this name in the following for clarity.


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
   packages<REF NEEDED>.
3. publish new package(s) to our usual, community controled distribution outlets
   (PyPI and conda-forge)
4. in a single PR to ``astropy``, add new package(s) as dependency(ies), remove
   duplicate extension code and associated test code, drastically simplify build
   system. <TODO: link POC PR ?>

As laid out, the whole process is relatively straightforward. However, we
anticipate that many extension modules may not be directly covered by astropy's
test suite, but instead rely on high level API tests, which we won't be able to
simply relocate. Thus, we anticipate stage 2. to be the most time consuming,
because we'll need to ensure all relocated modules are still thoroughly tested,
and we'll likely need to *create* new, lower level tests to achieve this goal.

Single repository, multiple packages
------------------------------------

While the authors recommend that, in order to avoid infrastructure fragmentation
and duplications, a single new repository be created, we also want to highlight
that this goal doesn't preclude the new code be implemented as *multiple*
packages, which would be useful to further refine the granulosity of (partial)
distributions. In the following, we provide an overview of how this secondary
goal could be achieved.

workspaces
^^^^^^^^^^

Modern Python packaging tools, such as ``uv`` and ``pixi``, allow for multiple
packages to be developed seamlessly within a single repository, forming a workspace.
Importantly, this notion does not, at the time of writing, abide to a standard,
which means that opting for this structure currently incurs a cost in flexibility;
workspaces are still a tool-specific notion, so migration from one tool to another
isn't guaranteed to be possible. However, standardisation is alread being discussed
<CITATION NEEDED>

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

The root ``pyproject.toml`` does not define a package. Instead, it provides
overarching structure and shared tool configurationq, including workspace
definition. CI and release infrastructure can be shared using local, reusable
GitHub Actions workflows, allowing for both targetted test jobs as well as
partial publications (where one or more packages can be published
independently).

<Q to TR: does this part need to be illustrated in the POC repo ?>


a PEP 420 namespace package
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Should we agree to publish subpackages separately, it might still be convenient
for regular development in the main ``astropy`` library to still *treat* core
packages as a unified namespace.

This is feasible thanks to `PEP 420 <https://peps.python.org/pep-0420/>`__
(accepted) allows use to use, for instance ``astropy_core`` as a namespace for
publishing other subpackages under, e.g., ``astropy_core.table``,
``astropy_core.wcs`` ...

.. note:: Security concerns
  
  At the time of writing, there is no way for the astropy
  organization to *reserve* a namespace on PyPI, meaning the namespace would
  theoritically be subject to typo-squating attacks. However 
  Also see `PEP 755 <https://peps.python.org/pep-0755/>`__


Runtime dependency management
-----------------------------

``astropy_core`` members are expected to have little to no runtime dependencies,
other than the standard library, NumPy, and, possibly, each other. ``astropy``
itself could pin every ``astropy_core`` members to an exact version, which would
result in a similar user experience we see today: in order to upgrade *any*
extension code from astropy, users would need to upgrade astropy itself.
However, the cost of *making* an astropy release, albeit only for the sake of
upgrading selected dependencies, would be far less than today.

Another possible strategy is to pin ``astropy_core`` members *loosely*, allowing
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
requires that ``astropy_core`` member packages adopt a clear semantic
versionning policy.


Additional benefits
===================

In addition to solving all aforementioned quality-of life issues, the proposed
separation between the main ``astropy`` package and its underlying extension
modules would create an opportunity to experiment with modern build backend
(included, but not restricted to meson-python or maturin), with little to no
maintainance overhead for the main package itself. Essentially, this creates a
greenfield for interested parties to revamp exisiting extensions, or experiment
with new ones, away from the very active main package.

It would also provide an avenue for distributors to create partial distributions
for astropy as they see fit, meaning we could start introducing new subpackages
with lower levels of support for exotic platform, without preventing
distributors from packaging *working* software.


Backward compatibility
======================

The proposed implementation doesn't create backward incompatibilities in that it
only affect packaging of private APIs. However, the authors recognize that not
all affected APIs may already be clearly marked as private (see APE 22), which
might result in breaking changes for any existing cusumer code relying on
private imports. In order to mitigate this issue, we recommend diligently
re-exporting any such APIs in the backward-compatible location within the
``astropy`` package and, crucially, to wrap the re-exports so deprecation
warnings are emitted at import time. This can be achieved with PEP 562 module
level ``__getattr__`` functions.

Alternatives
============

If there were any alternative solutions to solving the same problem, they should
be discussed here, along with a justification for the chosen approach.


Decision rationale
==================

<To be filled in by the coordinating committee when the APE is accepted or rejected>
