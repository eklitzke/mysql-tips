What Every Developer Needs To Know About MySQL
==============================================

These are some things that every developer should know about working with
MySQL. It principally covers two topics:

 * transactions and transaction isolation levels
 * the distinction between connections and cursors

Transactions & Transaction Isolation Levels
===========================================

Transactions are possibly the most misunderstood thing by new (and even
intermediate) MySQL users. MySQL operates under a model known as
[MVCC](http://en.wikipedia.org/wiki/Multiversion_concurrency_control). The way
that MVCC works is that multiple "transactions" can be open (on multiple
connections), and these transactions can have views of the database that are
isolated from each other. Consider the following example; the identifier on the
left of each line will be an identifier to distinguish transactions, so T1 is
one transaction, and T2 is a separate transaction.

    (T1) SELECT * FROM user WHERE id = 1
    (T2) DELETE FROM user WHERE id = 1
    (T2) DELETE FROM comment WHERE user_id = 1
    (T1) SELECT * FROM comment WHERE user_id = 1

For instance, the code specified by T1 might be code that is run when a user's
page is being rendered on a social networking site. The code specified by T2
might be code that executes when a user account is deleted.

In a DBMS without support for transactions, the first query would select a row
for the user; but the final query would select no rows, since all of the
comments had been deleted by T2.

In a DBMS with support for transactions, the first query will select the row for
a user like before. The final query will select all of that users comments (even
though T2 allegedly deleted those rows).

How does this work? When a transaction starts, the database makes a note of what
the current state of the database is. As long as the transaction is help open,
the database notes changes that are made to the database. It then ensures that
when it does queries on the behalf of the open transaction, it doesn't allow the
transaction to observe any side effects (inserts, deletes, updates) made by
other transactions. Typically this is implemented using an object versioning
system (although the implementation details don't really matter).

Transaction Isolation Levels
----------------------------

MySQL's InnoDB engine supports four different transaction isolation
levels. These are the four isolation levels specified by the SQL standard. They
are as follows (in order from least isolated and cheapest, to most isolated and
most expensive):

 * READ UNCOMMITTED -- in theory this means that you can observe data written to
   by other transactions that haven't yet committed. AFAIK this isn't actually
   implemented as such in MySQL, or any other system. This is equivalent to
   operating in a system that doesn't have transaction support.
 
 * READ COMMITTED -- this means that when reading data in the database, you'll
   always get the most recent copy of the data. In the example, suppose that T2
   committed after running the second delete statement. If T1 was operating in
   READ COMMITTED mode it wouldn't fetch any rows in its final query.
 
 * READ REPEATABLE -- this is the **default transaction isolation level in
   MySQL**. In this isolation level, each transaction is almost completely
   isolated from other transactions. Each transaction will not observe any
   changes made by any other transactions.
 
 * SERIALIZABLE -- this is the same as read repeatable, except all transactions
   logically occur as if they happened one after the other. This normally
   results in the same behavior as read repeatable, except during rare
   situations, which are probably usually code mistakes. See below for an
   example of how this can happen.

Common Mistakes
---------------

If you've altered data in the database, but aren't seeing any changes, several
things may have happened:

 * (common) the transaction with the changes never committed its changes,
   and therefore the second connection can't observe its changes.
 
 * (common) the transaction reading the changes never committed or rolled back,
   so it can't observe any common changes; if this is a connection that is just
   reading, try rolling back.

 * (rarely) you swear you committed the transaction, but another connection
   still can't observe the change; probably your framework/library thought it
   was alright to automagically rollback at some point, and the commit you think
   you did didn't actually commit any changes.

So the rule of thumb here is to always commit and rollback as appropriate.

One other thing to note is that each open transaction incurs some state/overhead
on the MySQL daemon, and this state/overhead increases as changes are made to
the database. So if you have a transaction that's held open for a long time (say
a long-running batch process did a select at the beginning, but never rolled
back) then this will incur a lot of overhead in MySQL. So always attempt to
rollback or commit whenever you're about to do an expensive operation (assuming
its OK to do so).

Advanced Things
---------------

Here's an example of how SERIALIZABLE might be different from READ REPEATABLE. A
good rule of thumb is that if you encounter problems like this the code is
probably structured poorly, and can be fixed to work in the READ REPEATABKE
level instead; but I'll assume you don't care about that.  Suppose there are two
tables, `user` and `user_stat`, where `user_stat` holds some denormalized data
about users (such as their comment count). Consider the following code:

    def insert_comment(conn, user_id, comment_text):
        c = conn.cursor()
        c.execute('SELECT COUNT(*) FROM comment WHERE user_id = %s' % (user_id,))
        count, = c.fetchone()
        c.execute('INSERT INTO comment (user_id, msg) VALUES (%s, %s)' % (user_id, comment_text)
        c.execute('UPDATE user_stat SET comment_count = %s WHERE user_id = %s' % (count + 1, user_id))
        conn.commit()

This code is generally awkward and strange, but it's also **broken** under READ
REPEATABLE. Imagine that the user starts out with 10 comments, and submits two
new comments one right after the other. Each copy of the `insert_comment`
function will observe 10 comments after selecting the comment count. It will
then update the `user_stat` to hold the value of 11. Since each transaction does
this, the final value in `user_stat` will be 11.

In the SERIALIZABLE transaction isolation level, the two copies of the function
will effectively run one after the other (instead of simultaneously) which
circumvents the problem.

N.B. Because of how locking works in InnoDB, if you reorder the SELECT and
INSERT, InnoDB locking will actually make this example work (because the INSERT
will get an X-lock). But this is an implementation detail, and isn't generally
true of these isolation levels.

Cursors Vs. Connections
=======================

A frequent source of confusion is what the difference is between a cursor and a
connection. In the database world, there are typically three kinds of relevant
things here:

 * Connections
 * Client side cursors (what you usually think of when seeing "cursor")
 * [Server side cursors](http://dev.mysql.com/doc/refman/5.0/en/cursors.html)

Connections
-----------

A connection is the most basic and most important thing. It represents an actual
connection (e.g. over a TCP socket) to an intance of MySQL. All
transaction-related things are local to *connections*, not cursors.

Server Side Cursors
-------------------

Cursors are most easily explained by first explaining what a server side cursor
is, although they're not commonly used in MySQL. A server side cursor is an
abstraction that MySQL (and other DBMS's) gives you to let you iterate over
results lazily.

Consider the following pseudo-code, for a social networking site. The intention
of the code is to select 20 comments from a particular user. However, we may not
want to show some comments (if the user "deleted" the comment, if the comment
has a low score, etc.). The criteria to determine this is too complex to encode
in a SQL query and needs to be done in the application logic or in a stored
procedure. We might have some pseudo-code like this (this would work the same
way if a stored procedure was being used):

    comments = []
    results = cursor.execute('SELECT * FROM comments WHERE user_id = %s', (user_id,))
    for row in cursor.results:
        if should_show_comment(row):
            comments.append(row)
            if len(comments) >= 20:
                break

The loop here will probably terminate after about 20 rows, assuming that most
comments are valid. However, the number of rows that need to be selected isn't
known up front, and any given user could have thousands of rows in the comment
table. It would be nice if the MySQL instance could only fetch results as they
are used, so that if only a few rows are used, the MySQL daemon won't need to do
all of the work to fetch those thousands of rows. This is exactly what a server
side cursor does -- it fetches rows from queries on demand.

Another common use for cursors is to allow code to iterate over two queries
independently:

    cursor_one.execute('SELECT * FROM user')
    for user_row in cursor_one.results:
        cursor_two.execute('SELECT * FROM review WHERE review.user_id = %s', (user_row.id,))
        for review_row in cursor_two.results:
            some_function(user_row, review_row)

This example is a bit contrived, but it illustrates an important principle:
cursors are used to encapsulate the results returned by particular queries. If
you execute two or more queries on a single connection, cursors provide an
abstraction to iterate over the results independently.

Client Side Cursors
-------------------

Client side cursors are what you're probably used to working with if you've any
of the Python database modules (in particular, the MySQLdb module and the SQLite
modules). In these modules, calling `.cursor()` on a connection object returns a
client side cursor.

A client side cursor is just an attempt to replicate the server side cursor
abstraction, without creating a real server side cursor. This is useful for a
few reasons:

 * Not all databases support the server side cursor abstraction (e.g. SQLite,
   where there isn't even a distinction between the client and the server)
 * In some databases, particularly MySQL, it may be awkward or expensive to work
   with sever side cursors. In MySQL the server side cursor usage is a bit
   obtuse, and it has some strange side effects (e.g. server side cursors cause
   MEMORY, and ocassionally even MyISAM tables to be created on the remote
   host).

The way a client side cursor is implemented is that when you send a query to the
DBMS, the DBMS sends the full result set back to the client (i.e. your
application). The cursor internally stores the full set of results in a list,
and then you can access the resulting rows using various methods (e.g. using
`.fetchone()` or `.fetchall()`.

Caveats
-------

It's a common mistake to try to rollback or commit on a cursor. You can't do
that. You can only rollback or commit on a connection. In a loose sense, this
will effectively "rollback" or "commit" all cursors created on the
connection. So DO NOT write code like this:

    c1 = conn.cursor()
    c2 = conn.cursor()
    c1.execute('DELETE * FROM user')
    c2.execute('UPDATE user SET first_name = %s WHERE id = %s', ('evan', 1))
    
    # oops, didn't really want to delete all of those users!
    conn.rollback()
    
    # ok, now I want to commit that update
    conn.commit()

What would happen here is that the rollback will have rolled back the update
statement that was run on `c2`, and therefore the final commit won't be
commiting anything at all.

Normally code that does this will be more subtle, but you should be aware
nonetheless.
