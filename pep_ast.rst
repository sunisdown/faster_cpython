.. _pep-ast:

+++++++++++++++++++++++++++++
PEP: API for AST transformers
+++++++++++++++++++++++++++++

:ref:`FAT Python <fat-python>` PEPs:

* PEP 509: :ref:`Add a private version to dict <pep-dict-version>`
* PEP 510: :ref:`Specialized functions with guards <pep-specialize>`
* PEP 511: :ref:`API for AST transformers <pep-ast>`

.. warning::
   This PEP is a draft, please wait until it's published on python-ideas
   or python-dev to discuss it. Or contact me privately.

::

    PEP: 511
    Title: API for AST transformers
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

Propose an API to support AST transformers. Add ``-o OPTIM_TAG`` command
line option to change ``.pyc`` filenames and raise an ``ImportError``
exception on import if the ``.pyc`` file is missing and the AST
transformers required to transform the code are missing.


Rationale
=========

Python does not provide a standard way to transform the code. Projects
transforming the code use various hooks. The MacroPy project uses an
import hook: it adds its own module finder in ``sys.meta_path`` to
hook its AST transformer. Another option is to monkey-patch the
builtin ``compile()`` function. There are even more options to
hook a code transformer.

Transforming the code allows to extend the Python language for specific
use cases. Transforming an Abstract Syntax Tree (AST) is a convenient
way to implement an optimizer. It's easier to work on the AST than
working on the bytecode, AST contains more information and is more high
level.

Python 3.6 optimizes the code using a peephole optimizer. By
definition, a peephole optimizer has a narrow view of the code and so
can only implement basic optimizations. The optimizer rewrites the
bytecode. It is difficult to enhance it, because it written in C.

This PEP proposes to add an API to register AST transformers.

A new ``-o OPTIM_TAG`` command line option is added to only load
transformed code: it changes the name of searched ``.pyc`` files. If the
``.pyc`` file of a module is missing and the ``.py`` is available, an
``ImportError`` exception is raised import if the AST transformers
required to transform the code are missing. The import behaviour with
the default optimizer tag (``opt``) is unchanged.

The transformation can done ahead of time. It allows to implement
powerful but expensive transformations.


Use Cases
=========

Interactive interpreter
-----------------------

It will be possible to use AST transformers with the interactive
interpreter which is popular in Python and commonly used to demonstrate
Python.

Build a transformed package
---------------------------

It will be possible to build a package of the transformed code.

A transformer can have a configuration. The configuration is not stored
in the package.

All ``.pyc`` files of the package must be transformed with the same AST
transformers and the same transformers configuration. It is possible to
build different ``.pyc`` files using different optimizer tags. Example:
``fat`` for the default configuration and ``fat_inline`` with function
inlining enabled.

A package can contain ``.pyc`` files with different optimizer tags.


Install a package containing transformed .pyc files
---------------------------------------------------

It will be possible to install a package which contains transformed
``.pyc`` files. All ``.pyc`` files contained in the package are
installed.


Build .pyc files when installing a package
------------------------------------------

If a package does not contain ``.pyc`` files of the current optimizer
tag (or some ``.pyc`` files are missing), the ``.pyc`` are created
during the installation.

AST transformers of the optimizer tag are required. Otherwise, the
installation fails with an error.


Execute transformed code
------------------------

It will be possible to execute transformed code.

Raise an ``ImportError`` exception on import if the ``.pyc`` file of the
current optimizer tag is missing and the AST transformers required to
transform the code are missing.


Changes
=======

API to support AST transformers:

* Add ``sys.ast_transformers``: list of callable with the prototype
  ``def ast_transformer(tree, filename)`` used to rewrite an AST tree.
  The list is empty by default (no AST transformer). The transformer is
  called after the creation of the AST by the parser and before the
  compilation to bytecode. It must return an AST tree. It can modify the
  AST tree in place, or create a new AST tree.
* Add ``sys.implementation.ast_transformers``: name of registered AST
  transformers
* Add ``sys.implementation.optim_tag``: optimization tag, default:
  ``'opt'``.
* Use the optimizer tag in ``.pyc`` filenames in ``importlib``.
  Remove also the special case for the optimizer level ``0`` with the
  default optimizer tag ``'opt'`` to simplify the code.
* Add a new ``-o OPTIM_TAG`` command line option to set
  ``sys.implementation.optim_tag``

AST transformer changes:

* Add a new compiler flag ``PyCF_TRANSFORMED_AST`` to get the
  transformed AST. ``PyCF_ONLY_AST`` returns the AST before the
  transformers.
* Add ``ast.Constant``: this type is not emited by the compiler, but
  can be used in an AST transformer to simplify the code. It does not
  contain line number and column offset informations on tuple or
  frozenset items.
* ``PyCodeObject.co_lnotab``: line number delta becomes signed to support
  moving instructions (note: need to modify MAGIC_NUMBER in importlib).
* Enhance the compiler to support ``tuple`` and ``frozenset`` constants.
  Currently, ``tuple`` and ``frozenset`` constants are created by the
  peephole transformer, after the bytecode compilation.
* ``marshal`` module: fix serialization of the empty frozenset singleton
* update ``Tools/parser/unparse.py`` to support the new ``ast.Constant``
  node type


Example
=======

.pyc filenames
--------------

Example of ``.pyc`` filenames of the ``os`` module.

With the default optimizer tag ``'opt'``:

===========================   ==================
.pyc filename                 Optimization level
===========================   ==================
``os.cpython-36.opt-0.pyc``                    0
``os.cpython-36.opt-1.pyc``                    1
``os.cpython-36.opt-2.pyc``                    2
===========================   ==================

With the ``'fat'`` optimizer tag:

===========================   ==================
.pyc filename                 Optimization level
===========================   ==================
``os.cpython-36.fat-0.pyc``                    0
``os.cpython-36.fat-1.pyc``                    1
``os.cpython-36.fat-2.pyc``                    2
===========================   ==================


AST transformer
----------------

Amazing AST transformer replacing all strings with ``"Ni! Ni! Ni!"``::

    import ast
    import sys


    class KnightsWhoSayNi(ast.NodeTransformer):
        def visit_Str(self, node):
            node.s = 'Ni! Ni! Ni!'
            return node


    def ast_transformer(tree, filename):
        KnightsWhoSayNi().visit(tree)
        return tree


    # first register the AST transformer
    sys.ast_transformers.append(ast_transformer)
    sys.implementation.ast_transformers.append('knights_who_say_ni')

    # then update optimizer tag (used to build .pyc filenames)
    sys.implementation.optim_tag = 'knights_who_say_ni'

    # execute code which will be transformed by ast_transformer()
    exec("print('Hello World!')")

Output::

    Ni! Ni! Ni!


Prior Art
=========

AST optimizers
--------------

In 2011, Eugene Toder proposed to rewrite some peephole optimizations in
a new AST optimizer: issue #11549, `Build-out an AST optimizer, moving
some functionality out of the peephole optimizer
<https://bugs.python.org/issue11549>`_.  The patch adds ``ast.Lit`` (it
was proposed to rename it to ``ast.Literal``).

In 2012, Victor Stinner wrote the `astoptimizer
<https://bitbucket.org/haypo/astoptimizer/>`_ project, an AST optimizer
implementing various optimizations. Most interesting optimizations break the
Python semantics since no guard is used to disable optimization if something
changes.

Issue #17515: `Add sys.setasthook() to allow to use a custom AST
optimizer <https://bugs.python.org/issue17515>`_.


Python Preprocessors
--------------------

* `MacroPy <https://github.com/lihaoyi/macropy>`_: MacroPy is an
  implementation of Syntactic Macros in the Python Programming Language.
  MacroPy provides a mechanism for user-defined functions (macros) to
  perform transformations on the abstract syntax tree (AST) of a Python
  program at import time.
* `pypreprocessor <https://code.google.com/p/pypreprocessor/>`_: C-style
  preprocessor directives in Python, like ``#define`` and ``#ifdef``


Modify the bytecode
-------------------

* `codetransformer <https://pypi.python.org/pypi/codetransformer>`_:
  Bytecode transformers for CPython inspired by the ``ast`` module’s
  ``NodeTransformer``.
* `byteplay <http://code.google.com/p/byteplay/>`_: Byteplay lets you
  convert Python code objects into equivalent objects which are easy to
  play with, and lets you convert those objects back into living Python
  code objects. It's useful for applying crazy transformations on Python
  functions, and is also useful in learning Python byte code
  intricacies. See `byteplay documentation
  <http://wiki.python.org/moin/ByteplayDoc>`_.

See also `BytecodeAssembler <http://pypi.python.org/pypi/BytecodeAssembler>`_.


Copyright
=========

This document has been placed in the public domain.
