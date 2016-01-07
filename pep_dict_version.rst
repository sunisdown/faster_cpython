.. _pep-dict-version:

+++++++++++++++++++++
PEP: dict.__version__
+++++++++++++++++++++

FAT Python PEPs:

* PEP 1/3: :ref:`dict.__version__ <pep-dict-version>`
* PEP 2/3: :ref:`AST optimizer API <pep-ast-optimizer>`
* PEP 3/3: :ref:`Specialized bytecode with guards <pep-fat-mode>`

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
``collections.UserDict`` types, incremented at each change.


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
                # another key was modified: cache the new dictionary version
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
   empty tuple, empty strings, Unicode strings of a single character in
   the range [U+0000; U+00FF], etc. When a key is set twice to the
   same singleton, the version is not modified.

``collections.Mapping`` is unchanged. The PEP is designed to implement
guards on namespaces, whereas generic ``collections.Mapping`` types
cannot be used for namespaces, only ``dict`` work in practice.
``collections.UserDict`` is modified because it must mimicks ``dict``.


Integer overflow
================

The implementation uses the C unsigned integer type ``size_t`` to store
the version.  On 32-bit systems, the maximum version is ``2**32-1``
(more than ``4.2 * 10 ** 9``, 4 billions). On 64-bit systems, the maximum
version is ``2**64-1`` (more than ``1.8 * 10**19``).

The C code uses ``version++``. The behaviour on integer overflow of the
version is undefined. The minimum guarantee is that the version always
changes when the dictionary is modified.

The check ``dict.__version__ == old_version`` can be true after an
integer overflow, so a guard can return false even if the value changed,
which is wrong. The bug occurs if the dict must be modified at least ``2**64``
times (on 64-bit system) between two checks of the guard.

Using a more complex type (ex: ``PyLongObject``) to avoid the overflow
would slow down operations on the ``dict`` type. Even if there is a
theorical risk missing a value change, the risk is considered too low
compared to the slow down of using a more complex type.


Alternatives
============

Add a version to each dict entry
--------------------------------

A single version per dictionary requires to keep a strong reference to
the value which can keep the value alive longer than expected. If we add
also a version per dictionary entry, the guard can rely on the entry
version and can avoid the strong reference to the value (only strong
references to the dictionary and key are needed).

Changes: add a ``getversion(key)`` method to dictionary which returns
``None`` if the key doesn't exist. When a key is created or modified,
the entry version is set to the dictionary version which is incremented
at each change (create, modify, delete).

Pseudo-code of an efficient guard to check if a dict key was modified
using ``getversion()``::

    UNSET = object()

    class Guard:
        def __init__(self, dict, key):
            self.dict = dict
            self.key = key
            self.dict_version = dict.__version__
            self.entry_version = dict.getversion(key)

        def check(self):
            """Return True if the dict value did not changed."""
            dict_version = self.dict.__version__
            if dict_version == self.version:
                # Fast-path: avoid the dict lookup
                return True

            entry_version = self.dict.getversion(self.key)
            if entry_version == self.entry_version:
                # another key was modified: cache the new dictionary version
                self.dict_version = dict_version
                return True

            return False

This main drawback of this option is the impact on the memory footprint.
It increases the size of each dictionary entry, so the overhead depends
on the number of buckets (dictionary entries, used or unused yet). For
example, it increases the size of each dictionary entry by 8 bytes on
64-bit system if we use ``size_t``.

In Python, the memory footprint matters and the trend is more to reduce
it. Examples:

* `PEP 412 -- Key-Sharing Dictionary
  <https://www.python.org/dev/peps/pep-0412/>`_
* `PEP 393 -- Flexible String Representation
  <https://www.python.org/dev/peps/pep-0393/>`_


Add a new dict subtype
----------------------

Add a new ``verdict`` type, subtype of ``dict``. When guards are needed,
use the ``verdict`` for namespaces (module namespace, type namespace,
instance namespace, etc.) instead of ``dict``.

Leave the ``dict`` type unchanged to not add any overhead (memory
footprint) when guards are not needed.

Technical issue: a lot of C code in the wild, including CPython core,
expect the exact ``dict`` type. Issues:

* Functions call directly ``PyDict_xxx()`` functions, instead of calling
  ``PyObject_xxx()`` if the object type is a ``dict`` subtype
* ``PyDict_CheckExact()`` check fails on ``dict`` subtype, whereas some
  functions require the exact ``dict`` type.
* ``Python/ceval.c`` does not completly supports dict subtypes for
  namespaces
* ``exec()`` requires a dict for globals and locals, many code uses
  ``globals={}``. It is not possible to cast the dict to a subtype
  because the caller expects the ``globals`` parameter to be modified
  (``dict`` is mutable).

Other issues:

* The garbage collector has a special code to "untrack" ``dict``
  instances. If a dict subtype is used for namespaces, the garbage
  collector may be unable to break some reference cycles.
* Some functions have a fast-path for ``dict`` which would not be taken
  for ``dict`` subtypes, and so it would make Python a little bit
  slower.


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
