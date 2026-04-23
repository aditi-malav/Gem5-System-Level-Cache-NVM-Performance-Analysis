# Gem5 Memory Subsystem & NVM Emulation

## 1. Project Overview
This project involves conducting a detailed architectural simulation of a memory subsystem using the gem5 simulator. The objective is divided into two main parts:
* **Part A:** Profiling demand hits and misses exclusively within the L2 cache during a specific execution window.
* **Part B:** Emulating STT-MRAM (Spin-Transfer Torque Magnetic RAM) to observe the performance impact of Non-Volatile Memory latencies compared to standard DDR3 DRAM.

## 2. System & Workload Configuration
This simulation was executed using the specific parameters assigned to **Group 24**.

* **Cache Configuration (CC3a):** 128 KiB L1-Instruction, 256 KiB L1-Data, 4 MiB L2.
* **Workload:** 510.parest_r (SPEC CPU 2017). This is a floating-point finite element solver.
* **Base Memory Technology (MT1):** DDR3-1600 8x8.
* **NVM Profile (NV2):** STT-MRAM (2× read, 5× write).
* **Simulation Scope:**
    * **Clock Cycle Conversion:** 1 clock cycle equals 500 ticks.
    * **Fast-Forward Phase:** 1,000,000 instructions to bypass OS boot.
    * **Max Instructions:** 10,000,000 instructions.
    * **Observation Window:** 8,000,000 to 9,000,000 clock cycles. This corresponds to simulation ticks 4,000,000,000 to 4,500,000,000.

## 3. Part A: L2 Cache Profiling (Task T3)

### Methodology
To successfully capture statistics specifically for the L2 cache, two primary filters were implemented in the C++ backend:
1.  **Time Window Filtering:** The `curTick()` function, which acts as gem5's global simulation clock, was used to restrict counting strictly to the assigned 4,000,000,000 to 4,500,000,000 tick window.
2.  **Object Filtering:** Because gem5 uses a single `CacheMemory` class for all cache levels, a string check (`name().find("L2cache") != std::string::npos`) was required to isolate L2 statistics and prevent L1 accesses from polluting the data.

### Simulation Results

| Metric | Total Global (Unfiltered L1+L2) | L2 Specific (Filtered) |
| :--- | :--- | :--- |
| **Hits** | 242,462 | 319 |
| **Misses** | 2,830 | 2,830 |
| **Miss Rate** | 1.15% | 89.87% |

### Conceptual Insights: The L1 Filter Effect
While an 89.87% miss rate at the L2 cache appears extremely high, the global miss rate of just 1.15% indicates a highly efficient memory hierarchy.
* **Large L1 Catchment:** The CC3a configuration possesses oversized L1 caches (128 KiB L1-I and 256 KiB L1-D). These caches absorbed 242,143 hits, effectively filtering ~99.87% of all memory requests before they could reach L2.
* **Spatial Locality:** The 510.parest_r workload operates on structured mesh data. This results in excellent spatial locality, allowing the L1 cache lines to service subsequent accesses efficiently.
* **L2 as a Last Resort:** The hierarchy rule dictates that only cold misses or capacity/conflict misses propagate down. Consequently, the requests the L2 cache receives are inherently "hard" misses, naturally driving up its local miss rate.

## 4. Part B: NVM Emulation (Task T4)

### Methodology
To emulate STT-MRAM, a new Python class `NVM_STTMRAM` was created inside `DRAMInterface.py`. This class inherited from the base `DDR3_1600_8x8` structure. Inheritance was strategically chosen to preserve standard DDR3 characteristics while exclusively overriding the read and write path latency timings.

The NV2 profile dictates a 2× latency penalty for read operations and a 5× penalty for write operations.

### Emulation Results

| Configuration | Total Execution Cycles | CPI | IPC |
| :--- | :--- | :--- | :--- |
| **DRAM Baseline** (DDR3-1600) | 17,024,410 | 4.040 | 0.247 |
| **NVM Emulation** (STT-MRAM) | 17,899,026 | 4.247 | 0.235 |

### Conceptual Insights: Latency Asymmetry
Replacing standard DRAM with STT-MRAM inflated total execution cycles by 874,616.
* **Read vs. Write Physics:** STT-MRAM stores data via the magnetic orientation of a magnetic tunnel junction (MTJ). Reading this state requires a small, non-destructive sense current, resulting in a 2× latency overhead vs DRAM.
* **The Write Penalty:** Writing to STT-MRAM requires physically flipping the magnetic layer using a spin-polarized current to overcome a thermal stability barrier. This probabilistic magnetic switching forces a massive 5× write latency penalty.
* **CPU Stalls:** Every L2 cache miss (2,830 in total) required an access to this slower emulated memory, accumulating significant CPU stall cycles and driving the final CPI from 4.040 to 4.247.

## 5. Source Code Modifications

The following files were modified in the gem5 source tree to accomplish this simulation.

**1. `gem5/src/mem/ruby/structures/CacheMemory.hh`**
```cpp
uint64_t window_hits = 0; 
uint64_t window_misses = 0;
```

**2. `gem5/src/mem/ruby/structures/CacheMemory.cc`**
```cpp
void CacheMemory::profileDemandHit() {
    cacheMemoryStats.m_demand_hits++; 
    if (curTick() >= 4000000000 && curTick() <= 4500000000) { 
        if (name().find("L2cache") != std::string::npos) {
            window_hits++; [cite: 251]
            std::cout << "L2_WINDOW_HIT: " << window_hits << std::endl;
        }
    }
}

void CacheMemory::profileDemandMiss() { 
    cacheMemoryStats.m_demand_misses++; 
    if (curTick() >= 4000000000 && curTick() <= 4500000000) {
        if (name().find("L2cache") != std::string::npos) { 
            window_misses++; [cite: 261]
            std::cout << "L2_WINDOW_MISS: " << window_misses << std::endl; 
        }
    }
}
```

**3. `gem5/src/mem/DRAMInterface.py`**
```python
class NVM_STTMRAM(DDR3_1600_8x8): 
    tRCD = '27.5ns'
    tCL = '27.5ns' 
    tRP = '27.5ns' 
    tRAS = '70ns' 
    tWR = '75ns' 
    tRTP = '37.5ns'
```







