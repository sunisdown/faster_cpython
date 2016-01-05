.. _pep-dict-version:

+++++++++++++++++++++
PEP: dict.__version__
+++++++++++++++++++++

FAT Python PEPs:

* PEP 1/3: :ref:`dict.__version__ <pep-dict-version>`
* PEP 2/3: :ref:`AST optimizer API <pep-ast-optimizer>`
* PEP 3/3: :ref:`FAT mode, specialized bytecode with guards <pep-fat-mode>`

.. warning::
   This PEP is a draft, please wait until it's published on python-ideas
   or python-dev to discuss it. Or contact me privately.

::

    PEP: xxx
    Title: Add dict.__version__
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

Add a new read-only ``__version__`` property to ``dict`` and
``collections.UserDict`` types.


Rationale
=========

In Python, the dict type is used by many instructions. For example, the
``LOAD_GLOBAL`` instruction searchs for a variable in the global namespace, or
in the builtins namespace if the variable in not found in globals (two dict
lookups). Python uses dict for the builtins namespace, globals namespace, type
namespaces, instance namespaces, etc. The local namespace is usually optimized
to an array, but it can be a dict too.

Python is hard to optimize because almost everything is mutable: builtin
functions, function code, global variables, local variables, ... can be
modified at runtime. Implement optimizations respecting the Python semantic
requires to detect when "something changes", we will call these checks
"guards".  The speedup of optimizations depends on the speed of guard checks.

This PEP proposes to add a version to dictionaries to implement efficient
guards on namespaces.

Example of optimization: replace loading a global variable with a constant.
This optimization requires a guard on the global variable to check if it was
modified. If the variable is modified, the variable must be loaded at runtime,
instead of using the old copy.


Guard example
=============

Pseudo-code of an efficient guard to check if a dict key was modified (created,
updated or deleted)::

    UNSET = object()

    class Guard:
        def __init__(self, dict, key):
            self.dict = dict
            self.key = key
            self.value = dict.get(key, UNSET)
            self.version = dict.__version__

        def check(self):
            """Return True if the dict value did not changed."""
            version = self.dict.__version__
            if version == self.version:
                # Fast-path: avoid the dict lookup
                return True

            value = self.dict.get(self.key, UNSET)
            if value == self.value:
                # another key was modified: update the version
                self.version = version
                return True

            return False


Changes
=======

Add a read-only ``__version__`` property to builtin dict type and
collections.UserDict.

New empty dictionaries are initilized to version 1. The version is incremented
at each change: new key is set, a key is modified or a key is removed.

Example::

    >>> d = {}
    >>> d.__version__
    1
    >>> d['key'] = 'value'
    >>> d.__version__
    2
    >>> d['key'] = 'new value'
    >>> d.__version__
    3
    >>> del d['key']
    >>> d.__version__
    4

The version is not incremented is an existing key is modified to the
same value, but only the identifier of the value is tested, not the
content of the value. Example::

    >>> d={}
    >>> value = object()
    >>> d['key'] = value
    >>> d.__version__
    2
    >>> d['key'] = value
    >>> d.__version__
    2

.. note::
   CPython uses some singleton like integers in the range [-5; 257],
   empty tuple, empty strings, Unicode strings of a single character
   in the range [U+0000; U+00FF], etc.

The behaviour on integer overflow of the version is undefined.

``collections.Mapping`` is unchanged. The PEP is designed to be used with FAT
mode to implement effecient guards on namespaces, so version is only needed on
dict. ``collections.UserDict`` is modified because it must mimicks ``dict``.


Alternatives
============

Add a version to each dict entry
--------------------------------

xxx


Add a new dict subtype
----------------------

Leave the dict type unchanged to not add any overhead (memory footprint) on
regular Python.

Technical issues: a lot of C code in the wild, including CPython code base,
expect the dict type. Issues:

* Call directly ``PyDict_xxx()`` functions
* ``PyDict_CheckExact()`` check
* Python/ceval.c doesn't completly supports dict subtypes for name lookup
  (in globals or builtins)
* exec() requires a dict for globals and locals, many code uses globals={},
  it's not possible to cast the dict to a subtype because the caller expects
  its globals parameter to be modified (dict is mutable)


Prior Art
=========

Cached globals+builtins lookup
------------------------------

In 2006, Andrea Griffini proposes a patch implementing a `Cached
globals+builtins lookup optimization <https://bugs.python.org/issue1616125>`_.
The patch adds a private ``timestamp`` field to dict.

See also the thread on python-dev: `About dictionary lookup caching
<https://mail.python.org/pipermail/python-dev/2006-December/070348.html>`_.


PySizer
-------

`PySizer <http://pysizer.8325.org/>`_: Google Summer of Code 2005 project by
Nick Smallbone.

This project has a patch for CPython 2.4 which adds ``key_time`` and
``value_time`` fields to each dict entry. It uses a global process-wide
counter for dict incremented each time that a dict is modified. The
times are used to decide when child objects first appeared in their
parent objects.


Copyright
=========

This document has been placed in the public domain.
