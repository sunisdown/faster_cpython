.. _pep-ast:

+++++++++++++++++++++++++++++
PEP: API for AST transformers
+++++++++++++++++++++++++++++

FAT Python PEPs:

* PEP 1/3: :ref:`dict.__version__ <pep-dict-version>`
* PEP 2/3: :ref:`API for AST transformers <pep-ast>`
* PEP 3/3: :ref:`Specialized functions with guards <pep-specialize>`

.. warning::
   This PEP is a draft, please wait until it's published on python-ideas
   or python-dev to discuss it. Or contact me privately.

::

    PEP: xxx
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

Propose an API to support AST transformers.


Rationale
=========

Python does not provide a standard way to transform the code. Various
hooks are used. The MacroPy project uses an import hook: it adds its own
module finder in ``sys.meta_path`` to inject its AST transformer.
Another option is to monkey-patch the builtin ``compile()`` function.

Transforming the code allows to extend the Python language for specific
needs. Transforming an Abstract Syntax Tree (AST) is a convenient way to
implement an optimizer. It's easier to work on the AST than working on
the bytecode, AST contains more information and is more high level.

Python 3.6 optimizes the code using a peephole optimizer. By
definition, a peephole optimizer has a narrow view of the code and so
can only implement basic optimizations. The optimizer rewrites the
bytecode. It is difficult to enhance because it written in C.

This PEP proposes to add an API to register an AST transformer.

A new ``-o OPTIM_TAG`` command line option is also added to enable AST
transformers and use different a ``.pyc`` filename.  The usage is to
produce the transformed code on one computer and use it a different
computer which does not have the transformer. It allows to implement
expensive but powerful transformations. Multiple transformers can be
specified, separated by ``-``. Example: ``'fat-pythran'`` to run FAT
Python and than Pythran transformers.


Changes
=======

AST transformer API:

* Add ``sys.asttransformers``: list of callable with the prototype
  ``def ast_transformer(tree, filename)`` used to rewrite an AST tree.
  The list of empty by default (no AST transformer). The transformer is
  called after the creation of the AST by the parser and before the
  compilation to bytecode.
* Add a new compiler flag ``PyCF_TRANSFORMED_AST`` to get the
  transformed AST, ``PyCF_ONLY_AST`` returns the AST before the
  transformers.
* Add ``ast.Constant``: this type is not emited by the compiler, but
  only used internally in an AST transformer to simplify the code. It
  doesn't contain line number and column offset informations on tuple or
  frozenset items.
* ``PyCodeObject.co_lnotab``: line number delta becomes signed to support
  moving instructions => need to modify MAGIC_NUMBER in importlib
* Add ``sys.implementation.ast_transformers``: list of transformer names

Optimization tag:

* Add a new ``-o OPTIM_TAG`` command line option to specify an "optimization tag"
* Add ``sys.implementation.optim_tag``: optimization tag
* ``importlib`` module: new filename for ``.pyc`` files when an
  optimizatin tag is used. Example:

  - ``Lib/__pycache__/os.cpython-36.pyc``: default filename
  - ``Lib/__pycache__/os.cpython-36.fat-0.pyc``: with the optimization
    tag ``"fat"``

AST transformer implementation changes:

* Enhance the compiler to support ``tuple`` and ``frozenset`` constants.
  Currently, ``tuple`` and ``frozenset`` constants are created by the
  peephole transformer, after the bytecode compilation.
* ``marshal`` module: fix serialization of the empty frozenset singleton
* update ``Tools/parser/unparse.py`` to support the new ``ast.Constant``
  node type


Example
=======

Optimizer replacing all strings with ``"Ni! Ni! Ni!"``::

    import ast
    import sys


    class KnightsWhoSayNi(ast.NodeTransformer):
        def visit_Str(self, node):
            node.s = 'Ni! Ni! Ni!'
            return node


    def ast_transformer(tree, filename):
        KnightsWhoSayNi().visit(tree)
        return tree


    sys.asttransformer = ast_transformer
    exec("print('Hello World!')")

Output::

    Ni! Ni! Ni!


Prior Art
=========

AST optimizers
--------------

Eugene Toder patch
^^^^^^^^^^^^^^^^^^

In 2011, Eugene Toder proposes to rewrite some peephole optimizations in
a new AST optimizer: issue #11549, `Build-out an AST optimizer, moving
some functionality out of the peephole optimizer
<https://bugs.python.org/issue11549>`_.  The patch adds ``ast.Lit`` (it
was proposed to rename it to ``ast.Literal``).

astoptimizer
^^^^^^^^^^^^

`astoptimizer <https://bitbucket.org/haypo/astoptimizer/>`_ is an AST
optimizer implementing various optimizations. Most interesting
optimizations break the Python semantic.

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
