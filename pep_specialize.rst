.. _pep-specialize:

+++++++++++++++++++++++++++++++++++++
PEP: Specialized bytecode with guards
+++++++++++++++++++++++++++++++++++++

FAT Python PEPs:

* PEP 1/3: :ref:`dict.__version__ <pep-dict-version>`
* PEP 2/3: :ref:`AST optimizer API <pep-ast-optimizer>`
* PEP 3/3: :ref:`Specialized bytecode with guards <pep-specialize>`

.. warning::
   This PEP is a draft, please wait until it's published on python-ideas
   or python-dev to discuss it. Or contact me privately.

::

    PEP: xxx
    Title: Specialized bytecode with guards
    Version: $Revision$
    Last-Modified: $Date$
    Author: Victor Stinner <victor.stinner@gmail.com>
    Status: Draft
    Type: Standards Track
    Content-Type: text/x-rst
    Created: 4-January-2016
    Python-Version: 3.6


Abstract
========

Add an API to add specialized bytecode with guards to functions, to
support static optimizers respecting the Python semantic.


Rationale
=========

Python is hard to optimize because almost everything is mutable: builtin
functions, function code, global variables, local variables, ... can be
modified at runtime. Implement optimizations respecting the Python
semantic requires to detect when "something changes", we will call these
checks "guards".

This PEP proposes to add a ``specialize()`` method to functions to add a
specialized bytecode with guards. When the function is called, guards
are checked. If guards checks of a specialized bytecode are ok, it is
used. The specialized bytecode is removed if guards cannot be true
anymore. Otherwise, guards will be checked again at the next function
call.

See also the PEP <verdict> which proposes the add a version to dictionaries
to implement fast guards on namespaces.


Example
=======

Replace a call to builtin function with its result::

    def func():
        return len("abc")

    def fast_func():
        return 3

    func.specialize(fast_func.__code__, [fat.GuardBuiltin("len")])

    del fast_func

``fat.GuardBuiltin("len")`` is a guard on the builtin ``len()`` function
and the ``len`` name in the global namespace. The guard is false if the
builtin function is replaced or if a ``len`` name is defined in the
global namespace.

Calling ``func()`` will simply return ``3``, but will switch back to
calling the builtin ``len()`` function if the guard becomes false.


Python Function Call
====================

Pseudo-code to call a Python function having specialized bytecode with
guards::

    def call_func(func, *args, **kwargs):
        # by default, call the regular bytecode
        bytecode = func.__code__.co_code
        specialized = func.get_specialized()
        nspecialized = len(specialized)

        index = 0
        while index < nspecialized:
            guard = specialized[index].guard
            # pass arguments, some guards need them
            check = guard(args, kwargs)
            if check == 1:
                # guard succeeded: we can use the specialized bytecode
                bytecode = specialized[index].bytecode
                break
            elif check == -1:
                # guard will always fail: remove the specialized bytecode
                del specialized[index]
            elif check == 0:
                # guard failed temporarely
                index += 1

        execute_bytecode(bytecode, args, kwargs)


Optimizer
=========

The bytecode specialization is out of the scope of this PEP.

The FAT Python project includes an AST optimizer which implements various
optimizations. The optimizer is expected to move faster than the release cycle
of CPython and so will be developed out of the CPython source code tree. It
also avoid to have to pay the price of backward compatibility.

This PEP is related to the PEP <astoptimizer> but this PEP is not strictly a
dependency. External optimizers are free to pick any method to produce
optimized bytecode.

Including one specific optimizer into CPython will require a separated PEP.


Changes
=======

* Add two new methods to functions: ``specialize()`` and ``get_specialized()``
* Implement guards on functions:

  - ``"arg_type"``: false if the type of a function argument does not
    match expected argument types
  - ``"builtins"``: false if ``builtins.__dict__[key]`` is replaced or
    if ``globals()[key]`` is created
  - ``"dict"``: false if ``dict[key]`` is modified
  - ``"func"``: false if ``func.__code__`` is replaced
  - ``"globals"``: false if ``globals()[key]`` is modified
  - ``"type_dict"``: false if ``MyClass.attr`` is modified

* Add ``code.replace_consts(mapping)`` method: create a new code object
  with new constants. Lookup in the mapping for each constant.
  Pseudo-code to create new constants::

    new_constants = tuple(mapping.get(constant, constant)
                          for constant in code.co_consts)

* Keep a private copy of builtins, created at the end of the Python
  initialization, used to check if a builtin symbol was replaced

When a function code is replaced (``func.__code__ = new_code``), all
specialized bytecodes are removed.


Effects on object lifetime
==========================

Guards keep strong references to different objects:

* dict guards (builtins, dict, globals, type_dict): strong reference to
  dict, watched keys and related values
* arg type guard: strong reference to argument types
* func guard: strong reference to func2.__code__

Weak references:

* func guard: weak reference to func2

.. note::
   It's not possible to create a weak reference to a dict.


Issues
======

The following issues must probably be fixed or decided before the PEP is
published:

* Keywords are not supported yet
* The list of supported guards is limited, new guards cannot be
  implemented at runtime :-/
* Functions must remain serializable: ignore specialization? serialize
  specialized?
* Python modules and python imports are not supported yet!


Copyright
=========

This document has been placed in the public domain.
