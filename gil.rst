.. _kill-gil:

*************
Kill the GIL?
*************

Why does CPython need a global lock?
====================================

Incomplete list:

* Python memory allocation is not thread safe (it should be easy to
  make it thread safe)
* The reference counter of each object is protected by the GIL.
* CPython has a lot of global C variables. Examples:

  - ``interp`` is a structure which contains variables of the Python
    interpreter: modules, list of Python threads, builtins, etc.
  - ``int`` singletons (-5..255)
  - ``str`` singletons (Python 3: latin1 characters)

* Some third party C libraries and even functions the C standard library are
  not thread safe: the GIL works around this limitation.

Kill the GIL
============

* Require deep changes of CPython code
* The current Python C API is too specific to CPython implementation details:
  need a new API. Maybe the stable ABI?
* Modify third party modules to use the stable ABI to avoid relying on CPython
  implementation details like reference couting
* Replace reference counting with something else? Atomic operations?
* Use finer locks on some specific operations (release the GIL)? like
  operations on builtin types which don't need to execute arbitrary Python
  code. Counter example: dict where keys are objects different than int and
  str.

See also pyparallel.
