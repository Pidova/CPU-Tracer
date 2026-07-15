# Block Building

Lightweight, header-only C++ tool used to construct basic blocks and control-flow edges manually, without reading serialized traces.

## What is Block Building?

[Loading & Analysis](loading_and_analysis.md) reconstructs blocks and edges from captured execution traces. 
Block building on the other hand lets you emit instructions, labels, and edges directly in memory, then constructs them into the same save_data, analyzed_data, and edge_data structures the loader, so downstream steps like 
[Graph Construction](graph.md) treat hand-built traces and captured traces identically.

Example uses: 
* Unit tests
* Synthetic control-flow experiments
* reproducing self-modifying-code (SMC) edge cases without instrumenting real hardware

**builder\<MAX_LEN>** holds three private stores:

* **insts** - instructions in program order; every emit() sets real_pc equal to the instruction's index, so real PC doubles as its position
* **label_map** – label -> index of the instruction the label precedes
* **label_idx_map** – index -> label, the reverse lookup used when detecting block leaders

## Idea

Given four emitted instructions, one label, one branch, and one signaled edge:

```
emit x4                  -> insts[0..3]
emit_label(0x200)        -> label 0x200 begins insts[2]
insts[1] jumps to 0x200  -> branch    1 -> 2
connect_edge(0, 3)       -> signaled  3 -> 0   (SMC re-entry)
```

build() cuts the flat stream into blocks at every leader, then resolves every emission into typed edges:

```
real_pc:  0   1  |  2   3        (marks the leader boundary)
blocks:   [A: 0,1]  [B: 2,3]

successors:
  A -> B          (fall-through 1 -> 2, merged with branch 1 -> 2)
  B -> A          (signaled 3 -> 0, the SMC edge)
```

## Algorithm

### Emission

Three entry points feed the builder before build() runs:

* **emit(bytes, interp_mode, [prev_realpc], [pc], [jump_to_labels])** – appends one instruction and returns its {real_pc, pc} pair. real_pc is set to the current instruction count, so it always equals the instruction's index; prev_real_pc defaults to real_pc - 1, chaining sequential flow, unless overridden.
* **emit_label(label)** – records that the next emitted instruction begins label; returns false when the label was already emitted.
* **connect_edge\<k>(dst_realpc, src_realpc, kind)** – stores one incoming edge on the destination instruction. This is how signaled / SMC transitions get tracked.

Outgoing edges (jump_to_labels) live on the source instruction; incoming edges (jump_to_realpc) live on the destination. build() folds both together.

* Time – O(1) per emission
* Memory – O(N) N instructions

### Leader detection (first pass)

build() resets all three outputs, then scans for **leaders**, instructions that begin new blocks. One index becomes leader when it is:

* The first instruction
* The target of some label
* The destination of some edge (its jump_to_realpc is non-empty)
* The fall-through directly after any branch source (label jump or connect_edge source).

* Time – O(N + E) over instructions and edges
* Memory – O(L) for the leader set

### Block cutting (second pass)

Instructions between consecutive leaders form one **block\<MAX_LEN>**. 
Each block receives the next running id, its loc (front instruction's virtual PC), inst_count, and interp mode (taken from its first instruction). 
Every instruction is marked fvalid, then the finished block is keyed into save_data::block_map by its front real PC.

* Time – O(N)
* Memory – O(N) for the block copies

### Indexing (reuses analyze)

With blocks in place, build() calls the same analyze::analyze(adata, sdata) the loader relies on, so biggest_block_id, sorted_blocks_global_id, real_pc_map, and inst_map come from identical, already-tested code.

* Time – O(N log N) dominated by block sort
* Memory – O(N)

### Edge resolution (final pass)

Two edge sources merge into edge_data:

1. **Sequential flow** – every instruction whose prev_real_pc differs from its own real PC (and whose fvalid_prev_real_pc stays set) emits one next edge from prev_real_pc to real_pc, matching the loader's cross-boundary stitching. Clearing fvalid_prev_real_pc, or pointing prev_realpc at the instruction itself, fall-through after unconditional jumps.
2. **Tracked emissions** – every jump_to_labels entry resolves its label through label_map and emits source -> target; every jump_to_realpc entry emits source -> destination. Both keep their original kind, so signaled / SMC edges stay signaled rather than collapsing into *next*.

Every resolved branch is also inserted into save_data::edge_map as one jmp_loc, so hand-built traces round-trip through save/load like captured ones.

* Time – O(N + E)
* Memory – O(E) for the successor/predecessor maps

## Determinism

Instruction indices, block IDs, and leader boundaries are assigned in strict emission order, so two runs over identical emissions have identical blocks and the same edge set every time. build() fully resets sdata, adata, and edata on entry, 
so one builder instance replays into fresh outputs repeatedly without it bleeding.

## Example

Hand-building two blocks joined by one branch and one signaled SMC edge, then reading the result out of edge_data:

```cpp
#include <iostream>
#include "cpu_tracer/common.hpp"

static constexpr auto MAX_LEN = 15u;

int main() {

    cpu_tracer::blocks::builder::builder<MAX_LEN> b;
	cpu_tracer::inst_bytes<MAX_LEN> nop{ 0x90 };

	/* Block A: two instructions, the second branches to label 0x200 */
	{
		b.emit(nop, cpu_tracer::archs::interpretation_mode::x64);                                       /* real_pc 0 */
	}

	/* Label container */
	boost::container::vector<std::pair<cpu_tracer::address, cpu_tracer::blocks::edges::kind>> to_label{ {0x200u,  cpu_tracer::blocks::edges::kind::next} };

	b.emit(nop, cpu_tracer::archs::interpretation_mode::x64, std::nullopt, std::nullopt, to_label); /* real_pc 1 */

	/* Label 0x200 begins the next emitted instruction (real_pc 2) */
	b.emit_label(0x200u);

	/* Block B: two instructions starting at label 0x200 */
	{
		b.emit(nop, cpu_tracer::archs::interpretation_mode::x64);                                       /* real_pc 2 */
		b.emit(nop, cpu_tracer::archs::interpretation_mode::x64);                                       /* real_pc 3 */
	}

	/* Signaled SMC re-entry: real_pc 3 signals back into real_pc 0 */
	b.connect_edge<cpu_tracer::blocks::edges::kind::signaled>(0u, 3u, cpu_tracer::blocks::edges::kind::signaled);

	/* Outputs */
	cpu_tracer::blocks::loader::save_data<MAX_LEN> sdata;
	cpu_tracer::blocks::loader::analyze::analyzed_data<MAX_LEN> adata;
	cpu_tracer::blocks::loader::analyze::edge_data edata;
	b.build(sdata, adata, edata);

	std::cout << "Built " << sdata.block_map.size() << " blocks\n";
	for (const auto& [src, dsts] : edata.successors) {
		std::cout << "rpc " << src << " ->";
		for (const auto& e : dsts) {
			std::cout << " " << e.dst_realpc;
		}
		std::cout << "\n";
	}
	std::cin.get();
    return 0;
}
```

Output:

```
Built 2 blocks
rpc 0 -> 1
rpc 1 -> 2
rpc 2 -> 3
rpc 3 -> 0
```
