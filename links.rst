Links
=====

Fully Python compliant
----------------------

* `PyPy <http://pypy.org/>`_
* `Jython <http://www.jython.org/>`_ based on the JVM
* `IronPython <http://ironpython.net/>`_ based on the .NET VM
* `Unladen Swallow <http://code.google.com/p/unladen-swallow/>`_ fork of CPython 2.6 using LLVM

  - `Unladen Swallow Retrospective
    <http://qinsb.blogspot.com.au/2011/03/unladen-swallow-retrospective.html>`_
  - `PEP 3146 <http://python.org/dev/peps/pep-3146/>`_


Fully Python compliant??
------------------------

* `psyco <http://psyco.sourceforge.net/>`_

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

Misc links
----------

* `"Need for speed" sprint <http://wiki.python.org/moin/NeedForSpeed>`_ (2006)
* ceval.c: use registers?

  * Java: `Virtual Machine Showdown: Stack Versus Registers <http://static.usenix.org/events/vee05/full_papers/p153-yunhe.pdf>`_
    (Yunhe Shi, David Gregg, Andrew Beatty, M. Anton Ertl, 2005)
  * Lua 5: `The Implementation of Lua 5.0 <http://www.tecgraf.puc-rio.br/~lhf/ftp/doc/sblp2005.pdf>`_
    (Roberto Ierusalimschy, Luiz Henrique de Figueiredo, Waldemar Celes, 2005)
  * `Python-ideas: Register based interpreter
    <http://mail.python.org/pipermail/python-ideas/2009-February/003092.html>`_
  * `unladen-swallow: ProjectPlan <https://code.google.com/p/unladen-swallow/wiki/ProjectPlan>`_:
    "Using a JIT will also allow us to move Python from a stack-based machine
    to a register machine, which has been shown to improve performance in other
    similar languages (Ierusalimschy et al, 2005; Shi et al, 2005)."

* Use a more efficient VM
* `WPython <http://code.google.com/p/wpython/>`_: 16-bit word-codes instead of byte-codes
* `Hotpy <http://code.google.com/p/hotpy/>`_ and
  `Hotpy 2 <https://bitbucket.org/markshannon/hotpy_2>`_: built using the
  `GVMT <http://code.google.com/p/gvmt/>`_ (The Glasgow Virtual Machine Toolkit)
* Search for Python issues of type performance: http://bugs.python.org/
* `Volunteer developed free-threaded cross platform virtual machines?
  <http://www.boredomandlaziness.org/2012/07/volunteer-supported-free-threaded-cross.html>`_



