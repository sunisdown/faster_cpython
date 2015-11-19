.. _new-ast-optimizer:

+++++++++++++++++
New AST Optimizer
+++++++++++++++++

The :ref:`FAT Python <fatpython>` projects comes with a new :ref:AST optimizer
<ast-optimizers>` project.

The :ref:`FAT Python <fatpython>` project provides guards checks at runtime. It
comes with an AST optimizer which produces guards to disable optimizations if
any assumption is no more true.

Implementation
==============

Files:

* Lib/astoptimizer.py
* Lib/test/test_astoptimizer.py
* Python/sysmodule.c: add sys.asthook
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
        def _func_ast_optimized0():
            return "A"
        func.specialize(_func_ast_optimized0)
        del _func_ast_optimized0
        func.add_builtin_guard(0, 'chr')
        func.add_dict_guard(0, globals(), 'chr')


Detection of free variables
===========================

VariableVisitor detects local and global variables of an ``ast.FunctionDef``
node. It is used by the ``FunctionOptimizer`` to detect free variables.


Corner cases
============

Calling the ``super()`` function creates a closure.
