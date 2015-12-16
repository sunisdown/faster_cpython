.. _new-ast-optimizer:

+++++++++++++++++
New AST Optimizer
+++++++++++++++++

The :ref:`FAT Python <fat-python>` projects comes with a new :ref:`AST optimizer
<ast-optimizers>` project.

Constants
=========

FAT Python introduced a new AST type: ``ast.Constant``. The optimizer starts by
converting ``ast.NameConstant``, ``ast.Num``, ``ast.Str``, ``ast.Bytes`` and
``ast.Tuple`` to ``ast.Constant``. Later, it can create constant of other
types. For example, ``frozenset('abc')`` creates a ``frozenset`` constant.

Supported constants:

* ``None`` singleton
* ``bool``: ``True`` and ``False``
* numbers: ``int``, ``float``, ``complex``
* strings: ``bytes``, ``str``
* containers:  ``tuple``, ``frozenset``


Literals
========

Literals are a superset of constants.

Supported literal types:

* (all constant types)
* containers: ``list``, ``dict``, ``set``


.. _new-ast-optimizer-limits:

Limitations
===========

* Decorators are not supported (yet?)
* Keywords are not supported yet

See also the `FAT Python TODO file
<https://hg.python.org/sandbox/fatpython/file/tip/TODO.rst>`_.


Implementation
==============

Files:

* Lib/astoptimizer.py
* Lib/test/test_astoptimizer.py
* Python/sysmodule.c: add sys.asthook
* Parser/Python.asdl: add ast.Constant
* PyParser_ASTFromStringObject() calls call_ast_hook()

Other changes:

* Lib/site.py: enable the AST optimizer in FAT mode

Different .pyc filename:

* Lib/__pycache__/os.cpython-36.pyc: default mode
* Lib/__pycache__/os.cpython-36.fat-0.pyc: FAT mode


FunctionOptimizer
=================

``FunctionOptimizer`` handles ``ast.FunctionDef`` and emits a specialized
function (protected with ``if __fat__:``) if a call to a builtin function can
be replaced with its result.

For example, this simple function::

    def func():
        return chr(65)

is optimized to::

    def func():
        return chr(65)

    if __fat__:
        _ast_optimized = func

        def func():
            return "A"
        _ast_optimized.specialize(func,
                                  [{'guard_type': 'builtins', 'name': 'chr'},
                                   {'guard_type': 'globals', 'name': 'chr'})

        func = _ast_optimized
        del _ast_optimized


Detection of free variables
===========================

VariableVisitor detects local and global variables of an ``ast.FunctionDef``
node. It is used by the ``FunctionOptimizer`` to detect free variables.


Corner cases
============

Calling the ``super()`` function requires a cell variables.
