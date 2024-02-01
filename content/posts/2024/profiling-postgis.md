---
title: "Profiling PostGIS with PLProfiler"
date: 2024-02-01T19:40:29+13:00
draft: false
summary: "PLProfiler is a great tool for profiling PL/PGSQL functions, but you need an extra trick for PostGIS native functions."
tags: ['PostgreSQL', 'PLProfiler', 'PostGIS', 'Database', 'Performance', 'Software Development']
---

When it comes to optimising code, you really want to have some actual data to guide your efforts. I've been working with PostgreSQL functions recently, trying to figure why a complex set of GIS-related functions was not scaling as we would like.

PostgreSQL has a number of different language extentions that stored procedures/functions can be coded in. This includes:

- native code, which is what PostGIS typically uses, as most of the GIS calculations are done using the GEOS library, which is written in C;
- 'plpgsql' (PL/PGSQL), which is probably the most common form of user-defined function;
- 'sql' for when you don't need multiple statements,
- and more (even including the likes of the v8 Javascript engine)

A PL/PGSQL function, with a little GIS flavouring in its input, might look something like the following:

```sql
CREATE OR REPLACE FUNCTION my_function(
    geoms geometry[])
RETURNS type AS
$$
DECLARE
    local_var1 type;
    ...
BEGIN
    local_var1 := ...;
    FOR i IN array_lower(geoms, 1) .. array_upper(geoms, 1)
    LOOP

        raise info 'Processing geometry % of %', i, array_upper(geoms, 1);

        ...
    END LOOP;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```

I'll leave it to the reader to imagine a lot more looping and conditions that could lead to difficulties in performance management.

PostgreSQL comes with no profiler support for this code... tools like slow query logging are much too coarse. So what is a developer to do when you need to figure where time is being spent?

- add lots of debug logging (eg. `raise info ...` etc.)
- or find something better

## Debugging Statements

Adding debug/info statements _can_ be useful, although it can be challenging to process, and certainly challenging to quickly enable or disable. Not something you would want to do in a production environment.

It's feels a bit clunky... better as a debugging aid instead of profiling. But it can be very useful if you don't mind the resulting text-processing to process timestamps and then report on your findings, such as an application exhibiting O(N<sup>2</sup>) time complexity.

Debugging statements don't require any additional componentry, but will require recreating functions etc.

Now let's find something better: plprofiler

## PLProfiler

(Note: when I was working this out, I was constrained to a much older version of PostgreSQL, so my version choices reflect a replease that still support that version; PLProfiler version 3.5. The current releases are 4.x)

Here's a useful presentation about plprofiler; just note that plprofiler has since moved.

- [plprofiler slide-deck from 2016](https://wiki.postgresql.org/images/1/16/Plprofiler_2016-11-03.pdf)
- [plprofiler repository](https://github.com/bigsql/plprofiler)

### Installing PLProfiler

Let's pretend I've taken a time-machine back to the mid 2010s, and I have Postgres 9.4 with Python 2. Which is great, because the reporting components for plprofiler 3.5 is coded in Python 2.

I'm going to assume that you have PostgreSQL already installed on a server, and are not using a cloud-hosted database. This is the blog post that was most helpful; just do the necessary rewrites for your particular Postgres version.

- [great install blog post](https://www.percona.com/blog/plprofiler-getting-a-handy-tool-for-profiling-your-pl-pgsql-code/)

#### Install Gotchas

When you clone the repository, be sure to checkout the appropriate release branch.

When it comes time to install the python-plprofiler code, note that the `setup.py` will need to be modified. This is because it references `pychopg2` as a dependency without specifying a version constraint, so even if you have `pychopg2` installed and usable in your virtualenv etc, you may find that `python setup.py install` will download the latest at the time (eg. 2.9.9 I think), and will fail because that version is now Python 3. I edited `setup.py` to include a dependency pin... removing the dependency would also work.

Don't neglect setting up your credentials in ~/.pgpass. You may need to specify database, host and username for authentication to work. This will become more relevant later.

## Your First PLProfiler attempt

Here's the very basic two-step approach to using the 'monitor' mode to profile the entire database, rather than giving plprofiler a command to profile.

(Hint: you might want to consider whether you need any other parts of the applicable running at the time, such as background processing)

Here I'm going to start monitoring for 300s (the activity I wish to profile takes a bit less)

```bash
plprofiler monitor -d mydb --duration 300
```

(Note: if using the `run` mode of operation, you may need to be more more careful to specify the hostname and user-name; this will be relevant later when we play with the `search_path`)

In this older version of plprofiler I'm using, it won't gracefully handle a ^C; perhaps current versions will.

Now run your workload; in my case this was an API request, but you might have some integration test-suite to run etc. Ideally something reproducible.

When that has completed, create a report. You can 'save' the dataset, but you can also take the shortcut and just report on the current shared dataset.

```bash
plprofiler report -d mydb --from-shared --output ./before.html \
  --title BUG-1234 --desc 'Profiling before optimisation'
```

Now open the HTML report in a browser. You'll see a useful flame-graph.... Ah, but when using PLProfiler with a PostGIS workload, you will initially be underwhelmed, because you won't see much at all. To give you an idea of what I was seeing:

- the function I was interested in made a lot of use of functions such as st_difference, st_area, st_intersects, st_union, st_transform, etc.
- all I saw in the first profiling attempt was st_area, and some other background activity, which didn't stack up.

The problem is that plprofiler works by adding instrumentation to plpgsql functions... but st_difference and friends are not plpgsql but native ('c') functions.

(There was a plpgsql function wrapper for st_area, which is why it showed up.)

## Making PostGIS Visible to PLProfiler

(This is what this post is really about)

What we need are some reasonably trivial wrapper functions that are coded in plplsql and call the 'internal' (native code) versions.

But we'd like to make a fairly minimal change, ideally that we can swap in and out easily.

### Changing search_path

For this, I created a new schema (I called it 'profiling') and will put all my wrapper functions in there. Each wrapper function will have the same name and type. To steer my application to these functions it will use the `search_path`.

Now is a good time to point you to a useful documentation about the complexities of `search_path`, because it's one of those configuration elements that can be set different places, and that has impact on what needs to be restarted and what is affected by it.

- [Schema and search_path surprises](https://www.postgresonline.com/article_pfriendly/279.html#:~:text=The%20PostgreSQL%20search_path%20variable%20allows,it%20to%20your%20schema%20search_path)

(Read that? Okay, let's continue)

I search the DDL statements for my application and noticed that it already sets `search_path` at the level of the particular users that need to see the `postgis` and other schemas, so I followed suit, adding my new `profiling` schema reasonably early:

```sql
ALTER ROLE app-user IN DATABASE mydb SET search_path = public, profiling, postgis;
```

Time for a bit of a sanity-check. Make sure ~/.pgpass allows access, and then try the following; not to do any profiling, but just to make sure that we know who we logged in as and what the search_path was.

```bash
plprofiler run -d mydb --host localhost --user app-user --command "select current_user; show search_path;" --name meh --title meh --description meh
```

Restart the application(s) that have a session open; otherwise they won't see the change to the search path.

### Now we just need some functions

(The manual way... the _smart_ way would be autogenerate it from PostGIS source code... but I'm not that invested in going down that rabbit hole; an hour of busy work will do just fine.)

Use `psql` to list functions and show their definitions.

Tips:

- [`\a`](https://www.postgresql.org/docs/current/app-psql.html#:~:text=If%20the%20current%20table%20output%20format%20is%20unaligned%2C%20it%20is%20switched%20to%20aligned.) and [`\x`](https://www.postgresql.org/docs/current/app-psql.html#:~:text=Sets%20or%20toggles%20expanded%20table%20formatting%20mode) are your friends if you haven't met them yet.
- `\df postgis.st_area`  will list functions; the documentation refers to these as regular expressions, although it seems to use glob expressions.
- `\df+` will include the comment
- Beware some IDEs; my older version of PyCharm was giving my documentation for a different function override.
- Beware 'geography' versus 'geometry' (trap for GIS newbies)
- `\sf postgis.st_area(postgis.geometry)` will output the source code for a function. When overrides are present you will need to specify the parameter types (not including any parameter names)

```
mydb=# \df+ postgis.st_area
List of functions

Schema|postgis
Name|st_area
Result data type|double precision
Argument data types|geometry
Type|normal
Security|invoker
Volatility|immutable
Owner|vagrant
Language|c
Source code|area
Description|args: g1 - Returns the area of the surface if it is a polygon or multi-polygon. For "geometry" type area is in SRID units. For "geography" area is in square meters.

mydb=# \sf postgis.st_area(postgis.geometry)
CREATE OR REPLACE FUNCTION postgis.st_area(geometry)
RETURNS double precision
LANGUAGE c
IMMUTABLE STRICT
AS '$libdir/postgis-2.1', $function$area$function$
```

So here's the general set of transformations:

- Our function will exist explicitly in the `profiling` schema
- The language will change from 'c' to 'plpgsql'
- You'll need to give the parameters names so you can refer to them in the body.
- I like to be explicit when creating such wrappers in this case when referring to anything in the `postgis` schema. **You MUST NOT refer to an unqualified `st_area` in this function; only `postgis.st_area`, or you will end up with unbounded recursion.
- I like to update the function comment also... but this is optional.
- Don't forget the ownership statement.
- The body of the function will be replaced.

Here's my wrapper for `st_area`

```sql
----- st_area(geometry)

CREATE OR REPLACE FUNCTION profiling.st_area(g1 postgis.geometry)
RETURNS double precision
LANGUAGE plpgsql
IMMUTABLE STRICT
AS
$$
BEGIN
  RETURN postgis.st_area(g1);
END;
$$;

COMMENT ON FUNCTION profiling.st_area(g1 postgis.geometry) IS 'args: g1 - [PROFILING] Returns the area of the surface if it is a polygon or multi-polygon. For "geometry" type area is in SRID units. For "geography" area is in square meters.';

ALTER FUNCTION profiling.st_area(g1 postgis.geometry) OWNER IS postgres;
```

With that loaded in you should now be able to `profiling.st_area` in your new profiling results.

Add wrappers for at least all the significant PostGIS functions you use; time spent in native functions does not show up at all otherwise.

I would suggest putting all of this on a couple of SQL files; one to uninstall (at least change the search_path back), and one to install. Beware that some of this could get you in trouble with any database migration scripts (eg. from SQLAlchemy) because of plprofiler dependencies; but this is not hard to deal with.

### More functions!

I found that (surprise!) `st_difference` accounted for about half of my processing time. Turns out that `st_difference` is computationaly expensive. I have three call-sites for `st_difference`, and the report doesn't tell me which, so I created three more wrappers:

- st_difference_A
- st_difference_B
- st_difference_C

And updated my function to use st_difference_A at the first call-site etc.

Now I know that st_difference_C is where I should be focussing my efforts on, then st_difference_B, and I can ignore st_difference_A.

I love good visibility, don't you?

# Summary

- Debugging statements (and their timestamps) can be a useful starting point, but for the purposes of profiling will be quite painful.
- Profiling PostGIS workloads is achievable, but you need to create some wrapper functions in plpgsql to make it visible to plprofiler.
- If you can reproduce the problem in a non-Production environment, this is an excellent way to gain visibility.
- You could also do this in a Production environment if you were suitably desperate, but you should probably avoid using 'monitor' mode and instead use the 'run' mode to reduce the scope of what you are measuring.
- It would be super if PostGIS (or some other tooling) could provide a ready-made set of wrappers for this purpose.
- Individual call-sites (eg. multiple calls to st_difference in the same function) become tangled, but we can overcome that with call-site specific wrappers.
