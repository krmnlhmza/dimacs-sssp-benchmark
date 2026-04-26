[README.md](https://github.com/user-attachments/files/27100623/README.md)
# DIMACS SSSP Benchmark

Single-Source Shortest Path (SSSP) algorithms — comparative implementation and benchmark on the **9th DIMACS Implementation Challenge** New York City road network (264,346 nodes, 733,846 directed arcs).

This repository implements three algorithms — **Dijkstra**, **Bellman-Ford**, and a simplified **BMSSP** (inspired by the 2025 STOC paper by Ran Duan et al.) — measures their runtime and memory consumption across five graph sizes, and verifies correctness via cross-validation.

---

## Author

**Muhammed Hamza Karamanlı** · Student No: **202328036**

Implementation, benchmarking, and reporting are entirely my own work.

---

## Table of Contents

1. [Why SSSP Algorithms?](#why-sssp-algorithms)
2. [What We Test](#what-we-test)
3. [Dijkstra vs. Bellman-Ford — Key Differences](#dijkstra-vs-bellman-ford--key-differences)
4. [BMSSP — Inspired by Duan et al. 2025](#bmssp--inspired-by-duan-et-al-2025)
5. [Repository Structure](#repository-structure)
6. [Libraries Used](#libraries-used)
7. [Functions Written](#functions-written)
8. [How to Run](#how-to-run)
9. [Hardware Used for Benchmarking](#hardware-used-for-benchmarking)
10. [Results Summary](#results-summary)

---

## Why SSSP Algorithms?

The **Single-Source Shortest Path** problem asks: given a weighted graph and a source vertex, what is the minimum-cost path from the source to every other vertex? This is a foundational problem in computer science with applications across the industry:

- **GPS navigation and routing** (Google Maps, Yandex)
- **Network routing protocols** (OSPF, IS-IS in Internet infrastructure)
- **Logistics and supply chain optimization**
- **Game AI pathfinding**
- **Currency arbitrage detection** (negative-weight cycles)

The two classical solutions — Dijkstra (1959) and Bellman-Ford (1958) — have very different complexity profiles, and the choice between them is non-trivial in practice. This benchmark quantifies that difference on real-world data.

---

## What We Test

Three things, on the same DIMACS NY road network:

1. **Correctness.** All three algorithms must produce identical distance vectors on test graphs (cross-validation).
2. **Runtime scalability.** How does execution time grow as the graph gets larger (1K → 50K nodes)?
3. **Memory consumption.** Peak heap memory during execution.

We extract BFS subgraphs of varying sizes (1K, 5K, 10K, 25K, 50K) from the full 264K-node NY graph, plus a stress test on the complete graph for Dijkstra and BMSSP.

---

## Dijkstra vs. Bellman-Ford — Key Differences

These two algorithms solve the same problem with fundamentally different strategies:

| Property | Dijkstra | Bellman-Ford |
|---|---|---|
| **Strategy** | Greedy: pick smallest-distance vertex, relax its edges | Brute force: relax all edges V−1 times |
| **Time complexity** | O((V + E) log V) | O(V · E) |
| **Negative weights** | ❌ Fails (greedy invariant breaks) | ✅ Handles correctly |
| **Negative cycle detection** | ❌ Cannot detect | ✅ Built-in |
| **Data structure** | Priority queue (min-heap) | Edge list, no extra structure |
| **Practical speed** | Very fast | Significantly slower |

**The trade-off:** Dijkstra is the default for non-negative weights because it is dramatically faster — in our benchmark, Bellman-Ford was over **100× slower** at n=50K. But Bellman-Ford is essential when negative weights or cycle detection are required (e.g., financial arbitrage, constraint graphs).

In this project we test both on identical inputs and quantify exactly how much faster Dijkstra is on real road-network data — and at what scale Bellman-Ford becomes impractical.

---

## BMSSP — Inspired by Duan et al. 2025

The third algorithm in this benchmark is a **simplified educational implementation of BMSSP**, adapted from the recent paper:

> **Ran Duan, Jiayi Mao, Xiao Mao, Xinkai Shu, Longhui Yin.**
> *"Breaking the Sorting Barrier for Directed Single-Source Shortest Paths."*
> **STOC 2025** — Best Paper Award.

For over four decades, the best-known time complexity for directed SSSP was **O(m + n log n)**, dominated by the cost of sorting vertices by distance. Duan and his team broke this long-standing barrier in 2025, presenting a deterministic algorithm with complexity **O(m · log^(2/3) n)** — the first asymptotic improvement on the problem since the 1980s.

The core idea — captured in this implementation — is to abandon strict priority ordering. Instead of extracting one minimum-distance vertex at a time (which forces a sort), the algorithm processes vertices in **batches** of size t ≈ log^(2/3) n. Each batch is then relaxed in a Bellman-Ford-style sweep.

**Important disclosure:** The implementation in this repository captures the batch-processing framework and produces correct results (verified by cross-validation against Dijkstra), but it does **not** achieve the paper's full asymptotic bound — that would require the recursive BMSSP procedure with the specialized data structure described in Section 3 of the paper. This educational simplification was a deliberate choice: a correct, honest implementation is more valuable than a fabricated one.


---

## Libraries Used

All libraries are part of Python's standard library or commonly pre-installed in scientific Python distributions (Anaconda, Jupyter):

| Library | Why we used it |
|---|---|
| `os`, `gzip`, `urllib.request` | Download and read the gzipped DIMACS file |
| `math` | Compute the BMSSP batch size t = log₂(n)^(2/3) |
| `time` | High-resolution timing via `perf_counter()` |
| `tracemalloc` | Measure peak memory consumption during algorithm execution |
| `heapq` | Built-in min-heap for Dijkstra and BMSSP priority queues |
| `random` | Seeded random subgraph extraction in cross-validation |
| `statistics` | Compute mean and standard deviation across trials |
| `collections.deque` | O(1) queue operations for BFS subgraph extraction |
| `pandas` | Tabulate and export results to CSV |
| `matplotlib` | Generate the runtime and memory charts |
| `numpy` | Bar-chart positioning in matplotlib |

---

## Functions Written

The notebook is organized into 10 cells. Here is what each function does and why it exists:

### `download_dimacs()` — Cell 2
Downloads the DIMACS-NY gzipped graph file from the official DIMACS server. Includes a `User-Agent` header to avoid 403 errors on some servers. Skips download if the file already exists locally.

### `load_dimacs(path)` — Cell 2
Parses the DIMACS `.gr` file format. Reads the problem-line (`p sp V E`) for graph dimensions, then iterates over all arc-lines (`a u v w`). Builds two complementary representations:
- **Adjacency list** (`adj[u] = [(v, w), ...]`) for Dijkstra and BMSSP
- **Edge list** (`(u, v, w)` tuples) for Bellman-Ford

We need both formats because Dijkstra/BMSSP query "what are u's neighbors?" while Bellman-Ford iterates over every edge each round.

### `bfs_subgraph(adj, n, target_size, start)` — Cell 3
Extracts a connected subgraph of approximately `target_size` nodes via breadth-first search starting from a fixed vertex. Re-indexes the subgraph nodes to 0..k-1 so they fit in fresh arrays. This lets us benchmark on multiple sizes without modifying the original graph.

### `dijkstra(graph, src, n)` — Cell 4
Heap-based Dijkstra. Uses Python's `heapq` for the priority queue, with the lazy-deletion technique (skip outdated entries with `if d > dist[u]: continue` rather than decrease-key). Returns the distance array.

### `bellman_ford(edges, src, n)` — Cell 5
Bellman-Ford with two improvements over the textbook version:
1. **Early-termination optimization**: exit as soon as a full pass produces no relaxation.
2. **Negative-cycle detection**: a final V-th pass checks if any edge can still be relaxed.

Returns `(dist, has_negative_cycle)`.

### `bmssp(graph, src, n)` — Cell 6
Simplified BMSSP, based on Duan et al. 2025. Like Dijkstra, but pulls a batch of t ≈ log^(2/3) n vertices from the heap before relaxing edges. Skips outdated entries. Returns the distance array.

### `cross_validate(size, seeds)` — Cell 7
Runs all three algorithms on four randomly-seeded BFS subgraphs (n=500) and asserts that their distance vectors are identical. This proves implementation correctness — particularly important for BMSSP, where it confirms that the simplified version still produces correct shortest-path distances.

### `measure(func, args, trials)` — Cell 8
Generic benchmarking wrapper. Runs the given function `trials` times, recording runtime (via `perf_counter`) and peak memory (via `tracemalloc`). Returns mean time, standard deviation, and mean memory. Avoids duplicating timing logic across three algorithms.

### Main benchmark loop — Cell 9
For each graph size in `{1K, 5K, 10K, 25K, 50K}`:
1. Extract BFS subgraph from full DIMACS-NY.
2. Measure each of the three algorithms over 3 trials.
3. Append results to a list.

Results are exported to `dimacs_benchmark.csv`.

### Plotting code — Cell 10
Produces a side-by-side figure: log-log line chart for runtime and bar chart for memory. Saves as `dimacs_charts.png`.

### Full-graph stress test — Cell 11
Runs Dijkstra and BMSSP on the **complete** 264,346-node DIMACS-NY graph. Bellman-Ford is omitted at this scale (would take hours). Verifies cross-validation between the two on the full graph.

---

## How to Run

### Prerequisites
- Python 3.10+
- Jupyter Notebook
- All libraries listed above (most are standard; `pandas`, `matplotlib`, `numpy` may need `pip install`)

### Steps
```bash
git clone https://github.com/<your-username>/dimacs-sssp-benchmark.git
cd dimacs-sssp-benchmark
jupyter notebook ceng383hw.ipynb
```

Then run all cells in order. The first cell downloads the DIMACS NY file (~3.7 MB, one-time) and the rest takes a few minutes total.

---

## Hardware Used for Benchmarking

All measurements in `dimacs_benchmark.csv` and the report were taken on the following workstation. They are reproducible — run the notebook on similar hardware to confirm.

| Component | Specification |
|---|---|
| **CPU** | AMD Ryzen 7 7800X3D (8 cores / 16 threads, base ~4.7 GHz, boost up to 5.5 GHz) |
| **CPU cooling** | 360 mm AIO liquid cooler with 3× 120 mm fans, ~3600 RPM |
| **Motherboard** | ASUS X870 |
| **Memory** | 32 GB DDR5 |
| **GPU** | NVIDIA RTX 5060 Ti 16 GB VRAM (unused — pure CPU benchmark) |
| **Storage** | 1 TB NVMe M.2 SSD |
| **PSU** | 750 W |
| **OS** | Windows 11 |
| **Ambient temperature** | 26 °C |
| **Software** | Python 3.12, Jupyter Notebook |

---

## Results Summary

Full results in `dimacs_benchmark.csv`. Quick summary:

| n | m | Dijkstra (ms) | Bellman-Ford (ms) | BMSSP (ms) | BF/Dij slowdown |
|---|---|---|---|---|---|
| 1,000 | 2,438 | 2.95 | 16.92 | 3.96 | 5.7× |
| 5,000 | 12,126 | 9.19 | 152.99 | 12.88 | 16.6× |
| 10,000 | 23,954 | 18.62 | 785.06 | 26.56 | 42.2× |
| 25,000 | 62,164 | 53.87 | 3,102.63 | 74.56 | 57.6× |
| 50,000 | 130,522 | **108.11** | **11,717.98** | **150.36** | **108.4×** |

**Full-graph stress test (264,346 nodes):**
- Dijkstra: 234.5 ms
- BMSSP: 253.8 ms
- Cross-validation passed ✓

The 108× slowdown of Bellman-Ford at n=50K is a clean empirical demonstration of the gap between O(V·E) and O((V+E) log V). Detailed analysis is in the accompanying report (`CENG383_SSSP_Final.pdf`).

---

## License

This is a coursework project. The DIMACS dataset is publicly available under the terms specified at https://www.diag.uniroma1.it/challenge9/.

---

## References

- Cormen, Leiserson, Rivest, Stein. *Introduction to Algorithms* (4th ed.), MIT Press, 2022.
- Duan, Mao, Mao, Shu, Yin. *Breaking the Sorting Barrier for Directed Single-Source Shortest Paths.* STOC 2025 (Best Paper Award).
- 9th DIMACS Implementation Challenge — Shortest Paths: https://www.diag.uniroma1.it/challenge9/
- Dijkstra, E. W. *A note on two problems in connexion with graphs.* Numerische Mathematik 1, 1959.
- Bellman, R. *On a routing problem.* Quarterly of Applied Mathematics, 1958.
