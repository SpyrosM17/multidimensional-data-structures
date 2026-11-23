<div align="center">

# 🎬 Multidimensional Movie Search Engine Project

### k-d Tree · Quadtree · R-tree · Range Tree · LSH

**Range queries over 946,000+ movies in 5D, followed by text similarity search with MinHash + LSH.**

![C++](https://img.shields.io/badge/C%2B%2B-17-00599C?logo=cplusplus&logoColor=white)
![Optimization](https://img.shields.io/badge/build-%2DO3-success)
![Dataset](https://img.shields.io/badge/dataset-946K%2B%20movies-blueviolet)



</div>

---

##  Overview

This project implements and experimentally compares **four multidimensional indexing structures** on the *Movies Metadata Cleaned* dataset (946K+ records). It follows a **two-phase pipeline**:

| Phase | Goal | Technique |
|-------|------|-----------|
| **1. Spatial filtering** | Range queries over 5 numeric attributes | k-d Tree, Quadtree (Hyperoctree), R-tree, 2D Range Tree |
| **2. Similarity search** | Find similar movies among the Phase 1 results by text | Shingling + MinHash + LSH |

**The 5 indexed dimensions:** `Budget` · `Revenue` · `Runtime` · `Release Year` · `Vote Average`

**The 3 searchable text fields:** `Title` · `Genres` · `Production Companies`

```
 ┌──────────────┐    range query     ┌────────────────┐   candidates   ┌──────────────────┐
 │ 946K movies  │ ─────────────────▶ │ Index (5D)     │ ─────────────▶ │ MinHash + LSH    │ ─▶ similar movies
 │ (CSV)        │  budget, revenue,  │ kd / quad /    │                │ 3-shingles       │
 └──────────────┘  runtime, year,    │ R-tree / range │                │ 100 hash funcs   │
                   vote avg          └────────────────┘                └──────────────────┘
```

---

##  Implemented Data Structures

### 1. k-d Tree (k = 5)
- Splits cyclically by axis: `cd = depth mod 5`.
- Built with `std::nth_element` (median split in **O(n)** per level instead of a full **O(n log n)** sort), producing a nearly perfectly balanced tree.
- Range search prunes whole subtrees when the query box doesn't extend across the node's splitting value.
- **Insert:** descends to an empty leaf position; ties are broken by `Movie ID`.
- **Delete:** replaces the node with the minimum node of the relevant subtree (`findMinRecursive`) to preserve the ordering invariant.
- **Update:** delete + re-insert.

### 2. Quadtree → Hyperoctree (2⁵ = 32 children)
- Generalization of the classic Quadtree to 5 dimensions.
- Root bounds are computed from the dimensions selected for the query.
- Each node is a **bucket** (`Q_BUCKET_SIZE`); when a full leaf receives a new movie it **splits** at the midpoint of its bounding box.
- The child index (0–31) is computed with bitwise OR + shift: the *i*-th bit is `1` if the value in dimension *i* is above the midpoint.
- Search recursively intersects the node box with the query box; non-intersecting subtrees are pruned. Leaves are scanned exhaustively.
- **Insert / Remove / Update** are supported (update = remove + insert).

### 3. R-tree
- Groups points into **Minimum Bounding Rectangles (MBRs)** (`min_coords` / `max_coords`, 5 values each).
- Range search skips every branch whose MBR doesn't overlap the query box.
- **Insert:** expands the parent MBRs (BVH-style hierarchy management).
- **Delete:** finds the leaf by `Movie ID`, removes it, then `recalculateMBR` propagates tightened bounds upward.
- **Update:** delete + re-insert.

### 4. 2D Range Tree (+ linear filter)
- A full 5D range tree is too memory hungry, so a **2D Range Tree** is built on `(Budget, Revenue)`.
- Primary tree: balanced BST on **Budget**. Each internal node holds an associated BST (`nextDimTree`) on **Revenue** for its subtree.
- Query steps: **find split node → collect canonical subtrees → 1D range query in each associated tree → linear filter** (`isMatchRest`) on `Runtime`, `Release Year`, `Vote Average`.
- **Insert** updates every associated tree along the path; **Remove** works by `Movie ID` and removes the movie from the main tree and from all ancestors' associated trees.
- Space **O(n log n)**, search **O(log² n + k)**.

---

##  Phase 2 — Similarity Search with LSH

After range filtering, the user can search by **Title**, **Genres** or **Production Companies**:

1. **Shingling** — text is converted to 3-shingles.
2. **MinHash** — 100 hash functions `h(x) = (a·x + b) mod p` compress each shingle set into a compact signature that approximates **Jaccard similarity**.
3. **LSH** — signatures are hashed into buckets so candidate pairs are found without comparing everything against everything.

### Example outputs

Filter used: `budget > 100M` and `release year > 2000` (2,081 movies indexed).

| Query | Field | Results | Highlights |
|-------|-------|:-------:|------------|
| `scary` | Title | 16 | *Scary Movie* (34%), *Madagascar* (33%), *Scary Movie 2/3/4* |
| `Universal` | Company | 125 | *American Reunion*, *Snow White and the Huntsman*, *Despicable Me*, *Minions* |
| `Horrr` *(deliberate typo)* | Genre | 102 | *The Conjuring: Last Rites*, *Freddy vs. Jason*, *Saw II*, *Pet Sematary*… |

> 💡 The `Horrr` query still returns Horror movies — shingling makes the search **tolerant to typos**.

---

## 📊 Experimental Results

Measured with `<chrono>` on the full dataset (946K+ movies).
Hardware: **Apple M2, 3.49 GHz**. Compiled with `-O3`.

### Build time

| Structure | Build time | Notes |
|-----------|-----------:|-------|
| R-tree | **58 ms** | MBR computation on insert |
| Quadtree | 122 ms | 2ᵏ children per node → higher memory use |
| k-d Tree | 190 ms | Simple median-based splitting |
| Range Tree | 2146 ms | Slowest: nested associated trees |

### Search time — 2 dimensions
`budget 10M–100M`, `release year 2000–2025` → **204,342 candidates** (identical across all structures ✅)

| Structure | Search time |
|-----------|------------:|
| Range Tree | **21 ms** |
| k-d Tree | 26 ms |
| Quadtree | 26 ms |
| R-tree | 45 ms |

### Search time — 5 dimensions
`budget 10M–100M`, `release year 2000–2025`, `revenue 1M–1B`, `runtime 90–120 min`, `vote average 7–10` → **4,491 candidates** (identical across all structures ✅)

| Structure | Search time |
|-----------|------------:|
| k-d Tree | **2 ms** |
| Quadtree | 5 ms |
| Range Tree | 30 ms |
| R-tree | 81 ms |

### Takeaways
- **k-d Tree** is the best all-rounder: fast to build and the fastest on the full 5D query.
- **Range Tree** wins on the 2D query it was designed for, but is by far the slowest to build and degrades once the remaining 3 dimensions need linear filtering.
- **Quadtree** is competitive on search, at the price of higher memory use from 32-way branching.
- **R-tree** builds fastest but searches slowest here — with only 5 dimensions and point data, MBR overlap hurts pruning.

---

##  Project Structure

> Adjust file names to match your repo.

```
.
├── data/
│   └── data_movies_clean.csv      # Movies Metadata Cleaned dataset
├── src/
│   ├── Movie.h                    # Movie struct (15 fields)
│   ├── FileReader.cpp             # CSV loader → std::vector<Movie>
│   ├── KDTree.*                   # k-d Tree (k = 5)
│   ├── Quadtree.*                 # Hyperoctree (32 children)
│   ├── RTree.*                    # R-tree with MBRs
│   ├── RangeTree.*                # 2D Range Tree + linear filter
│   ├── LSH.*                      # Shingling, MinHash, LSH
│   └── main.cpp                   # Interactive demo + benchmarks
├── report/
│   └── Report_2025-2026.pdf
└── README.md
```

---

##  Getting Started

### Requirements
- A C++17 compiler (`g++` / `clang++`)
- The dataset file `data_movies_clean.csv`

### Build

```bash
g++ -std=c++17 -O3 src/*.cpp -o movie_search
```

### Run

```bash
./movie_search
```

### Demo flow
1. The program loads `data_movies_clean.csv` into the `Movies` vector.
2. Choose an index structure and the dimensions/ranges to filter on (e.g. `budget 10M–100M`, `year 2000–2025`).
3. Build/search times and candidate counts are printed.
4. Optionally run **LSH** on the results: pick Title / Genres / Companies and type your query.

---

##  Complexity Summary

| Structure | Build | Range search | Space |
|-----------|-------|--------------|-------|
| k-d Tree | O(n log n) | pruned traversal | O(n) |
| Quadtree (bucketed) | O(n · depth) | prunes by box intersection | O(n) + overhead |
| R-tree | hierarchical MBR construction | O(log_M n) for balanced trees | O(n) |
| 2D Range Tree | O(n log n) | O(log² n + k) | O(n log n) |

---






---

<div align="center">

⭐ If you found this project useful, consider giving it a star!

</div>
