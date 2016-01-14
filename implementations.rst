+++++++++++++++++++++++++
Implementations of Python
+++++++++++++++++++++++++

Faster Python implementations
-----------------------------

* `PyPy <http://pypy.org/>`_

  - AST optimizer of PyPy:
    `astcompiler/optimize.py <https://bitbucket.org/pypy/pypy/src/default/pypy/interpreter/astcompiler/optimize.py>`_

* `Pyston <https://github.com/dropbox/pyston>`_
* `Hotpy <http://code.google.com/p/hotpy/>`_
  and `Hotpy 2 <https://bitbucket.org/markshannon/hotpy_2>`_,
  based on `GVMT <http://code.google.com/p/gvmt/>`_ (Glasgow Virtual
  Machine Toolkit)
* `Numba <http://numba.pydata.org/>`_: JIT implemented with LLVM, specialized
  to numeric types (numpy)
* `pymothoa <http://code.google.com/p/pymothoa/>`_ uses LLVM
  ("don't support classes nor exceptions")
* `WPython <http://code.google.com/p/wpython/>`_: 16-bit word-codes instead of byte-codes
* `Cython <http://www.cython.org/>`_

Fully Python compliant
----------------------

* `PyPy <http://pypy.org/>`_
* `Jython <http://www.jython.org/>`_ based on the JVM
* `IronPython <http://ironpython.net/>`_ based on the .NET VM
* `Unladen Swallow <http://code.google.com/p/unladen-swallow/>`_, fork of
  CPython 2.6, use LLVM. No more maintained

  - Project `announced
    <http://arstechnica.com/information-technology/2009/03/google-launches-project-to-boost-python-performance-by-5x/>`_
    in 2009, abandonned in 2011
  - `ProjectPlan
    <http://code.google.com/p/unladen-swallow/wiki/ProjectPlan>`_
  - `Unladen Swallow Retrospective
    <http://qinsb.blogspot.com.au/2011/03/unladen-swallow-retrospective.html>`_
  - `PEP 3146
    <http://python.org/dev/peps/pep-3146/>`_

* `Pyjion <https://github.com/microsoft/pyjion>`_


Other
-----

* Replace stack-based bytecode with register-based bytecode: old registervm
  project


Fully Python compliant??
------------------------

* `psyco <http://psyco.sourceforge.net/>`_: JIT. The author of pysco, Armin
  Rigo, co-created the PyPy project.

Subset of Python to C++
------------------------

* `Nuitka <http://www.nuitka.net/pages/overview.html>`_
* `Python2C <http://strout.net/info/coding/python/ai/python2c.py>`_
* `Shedskin <http://code.google.com/p/shedskin/>`_
* `pythran <https://github.com/serge-sans-paille/pythran>`_ (no class, set,
  dict, exception, file handling, ...)

Subset of Python
----------------

* `pymothoa <http://code.google.com/p/pymothoa/>`_: use LLVM;
  don't support classes nor exceptions.
* `unpython <http://code.google.com/p/unpython/>`_: Python to C
* `Perthon <http://perthon.sourceforge.net/>`_: Python to Perl
* `Copperhead <http://copperhead.github.com/>`_: Python to GPU (Nvidia)

Language very close to Python
-----------------------------

* `Cython <http://www.cython.org/>`_: "Cython is a programming language based
  on Python, with extra syntax allowing for optional static type declarations."

  - based on `Pyrex <http://www.cosc.canterbury.ac.nz/greg.ewing/python/Pyrex/>`_

