# dwstalk

A data web service, lightning talk.

> Leipzig Gophers #23, 2021-11-23, 19:00 CET

# A data web service

A few notes on KISS, sqlite and Go (and [citation graphs](https://arxiv.org/abs/2110.06595)).

* starting point was a project that should result in an API (queried by a
  frontend at [SLUB DD](https://www.slub-dresden.de/))
* fusing metadata from 200M documents (index) with graph data (citations, about 1B edges)

Outline and idea for a batch processing system existed, but we thought:

* Can we fuse data on-the-fly when it is requested?

## Initial Import

The graph data looks like this - two columns, like an simple edge list:

    a b
    a c
    c d
    d e

Or for real:

    10.1016/j.trf.2016.11.009       10.1080/15389588.2014.1003818
    10.1016/j.ttbdis.2015.11.004    10.1186/1756-3305-5-194
    10.1016/j.urology.2009.04.090   10.1097/01.ju.0000180413.98752.a1
    10.1016/j.vetimm.2011.06.014    10.1038/nbt.1496
    10.1016/j.visres.2004.12.017    10.1002/1521-1878(200008)22:8<685::aid-bies1>3.0.co;2-c
    10.1016/j.wear.2016.01.004      10.1016/j.phpro.2015.11.048
    10.1016/j.ydbio.2010.11.017     10.1016/s0002-9440(10)63336-6
    10.1016/j.ygcen.2020.113526     10.1637/11097-042015-reg.1
    10.1016/j.ympev.2006.11.027     10.1111/j.0300-3256.2004.00160.x
    10.1016/s0002-8703(05)80031-6   10.1161/01.cir.79.4.756

There are 1,186,958,897 rows, 10GB compressed, 57GB uncompressed - we'll need
simple queries only, akin to a key value store, e.g. getting all value for a
key or all keys for a value.

## The most deployed database

First idea was to keep it simple and to load the TSV into
[sqlite](https://sqlite.org/index.html) - the [most
deployed](https://www.sqlite.org/mostdeployed.html) database on the planet.

A lesser known fact about sqlite3:

> [sqlite](https://www.sqlite.org/locrsf.html) is a Recommended Storage Format
  for datasets according to the US Library of Congress (beside CSV, XML, JSON, etc.)

## Import data

* use `.mode tabs`
* using `.import` [command](https://sqlite.org/cli.html#importing_csv_files)

On first try, this worked well on a smaller dataset, but failed soon on the
bigger file; seemed that data would be loaded into memory first - and sqlite
got killed.

## Chop it up

* decided against implementing specific importing code through library

Minimal tool that:

* takes a two-column file
* breaks it up into smaller chunks (e.g. 64MB)
* spawns a sqlite3 call for each chunk and use `.import`

## Performance

Tweak various [pragma](https://www.sqlite.org/pragma.html):

```
PRAGMA journal_mode = OFF;
PRAGMA synchronous = 0;
PRAGMA cache_size = %d;
PRAGMA locking_mode = EXCLUSIVE;
```

> The `OFF journaling mode` disables the rollback journal completely. No rollback
> journal is ever created ...  On the other hand, commits can be orders of
> magnitude faster with `synchronous` OFF.

> `cache_size` ... maximum number of database disk pages that SQLite will hold
> in memory at once per open database file.

> There are three reasons to set the `locking-mode to EXCLUSIVE`.  ... The number
> of system calls for filesystem operations is reduced, possibly resulting in a
> small performance increase.

For query performance we'll need indices, too.

## Putting it all together

* minimal tool: [makta](https://github.com/miku/makta), turns (two column) TSV into an sqlite3 database

Example usage, 10M rows, 548MB file.

```
$ head fixtures/sample-10m.tsv # 548MB file
10.1001/amaguidesnewsletters.1996.novdec01      10.1016/s0363-5023(05)80265-5
10.1001/amaguidesnewsletters.1996.novdec01      10.1001/jama.1994.03510440069036
10.1001/amaguidesnewsletters.1997.julaug01      10.1097/00007632-199612150-00003
10.1001/amaguidesnewsletters.1997.mayjun01      10.1164/ajrccm/147.4.1056
10.1001/amaguidesnewsletters.1997.mayjun01      10.1136/thx.38.10.760
10.1001/amaguidesnewsletters.1997.mayjun01      10.1056/nejm199507133330207
10.1001/amaguidesnewsletters.1997.mayjun01      10.1378/chest.88.3.376
```

Create key-value database with both key and value indexed.

```
$ makta -version
makta 6dfae3b 2021-11-22T15:05:49Z

$ time makta -o data.db < fixtures/sample-10m.tsv
2021/11/22 16:06:57 [ok] initialized database · data.db
2021/11/22 16:07:11 [io] written 523M · 36.9M/s
2021/11/22 16:07:17 [ok] 1/2 created index · data.db
2021/11/22 16:07:35 [ok] 2/2 created index · data.db

real    0m38.339s
user    0m35.431s
sys     0m5.664s
```

Timings:

|               | elapsed (s)  | cumulative (s)    | rows/s    |
|-----------    |------------  |-----------------  |--------   |
| import        | 14           | 14                | 714,285   |
| index 1/2     | 6            | 20                | 500,000   |
| index 2/2     | 18           | 38                | 263,157   |
| total         | 38           | 38                | 263,157   |




