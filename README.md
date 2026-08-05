# GUFI animations

Slow, self-contained JS/canvas animations of how [GUFI](https://github.com/mar-file-system/gufi)
walks a filesystem. No build step, no libraries, no network — each page is one HTML file.

| | |
|---|---|
| **[▶ Full scan](https://nikhil-ghind.github.io/gufi-animation/)** (`index.html`) | building the index: work units, per-thread queues, work stealing |
| **[▶ Rollup, then query](https://nikhil-ghind.github.io/gufi-animation/rollup_query.html)** (`rollup_query.html`) | bottom-up rollup, then the query that rollup makes cheap |

The rest of this file describes the full-scan page; the rollup/query page is described
[further down](#rollup-then-query).

## Reading the screen

- **Top — caption.** The large line says what just happened; the smaller coloured line
  names the mechanism in the C code responsible for it.
- **Left top — source tree.** The filesystem being walked. One circle = one directory =
  one work item; the second line inside each circle (`27f`) is how many files/links that
  directory holds — the bigger that number, the longer its thread stays busy on it.
  Grey = not seen yet · blue = sitting in some thread's queue · **thread-coloured** = being
  scanned right now · green = scanned.
- **Left bottom — GUFI index tree.** The destination, built as the scan runs. A directory
  appears as a **blue outline ("empty")** the moment its thread does `mkdir` + copies the
  `db.db` template, and turns **green with its entry count** when the entries and summary
  row are committed. Compare the two trees: the index always lags the source by exactly the
  directories currently in flight.
- **Right — thread pool.** One lane per thread, showing what it is doing and its two
  queues: **waiting** (where other threads push work) and **claimed** (what this thread has
  taken ownership of). Chips are labelled with the directory name.

Flying chips are work items in transit: **blue** = a subdirectory being enqueued by
`descend()`, **pink** = an item being stolen.

## What to watch for

1. **Work division** — when a thread reads a directory, each subdirectory is enqueued to
   `next_queue`, which advances round-robin, so work spreads across threads as it is
   discovered. Files are never enqueued; they go straight into that directory's `db.db`
   transaction.
2. **Queue processing** — a thread moves its **entire** waiting queue into claimed in one
   operation, then works through claimed one item at a time.
3. **Work stealing** — an idle thread takes half of a neighbour's waiting queue, from the
   front. Only if every waiting queue is empty does it steal from a neighbour's *claimed*
   queue instead. The start (only T0 has the root) and the end (threads fight over the last
   directories) are where stealing shows up most.

Controls: **threads** (2–16), **speed** (0.25×–2×), **Pause**, **Restart** (or `space` / `r`).

Changing the thread count restarts on the *same* tree, so only the scheduling differs — run 2
then 16 back to back and the queues, the round-robin spread, and how much stealing happens are
the only things that change. The tree size is fixed at 22 directories, so at 16 threads most
threads spend the run stealing; that starvation is the point, not a bug.

The controls can also be preset from the URL, which is handy for a live demo:
`?threads=8`, `?skip=20` (fast-forward N seconds before drawing).
Restart generates a new random tree.

## Fidelity

The scheduler follows `src/QueuePerThreadPool.c` and the scan follows
`src/gufi_dir2index.c` + `src/descend.c`: round-robin `next_queue`, atomic
`waiting → claimed` splice, pop-before-execute, `steal.num/steal.denom = 1/2`, front-first
steal, round-robin `steal_from` cursor, waiting-queues-checked-before-claimed-queues, and
termination when `incomplete == 0`. Timings are stretched for watchability, not measured.

See [`GUFI_FULL_SCAN.md`](GUFI_FULL_SCAN.md) for the written walkthrough of the full scan
pipeline with file:line references into the GUFI source.

---

# Rollup, then query

`rollup_query.html` — two acts over one index, switched with the **① Rollup** / **② Query**
buttons (or `1` / `2`).

## Act 1 — rollup (bottom-up)

`gufi_rollup` walks *down* to discover the tree and does its work *on the way back up*
(`parallel_bottomup`). A directory's `ascend` cannot run until every child has finished, so
each node carries a `2/3`-style badge: that is `refs.remaining` counting down.

When `ascend` fires, two tests decide the directory's fate, and the panel on the right shows
the actual comparison:

- **`can_rollup()`** — every child must already be rolled up, and no child may carry
  permissions that differ from this directory's. A `0700` directory under a `0755` parent
  blocks the merge, because merging would let a query see rows the permissions hide.
- **`should_rollup()`** — the merged row count must stay under the limit. Past that point one
  huge `db.db` is slower to query than descending into several small ones.

Pass both and the descendants' entries are copied into `pentries_rollup` and
`summary.isrolledup` is set (purple, `Σ N`). Fail either and the directory is marked in red
with the reason: `perms`, `child` (something below it refused), or `too big`.

**Watch the poisoning.** A single `0700` directory refuses, which makes its parent refuse
(`perms`), which makes *its* parent refuse (`child`) — one odd permission can walk all the way
to the root. That is why rollup is usually partial, not all-or-nothing.

## Act 2 — query (top-down), two indexes side by side

The same `gufi_query -S … -E …` runs against the rolled-up index and a plain one, on the same
tree, with the same threads. Per directory: `opendir` + `attachdb(db.db)`, the `-S` summary
SQL, then the `-E` entries SQL. Descent happens *before* the SQL, to keep the pool fed.

The rolled-up side calls `get_isrolledup()` and, where it is set, **skips `gq_descend()`
entirely** — the subtree greys out as "never opened". The cost panel keeps both tallies.

The payoff is worth stating precisely, because it is easy to overclaim: **both sides scan the
same number of rows.** Rollup does not read less data. What it removes is the per-directory
`opendir` + `ATTACH` — in the default tree, 22 database opens become 10.

## Fidelity and honest limits

Act 1 follows `src/gufi_rollup.c` (`rollup_descend`, `rollup_ascend`, `can_rollup`,
`should_rollup`) and `src/BottomUp.c`; act 2 follows `src/gufi_query/processdir.c` and
`process_queries.c`. Both reuse the QPTPool shape of the scan page.

What is simplified, deliberately:

- The row limit is **88**, a number chosen so a 22-directory tree produces an interesting mix.
  Real GUFI's threshold is a tunable and much larger.
- Permissions are a single mode string per directory; real `can_rollup()` compares
  user/group/other bits properly.
- Timings are stretched for watchability. The 4.8s-vs-8.8s gap shows the *shape* of the
  speedup, not a benchmark.
- `-T` treesummary pruning, `-a` aggregation, external attach, and xattr views are not drawn.

URL presets, same as the scan page: `?act=2`, `?threads=8`, `?speed=0.5`, `?skip=10`.
