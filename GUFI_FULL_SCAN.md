# GUFI — The Full Scan Process

Source: <https://github.com/mar-file-system/gufi> (LANL / MarFS "Grand Unified File Index").
All file/line references below point into that repository's `src/` and `include/`.

---

## 1. What "full scan" means in GUFI

GUFI never scans a filesystem to answer a query. It performs a **full scan of the live
filesystem once** to build an index, and thereafter performs a **full scan of the index**
(a much cheaper, metadata-only tree of SQLite databases) to answer queries.

So there are really three distinct full-tree traversals in the codebase, all built on the
same threading primitive:

| # | Scan | Program | Direction | Reads | Writes |
|---|------|---------|-----------|-------|--------|
| 1 | **Index build (full source scan)** | `gufi_dir2index`, or `gufi_dir2trace` + `gufi_trace2index` | top-down | real filesystem (`readdir` + `stat`) | GUFI index tree (`db.db` per directory) |
| 2 | **Query (full index scan)** | `gufi_query` (and wrappers `gufi_find`, `gufi_ls`, `gufi_du`, `gufi_stats`) | top-down | index tree | stdout / output db |
| 3 | **Aggregation passes** | `gufi_treesummary`, `gufi_rollup` | bottom-up | index tree | index tree |

The shared shape of every one of these: **one directory = one unit of work**. A directory
is never processed by more than one thread, and directories are processed concurrently
with no global tree lock.

---

## 2. The engine underneath every scan: QueuePerThreadPool (QPTPool)

`include/QueuePerThreadPool.h`, `src/QueuePerThreadPool.c`

Classic thread pools share one queue and serialize on one mutex. At millions of
directories that mutex becomes the bottleneck. GUFI instead gives **every thread its own
pair of queues** and adds work stealing.

### Per-thread state (`QPTPoolThreadData_t`, `QueuePerThreadPool.c:83`)

```
waiting   — new work pushed here by producers (any thread)
claimed   — work this thread has taken ownership of and is executing from
next_queue— round-robin cursor: which thread I push my next item to
steal_from— cursor: which neighbour I try to steal from next
```

### The worker loop (`worker_function`, `QueuePerThreadPool.c:371`)

```
loop:
    maybe_steal_work()          # only if my waiting queue is empty and work remains
    wait_for_work()             # condvar sleep until work exists or pool is stopping
    if (stopping and pool->incomplete == 0) break
    claim_work()                # move my ENTIRE waiting queue into claimed, in one splice
    process_work()              # pop from claimed one at a time, call qi->func(ctx, work)
    pool->incomplete -= done    # global counter of outstanding work
```

Three details matter for understanding scan behaviour:

* **Bulk claim, single-item pop.** `claim_work` (`:311`) splices the whole waiting list at
  once — one lock acquisition for N items. `process_work` (`:324`) then pops items one at a
  time from `claimed` so that a *stealer* can take the tail of a still-executing thread's
  claimed list. That is the fix for the starvation case where one huge directory sits at the
  head of a long claimed list.
* **Work stealing, two tiers** (`steal`, `:166`). A thread with nothing to do walks its
  neighbours in round-robin order, `trylock`s their queue, and takes
  `size * steal.num / steal.denom` items (minimum 1). GUFI's scanners construct the pool
  with `steal_num=1, steal_denom=2` → **take half the neighbour's queue**. It first sweeps
  all `waiting` queues; only if every one is empty does it fall back to stealing from
  `claimed` queues (`steal_claimed`, `:222`) — deliberately rare, since it contends with
  running work.
* **Termination.** There is no "the tree is finished" signal. The pool exits when
  `state == STOPPING` (set by `QPTPool_stop`, called on the main thread right after the
  roots are enqueued) **and** `pool->incomplete == 0` **and** nothing is swapped. Because
  work spawns work, `QPTPool_stop` is effectively "no more *roots*; drain everything
  reachable."

### Memory back-pressure

A directory-heavy tree can enqueue faster than it is consumed. Three mechanisms cap that:

* `--target-memory-footprint` → `get_queue_limit()` (`src/utils.c:82`) =
  `target / sizeof(struct work) / nthreads`. Once a thread's `waiting` queue exceeds this,
  `QPTPool_enqueue_swappable` serializes the work item to a **swap file** instead of RAM
  (`write_swap`, `QueuePerThreadPool.c:903`); it is read back later.
* `--compress` (zlib) — work items are deflated in memory (`compress_struct` /
  `decompress_work`, `src/descend.c:297`).
* `--subdir-limit N` — see below; caps how many subdirectories of one directory become
  queue entries.

---

## 3. Scan #1 — Building the index (the full filesystem scan)

### 3.1 Entry point and startup (`gufi_dir2index.c:551`)

```
gufi_dir2index [options] src... GUFI_tree_parent
```

Sequence in `main`:

1. Parse args; last positional is the **index parent**, the rest are source roots.
2. `setup_dst()` — create the index parent directory if absent.
3. `create_dbdb_template()` — build **one prototype SQLite file** in a temp file with the
   `entries`, `summary`, `xattrs…` tables already created (`src/template_db.c:195`). Every
   per-directory database in the whole index is produced by *byte-copying this template*
   (`template_to_db`, `template_db.c:256`) rather than by running `CREATE TABLE` millions of
   times. This is one of the largest single wins in the index build.
4. `create_empty_dbdb()` — put an empty `db.db` at the index parent so queries against the
   parent don't error.
5. `create_xattrs_template()` — the same trick for per-user / per-group xattr databases.
6. `QPTPool_init_with_props(nthreads, &pa, NULL, NULL, queue_limit, swap_prefix, 1, 2, 1)`
   → steal half, stop-on-error on. `QPTPool_start()` spawns the threads.
7. For each source root: `validate_source()` (`lstat`, must be a directory; records
   `root_parent` so the index path can be computed by prefix substitution), then
   `QPTPool_enqueue(ctx, processdir, root)`.
8. `QPTPool_stop(ctx)` — blocks until the entire tree has drained.
9. Sum the per-thread `total_dirs` / `total_nondirs` counters and print
   Dirs/Sec, Non-Dirs/Sec, elapsed.

### 3.2 The per-directory unit of work (`processdir`, `gufi_dir2index.c:189`)

This function *is* the scan. Every directory in the source tree runs it exactly once.

```
processdir(ctx, work):
    decompress_work(work)                       # inflate/deserialize if needed
    dir = opendir(work->name)                   # give up quietly if unreadable
    lstat(work->name)                            # directory's own metadata

    # index path = index_parent + (source path with root_parent prefix stripped)
    topath = index_parent + "/" + (work->name + work->root_parent.len)
    mkdir(topath)                                # parent is guaranteed to exist already

    db = template_to_db(template, topath + "/db.db", uid, gid)   # byte copy + open
    zeroit(&summary)                             # reset running aggregates
    entries_res = insertdbprep(db, ENTRIES_INSERT)               # prepared statement
    startdb(db)                                  # BEGIN TRANSACTION

    descend(...)          # <<< enqueue subdirs, stream files/links into this transaction

    insertdbfin(entries_res)
    xattrs_get(work->name)                       # this dir's own xattrs
    insertsumdb(db, ..., &summary)               # one row into `summary`
    stopdb(db)                                   # COMMIT
    closedb(db)
    chmod(topath, mode); chown(topath, uid, gid) # index mirrors source permissions
    closedir(dir)
    total_dirs[id]++ ; total_nondirs[id] += ctrs.nondirs_processed
```

Ordering is deliberate and worth noting for an animation: **`descend` runs early**, before
the slow per-directory work (xattrs, summary insert, commit). Subdirectories are handed to
the pool as soon as they are discovered so the other threads never go hungry; the expensive
tail of the current directory happens afterwards.

### 3.3 `descend()` — the one loop that touches the filesystem (`src/descend.c:124`)

Shared by `gufi_dir2index` and `gufi_dir2trace`. For the open directory handle:

```
if work->level > in->max_level: return          # depth cut-off

while (entry = readdir(dir)):
    skip ".", "..", anything in --skip file (trie_search), and "db.db"
    child = new_work_with_name(parent_path, entry->d_name)
    try_skip_lstat(entry, child)                # see below
    child->level = parent.level + 1
    child->pinode = parent inode

    if S_ISDIR(child):
        if child->level <= max_level:
            if subdir_limit not hit:  QPTPool_enqueue(ctx, processdir, compress(child))
            else:                     processdir(ctx, child)      # recurse in-situ
        else: free(child)
        continue

    if S_ISLNK: readlink() into child_ed.linkname ; type='l'
    elif S_ISREG: type='f'
    else: free(child); continue                  # sockets/fifos/devices not indexed

    if work->level >= in->min_level:
        if process_xattrs: xattrs_get(child)
        processnondir(child, &child_ed, args)    # -> process_nondir, below
    free(child)
```

Key optimizations visible here:

* **`try_skip_lstat()`** (`descend.c:83`) — uses `dirent.d_type` to classify the entry
  without a syscall. `lstat` is only called when the filesystem reports `DT_UNKNOWN`. Files
  still get `fstatat` later (relative to the parent fd), but the *directory-vs-file*
  decision, which is what drives traversal, is usually syscall-free.
* **`--subdir-limit`** — for a directory with an enormous fan-out, only the first N
  subdirectories become queue items; the remainder are processed **in-situ** by recursively
  calling `processdir` on the current thread. This bounds peak memory (one child work item
  alive at a time instead of all of them) at the cost of some parallelism. The recursion
  depth is tracked in `child->recursion_level`.
* **`--min-level` / `--max-level`** — `max_level` prunes the traversal; `min_level`
  suppresses *recording* while still descending. Note the asymmetry in the code comments:
  files/links are at the same level as their directory, so the current level is checked for
  entries but `next_level` is checked before enqueuing a subdirectory.

### 3.4 Per-file handling (`process_nondir`, `gufi_dir2index.c:139`)

For each file or symlink discovered:

1. Plugin hook `plugins_stat_file` may supply metadata; otherwise `fstatat_wrapper`
   (`fstatat`/`statx` relative to the parent's directory fd — no path re-walk).
2. If the basename is the external-db marker file, `external_read_file()` registers it.
3. If `--xattrs`, xattrs are pulled and routed to `db.db` or to per-user/per-group external
   xattr databases depending on readability.
4. **`sumit(&summary, entry, ed)`** — folds the entry into the directory's running
   aggregate: counts, min/max/total of uid, gid, size, blocks, atime/mtime/ctime/crtime,
   zero-length count, size-bucket counters (`totltk`, `totmtk`, `totmtm`, `totmtg`,
   `totmtt`), xattr count.
5. **`insertdbgo(entry, ed, entries_res)`** — bind + step the prepared statement. All
   inserts for the directory are inside the single transaction opened by `startdb`, so the
   whole directory is one commit.

### 3.5 The result: the index tree

The index is a **shadow directory tree**. For source path `/proj/a/b`, with index parent
`/index`, GUFI creates `/index/proj/a/b/db.db`, with the same mode/uid/gid as the source
directory — so **UNIX permissions on the index reproduce permissions on the source**, which
is how GUFI can let unprivileged users query an index built by root.

Each `db.db` contains (`include/dbutils.h:98`+):

| Table / view | Contents |
|---|---|
| `entries` | one row per file/link **in this directory only** (name, type, inode, mode, nlink, uid, gid, size, blksize, blocks, atime, mtime, ctime, linkname, xattr_names, crtime, 4 `ossintN` + 2 `osstextN` user columns) |
| `summary` | exactly one row: this directory's own stat data **plus** the aggregates computed by `sumit` — `totfiles`, `totlinks`, min/max/tot of uid/gid/size/times/blocks, `depth`, `pinode`, `isroot`, `canrollup`, `isrolledup` |
| `pentries` | view: `entries` joined with the parent inode from `summary` |
| `xattrs_avail` / `xattrs_pwd` | xattr rows readable in this context |
| `treesummary` | *(only after `gufi_treesummary`/`gufi_rollup`)* aggregates for the **entire subtree** rooted here, including `totsubdirs`, `maxsubdirfiles`, `maxsubdirsize` |
| `vrsummary`, `vrpentries` | rollup-aware views used by queries |

### 3.6 The two-phase alternative: trace files

For very large or remote filesystems the walk and the database build are split:

* **`gufi_dir2trace`** (`gufi_dir2trace.c:145`) runs the *same* `descend()` but
  `process_nondir` just serializes each entry as a delimited text line into one **trace file
  per thread** (`worktofile`). Output is a set of stanzas: a directory line followed by its
  entry lines. Nothing SQLite-related happens, so the walk runs at raw `readdir`/`stat`
  speed and can be shipped elsewhere.
* **`gufi_trace2index`** (`gufi_trace2index.c:279`) reverses it. "Scout" threads
  (`enqueue_traces` / `scout_stream`) parse the trace files, find stanza boundaries, and
  enqueue one work item per directory carrying `(fd, offset, entry count)`. Half the threads
  start scouting while the rest already build databases. Each `processdir`
  (`gufi_trace2index.c:148`) does `dupdir()` (directories can appear in any order here, so
  full recursive mkdir is required — unlike `dir2index`, where the parent is guaranteed to
  exist), copies the template, `getline`s exactly its own entry lines, and performs the same
  `sumit` + `insertdbgo` + `insertsumdb` sequence. `MAXRECS` (default) rows per transaction
  triggers a commit/restart to bound the journal.

Same index comes out either way.

---

## 4. Scan #2 — The full index scan (`gufi_query`)

`gufi_query -S <summary SQL> -E <entries SQL> [-T <treesummary SQL>] [-a] index...`

Structurally identical to the index build — QPTPool, one directory per work item, descend
enqueues subdirectories — but the "directory" is an index directory and the work is running
the user's SQL against the local `db.db`.

### 4.1 Per-directory (`processdir`, `src/gufi_query/processdir.c:163`)

```
gqw = decompress(work)                      # gqw_t = work + sqlite3-escaped path
dbname = <index dir>/db.db
dir = opendir(index dir)                    # needed to find subdirectories
if keep_matime: save db.db's atime/mtime for restoration afterwards
if level >= min_level:
    db = attachdb(dbname, thread's out db, "tree", open_flags)
    addqueryfuncs_with_context(db)          # path(), epath(), fpath(), uidtouser(),
                                            #   gidtogroup(), human-readable size, …
    bind {n} -> basename, {i} -> full path  # user string substitution
    if -T given: verify a treesummary table exists, then run it; a miss can prune
    if xattrs enabled: build xattr views on the fly
    if -e/--external-attach: for each directory inode in `summary`, ATTACH the external
        db, create views, run queries, detach   (loop per inode)
    else: process_queries(...) once
detachdb ; restore matime ; closedir ; free
```

### 4.2 Descent and query, in order (`process_queries`, `process_queries.c:282`)

```
if descend:
    get_isrolledup(db)                     # summary.isrolledup
    if not rolled up:
        subdirs_walked = gq_descend(...)   # push child directories into the pool
if level >= min_level:
    register subdirs() SQL function        # rolled up -> stored count, else walked count
    if -S: run summary SQL   -> recs
    if recs > 0 (or OR-mode) and -E: run entries SQL
```

Two behaviours worth calling out:

* **Descent happens before the SQL runs**, again to keep the pool fed.
* **Rollup short-circuits the scan.** If `summary.isrolledup` is set, this directory's
  `db.db` already contains its descendants' entries, so `gq_descend` is skipped entirely —
  the traversal *stops here*. This is the single largest query-side speedup: a rolled-up
  tree turns thousands of directory visits into one database open.
* `-T` (treesummary) can prune an entire subtree with one query before any per-directory
  work happens.

`gq_descend` (`process_queries.c:149`) mirrors `descend()` but is index-specific: it skips
`db.db`, only enqueues directories, and applies `--dir-match-uid/gid` — with the
optimization that once an ancestor matched, `child->id_match` is inherited and no further
`lstat` is done.

### 4.3 Output

Each thread writes through its own `OutputBuffer` (`src/OutputBuffers.c`) and flushes under
a shared mutex, so per-directory results stream out as they are produced — output order is
non-deterministic by design. With `-a` and the `-I/-K/-J/-G/-F` aggregation flags, threads
instead write into per-thread in-memory databases, which `aggregate_intermediate()` unions
into one final database after the pool drains, and `-G` runs against that.

---

## 5. Scan #3 — Aggregation passes over the index

### 5.1 `gufi_treesummary` — top-down scan of one subtree

`gufi_treesummary` (`gufi_treesummary.c:130`) is *not* bottom-up. It reuses the same QPTPool
+ `descend()` machinery as the index build to sweep the subtree under one index directory,
accumulating into a per-thread `struct sum`:

* If the directory is **rolled up** (`isrolledup != 0`), everything is already local:
  `querytsdb()` aggregates from here and descent stops.
* Else if a `treesummary` table already exists (and this isn't level 0), its stored
  aggregates are consumed and descent stops.
* Otherwise `descend()` enqueues the subdirectories and this directory's `summary` row is
  folded in.

After the pool drains, `compute_treesummary()` merges the per-thread sums with `tsumit()`
and writes **one** `treesummary` row into the starting directory's `db.db`. So this program
answers "summarize this subtree", not "summarize every subtree".

### 5.2 Bottom-up passes: `gufi_treesummary_all` and `gufi_rollup`

Both use `parallel_bottomup` (`include/BottomUp.h`, `src/BottomUp.c`), which walks *down*
to discover the tree and then executes user work *on the way back up*: each `struct
BottomUp` holds a `refs.remaining` counter; a directory's `ascend` callback only fires once
all of its children have finished. That is the parallel equivalent of post-order traversal.

* **`gufi_treesummary_all`** (`treesummary_ascend`, `gufi_treesummary_all.c:137`) — creates a
  `treesummary` table in *every* directory of the index, each holding subtree-wide totals
  (`totsubdirs`, `maxsubdirfiles`, `maxsubdirlinks`, `maxsubdirsize`, all the min/max/tot
  columns). This is what lets `-T` prune whole subtrees during a query. With
  `--dont-reprocess`, `treesummary_descend` (`:96`) stops descending as soon as it finds an
  existing `treesummary` table.
* **`gufi_rollup`** (`rollup_descend` `:330`, `rollup_ascend` `:746`) — physically merges a
  directory's descendants' `entries` into its own database (into `pentries_rollup`) so a
  query can answer for the whole subtree from a single `db.db`. It only does so when it is
  legal and worthwhile:
  * `can_rollup()` (`:427`) compares each child's permissions with the parent's; a child
    whose permissions would let someone see rows they shouldn't blocks rollup.
  * `should_rollup()` (`:527`) additionally refuses when the merged `pentries` would exceed
    a row threshold — a rolled-up database that is too large is slower to query than
    descending.
  * Results are recorded in `summary.canrollup` / `summary.isrolledup`, which
    `process_queries` then reads to stop descending.
  * `gufi_unrollup` reverses the operation.

---

## 6. End-to-end picture

```
                          ┌──────────────────────────────────────────┐
   real filesystem  ──►   │ SCAN 1: gufi_dir2index                   │  ──►  index tree
   (readdir + stat)       │   QPTPool: 1 work item per directory     │       <dir>/db.db
                          │   descend(): subdirs -> queue            │       (entries +
                          │              files   -> sumit + INSERT   │        summary)
                          │   template copy instead of CREATE TABLE  │
                          └──────────────────────────────────────────┘
                                   ▲ (alternative)
                          gufi_dir2trace  ──trace files──►  gufi_trace2index

                          ┌──────────────────────────────────────────┐
   index tree      ──►    │ SCAN 3: gufi_treesummary_all / _rollup   │  ──►  treesummary
                          │   parallel_bottomup: post-order          │       + rolled-up
                          │   permission-safe merging of children    │         entries
                          │   (gufi_treesummary = top-down, 1 subtree)│
                          └──────────────────────────────────────────┘

                          ┌──────────────────────────────────────────┐
   index tree      ──►    │ SCAN 2: gufi_query                       │  ──►  stdout /
                          │   QPTPool: 1 work item per index dir     │       output db
                          │   -T prune, -S summary, -E entries       │
                          │   stop descending where isrolledup=1     │
                          └──────────────────────────────────────────┘
                                   ▲
                          gufi_find / gufi_ls / gufi_du / gufi_stat  (script wrappers
                          that translate familiar CLI flags into -S/-E SQL)
```

---

## 7. Why the full scan is fast — summary of the techniques

1. **Directory-granular parallelism.** The unit of work is a directory, which is exactly the
   granularity at which a filesystem's metadata is laid out. No lock is held across a
   directory.
2. **Queue per thread + work stealing (half the neighbour's queue).** Removes the shared-queue
   mutex; the stealing keeps threads busy on unbalanced trees. The two-tier steal (waiting,
   then claimed) prevents one slow directory from starving everyone.
3. **Enqueue before you work.** `descend()` is called early in `processdir`, so the pool is
   refilled before the current directory's expensive tail (xattrs, summary, commit) runs.
4. **`d_type` instead of `lstat` for traversal decisions** — the traversal itself is nearly
   syscall-free per entry.
5. **`fstatat` relative to the open directory fd** — no repeated path resolution.
6. **Template database byte-copy** instead of `CREATE TABLE` per directory.
7. **One transaction per directory**, prepared statements bound in a loop.
8. **Bounded memory:** queue limit + swap-to-disk, optional zlib compression of queued work,
   `--subdir-limit` in-situ recursion for huge fan-outs.
9. **Query-side pruning:** `--min-level`/`--max-level`, `-T` treesummary early-out,
   `isrolledup` truncating descent entirely.
10. **Index permissions mirror source permissions**, so no scan-time or query-time ACL
    engine is needed — the OS enforces it.

---

## 8. Tunables that change scan behaviour

| Flag | Applies to | Effect |
|---|---|---|
| `-n <threads>` | all | pool size |
| `--min-level` / `--max-level` | walk + query | record-from depth / prune-below depth |
| `--skip <file>` | walk + query | basenames to ignore (loaded into a trie) |
| `-x` / `--xattrs` | index build | pull xattrs; creates per-user/per-group external dbs |
| `--target-memory-footprint` | all | derives per-thread queue limit → swap-to-disk |
| `--swap-prefix` | all | where swapped work items go |
| `--compress` | all (zlib builds) | deflate queued work items |
| `--subdir-limit N` | index build | after N subdirs, recurse in-situ instead of enqueuing |
| `--path-list` | all | partial walk: start from an explicit list of subtree roots |
| `-S` / `-E` / `-T` | query | summary / entries / treesummary SQL |
| `-a` | query | AND/OR semantics between `-T`, `-S`, `-E` |
| `-I -K -J -G -F` | query | per-thread intermediate dbs + final aggregation |
| `--keep-matime` | query | restore each `db.db`'s atime/mtime after reading |

---

## 9. Quick reference — where to look in the source

| Concept | Location |
|---|---|
| Thread pool, stealing, swap | `src/QueuePerThreadPool.c` (`worker_function:371`, `steal:166`, `claim_work:311`, `process_work:324`) |
| The traversal loop | `src/descend.c:124`; `d_type` shortcut at `:83` |
| Index build driver | `src/gufi_dir2index.c` (`main:551`, `processdir:189`, `process_nondir:139`) |
| Trace walk | `src/gufi_dir2trace.c` (`processdir:145`) |
| Trace → index | `src/gufi_trace2index.c` (`main:279`, `processdir:148`) |
| Query driver | `src/gufi_query/main.c`, `processdir.c:163`, `process_queries.c:282`, `gq_descend:149` |
| Bottom-up framework | `include/BottomUp.h`, `src/BottomUp.c` |
| Treesummary (one subtree, top-down) | `src/gufi_treesummary.c` (`processdir:130`, `compute_treesummary:228`) |
| Treesummary (whole index, bottom-up) | `src/gufi_treesummary_all.c` (`treesummary_ascend:137`, `treesummary_descend:96`) |
| Rollup | `src/gufi_rollup.c` (`can_rollup:427`, `should_rollup:527`, `rollup_ascend:746`) |
| Schema | `include/dbutils.h:98`–`:200` |
| Template database | `src/template_db.c` (`create_dbdb_template:195`, `template_to_db:256`) |
| Aggregation (`sumit`) | `src/utils.c`, `include/bf.h` (`struct sum`) |
