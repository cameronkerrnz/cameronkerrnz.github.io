---
title: "Where Did That Sqlalchemy SQL Originate?"
date: 2025-04-19T11:34:45+12:00
summary: "ORMs such as SQLAlchemy are very useful, but can easily generate very inefficient unrecognisable code. Here's an easy way to log where the query originates in your code."
tags: ['Database', 'SQL', 'SQLAlchemy', 'Python', 'PostgreSQL', 'Programming', 'Software Development', 'SRE', 'logging']
---

I submitted a PR the other day, and found that when it hit the Test environment, my code was taking 4 seconds to complete a request... much longer than 200ms expected. Actually, 4 seconds could be considered the minimal amount of time it might take. Odd, but certainly not unheard of.

We have SQLAlchemy logging the queries (and arguments) that it issues in Dev, so I repeated the operation in Dev. Looking closely at the log timestamps, I eventually found a SQL statement that could be described as:

- joining several (about 5) tables
- seemed to retrieve all the columns it knew of (not `SELECT *`, but equivalent)
- some of the tables are known to have many columns, and some of those columns contain JSON

```
2025-04-16 14:23:26,126 INFO sqlalchemy.engine.base.Engine \
  SELECT lots of columns FROM lots of tables ...
```

Having harvested the SQL, I then made the equivalent query in our Test environment:

- executing the query did indeed take at least 4 seconds
- `EXPLAIN ANALYSE ...` however took only 270ms!

The query plan that was generated was quite reasonable, so why the great difference?
The difference is that `EXPLAIN ANALYSE` runs the query, **but does not transfer the results**.
In other words, the time to serialise and transmit the results, greatly outweighed the time to calculate the result set.
The key takeaway was that the ORM-generated query was doing essentially a `SELECT lots-of-colums` when it only really needed to `SELECT just-a-few-columns`.

I also validated this result by running my Dev environment against our (read-only) Test database. This was helpful to further check performance the impact of performance changes without having to go through the CI/PR process every time.

## Where in my code does this SQL originate?

Even when you have the generated SQL being logged (in SQLAlchemy this is configured via the [echo engine parameter](https://docs.sqlalchemy.org/en/20/core/engines.html#sqlalchemy.create_engine.params.echo)) it only tells you about the generated SQL, and _not_ the location in your codebase where the OAM functionality was invoked... your code lies somewhere in a higher stack frame.

So to be able to identify our own code, we have to climb the stack, ignoring code that is not our own, until we find the code we might recognise as our own.

I certainly don't want to enable something like stack introspection in a production environment if I don't have to... what if loading the required `inspect` module does something like change how the stack mechanics work in Python? So I wanted a conditional import... but with a global name. Here's what I came up with (with _some_ help from ChatGPT)

```python
# Want 'import inspect', but only when feature-flagged in a Dev environment, so declare the global
# and conditionally assign it later.
inspect = None

def maybe_enable_log_sql_origin(engine):
    global inspect
    if IN_DEV_MODE_AND_OR_FEATURE_FLAGGED:
        import inspect  as _inspect
        # assign it globally so log_sql_origin doesn't need to import all the time
        inspect = _inspect
        event.listen(engine, 'before_cursor_execute', log_sql_origin)

def log_sql_origin(conn, cursor, statement, parameters, context, executemany):
    """Log where in our code any generated SQL originates.

    Should only ever be present in Dev because it
    inspects the stack (side-effects of enabling?).
    """
    global inspect
    if inspect is None:
        return

    # Skip the SQLAlchemy internals and get to your app code
    try:
        stack = inspect.stack()
        for frame, filename, lineno, function, code_context, index in stack:
            if function == "log_sql_origin":
                continue
            if "site-packages" not in filename:  # crude filter to skip library code
                log().info("SQL from {}:{} in {}".format(filename, lineno, function))
                break
    except Exception:
        pass  # oh well, we tried.
```

There is an element of crudeness here in that we need have some way of determining which code is "ours". We have to skip over the frame containing our `log_sql_origin` callback function, then on up through SQLAlchemy etc. until we get back to 'our' code.

In my Dev environment, at least, I can safely make the assumption that if something is inside a `site-packages` folder, then its not 'our' code. That's not true outside of Development, because our code is also packaged... you may need to adjust this part of the code.

So now we have loader and callback functions, and we only need to call the loader. This is achieved via one additional line where you create the database engine for SQLAlchemy:

```python
engine = engine_from_config(...)
maybe_enable_log_sql_origin(engine)  # added
bind_engine(engine)
```

And now your logging will be much more useful, just look for the 'SQL from' line:

```
2025-04-16 14:23:26,126 INFO  [12697-MainThread][....__init__.log_sql_origin:155] \
  SQL from .../comfy_chair.py:157 in something_unexpected
2025-04-16 14:23:26,126 INFO sqlalchemy.engine.base.Engine \
  SELECT lots of columns FROM lots of tables ...
```

## So how useful was this?

My code was of the type that checked a bunch of conditions to issue validation warnings, and so numerous different models were being consulted, but only very few things in each model. Having learned of the slowness I started changing my code to turn my nice readable code from freely using the ORM models into a fairly complex SQL query which would satisfy all the needs of this particular part of the code-base...  at least I had my unit-tests to help me get it right!

I was particularly thankful my senior colleague pressed me to find out the root cause of the slowness, which led me to the technique showcased in this blog post.

Stashing the code I had already started changing, I added this additional logging... and was surprised to find out that the slow-down was not due to my PR code at all, but was instead in a request factory attached to this request handler. That request factory had recently been changed to use eager loading, to reduce the chattiness that ORM code will often cause.

> A **request factory** is something that is attached to a request handler that will do tasks such as making model data available. For example, your request handler might take an Order ID(s) as a parameter; a OrderFactory or OrderListFactory might then make the related Order model instances available to the request handler. Sometimes, this is useful, but they can end up violating separation of concerns. For example, a request factory might also contain authorisation related code that would make it risky to remove.

## Lessons learned

I enjoyed this particular adventure, even though it had to delay the release. It gave me useful experience in using ORMs in general, and SQLAlchemy in particular:

- Always ask yourself if you have the visibility you need around which SQL queries are being executed, and where those queries come from in your code.
- Does your observability expose both execution time _and_ transfer time? Execution time is readily available and usable for slow query logging, but you will likely need to get the transfer time statistics from your database client; your APM solution should be useful there.
- Being able to run Dev code against a **read-only** database in Test can be very productive when validating performance improvements.
- Be on the lookout for the use of `joinedload` without `load_only`: `load_only` is a way to limit which fields will be eagerly loaded, and is particularly useful when querying or joining wide tables, or tables with large columnms (eg. containing the likes of JSON, CLOBs, etc.)
- When using `load_only`, also consider using [`raiseload`](https://docs.sqlalchemy.org/en/20/orm/queryguide/columns.html#orm-queryguide-deferred-raiseload) to raise an exception when a column would otherwise be lazy-loaded; at least temporarily.
