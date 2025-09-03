Astropy as a pure Python package
--------------------------------

author: Clément Robert, Thomas Robitaille

date-created: 2025 September 03

date-last-revised: <TODO>

date-accepted: <TODO>

type: Standard Track

status: Discussion

revised-by:

* ...


Abstract
--------

This APE proposes that the ``astropy`` package be revamped as a pure Python
library, pushing all low-level extension modules (primarily C and Cython code)
into one or several core package(s) that astropy itself would depend on.


Detailed description
--------------------

The contents of the core ``astropy`` package has historically be primarily pure
Python. In particular, all user-exposed APIs are found in Python modules; any
low level code developed in astropy is regarded as an implementation detail and
aims at improving performance critical sections, but is not intended as
user-facing. In practice, only a fraction of the entire package is implemented
in low-level, assembly-compilable languages, and are rarely updated.
Nevertheless, their inclusion in the core package creates a least three sailent
points of friction for developers and maintainers: * complex build systems are
required to build extension modules (while ``extension-helpers``
  does the heavy lifting for us, it still requires maintainance and only works
  with ``setuptools``)
* all extensions, albeit essentially invariant, must be compiled over and over
  again in all CI jobs, essentially wasting CPU as well as developer time, when
  running the actual test suite takes a comparable amount of time.
* releasing astropy requires complex infrastructure and storage, creating an
  incentive for cutting bugfix releases *infrequently*

user side:
* release artifacts are much larger than they need to be, for the majority of times that
  users upgrade astropy but extension modules didn't change at all in between
  releases.
* as hinted to earlier, bugfix releases may be delayed, pushing more users to
  install from source, manually patching their installations, or working around
  bugs with known solutions in less-than-maintainable, expeditive ways.

We can resolve all these issues, improving developer, contributor, and user
experiences by pushing all extension modules up the stack, and into a new
``astropy_core`` package. The exact name and shape of this package are
implementation details, subject to change, and crucially *not intended for
end-user consumption*, but we'll use this name in the following for clarity.


Branches and pull requests
--------------------------

<TODO>
insert Tom's proof-of-concept
</TODO>

note: we could go as far as implementing the PR from the Implementation section,
using ``[tool.uv.source]`` to temporarily work around the fact that the core package
isn't published yet.


Implementation
--------------

This section lists the major steps required to implement the APE.  Where
possible, it should be noted where one step is dependent on another, and which
steps may be optionally omitted.  Where it makes sense, each  step should
include a link related pull requests as the implementation progresses.

The implementation can be performed in 4 stages:

1. copy all extension code to a separate repository, making sure contribution
   history is retained.
2. re-organize this new repository into one, or optionally multiple new
   packages<REF NEEDED>.
3. publish new package(s) to our usual, community controled distribution outlets
   (PyPI and conda-forge)
4. in a single PR to ``astropy``, add new package(s) as dependency(ies), remove
   duplicate extension code and associated test code, drastically simplify build
   system.

As laid out, the whole process is relatively straightforward. However, we
anticipate that many extension modules may not be directly covered by astropy's
test suite, but instead rely on high level API tests, which we won't be able to
simply relocate to their new home. Thus, we anticipate stage 2. to be the most
time consuming, because we'll need to ensure all relocated modules are still
thoroughly tested, and we'll likely need to *create* new, lower level tests to
achieve this goal.

While the authors recommend that a single new repository be used, in order to
avoid infrastructure fragmentation and duplications, we also want to highlight
that this goal doesn't preclude the new code be implemented as *several*
packages, which might be useful to further refine the granulosity of (partial)
distributions. In the following, we provide an overview of how this secondary
goal could be achieved.

repository layout

.. code-block:: shell

  .
  ├── .github
  │   └── workflows # common CI infrastructure
  ├── packages
  │   ├── astropy_core.units
  │   │   ├── pyproject.toml
  │   │   ├── README.md
  │   │   └── src
  │   │       └── astropy_core_units
  │   │           ├── __init__.py
  │   │           └── ...
  │   └── astropy_core.wcs
  │       ├── pyproject.toml
  │       ├── README.md
  │       └── src
  │           └── astropy_core_wcs
  │               ├── __init__.py
  │               └── ...
  └── pyproject.toml # root project

The root ``pyproject.toml`` doesn't define a package, but provides overarching
structure and shared tool configuration, crucially, including workspace
definition. CI and release infrastructure can be shared using local, reusable
GitHub Actions workflows, allowing for targetted test runs and partial
distributions.


Backward compatibility
----------------------

This section describes the ways in which the APE breaks backward compatibility.

Alternatives
------------

If there were any alternative solutions to solving the same problem, they should
be discussed here, along with a justification for the chosen approach.


Decision rationale
------------------

<To be filled in by the coordinating committee when the APE is accepted or rejected>
