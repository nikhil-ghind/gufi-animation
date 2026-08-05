# GUFI full-scan animation

A slow, self-contained JS/canvas animation of how a
[GUFI](https://github.com/mar-file-system/gufi) full scan works: how the directory tree is
turned into work units, how those units are divided among threads, how each thread drives
its own two queues, and how an idle thread steals work from a neighbour.

**[▶ Watch it live](https://nikhil-ghind.github.io/gufi-animation/)** — or open `index.html` in a
browser. No build step, no libraries, no network.

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

Controls: **speed** (0.25×–2×), **Pause**, **Restart** (or `space` / `r`).
Restart generates a new random tree.

## Fidelity

The scheduler follows `src/QueuePerThreadPool.c` and the scan follows
`src/gufi_dir2index.c` + `src/descend.c`: round-robin `next_queue`, atomic
`waiting → claimed` splice, pop-before-execute, `steal.num/steal.denom = 1/2`, front-first
steal, round-robin `steal_from` cursor, waiting-queues-checked-before-claimed-queues, and
termination when `incomplete == 0`. Timings are stretched for watchability, not measured.

See [`GUFI_FULL_SCAN.md`](GUFI_FULL_SCAN.md) for the written walkthrough of the full scan
pipeline with file:line references into the GUFI source.
