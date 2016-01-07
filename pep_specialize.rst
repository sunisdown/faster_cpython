.. _pep-specialize:

+++++++++++++++++++++++++++++++++++++
PEP: Specialized bytecode with guards
+++++++++++++++++++++++++++++++++++++

FAT Python PEPs:

* PEP 1/3: :ref:`dict.__version__ <pep-dict-version>`
* PEP 2/3: :ref:`AST optimizer API <pep-ast>`
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
specialized bytecode with guards. When the function is called, the
specialized bytecode if used if nothing changed, otherwise use the
original bytecode.

See also the PEP <dict_version> which proposes the add a version to
dictionaries to implement fast guards on namespaces.


Example
=======

Replace a call to builtin function with the result using an hypothetical
``myoptimizer`` module::

    import myoptimizer

    def func():
        return len("abc")

    def fast_func():
        return 3

    func.specialize(fast_func.__code__, [myoptimizer.GuardBuiltin("len")])

    del fast_func

``GuardBuiltin("len")`` is a guard on the builtin ``len()`` function and
the ``len`` name in the global namespace. The guard is false if the
builtin function is replaced or if a ``len`` name is defined in the
global namespace.

Calling ``func()`` will directly return ``3``, but will switch back to
calling the builtin ``len()`` function if something changed.


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


Changes
=======

* Add two new methods to functions:

  - ``specialize(bytecode: code, guards: list)``: add specialized
    bytecode with guard. The specialization can be ignored if a guard
    already fails.
  - ``get_specialized()``: get the list of specialized bytecodes with
    guards

* Base ``Guard`` type which can be used as parent type to implement
  guards. It requires to implement a ``check()`` function, with an
  optional ``first_check()`` function. API:

  * ``int check(PyObject *guard, PyObject **stack)``: return 1 on
    success, 0 if the guard failed temporarely, -1 if the guard will
    always fail
  * ``int first_check(PyObject *guard, PyObject *func)``: return 0 on
    success, -1 if the guard will always fail

* Add ``code.replace_consts(mapping)`` method: create a new code object
  with new constants. Lookup in the mapping for each constant.
  Pseudo-code to create new constants::

    new_constants = tuple(mapping.get(constant, constant)
                          for constant in code.co_consts)

* Keep a private copy of builtins, created at the end of the Python
  initialization, used to check if a builtin symbol was replaced

When a function code is replaced (``func.__code__ = new_code``), all
specialized bytecodes are removed.


Issues
======

The following issues must probably be fixed or decided before the PEP is
published:

* Keywords are not supported yet
* Functions must remain serializable: ignore specialization? serialize
  specialized?


Copyright
=========

This document has been placed in the public domain.
