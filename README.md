# Gem5-System-Level-Cache-NVM-Performance-Analysis

This repository contains ongoing implementation and research for analyzing the performance trade-offs between conventional **DRAM** and **Non-Volatile Memory (NVM)** architectures using the **Gem5 Simulator**.

##  Status: Work in Progress (WIP)
*Currently under active development for the CS322: Computer Architecture curriculum at IIT Guwahati.*

## Project Objective
The goal is to quantify how memory latency affects system-level throughput when executing **SPEC CPU 2017** benchmarks. The project focuses on:
* Implementing precise **cache hit/miss counting** within specific clock-cycle observation windows.
* Emulating emerging memory technologies like **PCM (Phase Change Memory)** by overriding hardware timing parameters.

## Technical Stack
* **Simulator:** Gem5 (v20.1+ / stdlib 23.x)
* **Languages:** C++ (Core logic), Python (Hardware configuration)
* **Tools:** Linux, Git, Makefile

## Current Implementation Details

### Part A: Cache Hierarchy Analysis (C++)
Modifying `gem5/src/mem/ruby/structures/CacheMemory.cc` to count demand hits/misses for **L1-I, L1-D, and L2 caches**.
* **Gated Profiling:** Using `curTick()` to track statistics only within assigned cycle windows (TS to TE).
* **Cycle-to-Tick Conversion:** Implementing logic to handle the 500-tick-per-cycle conversion required for Gem5's Ruby memory system.

### Part B: NVM Emulation (Python)
Developing custom classes in `DRAMInterface.py` to emulate **NV1 (PCM)**, **NV2 (STT-MRAM)**, and **NV3 (ReRAM)** profiles.
* **Latency Scaling:** Overriding `tRCD`, `tCL`, `tRP`, and `tWR` parameters to model NVM's significant write-path penalties (up to 15x base DRAM).
* **Throughput Comparison:** Measuring the delta in total execution cycles between **DDR4-2400** baselines and NVM configs.

---
*Note: This repository is intended for academic research and evaluation within the IIT Guwahati Computer Science department.*
