+++++++++++++++
Python bytecode
+++++++++++++++

.. _cpython-peephole:

CPython peephole optimizer
==========================

Examples:

* x+0 => x if x is an int
* x*0 => 0 if x is an int
* x*1 => x if x is an int, str or a tuple
* x and True
* x or False
* x = x + 1 => x += 1 if x is an int

Should be rewritten as an :ref:`AST optimizer <ast-optimizers>`.


Bytecode
========

* `byteplay <http://code.google.com/p/byteplay/>`_
* `diving-into-byte-code-optimization-in-python
  <http://www.slideshare.net/cjgiridhar/diving-into-byte-code-optimization-in-python>`_
* `BytecodeAssembler <http://pypi.python.org/pypi/BytecodeAssembler>`_
* `ByteplayDoc <http://wiki.python.org/moin/ByteplayDoc>`_
* `Hacking Python bytecode <http://geofft.mit.edu/blog/sipb/73>`_

