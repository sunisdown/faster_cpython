****
Misc
****

Ideas
=====

* PyPy CALL_METHOD instructor
* Lazy formatting of Exception message: in most cases, the message is not used.
  AttributeError(message) => AttributeError(attr=name), lazy formatting
  for str(exc) and exc.args.

Plan
====

* Modify CPython to be :ref:`notified when the Python code is changed <py-notify-code-change>`
* :ref:`Learn types <learn-types>` of function parameters and variables
* Compile a specialized version of a function using types and platform
  informations: more efficient bytecode using an :ref:`AST optimizer <py-astoptimizer>`, or even :ref:`emit machine code <py-machine-code>`. The
  compilation is done once and not during the execution, it's not a JIT.
* :ref:`Choose between bytecode and specialized code <py-choose-specialized>`
  at runtime

Other idea:

* `registervm <http://hg.python.org/sandbox/registervm>`_: My fork of Python
  3.3 using register-based bytecode, instead of stack-code bytecode. Read
  `REGISTERVM.txt <http://hg.python.org/sandbox/registervm/file/tip/REGISTERVM.txt>`_
* :ref:`Kill the GIL? <kill-gil>`


Status
======

.. _py-notify-code-change:
.. _py-astoptimizer:


See also the status of individual projects:

* `READONLY.txt <http://hg.python.org/sandbox/readonly/file/tip/READONLY.txt>`_
* `REGISTERVM.txt <http://hg.python.org/sandbox/registervm/file/tip/REGISTERVM.txt>`_
* `astoptimizer TODO list <https://bitbucket.org/haypo/astoptimizer/src/tip/TODO>`_

Done
----

* astoptimizer project exists:
  `astoptimizer <https://bitbucket.org/haypo/astoptimizer>`_.
* Fork of CPython 3.5: be notified when the Python code is changed:
  modules, types and functions are tracked. My fork of CPython 3.5: `readonly
  <http://hg.python.org/sandbox/readonly>`_; read `READONLY.txt
  <http://hg.python.org/sandbox/readonly/file/tip/READONLY.txt>`_
  documentation.

.. note::

   "readonly" is no more a good name for the project. The name comes from
   a first implementation using ead-only code.

To do
-----

* Learn types
* Enhance astoptimizer to use the type information
* Emit machine code


Why Python is slow?
===================

Why the CPython implementation is slower than PyPy?
---------------------------------------------------

* everything is stored as an object, even simple types like integers or
  characters. Computing the sum of two numbers requires to "unbox" objects,
  compute the sum, and "box" the result.
* Python maintains different states: thread state, interperter state, frames,
  etc. These informations are available in Python. The common usecase is
  to display a traceback in case of a bug. PyPy builds frames on demand.
* Cost of maintaince the reference counter: Python programs rely on the
  garbage collector
* ceval.c uses a virtual stack instead of CPU registers

Why the Python language is slower than C?
-----------------------------------------

* modules are mutable, classes are mutable, etc. Because of that, it is not
  possible to inline code nor replace a function call by its result (ex:
  len("abc")).
* The types of function parameters and variables are unknown. Example of
  missing optimizations:

  * "obj.attr" instruction cannot be moved out of a loop: "obj.attr" may
    return a different result at each call, or execute arbitrary Python code
  * x+0 raises a TypeError for "abc", whereas it is a noop for int (it
    can be replaced with just ``x``)
  * conditional code becomes dead code when types are known

* obj.method creates a temporary bounded method


Why improving CPython instead of writing a new implementation?
--------------------------------------------------------------

* There are already a lot of other Python implementations. Some examples:
  PyPy, Jython, IronPython, Pyston.
* CPython remains the reference implementation: new features are first
  implemented in CPython. For example, PyPy doesn't support Python 3 yet.
* Important third party modules rely heavily on CPython implementation details,
  especially the :ref:`Python C API <c-api>`. Examples: numpy and PyQt.


Why not a JIT?
--------------

* write a JIT is much more complex, it requires deep changes in CPython;
  CPython code is old (+20 years)
* cost to "warn up" the JIT: Mercurial project is concerned by the Python
  startup time
* Store generated machine code?


.. _learn-types:

Learn types
===========

* Add code in the compiler to record types of function calls. Run your program.
  Use recorded types.
* Range of numbers (predict C int overflow)
* Optional paramters: forceload=0. Dead code with forceload=0.
* Count number of calls to the function to decide if it should be optimized
  or not.
* Measure time spend in a function. It can be used to decide if it's useful
  to release or not the GIL.
* Store type information directly in the source code? Manual type annotation?


.. _py-machine-code:

Emit machine code
=================

* Limited to simple types like integers?
* Use LLVM?
* Reuse Cython or numba?
* Replace bytecode with C functions calls. Ex: instead of PyNumber_Add(a, b)
  for a+b, emit PyUnicode_Concat(a, b), long_add(a, b) or even simpler code
  without unbox/box
* Calling convention: have two versions of the function? only emit the C
  version if it is needed?

  - Called from Python: Python C API, ``PyObject* func(PyObject *args, PyObject *kwargs)``
  - Called from C (specialized machine code): C API, ``int func(char a, double d)``
  - Version which doesn't need the GIL to be locked?

* Option to compile a whole application into machine code for proprietary
  software?


Example of (specialized) machine code
-------------------------------------

Python code::

    def mysum(a, b):
        return a + b

Python bytecode::

    0 LOAD_FAST                0 (a)
    3 LOAD_FAST                1 (b)
    6 BINARY_ADD
    7 RETURN_VALUE

C code used to executed bytecode (without code to read bytecode and handle
signals)::

    /* LOAD_FAST */
    {
        PyObject *value = GETLOCAL(0);
        if (value == NULL) {
            format_exc_check_arg(PyExc_UnboundLocalError, ...);
            goto error;
        }
        Py_INCREF(value);
        PUSH(value);
    }

    /* LOAD_FAST */
    {
        PyObject *value = GETLOCAL(1);
        if (value == NULL) {
            format_exc_check_arg(PyExc_UnboundLocalError, ...);
            goto error;
        }
        Py_INCREF(value);
        PUSH(value);
    }

    /* BINARY_ADD */
    {
        PyObject *right = POP();
        PyObject *left = TOP();
        PyObject *sum;
        if (PyUnicode_CheckExact(left) &&
                 PyUnicode_CheckExact(right)) {
            sum = unicode_concatenate(left, right, f, next_instr);
            /* unicode_concatenate consumed the ref to v */
        }
        else {
            sum = PyNumber_Add(left, right);
            Py_DECREF(left);
        }
        Py_DECREF(right);
        SET_TOP(sum);
        if (sum == NULL)
            goto error;
    }

    /* RETURN_VALUE */
    {
        retval = POP();
        why = WHY_RETURN;
        goto fast_block_end;
    }

Specialized and simplified C code if both arguments are Unicode strings::

    /* LOAD_FAST */
    PyObject *left = GETLOCAL(0);
    if (left == NULL) {
        format_exc_check_arg(PyExc_UnboundLocalError, ...);
        goto error;
    }
    Py_INCREF(left);

    /* LOAD_FAST */
    PyObject *right = GETLOCAL(1);
    if (right == NULL) {
        format_exc_check_arg(PyExc_UnboundLocalError, ...);
        goto error;
    }
    Py_INCREF(right);

    /* BINARY_ADD */
    PyUnicode_Append(&left, right);
    Py_DECREF(right);
    if (sum == NULL)
        goto error;

    /* RETURN_VALUE */
    retval = left;
    why = WHY_RETURN;
    goto fast_block_end;


.. _py-choose-specialized:

Test if the specialized function can be used
============================================

Write code to choose between the bytecode evaluation and the machine code.

Preconditions:

* Check if os.path.isabs() was modified:

  - current namespace was modified? (os name cannot be replaced)
  - namespace of the os.path module was modified?
  - os.path.isabs function was modified?
  - compilation: checksum of the os.py and posixpath.py?

* Check the exact type of arguments

  - x type is str: in C, PyUnicode_CheckExact(x)
  - list of int: check the whole array before executing code? fallback
    in the specialized code to handle non int items?

* Callback to use the slow-path if something is modified?
* Disable optimizations when tracing is enabled
* Online benchmark to decide if preconditions and optimized code is faster than
  the original code?
