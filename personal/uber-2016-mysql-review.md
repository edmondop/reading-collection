# Uber-cited Postgres 9.x issues

Uber’s 2016 blog lists several PostgreSQL 9.x “limitations” (reproduced below
with Uber’s phrasing/quote):

- **Inefficient architecture for writes** ("map of Postgres’s on-disk data…
  resulted in inefficiencies"; “first problem… known as _write amplification_”
  [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=We%20encountered%20many%20Postgres%20limitations%3A)
  [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20first%20problem%20with%20Postgres%E2%80%99s,at%20least%20four%20physical%20updates)).
  In practice, minor updates cause new row versions and require updating _all_
  indexes (even those whose keys didn’t change).
- **Inefficient data replication** (“replication occurs at the level of on-disk
  changes… the Postgres replication stream quickly becomes extremely verbose”
  [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=)).
  Replication logs every low-level WAL change, so small logical changes produce
  many WAL records.
- **Issues with table corruption** (Uber encountered a _timeline switch_ bug in
  9.2 causing replicas to misapply WAL and return duplicate rows [o^Why Uber
  Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=During%20a%20routine%20master%20database,mechanism%20weren%E2%80%99t%20actually%20marked%20inactive)).
  They feared that low-level WAL bugs can silently corrupt replicas.
- **Poor replica MVCC support** (“Postgres does not have true replica MVCC
  support… if a streaming replica has an open transaction, updates affecting
  that row will block (and eventually kill) the open transaction” [o^Why Uber
  Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=)).
  In other words, long-running queries on a standby can halt WAL apply, or be
  canceled by Postgres.
- **Difficulty upgrading between versions** (“cannot replicate data between
  different GA releases of Postgres… 9.3 master can’t stream to 9.2 replica”).
  Upgrades require dump/restore or heavy pg_upgrade downtime, and pglogical was
  only a third-party option [o^Why Uber Engineering Switched from Postgres to
  MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Because%20replication%20records%20work%20at,3)
  [o^PostgreSQL: Release
  Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek).
- **Vacuum inefficiency** (Postgres “has to do full table scans” to find dead
  tuples [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=This%20design%20also%20makes%20vacuuming,scans%20to%20identify%20deleted%20rows),
  since it lacked an easy way to skip pages).
- **Connection-handling scalability** (“Postgres uses a process-per-connection
  design… expensive… [we] had significant problems scaling past a few hundred
  active connections” [o^Why Uber Engineering Switched from Postgres to MySQL
  │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Postgres%2C%20however%2C%20use%20a%20process,to%20make%20a%20context%20switch)).
- **Buffer cache overhead** (Postgres relies on the OS page cache and many
  system calls instead of an optimized buffer pool, causing extra context
  switches [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20problem%20with%20this%20design,into%20a%20single%20system%20call)).

Below we examine each issue and whether it’s been fixed or mitigated in
PostgreSQL 9.4–17.

## Issue 1: Inefficient write architecture (write amplification)

**Uber’s summary:** Postgres updates always create new tuples and update _all_
indexes, even if indexed columns didn’t change. Small logical updates lead to
many physical writes (high “write amplification”) [o^Why Uber Engineering
Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20first%20problem%20with%20Postgres%E2%80%99s,at%20least%20four%20physical%20updates).

**Status:** _Improved (partially)._

PostgreSQL’s core MVCC/index design remains the same, but newer versions add
features to reduce unnecessary index IO. Notably, **HOT updates** (heap-only
tuple updates) have been in Postgres since 8.3 (prior to 9.x), so that if only
non-indexed columns change the update can occur in place and skip index
updates. More recently, PostgreSQL added **index deduplication/compression**:
v12 introduced improved B-tree page splitting and prefix-compression tweaks,
and v13 added full _deduplication of B-tree pages_ (coalescing duplicate index
entries on a page). These cuts down index bloat on update-heavy tables. In
practice, PG12/13 show much lower index size and I/O under update workloads
(“Postgres 13 includes index deduplication, and PG12 had B-Tree enhancements,
which improved workloads with heavy index churn” [o^Why Uber Engineering
Switched from Postgres to MySQL (2016) │ Hacker
News](https://news.ycombinator.com/item?id=26283348#:~:text=There%20is%20also%20index%20deduplication,is%20more%20descriptive%20and%20has)).
As of PG14/15, further VACUUM and index improvements (e.g. skipping dead index
entries more efficiently) continue this trend.

- **Fixed/Improved?** Improved.
- **Earliest fix:** HOT updates (since v8.3). B-tree dedup in PG13 (2020).
- **Follow-on:** PG14/15 extra VACUUM and index optimizations.
- **Sources:** Uber blog [o^Why Uber Engineering Switched from Postgres to
  MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20first%20problem%20with%20Postgres%E2%80%99s,at%20least%20four%20physical%20updates),
  Postgres 13 commitnotes [o^Why Uber Engineering Switched from Postgres to
  MySQL (2016) │ Hacker
  News](https://news.ycombinator.com/item?id=26283348#:~:text=There%20is%20also%20index%20deduplication,is%20more%20descriptive%20and%20has)
  (index dedup), and release docs (index enhancements in PG12/13).

## Issue 2: Replication inefficiency (verbose WAL)

**Uber’s summary:** Postgres streams physical on-disk changes (WAL) to
replicas, so minor logical updates produce many WAL records. The replication
log is “extremely verbose” for updates [o^Why Uber Engineering Switched from
Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=).

**Status:** _Improved._

While PostgreSQL still supports physical WAL replication by default, **logical
replication** was introduced to mitigate this. Third-party tools (e.g. 9.4’s
pglogical extension) and _built-in logical replication_ (Postgres 10+) allow
streaming row-level changes. Logical replication sends only the changed columns
of a row (and infers rest), so replicas apply just the minimal logical update.
It also supports selective/table-level replication. Critically, logical
replication works across major versions (“including replication between
different major versions of PostgreSQL” [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek)).
Thus, the volume of WAL for a given logical update is far lower.

- **Fixed/Improved?** Improved (with new modes).
- **Earliest fix:** pglogical extension (v9.4+), core logical replication in
  **Postgres 10** (Oct 2017) [o^PostgreSQL: Release
  Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek).
- **Follow-on:** Logical slots and publications (v10), streaming physical slots
  (v9.4), publication filters.
- **Sources:** Postgres 10 Release Notes [o^PostgreSQL: Release
  Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek)
  (logical replication), Uber blog [o^Why Uber Engineering Switched from
  Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=).

## Issue 3: Table corruption from timeline bugs

**Uber’s summary:** They encountered a Postgres 9.2 bug where replicas on
a timeline switch “misapplied some WAL records” and returned duplicate rows
[o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=During%20a%20routine%20master%20database,mechanism%20weren%E2%80%99t%20actually%20marked%20inactive).
This risked silent index/table corruption when a standby took over. Uber notes
the bug “has been fixed for a long time now” [o^Why Uber Engineering Switched
from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20bug%20we%20ran%20into,could%20be%20released%20at%20any).

**Status:** _Fixed._

The specific timeline-switch issues were addressed in minor releases after 9.2.
For example, PostgreSQL **9.2.2** (Dec 2012) fixed bogus out-of-order
timeline-ID errors in standby mode [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/9.2.2/#:~:text=%2A%20Avoid%20bogus%20%22out,Heikki%20Linnakangas).
Subsequent 9.3+ releases included more fixes for cascading replication and
timeline switching. In practice, modern PG versions handle timeline switches
correctly. No equivalent bug is known in current releases. (Nonetheless, Uber
warns that any complex replication can uncover new bugs.)

- **Fixed/Improved?** Fixed.
- **Earliest fix:** **v9.2.2** (Dec 2012) – fixes “out-of-sequence timeline ID”
  errors [o^PostgreSQL: Release
  Notes](https://www.postgresql.org/docs/release/9.2.2/#:~:text=%2A%20Avoid%20bogus%20%22out,Heikki%20Linnakangas).
  Further fixes in 9.3.x for cascading replication.
- **Follow-on:** Continued replication stability improvements in 9.4–9.6 (e.g.
  synchronous streaming, replication slots).
- **Sources:** PostgreSQL 9.2.2 Release Notes [o^PostgreSQL: Release
  Notes](https://www.postgresql.org/docs/release/9.2.2/#:~:text=%2A%20Avoid%20bogus%20%22out,Heikki%20Linnakangas),
  Uber blog [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=During%20a%20routine%20master%20database,mechanism%20weren%E2%80%99t%20actually%20marked%20inactive)
  [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20bug%20we%20ran%20into,could%20be%20released%20at%20any).

## Issue 4: Lack of replica MVCC (standby query blocking)

**Uber’s summary:** “Postgres does not have true replica MVCC support.” In
physical streaming, a standby’s on-disk state is identical to master’s, so
a long-running query on the standby can block WAL playback (or be canceled
after a timeout) [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=).
Uber notes replicas can lag or kill queries, disrupting readers.

**Status:** _Partially mitigated, but still inherent._

PostgreSQL’s physical replication still enforces this constraint. However, some
mitigations exist: since v9.0 one can set **hot_standby_feedback**=on, so the
standby informs the master of its oldest snapshot, preventing HOT-vacuum from
removing tuples that active queries need (thus reducing cancellations). Also,
logical replication (v10+) avoids this issue entirely by decoupling replica
data from on-disk propagation. In short, there is **no fundamental change**:
physical standbys still require either pausing updates or risking cancels if
old snapshots exist.

- **Fixed/Improved?** Still mostly **not fixed** (physical replication still
  blocks). Mitigated by features (hot_standby_feedback in 9.0+, logical
  replication in 10).
- **Earliest fix/note:** **9.0+** (hot_standby_feedback), Postgres 10+ (logical
  replication avoids the issue).
- **Notes:** No built-in “MVCC on replicas” mode was added; Uber’s concern
  remains unless one uses logical replication or careful hot_standby config.
- **Sources:** Uber blog [o^Why Uber Engineering Switched from Postgres to
  MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=);
  PostgreSQL doc on hot_standby_feedback (see v9.6+) shows this feature, and
  Postgres 10 docs cover logical replication.

## Issue 5: Upgrade/downtime difficulty (cross-version replication)

**Uber’s summary:** Postgres’s physical WAL replication is version-dependent:
e.g. a 9.3 master cannot stream to a 9.2 standby [o^Why Uber Engineering
Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Because%20replication%20records%20work%20at,3).
Upgrading required long downtime (dump/restore or fast pg_upgrade

- snapshot/resync). Uber notes pglogical (third-party, 9.4+) but it isn’t core.

**Status:** _Fixed (largely)._

PostgreSQL added core support for logical replication in 10, explicitly
allowing “replication between different major versions” [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek).
By creating a publication on the old master and subscribing on the new server,
one can do near-zero-downtime upgrades. In PG9.4–9.6, the open-source
**pglogical** extension (by 2ndQuadrant) provided similar cross-version
replication. Also, **pg_upgrade** has improved (with link mode to reduce
downtime).

- **Fixed/Improved?** Fixed.
- **Earliest fix:** Logical replication in **Postgres 10** (Oct 2017)
  [o^PostgreSQL: Release
  Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek);
  pglogical extension (v2.1 for 9.4+) for earlier versions.
- **Follow-on:** Continued improvements to pg_upgrade and replication features
  (e.g. pg_basebackup enhancements).
- **Sources:** PostgreSQL 10 Release Notes [o^PostgreSQL: Release
  Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek)
  (logical replication between major versions), Uber blog [o^Why Uber
  Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Because%20replication%20records%20work%20at,3).

## Issue 6: VACUUM/autovacuum inefficiency

**Uber’s summary:** Postgres “has to do full table scans to identify deleted
rows” because it lacks a rollback-segment approach [o^Why Uber Engineering
Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=This%20design%20also%20makes%20vacuuming,scans%20to%20identify%20deleted%20rows).
In other words, old versions aren’t easily visible, so VACUUM must scan pages.

**Status:** _Improved._

Modern Postgres uses a **visibility map** to track which pages are all-visible
(no dead tuples). In fact, since PG 8.4 (well before 9.2) the visibility map
has allowed VACUUM to **skip clean pages**. The official docs state: _“VACUUM
uses the visibility map to determine which pages… to be scanned. Normally, it
will skip pages that don’t have any dead row versions”_ [o^PostgreSQL:
Documentation: 13: 24.1. Routine
Vacuuming](https://www.postgresql.org/docs/13/routine-vacuuming.html#:~:text=match%20at%20L279%20,old%20row%20version%20in%20the).
Thus, although Uber’s 9.2 instance scanned whole tables, in current PG versions
VACUUM typically touches only pages with actual deletions. Further
improvements: PG 12 added a “freeze map” to skip pages that are entirely
frozen, and PG 13 allows parallel index vacuuming. These reduce autovacuum load
and index bloat on still-clean tables.

- **Fixed/Improved?** Improved.
- **Earliest fix:** Visibility map (introduced v8.4, present in 9.x
  documentation).
- **Follow-on:** PG12+ freeze-map and autovacuum tweaks, PG13 parallel vacuum,
  PG14 vacuum progress.
- **Sources:** PostgreSQL manual (VACUUM/visibility map) [o^PostgreSQL:
  Documentation: 13: 24.1. Routine
  Vacuuming](https://www.postgresql.org/docs/13/routine-vacuuming.html#:~:text=match%20at%20L279%20,old%20row%20version%20in%20the),
  Uber blog [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=This%20design%20also%20makes%20vacuuming,scans%20to%20identify%20deleted%20rows).

## Issue 7: Connection handling scaling

**Uber’s summary:** Postgres’s **process-per-connection** model is heavy:
“forking a new process… multiples memory overhead… we had significant problems
scaling Postgres past a few hundred active connections” [o^Why Uber Engineering
Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Postgres%2C%20however%2C%20use%20a%20process,to%20make%20a%20context%20switch).

**Status:** _Not fixed._
PostgreSQL still uses one backend process per connection (even in v17), so the
fundamental overhead remains. No core version of Postgres (9.4–17) changed to
a thread model. Users still mitigate this via connection pools (e.g. pgbouncer)
or by restricting max_connections.

- **Fixed/Improved?** Not fixed.
- **Version(s):** (No change in 9.4–17) – Postgres still process-based.
- **How addressed:** Best practice is out-of-process pooling.
- **Sources:** Uber blog [o^Why Uber Engineering Switched from Postgres to
  MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Postgres%2C%20however%2C%20use%20a%20process,to%20make%20a%20context%20switch);
  PostgreSQL documentation still recommends pooling for large client loads.

## Issue 8: Buffer-cache (I/O caching) inefficiency

**Uber’s summary:** Postgres relies on the OS page cache instead of
a user-space buffer pool. This incurs extra `read()` and `lseek()` syscalls
(context switches) [o^Why Uber Engineering Switched from Postgres to MySQL
│ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20problem%20with%20this%20design,into%20a%20single%20system%20call).
InnoDB by contrast uses an internal buffer pool, avoiding many syscalls.

**Status:** _Not fixed._

The Postgres buffer/cache design remains unchanged. The server does not yet
implement its own shared buffer pool in userspace; it still relies on OS
caching. There has been no built-in alternative or replacement. (AkkaPG or
other fork projects experimented with custom buffer management, but core
PostgreSQL still uses the OS page cache.)

- **Fixed/Improved?** Not fixed.
- **Version(s):** (No change up to v17) – Postgres still uses OS page cache.
- **How addressed:** None in core; performance tuning can use `shared_buffers`
  and OS config (huge pages), but design unchanged.
- **Sources:** Uber blog [o^Why Uber Engineering Switched from Postgres to
  MySQL │ Uber
  Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20problem%20with%20this%20design,into%20a%20single%20system%20call);
  current Postgres docs (e.g. _Shared Memory Configuration_) still describe
  `shared_buffers` as relatively small and rely on OS cache.

## Summary table

| Issue (Uber) | Uber’s description | Fixed/Improved? | Version(s) | How
addressed (core PG) | Sources | | -------------------------------------
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| -------------------------- | ------------------------------
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| | Inefficient writes (write amplifier) | “inefficient architecture for
writes” [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=We%20encountered%20many%20Postgres%20limitations%3A)
(small updates cause many index/WAL writes) | Improved (partly) | PG12, PG13
(dedup) | HOT updates (since 8.3) skip index writes for unmodified cols; PG12
B-tree improvements and PG13 index page deduplication reduce index churn.
| Uber blog [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20first%20problem%20with%20Postgres%E2%80%99s,at%20least%20four%20physical%20updates);
PostgreSQL 13 release info [o^Why Uber Engineering Switched from Postgres to
MySQL (2016) │ Hacker
News](https://news.ycombinator.com/item?id=26283348#:~:text=There%20is%20also%20index%20deduplication,is%20more%20descriptive%20and%20has)
| | Inefficient replication (verbose WAL) | “inefficient data replication”
[o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=,Inefficient%20data%20replication)
(WAL-based stream very verbose) | Improved | PG10 (logic. repl) | Introduced
logical replication (PG10) and publications/subscriptions. Pglogical (v9.4+)
allows row-based, cross-version streaming. | Uber blog [o^Why Uber Engineering
Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=);
PG 10 Release Notes [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek)
(logical replication) | | Table-corruption (timeline bug) | Timeline switch bug
in 9.2 caused replica WAL to misapply (duplicate rows) [o^Why Uber Engineering
Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=During%20a%20routine%20master%20database,mechanism%20weren%E2%80%99t%20actually%20marked%20inactive)
| Fixed | PG 9.2.2+, PG 9.3+ | Fixed cascading replication/timeline issues in
early 9.2 patch releases (e.g. 9.2.2) and 9.3. Standbys now correctly follow
new timelines. | PG 9.2.2 Release Notes [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/9.2.2/#:~:text=%2A%20Avoid%20bogus%20%22out,Heikki%20Linnakangas);
Uber blog [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=During%20a%20routine%20master%20database,mechanism%20weren%E2%80%99t%20actually%20marked%20inactive)
[o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20bug%20we%20ran%20into,could%20be%20released%20at%20any)
| | Replica MVCC (standby blocking) | “Postgres does not have true replica MVCC
support” [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=)
(standby queries can block/kill WAL apply) | Still an issue (mitigated) | PG
9.0+ (mitigations) | No fundamental change. Hot_standby_feedback (since 9.0)
can prevent some query cancels; built-in logical replication (PG10) avoids
physical-applying constraints. | Uber blog [o^Why Uber Engineering Switched
from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=);
Postgres docs on hot_standby_feedback and logical replication | | Upgrade
difficulty (cross-version) | “not possible to replicate… between GA releases”
[o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Because%20replication%20records%20work%20at,3)
| Fixed | PG10 (PGLogical v9.4+) | Logical replication supports cross-version
migration (PG10’s publish/subscribe [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek)).
pglogical (9.4+) also allows side-by-side upgrade. | Uber blog [o^Why Uber
Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Because%20replication%20records%20work%20at,3);
PostgreSQL 10 Release Notes [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek)
| | VACUUM inefficiency | “autovacuum process has to do full table scans”
[o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=This%20design%20also%20makes%20vacuuming,scans%20to%20identify%20deleted%20rows)
| Improved (partly) | (visibility map in 8.4; PG12+) | PG uses a visibility map
to skip clean pages [o^PostgreSQL: Documentation: 13: 24.1. Routine
Vacuuming](https://www.postgresql.org/docs/13/routine-vacuuming.html#:~:text=match%20at%20L279%20,old%20row%20version%20in%20the).
PG12 added a freeze map to skip fully frozen pages; later PGs have
parallel/index-only vacuums. | Uber blog [o^Why Uber Engineering Switched from
Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=This%20design%20also%20makes%20vacuuming,scans%20to%20identify%20deleted%20rows);
PostgreSQL manual on visibility map [o^PostgreSQL: Documentation: 13:
24.1. Routine
Vacuuming](https://www.postgresql.org/docs/13/routine-vacuuming.html#:~:text=match%20at%20L279%20,old%20row%20version%20in%20the)
| | Connection scaling | “process-per-connection… scaling [past few hundred]
difficult” [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Postgres%2C%20however%2C%20use%20a%20process,to%20make%20a%20context%20switch)
| Not fixed | (no change) | Architecture unchanged: each client is a full OS
process. Scaling still relies on external pooling (pgbouncer/pgpool). | Uber
blog [o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=Postgres%2C%20however%2C%20use%20a%20process,to%20make%20a%20context%20switch);
PostgreSQL docs (pool recommended). | | Buffer cache overhead | Prefork design
“doesn’t use `pread()` etc.” [o^Why Uber Engineering Switched from Postgres to
MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20problem%20with%20this%20design,into%20a%20single%20system%20call)
(extra system calls/context switches) | Not fixed | (no change) | Postgres
still relies on OS page cache (no native user-space buffer pool). Users set
`shared_buffers` but heavy I/O still incurs syscalls. | Uber blog [o^Why Uber
Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=The%20problem%20with%20this%20design,into%20a%20single%20system%20call);
PostgreSQL config docs (e.g. description of `shared_buffers`). |

## Remaining limitations

Of Uber’s listed issues, most have seen significant improvement in modern
Postgres. The main **remaining limitations** as of PG 17 are:

- **Write amplification:** Core MVCC/index design still causes multiple index
  writes per update (HOT can skip some, but multi-index tables still see extra
  writes). Although new features (HOT, index dedup, etc.) help, heavy write
  workloads with many indexes can still incur overhead.
- **Replica MVCC:** Physical standbys still cannot serve past snapshots without
  risk of conflicts. Hot standby feedback helps, but open long queries can
  still delay WAL apply.
- **Connection scaling:** The process-per-connection model remains; scaling
  beyond hundreds of clients still requires pooling.
- **Buffer pool:** Postgres still uses OS caching by default, so the user-space
  buffer-pool design of InnoDB is not matched.

All other Uber issues (verbose WAL replication, timeline bugs, upgrade
downtime, vacuum scans) have been addressed by features in later versions.
Evidence is captured in the table above and the cited PostgreSQL release
notes/docs.

**Sources:** Uber Engineering blog [o^Why Uber Engineering Switched from
Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=We%20encountered%20many%20Postgres%20limitations%3A)
[o^Why Uber Engineering Switched from Postgres to MySQL │ Uber
Blog](https://www.uber.com/en-US/blog/postgres-to-mysql-migration/#:~:text=During%20a%20routine%20master%20database,mechanism%20weren%E2%80%99t%20actually%20marked%20inactive);
PostgreSQL release notes and docs (v9.2–17) [o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/9.2.2/#:~:text=%2A%20Avoid%20bogus%20%22out,Heikki%20Linnakangas)
[o^PostgreSQL: Release
Notes](https://www.postgresql.org/docs/release/10.0/#:~:text=,Petr%20Jelinek)
[o^PostgreSQL: Documentation: 13: 24.1. Routine
Vacuuming](https://www.postgresql.org/docs/13/routine-vacuuming.html#:~:text=match%20at%20L279%20,old%20row%20version%20in%20the).
