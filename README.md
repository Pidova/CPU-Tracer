# CPU-Tracer

A header-only C++ library for loading, reconstructing, and analyzing CPU execution traces.

## Example Usage: Instruction Frequency Profile

Below is the distribution of the 25 most common x86 instructions executed during a Windows Vista boot-to-usage trace. 
This profile provides insight into the instruction composition across more than 8.3 million tracked instructions.
```text
mov    [ 35.9%] ##############                           1588665
call   [  6.3%] ##                                       278470
lea    [  6.3%] ##                                       277325
cmp    [  5.7%] ##                                       251354
je     [  4.7%] #                                        207601
test   [  4.5%] #                                        197455
add    [  3.6%] #                                        158900
xor    [  3.5%] #                                        154274
jne    [  3.4%] #                                        150383
push   [  3.1%] #                                        135227
dec    [  2.8%] #                                        122873
pop    [  2.6%] #                                        115388
sub    [  2.4%]                                          106038
jmp    [  2.0%]                                          86519
and    [  1.7%]                                          76460
ret    [  1.5%]                                          67016
movzx  [  1.4%]                                          60790
inc    [  1.2%]                                          54734
js     [  0.6%]                                          25237
jb     [  0.6%]                                          24856
or     [  0.5%]                                          24151
shr    [  0.4%]                                          19847
nop    [  0.4%]                                          19652
jae    [  0.4%]                                          16983
shl    [  0.4%]                                          16173
```


## Overview

Lightweight, header-only C++ library for working with raw per-block execution traces emitted using a dynamic binary instrumentation.
It provides reusable tools that turn a serialized trace (basic blocks, edges, interrupts, VCPU state changes, MMIO accesses) into structured
data and a Control Flow Graph (CFG) using Boost Graph Library (BGL) to make integration into existing projects easier.

Includes:
* **Block Format** – Self-describing, tagged binary format for basic blocks, instructions, edges, interrupts, VCPU states, and MMIO/Special Memory accesses, with LZ4-compressed edges.
* **Loading & Analysis** – Stream a trace file into memory, sort and index blocks by global ID and real PC, and reconstruct successor/predecessor edges from raw execution data, interrupts, and self-modifying/retranslated blocks.
* **Graph Construction** – Build a deterministic ```boost::adjacency_list``` CFG from analyzed blocks and edges.

## Importing

Importing all library functions just requires
```
#include "cpu_tracer/common.hpp"
```

## Libraries

Some libraries are used to reduce ABI reliant issues and increase speed

* [BoostPP flat vector](https://github.com/Pidova/BoostPP/blob/main/vector.hpp) **e.g. boost::fixed_vector<std::uint8_t, MAX_LEN, std::uint8_t>** - 
MAX Len is maximum size the data is expected to have, stored as an contigous array with the 3rd template argument describing internal size tracker type.

- [Boost smart pointers, vectors, maps](https://www.boost.org/)

## Documentation and Examples

All documentation and examples for each can be found:

- [Block Format](docs/blocks.md) – The on-disk layout of blocks, instructions, trace records, and terminology
- [Loading & Analysis](docs/loading_and_analysis.md) – Stream a trace file into memory and reconstruct its edges
- [Graph Construction](docs/graph.md) – Build a Boost Graph Library CFG from analyzed trace data
