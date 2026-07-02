# Blocks
Lightweight, header-only, fast C++ tool used to serialize basic blocks and trace data to disk.

## What is a Block?

During instrumentation, every translated basic block (TB) executed by a virtual CPU is captured along with the instructions inside it. 
A **block** is the unit of trace data: contiguous execution of instructions starting at a virtual PC, tagged with the time it was translated,
 a monotonically incrementing global ID, and per-instruction metadata describing how each instruction actually behaved at runtime (executed, taken branch, self-modified, etc).

Each trace gives execution data to help further context:

* **Edges** (jmp_loc) – unsafe, pre-disassembly source -> destination PC transitions captured directly from execution or special CPU jumps (e.g. X86s APIC ICR)
* **Interrupts** – exceptions/interrupts
* **VCPU states** – pause/resume snapshots of every other active VCPU, captured whenever one VCPU changes state
* **MMIO accesses** – memory-mapped I/O / special memory access reads/writes linked back to the instruction that caused it
* **Real PC** – logical PC defined thread-safe unique PC per new translation or new instruction/interpretation data; real PC will change

## Idea

Every record written to the trace file is prefixed with a one-byte **save_type** tag so the stream is self-describing and can be read in order without an external index:

```
[ save_type ][ payload ][ save_type ][ payload ]
```

## Layout

Each piece of data is designed to have the same time complexity:

* **Time** – O(n), number of data per vector used
* **Memory** – O(n), number of data per vector

### Tag dispatch (save_type)

Every record begins with one byte identifying its kind: **block**, **edge_map**, **vcpu_states**, **interrupts**, **MMIO**, or **none** (step/skip). 
**blocks::fs::get_save_type** peeks (or consumes) this tag so a reader can branch per-record without knowing the stream layout in advance.

### Block

A **block\<MAX_LEN>** payload is a fixed header followed by a flat array of per-instruction entries:

```
| time | id | fretranslated | loc | inst_count | interp | vcpu_n |
| insts[0]: flags | fvalid | inst { pc, real_pc, prev_real_pc, vcpu, bytes[] } |
| insts[1]: ...                                                                |
```

**Header fields:**

* **time** – timestamp the block was translated at
* **id** – block's monotonically incrementing global ID
* **fretranslated** – marks whether this block replaced an earlier translation at the same location (e.g. after self-modifying code)
* **loc** – block's starting virtual PC
* **inst_count** – number of instruction entries that follow the header
* **interp** – tells a disassembler which decoder to use for the bytes that follow
* **vcpu_n** – number of VCPUs referenced by this block

**Per-instruction fields (inst_data<MAX_LEN>):**

* **flags** – a **flag_storage** bitfield (see **flags::finsts**)
* **fvalid** – marks whether the instruction executed
* **inst** – a **basic_info::inst\<MAX_LEN>** holding:
  * **pc** – the virtual program counter
  * **real_pc** / **prev_real_pc** – real-PC pair
  * **vcpu** – the executing VCPU
  * **bytes[]** – the raw instruction bytes, capped at **MAX_LEN**

**block::write** / **block::read** serialize every header field with fixed-width **io::write** / **io::read** in declaration order, then walk the instruction vector the same way.

### Edges

Raw edges (**edges::jmp_loc**: a **kind** of either **signaled** or **next/unknown**, plus source/destination real PCs) are the highest-volume record in a trace, 
so rather than being written one at a time they're batched into a single **unordered_flat_set** and flushed as one LZ4-compressed blob.

* **edges::write** – packs the set into a tightly-packed **packed_jmp_loc** array (**#pragma pack(push, 1)**), compresses it with **LZ4_compress_fast**, and writes out the element count, compressed size, and compressed bytes
* **edges::read** – reverses this: reads both sizes, decompresses with **LZ4_decompress_safe**, checks the decompressed byte count matches **vector_size * sizeof(packed_jmp_loc)** exactly, and rebuilds the set, rewinding the stream and returning **false** if the decompressed size doesn't line up, so a corrupt or truncated edge blob never silently produces a partial edge set

### Interrupts, VCPU states, and MMIO

These three record kinds share the same shape: a **save_type** tag, a **std::size_t** element count, then that many fixed-size entries written back to back with **io::write** / **io::read** no compression,
since these records are relatively low-volume.

* **interrupts::interrupt** – carries the interrupt **type** (exception/interrupt/misc), the source and destination virtual/real PCs, the global block **id** it occurred during, and the VCPU it occurred on
* **vcpu::captured_block_state** – carries which VCPU changed state, whether it has a recent real PC and what that real PC is, the new **state** (**PAUSED**/**RESUME**), and the current global block ID at the moment of the transition
* **mmio::data** – the smallest record, carrying only the destination address and the real PC of the instruction that issued the access

## Determinism

Every field in a block, interrupt, VCPU state, or MMIO record is written and read back in the exact same order with fixed-width **io::write** / **io::read** calls, so round-tripping any of these
 records to disk and back produces logically identical data every time.

The one exception is the edge blob: the **set of edges** recovered after decompression is always identical, but the compressed bytes on disk are not guaranteed to match byte-for-byte between two 
writes of the same logical edge set, since LZ4's output and the iteration order of an **unordered_flat_set** are not required to be stable across runs.

## Example

Reading a single block back out of an open trace stream and inspecting its instructions:

```cpp
#include "cpu_tracer/common.hpp"
#include <iostream>

static constexpr auto MAX_LEN = 15u; /* Max x86 instruction length */

std::ifstream ifs("trace.bin", std::ios::binary);

while (ifs.good()) {
      switch (cpu_tracer::blocks::fs::get_save_type(ifs, true)) {
            case cpu_tracer::save_type::block: {
                  cpu_tracer::blocks::block<MAX_LEN> b;
                  b.read(ifs);
                  std::cout << "Block " << b.id << " 0x" << std::hex << b.loc << " (" << std::dec << b.inst_count << " insts)\n";
                  for (const auto &i : b.insts) {
                        std::cout << "  pc=0x" << std::hex << i.inst.pc
                                  << " real_pc=0x" << i.inst.real_pc
                                  << " valid=" << std::dec << i.fvalid << "\n";
                  }
                  break;
            }
            case cpu_tracer::save_type::none: {
                  ifs.ignore(sizeof(cpu_tracer::save_type));
                  break;
            }
            default: {
                  /* Skip */
                  break;
            }
      }
}
```

Output:

```
Block 0 0x401000 (3 insts)
  pc=0x401000 real_pc=0x7f2a10001000 valid=1
  pc=0x401003 real_pc=0x7f2a10001003 valid=1
  pc=0x401007 real_pc=0x7f2a10001007 valid=1
```
