Tips
****

.. currentmodule:: apsw

These tips are based on mailing list postings, issues, and emails.
You are recommended to read all the documentation as well.

.. _version_stuff:

About Python, APSW, and SQLite versions
=======================================

SQLite has approximately quarterly releases.  These include tweaks,
bug fixes, and new functionality based on the billions of SQLite
databases in use, and the many many programs that use SQLite (eg
almost every browser, mail client, photo library, mobile and desktop
OS).  Despite these changes, SQLite retains backwards and forwards
compatibility with the `file format
<https://www.sqlite.org/onefile.html>`__ and APIs.

APSW wraps the `SQLite C API
<https://www.sqlite.org/c3ref/intro.html>`__.  That means when SQLite
adds new constant or API, then so does APSW.  You can think of APSW as
the Python expression of SQLite's C API.  You can `lookup
<genindex.html#S>`__ SQLite APIs to find which APSW functions and
attributes call them.

Consequently the APSW version mirrors the SQLite version.  You can use
APSW with the corresponding version of SQLite, or any newer version of
SQLite.  You could use the original 2004 release of APSW with today's
SQLite just fine, although it wouldn't know about the new APIs and
constants.

APSW has compatibility with a broad range of Python versions.  This is
so that you can update the SQLite version you use, access new
constants and APIs (if desired), all without having to change your
Python version.

SQLite is different
===================

While SQLite provides a SQL database like many others out there, it is
also unique in many ways.  Read about the unique features at the
`SQLite website <https://sqlite.org/different.html>`__.

.. note::

  SQLite 3 has been available for two decades, improving and adding
  features over time.  Because of strong compatibility guarantees, you
  may need to opt-in to some like `foreign key enforcement
  <https://www.sqlite.org/foreignkeys.html>`__.  It is a good idea to
  review the `pragmas <https://www.sqlite.org/pragma.html#toc>`__ and
  consider using :attr:`apsw.connection_hooks` to configure each
  :class:`Connection`.

Transactions
============

Transactions are the changes applied to a database file as a whole.
They either happen completely, or not at all.  SQLite notes all the changes
made during a transaction, and at the end when you do a commit will cause
them to permanently end up in the database.  If you do not commit, or
just exit, then other/new connections will not see the changes and SQLite
handles tidying up the work in progress automatically.

Committing a transaction can be quite time consuming.  SQLite uses a robust
multi-step process that has to handle errors that can occur at any point,
and asks the operating system to ensure that data is on storage and would
survive a power cycle.  This will `limit the rate at which you can do
transactions <https://www.sqlite.org/faq.html#q19>`__.

If you do nothing, then each statement is a single transaction::

   # this will be 3 separate transactions
   db.execute("INSERT ...")
   db.execute("INSERT ...")
   db.execute("INSERT ...")

You can use BEGIN/END to set the transaction boundary::

   # this will be one transaction
   db.execute("BEGIN")
   db.execute("INSERT ...")
   db.execute("INSERT ...")
   db.execute("INSERT ...")
   db.execute("COMMIT")

However that is extra effort, and also requires error handling.  For example
if the second INSERT failed then you likely want to ROLLBACK the incomplete
transaction, so that additional work on the same connection doesn't see the
partial data.

If you use :meth:`with Connection <Connection.__enter__>` then the transaction
will be automatically started, and committed on success or rolled back if
exceptions occur::

   # this will be one transaction with automatic commit and rollback
   with db:
       db.execute("INSERT ...")
       db.execute("INSERT ...")
       db.execute("INSERT ...")

There are `technical details <https://www.sqlite.org/lang_transaction.html>`__
at the `SQLite site <https://www.sqlite.org/docs.html>`__.

Cursors
=======

SQLite only calculates each result row as you request it.  For example
if your query returns 10 million rows SQLite will not calculate all 10
million up front.  Instead the next row will be calculated as you ask
for it.

Cursors on the same :ref:`Connection <connections>` are not isolated
from each other.  Anything done on one cursor is immediately visible
to all other Cursors on the same connection.  This still applies if
you start transactions.  Connections are isolated from each other.

Cursor objects are obtained by :meth:`Connection.cursor` and are very
cheap.  It is best practise to not re-use them, and instead get a new one
each time.  If you don't, code refactoring and nested loops can unintentionally
use the same cursor object which will not crash but will cause hard to
diagnose behaviour in your program.

Read more about :ref:`Cursors <cursors>`.

Bindings
========

When using a cursor, always use bindings.  `String interpolation
<http://docs.python.org/library/stdtypes.html#string-formatting-operations>`_
may seem more convenient but you will encounter difficulties.  You may
feel that you have complete control over all data accessed but if your
code is at all useful then you will find it being used more and more
widely.  The computer will always be better than you at parsing SQL
and the bad guys have years of experience finding and using `SQL
injection attacks <http://en.wikipedia.org/wiki/SQL_injection>`_ in
ways you never even thought possible.

The :ref:`documentation <cursors>` gives many examples of how to use
various forms of bindings.

.. _diagnostics_tips:

Diagnostics
===========

Both SQLite and APSW provide detailed diagnostic information.  Errors
will be signalled via an :doc:`exception <exceptions>`.

APSW ensures you have :ref:`detailed information
<augmentedstacktraces>` both in the stack trace as well as what data
APSW/SQLite was operating on.

SQLite has a `warning/error logging facility
<http://www.sqlite.org/errlog.html>`__.  You can call
:meth:`apsw.ext.log_sqlite` which installs a handler that forwards
SQLite messages to the `logging module
<https://docs.python.org/3/library/logging.html>`__.

To do it yourself::

    def handler(errcode, message):
        errstr=apsw.mapping_result_codes[errcode & 255]
        print (f"SQLITE_LOG: { message } ({ errcode }) { errstr } "
               + apsw.mapping_extended_result_codes.get(errcode, ""))

    apsw.config(apsw.SQLITE_CONFIG_LOG, handler)

.. note::

   The handler **must** be set before any other calls to SQLite.
   Once SQLite is initialised you cannot change the logger - a
   :exc:`MisuseError` will happen (this restriction is in SQLite not
   APSW).

This is an example of what gets printed when I use ``/dev/null`` as
the database name in the :class:`Connection` and then tried to create
a table.

.. code-block:: output

    SQLITE_LOG: cannot open file at line 28729 of [7dd4968f23] (14) SQLITE_CANTOPEN
    SQLITE_LOG: os_unix.c:28729: (2) open(/dev/null-journal) - No such file or directory (14) SQLITE_CANTOPEN
    SQLITE_LOG: statement aborts at 38: [create table foo(x,y);] unable to open database file (14) SQLITE_CANTOPEN

Managing and updating your schema
=================================

If your program uses SQLite for `data
<https://sqlite.org/appfileformat.html>`__ then you'll need to manage
and update your schema.  The hard way of doing this is to test for the
existence of tables and their columns, and doing that maintenance
programmatically.  The easy way is to use `pragma user_version
<https://sqlite.org/pragma.html#pragma_user_version>`__ as in this example::

  def ensure_schema(db):
    # a new database starts at user_version 0
    if db.pragma("user_version") == 0:
      with db:
        db.execute("""
          CREATE TABLE IF NOT EXISTS foo(x,y,z);
          CREATE TABLE IF NOT EXISTS bar(x,y,z);
          PRAGMA user_version = 1;""")

    if db.pragma("user_version") == 1:
      with db:
        db.execute("""
        CREATE TABLE IF NOT EXISTS baz(x,y,z);
        CREATE INDEX ....
        PRAGMA user_version = 2;""")

    if db.pragma("user_version") == 2:
      with db:
        db.execute("""
        ALTER TABLE .....
        PRAGMA user_version = 3;""")

This approach will automatically upgrade the schema as you expect.
You can also use `pragma application_id
<https://sqlite.org/pragma.html#pragma_application_id>`__ to mark the
database as made by your application.

Parsing SQL
===========

Sometimes you want to know what a particular SQL statement does.  Use
:func:`apsw.ext.query_info` which will provide as much detail as you
need.

Unexpected behaviour
====================

Occasionally you may get different results than you expected.  Before
littering your code with *print*, try :ref:`apswtrace <apswtrace>`
with all options turned on to see exactly what is going on. You can
also use the :ref:`SQLite shell <shell>` to dump the contents of your
database to a text file.  For example you could dump it before and
after a run to see what changed.

One fairly common gotcha is using double quotes instead of single
quotes.  (This wouldn't be a problem if you use bindings!)  SQL
strings use single quotes.  If you use double quotes then it will
mostly appear to work, but they are intended to be used for
identifiers such as column names.  For example if you have a column
named ``a b`` (a space b) then you would need to use::

  SELECT "a b" from table

If you use double quotes and happen to use a string whose contents are
the same as a table, alias, column etc then unexpected results will
occur.

.. _customizing_connection_cursor:

Customizing Connections
=======================

:attr:`apsw.connection_hooks` is a list of callbacks for when
each :class:`Connection` is created.  They are called in turn, with
the new connection as the only parameter.

For example if you wanted to add an `executescript` method to
Connections that is like :meth:`Connection.execute` but ignores all
returned rows::

  def executescript(self, sql, bindings=None):
    for _ in self.execute(sql, bindings):
      pass

  def my_hook(connection):
    connection.executescript = executescript

  apsw.connection_hooks.append(my_hook)

Customizing Cursors
===================

You can customize the behaviour of cursors.  An example would be
wanting a :ref:`rowcount <rowcount>` or batching returned rows.
(These don't make any sense with SQLite but the desire may be to make
the code source compatible with other database drivers).

Set :attr:`Connection.cursor_factory` to any callable, which will be
called with the connection as the only parameter, and return the
object to use as a cursor.

For example instead of returning rows as tuples, we can return them as
dictionaries using a :ref:`row tracer <rowtracer>` with
:meth:`Cursor.getdescription`::

  def dict_row(cursor, row):
    return {k[0]: row[i] for i, k in enumerate(cursor.getdescription())}

  def my_factory(connection):
    cursor = apsw.Cursor(connection)
    cursor.rowtrace = dict_row
    return cursor

  connection.cursor_factory = my_factory


.. _busyhandling:

Busy handling
=============

SQLite uses locks to coordinate access to the database by multiple
connections (within the same process or in a different process).  The
general goal is to have the locks be as lax as possible (allowing
concurrency) and when using more restrictive locks to keep them for as
short a time as possible.  See the `SQLite documentation
<https://sqlite.org/lockingv3.html>`__ for more details.

By default you will get a :exc:`BusyError` if a lock cannot be
acquired.  You can set a :meth:`timeout <Connection.setbusytimeout>`
which will keep retrying or a :meth:`callback
<Connection.setbusyhandler>` where you decide what to do.

Database schema
===============

When starting a new database, it can be quite difficult to decide what
tables and fields to have and how to link them.  The technique used to
design SQL schemas is called `normalization
<http://en.wikipedia.org/wiki/Database_normalization>`_.  The page
also shows common pitfalls if you don't normalize your schema.

.. _sharedcache:

Shared Cache Mode
=================

SQLite supports a `shared cache mode
<https://sqlite.org/sharedcache.html>`__ where multiple connections
to the same database can share a cache instead of having their own.
It is not recommended that you use this mode.

A big issue is that :ref:`busy handling <busyhandling>` is not done
the same way.  The timeouts and handlers are ignored and instead
*SQLITE_LOCKED_SHAREDCACHE* extended error is returned.
Consequently you will have to do your own busy handling.  (`SQLite
ticket
<https://sqlite.org/src/tktview/ebde3f66fc64e21e61ef2854ed1a36dfff884a2f>`__,
:issue:`59`)

The amount of memory and I/O saved is trivial compared to Python's
overall memory and I/O consumption.  You may also need to tune the
shared cache's memory back up to what it would have been with separate
connections to get the same performance.

The shared cache mode is targeted at embedded systems where every
byte of memory and I/O matters.  For example an MP3 player may only
have kilobytes of memory available for SQLite.

.. _wal:

Write Ahead Logging
===================

SQLite 3.7 introduced `write ahead logging
<https://sqlite.org/wal.html>`__ which has several benefits, but
also some drawbacks as the page documents.  WAL mode is off by
default.  In addition to turning it on manually for each database, you
can also turn it on for all opened databases by using
:attr:`connection_hooks`::

  def setwal(db):
      db.execute("pragma journal_mode=wal")
      # custom auto checkpoint interval (use zero to disable)
      db.wal_autocheckpoint(10)

  apsw.connection_hooks.append(setwal)

Note that if wal mode can't be set (eg the database is in memory or
temporary) then the attempt to set wal mode will be ignored.  The
pragma will return the mode in effect.  It is also harmless to call
functions like :meth:`Connection.wal_autocheckpoint` on connections
that are not in wal mode.

If you write your own VFS, then inheriting from an existing VFS that
supports WAL will make your VFS support the extra WAL methods too.
(Your VFS will point directly to the base methods - there is no
indirect call via Python.)
