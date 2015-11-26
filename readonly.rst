.. _readonly:

****************
Read-only Python
****************

Intro
=====

A first attempt to implement guards was the `readonly PoC
<https://hg.python.org/sandbox/readonly>`_ (fork of CPython 3.5) which
registered callbacks to notify all guards. The problem is that modifying a
watched dictionary gets a complexity of O(n) where n is the number of
registered guards.

readonly adds a ``modified`` flag to types and a ``readonly`` property to
dictionaries. The guard was notified with the modified key to decide to disable
or not the optimization.

More information: `READONLY.txt
<http://hg.python.org/sandbox/readonly/file/tip/READONLY.txt>`_

Thread on the python-ideas mailing list: `Make Python code read-only
<https://mail.python.org/pipermail/python-ideas/2014-May/027870.html>`_.

The project was mostly developed in May 2014. The project is now dead,
replaced with :ref:`FAT Python <fat-python>`.

READONLY
========

This fork on CPython 3.5 adds a machinery to be notified when the Python code
is modified. Modules, classes (types) and functions are tracked. At the first
modification, a callback is called with the object and the modified attribute.

This machinery should help static optimizers. See this article for more
information:
http://haypo-notes.readthedocs.org/faster_cpython.html

Examples of such optimizers:

* astoptimizer project: replace a function call by its result during the AST
  compilation
* Learn types of function paramters and local variables, and then compile
  Python (byte)code to machine code specialized for these types (like Cython)


Issues with read-only code
==========================

* Currently, it's not possible to allow again to modify a module,
  class or function to keep my implementation simple. With a registry of
  callbacks, it may be possible to enable again modification and call
  code to disable optimizations.

* PyPy implements this but thanks to its JIT, it can optimize again
  the modified code during the execution. Writing a JIT is very complex,
  I'm trying to find a compromise between the fast PyPy and the slow
  CPython. Add a JIT to CPython is out of my scope, it requires too much
  modifications of the code.

* With read-only code, monkey-patching cannot be used anymore. It's
  annoying to run tests. An obvious solution is to disable read-only
  mode to run tests, which can be seen as unsafe since tests are usually
  used to trust the code.

* The sys module cannot be made read-only because modifying sys.stdout
  and sys.ps1 is a common use case.

* The warnings module tries to add a __warningregistry__ global
  variable in the module where the warning was emited to not repeat
  warnings that should only be emited once. The problem is that the
  module namespace is made read-only before this variable is added. A
  workaround would be to maintain these dictionaries in the warnings
  module directly, but it becomes harder to clear the dictionary when a
  module is unloaded or reloaded. Another workaround is to add
  __warningregistry__ before making a module read-only.

* Lazy initialization of module variables does not work anymore. A
  workaround is to use a mutable type. It can be a dict used as a
  namespace for module modifiable variables.

* The interactive interpreter sets a "_" variable in the builtins
  namespace. I have no workaround for this. The "_" variable is no more
  created in read-only mode. Don't run the interactive interpreter in
  read-only mode.

* It is not possible yet to make the namespace of packages read-only.
  For example, "import encodings.utf_8" adds the symbol "utf_8" to the
  encodings namespace. A workaround is to load all submodules before
  making the namespace read-only. This cannot be done for some large
  modules. For example, the encodings has a lot of submodules, only a
  few are needed.


STATUS
======

* Python API:

  - new function.__modified__ and type.__modified__ properties: False by
    default, becomes True when the object is modified
  - new module.is_modified() method
  - new module.set_initialized() method

* C API:

  - PyDictObject: new "int ma_readonly;" field
  - PyTypeObject: a new "int tp_modified;" field
  - PyFunctionObject: new "int func_module;" and "int func_initialized;" fields
  - PyModuleObject: new "int md_initialized;" field


Modified modules, classes and functions
=======================================

* It's common to modify the following attributes of the sys module:

  - sys.ps1, sys.ps2
  - sys.stdin, sys.stdout, sys.stderr

* "import encodings.latin_1" sets "latin_1" attribute in the namespace of the
  "encodings" module.

* The interactive interpreter sets the "_" variable in builtins.

* warnings: global variable __warningregistry__ set in modules

* functools.wraps() modifies the wrapper to copy attributes of the wrapped
  function


TODO
====

* builtins modified in initstdio(): builtins.open modified
* sys modified in initstdio(): sys.__stdin__ modified
* structseq: types are created modified; same issue with _ast types (Python-ast.c)
* module, type and function __dict__:

  - Drop dict.setreadonly()
  - Decide if it's better to use dict.setreadonly() or a new subclass
    (ex: "dict_maybe_readonly" or "namespace").
  - Read only dict: add a new ReadOnlyError instead of ValueError?
  - sysmodule.c: PyDict_DelItemString(FlagsType.tp_dict, "__new__") doesn't mark
    FlagsType as modified
  - Getting func.__dict__ / module.__dict__ marks the function/module as
    modified, this is wrong.  Use instead a mapping marking the function as
    modified when the mapping is modified.
  - module.__dict__ is read-only: similar issue for functions.

* Import submodule. Example: "import encodings.utf_8" modifies "encoding"
  to set a new utf_8 attribute


TODO: Specialized functions
===========================

Environment
-----------

* module and type attribute values:

  - ("module", "os", OS_CHECKSUM)
  - ("attribute", "os.path")
  - ("module", "path", PATH_CHECKSUM)
  - ("attribute", "path.isabs")
  - ("function", "path.isabs")

* function attributes
* set of function parameter types (passed as indexed or keyword arguments)

Read-only state
---------------

Scenario:

* 1: application.py is compiled. Function A depends on os.path.isabs,
  function B depends on project.DEBUG
* 2: application is started, "import os.path"
* 3: os.path.isabs is modified
* 4: optimized application.py is loaded
* 5: project.DEBUG is modified

When the function is created, os.path.isabs was already modified compared
to the OS_CHECKSUM.

Example of environments
-----------------------

* The function calls "os.path.isabs":

  - rely on "os.path" attribute
  - rely on "os.path.isabs" attribute
  - rely on "os.path.isabs" function attributes (except __doc__)

* The function "def mysum(x, y):" has two parameters

  - x type is int and y type is int
  - or: x type is str and y type is str
  - ("type is": check the exact type, not a subclass)

* The function uses "project.DEBUG" constant

  - rely on "project.DEBUG" attribute

Content of a function
---------------------

* classic attributes: doc, etc.
* multiple versions of the code:

  - required environment of the code
  - bytecode

Create a function
-----------------

* build the environment
* register on module, type and functions modification

Callback when then environment is modified
------------------------------------------

xxx

Call a function
---------------

xxx


LINKS
=====

* http://legacy.python.org/dev/peps/pep-0351/ : Get an immutable copy of
  arbitrary objects
* http://legacy.python.org/dev/peps/pep-0416/ : add a new frozendict type
  => types.MappingProxy added to Python 3.3
