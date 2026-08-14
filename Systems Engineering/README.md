# Distributed HPC Systems Engineering 

This reading and topic list bypasses traditional web engineering and focuses entirely on the three layers of HPC: **Compute (The Processor), Interconnect (The Network), and Programming Models (The Parallelism).**

---

## 1. The Compute Layer: Computer Architecture & Memory

HPC engineers must understand exactly how a single CPU/GPU manages memory. If code causes memory stalls or cache misses, parallelizing it across a thousand nodes won't help.

### Core Topics:

* **The Memory Hierarchy:** Understanding the massive latency gaps between L1, L2, L3 caches, Main Memory (DRAM), High-Bandwidth Memory (HBM), and Disk.
* **Cache Locality:** Spatial vs. Temporal locality. Row-major vs. Column-major matrix traversal (and how it affects the CPU's prefetcher).
* **SIMD & Vectorization:** Single Instruction, Multiple Data. Forcing the CPU to compute multiple mathematical operations in a single clock cycle using AVX-512 or ARM Neon instruction sets.
* **GPU Architecture:** Stream Multiprocessors (SMs), Warps and Warp Divergence, Global Memory vs. Shared/Texture Memory.

### Key Terms to Google/Research:

> `False Sharing`, `Cache Lines`, `Memory Bandwidth Bound vs. Compute Bound`, `Roofline Model`, `AVX-512 Vectorization`, `Structure of Arrays (SoA) vs. Array of Structures (AoS)`, `Coalesced Memory Access`.

---

## 2. The Network Layer: High-Speed Interconnects

In an enterprise system, servers talk via HTTP over standard Ethernet. In HPC and Quant trading, standard networking is far too slow and creates a bottleneck.

### Core Topics:

* **Kernel Bypass:** Preventing the Linux Operating System kernel from touching network packets, allowing your application to talk directly to the Network Interface Card (NIC) to save microseconds.
* **Remote Direct Memory Access (RDMA):** Technologies that allow Server A to read or write directly into the RAM of Server B over the network without involving either server's CPU or OS.
* **Network Topologies:** How supercomputing clusters are physically wired to reduce data hopping (Fat-Tree, Torus, Dragonfly networks).

### Key Terms to Google/Research:

> `InfiniBand (NDR/EDR)`, `RoCE (RDMA over Converged Ethernet)`, `DPDK (Data Plane Development Kit)`, `SRIOV`, `NVLink`, `NCCL (NVIDIA Collective Communications Library)`.

---

## 3. The Parallel Programming Models (The Holy Grail)

This is the core software engineering toolkit of an HPC or Quant Infrastructure developer. You must learn how to write code that splits work across cores and machines.

### Core Topics:

* **Shared-Memory Parallelism (Single Machine):** Fork-join execution models, assigning loops to multiple CPU threads simultaneously.
* **Distributed-Memory Parallelism (Multi-Machine):** Explicitly passing chunks of memory over the network between isolated system nodes.
* **Heterogeneous Computing:** Offloading the mathematically dense, parallel parts of your program from the host CPU to an accelerator GPU.

### Key Terms & Frameworks to Learn:

* **OpenMP:** The industry standard for shared-memory multi-threading in C/C++ via compiler directives.
* **MPI (Message Passing Interface):** The absolute bedrock of cluster computing. Learn primitives like: `MPI_Send`, `MPI_Recv`, `MPI_Bcast`, `MPI_Scatter`, `MPI_Gather`, and `MPI_Allreduce`.
* **CUDA / Triton:** Writing custom code kernels to run directly on NVIDIA graphics cards.
* **POSIX Threads (pthreads):** Low-level, explicit thread management in C.

---

## 4. Systems Tooling & Performance Profiling

HPC developers spend half their lives using profilers to find out *why* a script is running slowly on hardware.

### Core Topics & Tools:

* **Compiler Optimizations:** Flags used when compiling C++ code to let the compiler automatically vectorize loops or reorder instructions safely.
* **Hardware Profilers:** Tools that look directly into the CPU/GPU counters to tell you exactly how many cache misses, branch mispredictions, or floating-point operations your code triggered.

### Key Terms & Tools to Learn:

> `gcc -O3 -march=native`, `Intel VTune Profiler`, `NVIDIA Nsight Systems`, `perf (Linux)`, `Google's magic-trace`, `Valgrind (Cachegrind)`.

---

## Recommended Reading List

1. [**"Computer Architecture: A Quantitative Approach"**](./Computer%20Architecture%20-%20A%20Quantitative%20Approach.pdf) *by John L. Hennessy and David A. Patterson* (The definitive bible on how modern computer hardware actually works).
2. [**"An Introduction to Parallel Programming"**](./An%20Introduction%20to%20Parallel%20Programming%20-%20Peter%20Pacheco.pdf) *by Peter Pacheco* (The best practical guide for learning MPI and OpenMP code design).
3. [**"Programming Massively Parallel Processors"**](./Programming%20Massively%20Parallel%20Processors%20-%20A%20Hand%20on%20Approach.pdf) *by David B. Kirk and Wen-mei W. Hwu* (The gold-standard book for learning CUDA and GPU system limitations).
