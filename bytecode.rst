+++++++++++++++
Python bytecode
+++++++++++++++

.. _cpython-peephole:

CPython peephole optimizer
==========================

Implementation: Python/peephole.c

Optmizations:

* :ref:`Constant folding <const-fold>`
* :ref:`Dead code elimination <dead-code>`
* Some other optimizations more specific to the bytecode, like removal
  of useless jumps and optimizations on conditional jumps

Latest enhancement::

    changeset:   68375:14205d0fee45
    user:        Antoine Pitrou <solipsis@pitrou.net>
    date:        Fri Mar 11 17:27:02 2011 +0100
    files:       Lib/test/test_peepholer.py Misc/NEWS Python/peephole.c
    description:
    Issue #11244: The peephole optimizer is now able to constant-fold
    arbitrarily complex expressions.  This also fixes a 3.2 regression where
    operations involving negative numbers were not constant-folded.

Should be rewritten as an :ref:`AST optimizer <ast-optimizers>`.


Bytecode
========

* `byteplay <http://code.google.com/p/byteplay/>`_
* `diving-into-byte-code-optimization-in-python
  <http://www.slideshare.net/cjgiridhar/diving-into-byte-code-optimization-in-python>`_
* `BytecodeAssembler <http://pypi.python.org/pypi/BytecodeAssembler>`_
* `ByteplayDoc <http://wiki.python.org/moin/ByteplayDoc>`_
* `Hacking Python bytecode <http://geofft.mit.edu/blog/sipb/73>`_

