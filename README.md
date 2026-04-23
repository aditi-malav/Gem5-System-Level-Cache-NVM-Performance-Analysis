# Gem5 Memory Subsystem & NVM Emulation

## 1. Project Overview
[cite_start]This project involves conducting a detailed architectural simulation of a memory subsystem using the gem5 simulator[cite: 124, 268]. The objective is divided into two main parts:
* [cite_start]**Part A:** Profiling demand hits and misses exclusively within the L2 cache during a specific execution window[cite: 5, 273].
* [cite_start]**Part B:** Emulating STT-MRAM (Spin-Transfer Torque Magnetic RAM) to observe the performance impact of Non-Volatile Memory latencies compared to standard DDR3 DRAM[cite: 61, 212, 273].

## 2. System & Workload Configuration
[cite_start]This simulation was executed using the specific parameters assigned to **Group 24**[cite: 2, 5, 123].

* [cite_start]**Cache Configuration (CC3a):** 128 KiB L1-Instruction, 256 KiB L1-Data, 4 MiB L2[cite: 5, 126].
* [cite_start]**Workload:** 510.parest_r (SPEC CPU 2017)[cite: 5, 125]. [cite_start]This is a floating-point finite element solver[cite: 5, 84].
* [cite_start]**Base Memory Technology (MT1):** DDR3-1600 8x8[cite: 5, 295].
* [cite_start]**NVM Profile (NV2):** STT-MRAM (2× read, 5× write)[cite: 5, 297].
* **Simulation Scope:**
    * [cite_start]**Clock Cycle Conversion:** 1 clock cycle equals 500 ticks[cite: 27, 271].
    * [cite_start]**Fast-Forward Phase:** 1,000,000 instructions to bypass OS boot[cite: 5, 93, 127].
    * [cite_start]**Max Instructions:** 10,000,000 instructions[cite: 5, 271].
    * [cite_start]**Observation Window:** 8,000,000 to 9,000,000 clock cycles[cite: 5, 29]. [cite_start]This corresponds to simulation ticks 4,000,000,000 to 4,500,000,000[cite: 30, 34].

## 3. Part A: L2 Cache Profiling (Task T3)

### Methodology
To successfully capture statistics specifically for the L2 cache, two primary filters were implemented in the C++ backend:
1.  [cite_start]**Time Window Filtering:** The `curTick()` function, which acts as gem5's global simulation clock, was used to restrict counting strictly to the assigned 4,000,000,000 to 4,500,000,000 tick window[cite: 32, 34, 250, 260].
2.  [cite_start]**Object Filtering:** Because gem5 uses a single `CacheMemory` class for all cache levels [cite: 43][cite_start], a string check (`name().find("L2cache") != std::string::npos`) was required to isolate L2 statistics and prevent L1 accesses from polluting the data[cite: 54, 56, 58, 251, 261].

### Simulation Results

| Metric | Total Global (Unfiltered L1+L2) | L2 Specific (Filtered) |
| :--- | :--- | :--- |
| **Hits** | [cite_start]242,462 [cite: 10, 133] | [cite_start]319 [cite: 10, 170] |
| **Misses** | [cite_start]2,830 [cite: 10, 133] | [cite_start]2,830 [cite: 10, 170] |
| **Miss Rate** | [cite_start]1.15% [cite: 22, 133] | [cite_start]89.87% [cite: 84, 170] |

### Conceptual Insights: The L1 Filter Effect
[cite_start]While an 89.87% miss rate at the L2 cache appears extremely high, the global miss rate of just 1.15% indicates a highly efficient memory hierarchy[cite: 21, 22, 106].
* [cite_start]**Large L1 Catchment:** The CC3a configuration possesses oversized L1 caches (128 KiB L1-I and 256 KiB L1-D)[cite: 12, 13, 207]. [cite_start]These caches absorbed 242,143 hits, effectively filtering ~99.87% of all memory requests before they could reach L2[cite: 10, 209].
* [cite_start]**Spatial Locality:** The 510.parest_r workload operates on structured mesh data[cite: 16]. [cite_start]This results in excellent spatial locality, allowing the L1 cache lines to service subsequent accesses efficiently[cite: 16, 109, 110].
* [cite_start]**L2 as a Last Resort:** The hierarchy rule dictates that only cold misses or capacity/conflict misses propagate down[cite: 18, 19]. [cite_start]Consequently, the requests the L2 cache receives are inherently "hard" misses, naturally driving up its local miss rate[cite: 21, 118].

## 4. Part B: NVM Emulation (Task T4)

### Methodology
[cite_start]To emulate STT-MRAM, a new Python class `NVM_STTMRAM` was created inside `DRAMInterface.py`[cite: 237, 238]. This class inherited from the base `DDR3_1600_8x8` structure[cite: 95, 238]. [cite_start]Inheritance was strategically chosen to preserve standard DDR3 characteristics while exclusively overriding the read and write path latency timings[cite: 96, 97].

[cite_start]The NV2 profile dictates a 2× latency penalty for read operations and a 5× penalty for write operations[cite: 63, 67, 297].

### Emulation Results

| Configuration | Total Execution Cycles | CPI | IPC |
| :--- | :--- | :--- | :--- |
| **DRAM Baseline** (DDR3-1600) | [cite_start]17,024,410 [cite: 80, 214] | [cite_start]4.040 [cite: 80, 214] | [cite_start]0.247 [cite: 214] |
| **NVM Emulation** (STT-MRAM) | [cite_start]17,899,026 [cite: 80, 214] | [cite_start]4.247 [cite: 80, 214] | [cite_start]0.235 [cite: 214] |

### Conceptual Insights: Latency Asymmetry
[cite_start]Replacing standard DRAM with STT-MRAM inflated total execution cycles by 874,616[cite: 80, 81, 233].
* [cite_start]**Read vs. Write Physics:** STT-MRAM stores data via the magnetic orientation of a magnetic tunnel junction (MTJ)[cite: 61]. [cite_start]Reading this state requires a small, non-destructive sense current, resulting in a 2× latency overhead vs DRAM[cite: 64, 65, 66].
* [cite_start]**The Write Penalty:** Writing to STT-MRAM requires physically flipping the magnetic layer using a spin-polarized current to overcome a thermal stability barrier[cite: 68, 71]. [cite_start]This probabilistic magnetic switching forces a massive 5× write latency penalty[cite: 67, 69, 73].
* [cite_start]**CPU Stalls:** Every L2 cache miss (2,830 in total) required an access to this slower emulated memory, accumulating significant CPU stall cycles and driving the final CPI from 4.040 to 4.247[cite: 77, 79, 82, 235].

## 5. Source Code Modifications

The following files were modified in the gem5 source tree to accomplish this simulation.

```cpp
//1. In CacheMemory.hh
uint64_t window_hits = 0;
uint64_t window_misses = 0;
```
```cpp
// 2. In CacheMemory.cc
void CacheMemory::profileDemandHit() {
    cacheMemoryStats.m_demand_hits++;
    if (curTick() >= 4000000000 && curTick() <= 4500000000) {
        if (name().find("L2cache") != std::string::npos) {
            window_hits++;
            std::cout << "L2_WINDOW_HIT: " << window_hits << std::endl;
        }
    }
}

void CacheMemory::profileDemandMiss() {
    cacheMemoryStats.m_demand_misses++;
    if (curTick() >= 4000000000 && curTick() <= 4500000000) {
        if (name().find("L2cache") != std::string::npos) {
            window_misses++;
            std::cout << "L2_WINDOW_MISS: " << window_misses << std::endl;
        }
    }
}
```
```python
# In DRAMInterface.py
class NVM_STTMRAM(DDR3_1600_8x8):
    tRCD = '27.5ns'
    tCL = '27.5ns'
    tRP = '27.5ns'
    tRAS = '70ns'
    tWR = '75ns'
    tRTP = '37.5ns'
```





