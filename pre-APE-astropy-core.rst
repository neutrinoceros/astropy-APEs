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

We can resolve all these issues by pushing all extension modules up the stack,
into a new ``astropy-core`` package. The exact name and shape of this package
are implementation details, subject to change, and discussed in the following,
but we'll use this name in the following for clarity.


Branches and pull requests
--------------------------

Any pull requests or development branches containing work on this APE should be
linked to from here.  (An APE does not need to be implemented in a single pull
request if it makes sense to implement it in discrete phases). If no code is yet
implemented, just put "N/A"


Implementation
--------------

This section lists the major steps required to implement the APE.  Where
possible, it should be noted where one step is dependent on another, and which
steps may be optionally omitted.  Where it makes sense, each  step should
include a link related pull requests as the implementation progresses.


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
