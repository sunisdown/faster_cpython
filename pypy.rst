+++++++++++++++++++++++
Random notes about PyPy
+++++++++++++++++++++++

What is the problem with PyPy?
==============================

PyPy is fast, much faster than CPython, but it's still not widely used by
users. What is the problem? Or what are the problems?

* Bad support of the :ref:`Python C API <c-api>`: PyPy was written from scratch
  and uses different memory structures for objects. The cpyext module emulates
  the Python C API but it's slow.

* New features are first developped in CPython. In january 2016, PyPy only
  supports Python 2.7 and 3.2, whereas CPython is at the version 3.5. It's hard
  to have a single code base for Python 2.7 and 3.2, Python 3.3 reintroduced
  ``u'...'`` syntax for example.

* Not all modules are compatible with PyPy: see `PyPy Compatibility Wiki
  <https://bitbucket.org/pypy/compatibility/wiki/Home>`_. For example, numpy
  is not compatible with PyPy, but there is a project
  under development: `pypy/numy <https://bitbucket.org/pypy/numpy>`_. PyGTK,
  PyQt, PySide and wxPython libraries are not compatible with PyPy; these
  libraries heavily depend on the Python C API.  GUI applications (ex: gajim,
  meld) using these libraries don't work on PyPy :-( Hopefully, a lot of
  popular modules are compatible with PyPy (ex: Django, Twisted).

* PyPy is slower than CPython on some use cases. Django: "PyPy runs the
  templating engine faster than CPython, but so far DB access is slower for
  some drivers." (`source of the quote
  <https://code.djangoproject.com/wiki/DjangoAndPyPy>`_)

If I understood correctly, Pyjston will have same problems than PyPy since it
doesn't support the Python C API neither. Same issue for Pyjion?
