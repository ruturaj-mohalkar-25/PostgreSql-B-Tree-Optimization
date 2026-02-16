# PostgreSQL B-Tree Micro-Optimizations: Prefetching and Linear Scan

<p align="center">
  <img src="https://img.shields.io/badge/Language-C-00599C?style=for-the-badge&logo=c&logoColor=white"/>
  <img src="https://img.shields.io/badge/Database-PostgreSQL_17.4-4169E1?style=for-the-badge&logo=postgresql&logoColor=white"/>
  <img src="https://img.shields.io/badge/Benchmark-IMDB%2FJOB-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Queries_Tested-113-blue?style=for-the-badge"/>
</p>

---

## 📋 Overview

This project extends PostgreSQL 17.4's B-Tree index implementation with two micro-optimizations:

1. **Leaf-page lookahead prefetch** — Issues async I/O hints for sibling pages during range scans
2. **Linear scan on tiny leaf pages** — Bypasses binary search overhead when pages have ≤4 items

Both optimizations are controlled by runtime-configurable GUC parameters, making them safe to enable/disable without server restart.

---

## 🎯 Problem Statement

PostgreSQL's B-Tree implementation follows the classic Lehman-Yao design, which is highly optimized for concurrent operations. However, two potential inefficiencies exist:

1. **I/O Stalls During Range Scans** — When scanning across leaf pages, the system waits for each page to load before proceeding
2. **Binary Search Overhead on Small Pages** — For pages with only 2-4 items, binary search has unnecessary loop overhead and branch mispredictions

---

## 💡 Solution

### 1. Leaf-Page Lookahead Prefetch

```
Current Page Processing          Next Page Loading
        │                              │
        ▼                              ▼
   ┌─────────┐    PrefetchBuffer()  ┌─────────┐
   │ Process │ ──────────────────►  │  Load   │
   │  Page   │     (async I/O)      │  Page   │
   └─────────┘                      └─────────┘
        │                              │
        └──────────► Overlap ◄─────────┘
```

- After processing current leaf page, issue non-blocking prefetch for sibling
- Forward scans prefetch `btpo_next`, backward scans prefetch `btpo_prev`
- If prefetch is unnecessary (e.g., concurrent split), correctness unaffected

### 2. Linear Scan for Tiny Pages

```
Page has ≤ threshold items?
        │
   ┌────┴────┐
   │         │
  Yes        No
   │         │
   ▼         ▼
Linear    Binary
 Scan     Search
```

- For leaf pages with 2 to `threshold` items, use sequential scan
- Avoids binary search loop overhead and branch mispredictions
- Falls back to binary search if conditions not met

---

## ⚙️ Configuration (GUC Parameters)

Three new runtime parameters added:

```sql
-- Enable leaf-page prefetching (default: off)
SET btree_leaf_prefetch = on;

-- Enable linear scan for small pages (default: off)
SET btree_binsrch_linear = on;

-- Set threshold for linear scan (default: 4, range: 1-32)
SET btree_binsrch_linear_threshold = 4;

-- Check current values
SHOW btree_leaf_prefetch;
SHOW btree_binsrch_linear;
SHOW btree_binsrch_linear_threshold;
```

All parameters are `USERSET`, so they can be toggled per-session without restart.

---

## 📁 Files Modified

| File | Purpose |
|------|---------|
| `src/include/access/nbtree.h` | GUC variable declarations |
| `src/backend/access/nbtree/nbtutils.c` | GUC variable definitions with defaults |
| `src/backend/access/nbtree/nbtsearch.c` | Core optimization logic |
| `src/backend/utils/misc/guc_tables.c` | GUC parameter registration |

---

## 📊 Results

Tested on **IMDB dataset** with **113 JOB queries**.

### Performance Summary

| Configuration | Faster (%) | Slower (%) | Avg Gain (ms) | Avg Loss (ms) |
|---------------|------------|------------|---------------|---------------|
| Linear Scan (on) | **57.5%** | 36.3% | 79.3 | 131.8 |
| Prefetch (on) | 29.2% | 66.4% | 107.5 | 88.1 |
| Both (on) | 30.1% | 63.7% | 14041.9 | 141.7 |

### Key Findings

- ✅ **Correctness preserved** — All queries returned correct results
- ✅ **No crashes or errors** — Stable under all test conditions
- ✅ **Code paths verified** — GDB debugging confirmed optimizations triggered
- ⚠️ **Mixed performance** — Small gains in some queries, small regressions in others
- ℹ️ **Expected outcome** — PostgreSQL's B-Tree is already highly optimized after decades of development

---

## 🚀 Getting Started

### Prerequisites

- Docker (recommended) or Linux environment
- GCC compiler
- ~4GB disk space for PostgreSQL + IMDB data

### Build

```bash
# Step 1: Clone PostgreSQL 17.4
wget https://ftp.postgresql.org/pub/source/v17.4/postgresql-17.4.tar.gz
tar xzf postgresql-17.4.tar.gz
cd postgresql-17.4

# Step 2: Apply the patch
git init
git add .
git commit -m "Initial PostgreSQL 17.4"
git apply /path/to/btree_prefetch_linear.patch

# Step 3: Configure with debug symbols
CFLAGS="-O0" ./configure --prefix=$HOME/postgresql-17.4 --enable-debug

# Step 4: Build and install
make -j$(nproc)
make install

# Step 5: Add to PATH
export PATH=$HOME/postgresql-17.4/bin:$PATH
```

### Initialize Database

```bash
# Initialize cluster
pg_ctl -D ~/databases initdb

# Start server
pg_ctl -D ~/databases -l logfile start

# Load IMDB dataset (see project repo for scripts)
```

### Run Benchmarks

```bash
# Connect to database
psql -U $USER -d imdbload

# Enable optimizations
SET btree_leaf_prefetch = on;
SET btree_binsrch_linear = on;
SET btree_binsrch_linear_threshold = 4;

# Run a query
\i /path/to/job_queries/1a.sql
```

---

## 🔮 When to Use

| Scenario | Prefetch | Linear Scan |
|----------|----------|-------------|
| Long range scans with I/O-bound workload | ✅ May help | — |
| Small buffer pool / cold cache | ✅ May help | — |
| Tables with very small leaf pages | — | ✅ May help |
| Already memory-resident data | ⚠️ Overhead | ⚠️ Overhead |
| Production workloads | ⚠️ Benchmark first | ⚠️ Benchmark first |

---

## 🔮 Future Work

- [ ] Test on I/O-bound workloads (spinning disks, limited memory)
- [ ] Evaluate with larger-than-memory datasets
- [ ] Measure CPU cache effects with perf counters
- [ ] Explore adaptive threshold tuning

---

## 📚 References

- [Lehman & Yao (1981) — Efficient Locking for Concurrent Operations on B-Trees](https://dl.acm.org/doi/10.1145/319628.319663)
- [PostgreSQL 17.4 Documentation](https://www.postgresql.org/docs/17/index.html)
- [IMDB/JOB Benchmark](https://github.com/gregrahn/join-order-benchmark)

---

## 📄 License

This project is for educational purposes as part of USC CS543 (Fall 2025).

---

<p align="center">
  <b>⭐ Star this repo if you found it helpful!</b>
</p>
