plpgsql_check
=============

I founded this project, because I wanted to publish the code I wrote in the last two years,
when I tried to write enhanced checking for PostgreSQL upstream. It was not fully
successful - integration into upstream requires some larger plpgsql refactoring - probably
it will not be done in next two years (now is Dec 2013). But written code is fully functional
and can be used in production. So, I created this extension to be available for all
plpgsql developers.

If you like it and if you would to join to development of this extension, register
yourself to [postgresql extension hacking](https://groups.google.com/forum/#!forum/postgresql-extensions-hacking)
google group.

I invite any ideas, patches, bugreports

plpgsql_check is next generation of plpgsql_lint. It allows to check source code by explicit call
<i>plpgsql_check_function</i>.

PostgreSQL 9.2 is required, PostgreSQL 9.3 and 9.4 are supported

The SQL statements inside PL/pgSQL functions are checked by validator for semantic errors. These errors
can be found by plpgsql_check_function:

# Active mode

    postgres=# load 'plpgsql';
    LOAD
    postgres=# CREATE EXTENSION plpgsql_check;
    LOAD
    postgres=# CREATE TABLE t1(a int, b int);
    CREATE TABLE

    postgres=#
    CREATE OR REPLACE FUNCTION public.f1()
    RETURNS void
    LANGUAGE plpgsql
    AS $function$
    DECLARE r record;
    BEGIN
      FOR r IN SELECT * FROM t1
      LOOP
        RAISE NOTICE '%', r.c; -- there is bug - table t1 missing "c" column
      END LOOP;
    END;
    $function$;

    CREATE FUNCTION

    postgres=# select f1(); -- execution doesn't find a bug due to empty table t1
      f1 
     ────
       
     (1 row)

    postgres=# \x
    Expanded display is on.
    postgres=# select * from plpgsql_check_function_tb('f1()');
    ─[ RECORD 1 ]───────────────────────────
    functionid │ f1
    lineno     │ 6
    statement  │ RAISE
    sqlstate   │ 42703
    message    │ record "r" has no field "c"
    detail     │ [null]
    hint       │ [null]
    level      │ error
    position   │ 0
    query      │ [null]

    postgres=# \sf+ f1
        CREATE OR REPLACE FUNCTION public.f1()
         RETURNS void
         LANGUAGE plpgsql
    1       AS $function$
    2       DECLARE r record;
    3       BEGIN
    4         FOR r IN SELECT * FROM t1
    5         LOOP
    6           RAISE NOTICE '%', r.c; -- there is bug - table t1 missing "c" column
    7         END LOOP;
    8       END;
    9       $function$


Function plpgsql_check_function() has two possible formats: text or xml

    select * from plpgsql_check_function('f1()', fatal_errors := false);
                             plpgsql_check_function                         
    ------------------------------------------------------------------------
     error:42703:4:SQL statement:column "c" of relation "t1" does not exist
     Query: update t1 set c = 30
     --                   ^
     error:42P01:7:RAISE:missing FROM-clause entry for table "r"
     Query: SELECT r.c
     --            ^
     error:42601:7:RAISE:too few parameters specified for RAISE
    (7 rows)

    postgres=# select * from plpgsql_check_function('fx()', format:='xml');
                     plpgsql_check_function                     
    ────────────────────────────────────────────────────────────────
     <Function oid="16400">                                        ↵
       <Issue>                                                     ↵
         <Level>error</level>                                      ↵
         <Sqlstate>42P01</Sqlstate>                                ↵
         <Message>relation "foo111" does not exist</Message>       ↵
         <Stmt lineno="3">RETURN</Stmt>                            ↵
         <Query position="23">SELECT (select a from foo111)</Query>↵
       </Issue>                                                    ↵
      </Function>
     (1 row)


## Triggers

When you would to check any trigger, you have to enter a relation that will be
used together with trigger function

    CREATE TABLE bar(a int, b int);

    postgres=# \sf+ foo_trg
        CREATE OR REPLACE FUNCTION public.foo_trg()
             RETURNS trigger
             LANGUAGE plpgsql
    1       AS $function$
    2       BEGIN
    3         NEW.c := NEW.a + NEW.b;
    4         RETURN NEW;
    5       END;
    6       $function$

Missing relation specification

    postgres=# select * from plpgsql_check_function('foo_trg()');
    ERROR:  missing trigger relation
    HINT:  Trigger relation oid must be valid

Correct trigger checking (with specified relation)

    postgres=# select * from plpgsql_check_function('foo_trg()', 'bar');
                     plpgsql_check_function                 
    --------------------------------------------------------
     error:42703:3:assignment:record "new" has no field "c"
    (1 row)

## Mass check

You can use the plpgsql_check_function for mass check functions and mass check
triggers. Please, test following queries:

    -- check all nontrigger plpgsql functions
    SELECT p.oid, p.proname, plpgsql_check_function(p.oid)
       FROM pg_catalog.pg_namespace n
       JOIN pg_catalog.pg_proc p ON pronamespace = n.oid
       JOIN pg_catalog.pg_language l ON p.prolang = l.oid
      WHERE l.lanname = 'plpgsql' AND p.prorettype <> 2279;

or

    -- check all plpgsql functions (functions or trigger functions with defined triggers)
    SELECT
        (pcf).functionid::regprocedure, (pcf).lineno, (pcf).statement,
        (pcf).sqlstate, (pcf).message, (pcf).detail, (pcf).hint, (pcf).level,
        (pcf)."position", (pcf).query, (pcf).context
    FROM
    (
        SELECT
            plpgsql_check_function_tb(pg_proc.oid, COALESCE(pg_trigger.tgrelid, 0)) AS pcf
        FROM pg_proc
        LEFT JOIN pg_trigger
            ON (pg_trigger.tgfoid = pg_proc.oid)
        WHERE
            prolang = (SELECT lang.oid FROM pg_language lang WHERE lang.lanname = 'plpgsql') AND
            pronamespace <> (SELECT nsp.oid FROM pg_namespace nsp WHERE nsp.nspname = 'pg_catalog') AND
            -- ignore unused triggers
            (pg_proc.prorettype <> (SELECT typ.oid FROM pg_type typ WHERE typ.typname = 'trigger') OR
             pg_trigger.tgfoid IS NOT NULL)
        OFFSET 0
    ) ss
    ORDER BY (pcf).functionid::regprocedure::text, (pcf).lineno

# Passive mode

Functions should be checked on start - plpgsql_check module must be loaded.

## Configuration

    plpgsql_check.mode = [ disabled | by_function | fresh_start | every_start ]
    plpgsql_check.fatal_errors = [ yes | no ]

    plpgsql_check.show_nonperformance_warnings = false
    plpgsql_check.show_performance_warnings = false

Default mode is <i>by_function</i>, that means that the enhanced check is done only in
active mode - by <i>plpgsql_check_function</i>.

You can enable passive mode by

    load 'plpgsql';
    load 'plpgsql_check';
    set plpgsql_check.mode = 'every_start';

    SELECT fx(10); -- run functions - function is checked before runtime starts it

# Limits

<i>plpgsql_check</i> should find almost all errors on really static code. When developer use some
PLpgSQL's dynamic features like dynamic SQL or record data type, then false positives are
possible. These should be rare - in well written code - and then the affected function
should be redesigned or plpgsql_check should be disabled for this function.


    CREATE OR REPLACE FUNCTION f1()
    RETURNS void AS $$
    DECLARE r record;
    BEGIN
      FOR r IN EXECUTE 'SELECT * FROM t1'
      LOOP
        RAISE NOTICE '%', r.c;
      END LOOP;
    END;
    $$ LANGUAGE plpgsql SET plpgsql.enable_check TO false;

<i>A usage of plpgsql_check adds a small overhead (in enabled passive mode) and you should use
it only in develop or preprod environments.</i>

## Dynamic SQL

This module doesn't check queries that are assembled in runtime. It is not possible
to identify results of dynamic queries - so <i>plpgsql_check</i> cannot to set correct type to record
variables and cannot to check a dependent SQLs and expressions. Don't use record variable
as target for dynamic queries or disable <i>plpgsql_check</i> for functions that use dynamic
queries.

## Temporary tables

<i>plpgsql_check</i> cannot verify queries over temporary tables that are created in plpgsql's function
runtime. For this use case it is necessary to create a fake temp table or disable <i>plpgsql_check</i> for this
function.

# Compilation

You need a development environment for PostgreSQL extensions:

    make USE_PGXS=1 clean
    make USE_PGXS=1 install

result:

    [pavel@localhost plpgsql_check]$ make USE_PGXS=1 clean
    rm -f plpgsql_check.so   libplpgsql_check.a  libplpgsql_check.pc
    rm -f plpgsql_check.o
    rm -rf results/ regression.diffs regression.out tmp_check/ log/
    [pavel@localhost plpgsql_check]$ make USE_PGXS=1 clean
    rm -f plpgsql_check.so   libplpgsql_check.a  libplpgsql_check.pc
    rm -f plpgsql_check.o
    rm -rf results/ regression.diffs regression.out tmp_check/ log/
    [pavel@localhost plpgsql_check]$ make USE_PGXS=1 all
    clang -O2 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fpic -I/usr/local/pgsql/lib/pgxs/src/makefiles/../../src/pl/plpgsql/src -I. -I./ -I/usr/local/pgsql/include/server -I/usr/local/pgsql/include/internal -D_GNU_SOURCE   -c -o plpgsql_check.o plpgsql_check.c
    clang -O2 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fpic -I/usr/local/pgsql/lib/pgxs/src/makefiles/../../src/pl/plpgsql/src -shared -o plpgsql_check.so plpgsql_check.o -L/usr/local/pgsql/lib -Wl,--as-needed -Wl,-rpath,'/usr/local/pgsql/lib',--enable-new-dtags  
    [pavel@localhost plpgsql_check]$ su root
    Password: *******
    [root@localhost plpgsql_check]# make USE_PGXS=1 install
    /usr/bin/mkdir -p '/usr/local/pgsql/lib'
    /usr/bin/mkdir -p '/usr/local/pgsql/share/extension'
    /usr/bin/mkdir -p '/usr/local/pgsql/share/extension'
    /usr/bin/install -c -m 755  plpgsql_check.so '/usr/local/pgsql/lib/plpgsql_check.so'
    /usr/bin/install -c -m 644 plpgsql_check.control '/usr/local/pgsql/share/extension/'
    /usr/bin/install -c -m 644 plpgsql_check--0.9.sql '/usr/local/pgsql/share/extension/'
    [root@localhost plpgsql_check]# exit
    [pavel@localhost plpgsql_check]$ make USE_PGXS=1 installcheck
    /usr/local/pgsql/lib/pgxs/src/makefiles/../../src/test/regress/pg_regress --inputdir=./ --psqldir='/usr/local/pgsql/bin'    --dbname=pl_regression --load-language=plpgsql --dbname=contrib_regression plpgsql_check_passive plpgsql_check_active plpgsql_check_active-9.5
    (using postmaster on Unix socket, default port)
    ============== dropping database "contrib_regression" ==============
    DROP DATABASE
    ============== creating database "contrib_regression" ==============
    CREATE DATABASE
    ALTER DATABASE
    ============== installing plpgsql                     ==============
    CREATE LANGUAGE
    ============== running regression test queries        ==============
    test plpgsql_check_passive    ... ok
    test plpgsql_check_active     ... ok
    test plpgsql_check_active-9.5 ... ok
    
    =====================
     All 3 tests passed. 
    =====================

Checked on

* gcc on Linux (against all supported PostgreSQL)
* clang 3.4 on Linux (against PostgreSQL 9.5)


# Licence

Copyright (c) Pavel Stehule (pavel.stehule@gmail.com)

 Permission is hereby granted, free of charge, to any person obtaining a copy
 of this software and associated documentation files (the "Software"), to deal
 in the Software without restriction, including without limitation the rights
 to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 copies of the Software, and to permit persons to whom the Software is
 furnished to do so, subject to the following conditions:

 The above copyright notice and this permission notice shall be included in
 all copies or substantial portions of the Software.

 THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 THE SOFTWARE.

# Note

If you like it, send a postcard to address

    Pavel Stehule
    Skalice 12
    256 01 Benesov u Prahy
    Czech Republic


I invite any questions, comments, bug reports, patches on mail address pavel.stehule@gmail.com
