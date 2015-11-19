.. _c-api:

************
Python C API
************

Intro
=====

CPython comes with a C API called the "Python C API". The most common type is
``PyObject*`` and functions are prefixed with ``Py`` (and ``_Py`` for private
functions but you must not use them!).


Historical design choices
=========================

CPython was created in 1991 by Guido van Rossum. Some design choices made sense
in 1991 but don't make sense anymore in 2015. For example, the :ref:`GIL <gil>`
was a simple and safe choice to implement multithreading in CPython. But in
2015, smartphones have 2 or 4 cores, and desktop PC have between 4 and 8 cores.
The GIL restricts peek performances on multithreaded applications, even when
it's possible to release the GIL.


.. _gil:

GIL
===

CPython uses a Global Interpreter Lock called "GIL" to avoid concurrent
accesses to CPython internal structures (shared resources like global
variables) to ensure that Python internals remain consistent.

See also :ref:`Kill the GIL <kill-gil>`.


Reference counting and garbage collector
========================================

The C structure of all Python objects inherit from the ``PyObject`` structure
which contains the field ``Py_ssize_t ob_refcnt;``. This is a simple counter
initialized to ``1`` when the object is created, increased each time that a
variable has a strong reference to the object, and decreased each time that a
strong reference is removed. The object is removed when the counter reached
``0``.

In some cases, two objects are linked together. For example, A has a strong
reference to B which has a strong reference to A. Even if A and B are no more
referenced outside, these objects are not destroyed because their reference
counter is still equal to ``1``. A garbage collector is responsible to
find and break *reference cycles*.

See also the `PEP 442: Safe object finalization
<https://www.python.org/dev/peps/pep-0442/>`_ implemented in Python 3.4 which
helps to break reference cycles.


Popular projects using the Python C API
=======================================

* numpy
* PyQt
* Mercurial

