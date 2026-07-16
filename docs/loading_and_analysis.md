# Loading & Analysis

A lightweight, header-only C++ tool designed to stream a serialized trace file into memory and reconstruct its execution edges into a successor/predecessor map.

## What is Loading & Analysis?

A raw trace file is a flat, tagged stream of blocks, edges, interrupts, VCPU states, and MMIO records (see Block Format, blocks.md). 
On its own that stream isn't useful: blocks are unordered, edges are still unsafe (raw PC transitions, not validated control flow), 
and there's no way to look a block up by address or walk forward/backward from it.

Loading consumes the tagged stream once into a save_data container, keyed by record kind. Analysis then runs over that container twice: 
once to index and sort the blocks (analyzed_data), and once to fold raw edges, instruction-to-instruction transitions, and 
interrupts into one deduplicated successor/predecessor edge map (edge_data).

## Idea

Given a raw trace stream:

```
[block A][block B][edge_map][interrupts][block C][vcpu_states] ...
```

Loading produces an in-memory record set:

```
save_data {
  block_map:    { realpc(A) -> A, realpc(B) -> B, realpc(C) -> C }
  edge_map:     { jmp_loc, jmp_loc, ... }
  interrupts:   [ ... ]
  block_states: [ ... ]
  mmio:         [ ... ]
}
```

Analysis then turns that into an indexed, edge-resolved view:

```
analyzed_data {
  sorted_blocks_global_id: [A, B, C]   (sorted by block.id)
  real_pc_map:             { realpc -> block }
  inst_map:                { vaddr  -> [(block, inst), ...] }
}
edge_data {
  successors:   { realpc(A) -> {realpc(B)}, realpc(B) -> {realpc(C)} }
  predecessor:  { realpc(B) -> {realpc(A)}, realpc(C) -> {realpc(B)} }
}
```

## Algorithm

### Streaming load (loader::load)

A single loop peeks the next save_type tag and dispatches: block records are heap-allocated and keyed into block_map by the real PC of their first instruction; 
edge_map, vcpu_states, interrupts, and MMIO records are appended into their own containers; a none tag steps past a skip marker. 
Loading is one linear pass over the file, with backtracking only where a record's own reader rewinds.

- Time - O(N) in total bytes read
- Memory - O(N) for the resulting in-memory containers

### Block indexing (analyze::analyze over analyzed_data)

Each loaded block's instructions are indexed into inst_map by virtual PC, and the running maximum id is tracked as biggest_block_id 
(used later to assign new IDs during splitting). Blocks are copied into sorted_blocks_global_id and sorted by id; 
interrupts and VCPU states are separately sorted by curr_global_block_id so later passes can walk them in execution order. 
Finally, real_pc_map is built so any instruction's real PC resolves directly to its owning block.

- Time - O(N log N), dominated by the three sorts; O(N) for the indexing passes
- Memory - O(N) for the index and sorted-copy containers

### Edge reconstruction (analyze::analyze over edge_data)

This pass merges three sources of control flow into one edge map:

1. Raw edge records - every jmp_loc loaded from the trace's edge_map is emitted directly as a next edge.
2. Sequential execution - walking blocks in sorted global-ID order, an instruction whose prev_real_pc doesn't chain from the previous instruction is emitted as a next edge from prev_real_pc to real_pc. This stitches control flow across block boundaries.
3. Interrupts - walked alongside the sorted blocks by curr_global_block_id. An interrupt's source virtual address is resolved to a real PC and emitted as a signaled edge to the interrupt's destination. If the source vaddr isn't resolved yet, it's parked and retried once more instructions are indexed.

Each edge is inserted into both successors and predecessor at once; if the same source/destination pair already exists, its kind bits are updated rather than 
duplicated, so an edge that's both directly observed and inferred is recorded once.

- Time - O(N) amortized; pending-interrupt retries are bounded by the number of unresolved addresses
- Memory - O(E) for the successor/predecessor maps, where E is the number of distinct edges

### Block splitting (tools::fix_blocks)

Because edges are reconstructed independently of the original block boundaries, a destination address sometimes lands mid-block rather than at its start 
(e.g. a backward branch into a loop body traced as one larger block). For every successor address that resolves into the interior of a block, fix_blocks 
finds its instruction offset, splits the block at that offset, re-registers the new block in real_pc_map, and assigns it the next available ID.

- Time - O(```S * N```), where S is the number of split sites and N is the average block size
- Memory - O(S) for the newly created blocks

## Determinism

Block and edge order is deterministic as long as sorting uses a consistent comparison (id or curr_global_block_id) and containers are iterated consistently within one run. 
Tag types have assetions to make sure they are backwards compatible to older versions.

## Example

Loading a trace file, analyzing it, and reconstructing its edges:

```cpp
#include "cpu_tracer/common.hpp"
#include <iostream>

constexpr std::uint8_t MAX_LEN = 15u;

std::ifstream ifs("trace.bin", std::ios::binary);

/* Load save data */
cpu_tracer::blocks::loader::save_data<MAX_LEN> sdata;
cpu_tracer::blocks::loader::load(sdata, ifs);

/* Analyze save data */
cpu_tracer::blocks::loader::analyze::analyzed_data<MAX_LEN> adata;
cpu_tracer::blocks::loader::analyze::analyze(adata, sdata);

/* Analyze Edges */
cpu_tracer::blocks::loader::analyze::edge_data edata;
cpu_tracer::blocks::loader::analyze::analyze(edata, sdata, adata);

/* Repair any blocks that edges split mid-way through */
cpu_tracer::blocks::loader::tools::fix_blocks(edata, adata, sdata);

std::cout << "Loaded " << sdata.block_map.size() << " blocks, " << adata.biggest_block_id + 1 << " after splitting\n";

for (const auto &[src, dsts] : edata.successors) {
      std::cout << "0x" << std::hex << src << " ->";
      for (const auto &i : dsts) {
            std::cout << " 0x" << i.dst_realpc;
      }
      std::cout << std::dec << "\n";
}
```

Output:

```
Loaded 3 blocks, 4 after splitting
0x7f2a10001000 -> 0x7f2a10001010
0x7f2a10001010 -> 0x7f2a10001020 0x7f2a10001040
```
