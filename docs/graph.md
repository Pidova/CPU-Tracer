# Graph Construction

A lightweight, header-only C++ tool designed to formalize analyzed trace data into a Boost Graph Library (BGL) Control Flow Graph (CFG).

## What is Graph Construction?

[Loading & Analysis](loading_and_analysis.md) produces analyzed_data (every block, indexed and sorted) and 
edge_data (a deduplicated successor/predecessor map keyed by real PC). Neither of these is a graph on its own, they're flat containers. 
Graph construction is the final step that turns them into a standard boost::adjacency_list, so it can be easily integrated into existing projects and
can operate on the trace as an normal CFG.

## Idea

Given analyzed blocks and resolved successor edges:

```
real_pc_map:  { 0x1000 -> A, 0x1010 -> B, 0x1020 -> C, 0x1040 -> D }
successors:   { 0x1000 -> {0x1010}, 0x1010 -> {0x1020, 0x1040} }
```

generate_graph produces a BGL graph whose nodes carry the original block pointers as properties:

```
   A
   |
   B
  / \
 C   D
```

## Algorithm

### Vertex creation

generate_graph iterates analyzed_data::real_pc_map once, adding one vertex per real PC with boost::add_vertex(blk, g) and recording the resulting 
VertexDesc in a local addr_to_vertex lookup. Each vertex's bundled property is the block's block_ptr<MAX_LEN> itself (a boost::shared_ptr), 
so no block data is copied, the graph just holds references to the same blocks produced by analysis.

* Time – O(V), where V is the number of distinct real PCs
* Memory – O(V), for the vertex set and the address lookup table

### Edge creation

The function then iterates edge_data::successors once. 
For each source address it resolves both endpoints through addr_to_vertex; 
if either endpoint wasn't registered as a vertex (the destination address never appeared in real_pc_map, e.g. an edge into code that wasn't captured by the trace), 
that edge is silently skipped rather than creating a dangling vertex. Surviving edges are added with boost::add_edge(u, v, g).

* Time – O(E), where E is the number of successor edges
* Memory – O(E), for the edge set added to the graph

Combined, building the full graph from already-analyzed data is linear in the size of the trace:

* Time – O(V + E)
* Memory – O(V + E)

## Determinism

The graph uses boost::vecS for both its vertex and edge containers, so vertex descriptors are assigned in insertion order and 
iteration over g is reproducible within a single build run.

Because vertex insertion order follows iteration over real_pc_map (a boost::unordered_flat_map), the descriptor numbering itself is only as 
deterministic as that map's iteration order, the set of vertices and edges recovered is always the same for the same analyzed data, but which integer descriptor a 
given block ends up with may not match between runs unless real_pc_map is iterated in a fixed order beforehand (e.g. via sorted_blocks_global_id).

## Example

Building a graph from a loaded and analyzed trace, then walking it:

```cpp
#include "cpu_tracer/common.hpp"
#include "cpu_tracer/blocks/tools/blocks_graph.hpp"
#include <iostream>

constexpr std::uint8_t MAX_LEN = 15u;

/*  [[INSERT ANALYSIS AND EDGE DATA BEFORE HAND]] */

auto g = cpu_tracer::blocks::graph::generate_graph<MAX_LEN>(adata, edata);

std::cout << "Graph has " << boost::num_vertices(g) << " vertices, " << boost::num_edges(g) << " edges\n";

for (auto v : boost::make_iterator_range(boost::vertices(g))) {

      const auto &b = g[v];
      std::cout << "Block " << b->id << " @ 0x" << std::hex << b->loc << std::dec << " -> ";
      for (auto e : boost::make_iterator_range(boost::out_edges(v, g))) {
	  
            const auto &dst = g[boost::target(e, g)];
            std::cout << "0x" << std::hex << dst->loc << std::dec << " ";
      }
      std::cout << "\n";
}
```

Output:

```
Graph has 4 vertices, 3 edges
Block 0 @ 0x1000 -> 0x1010
Block 1 @ 0x1010 -> 0x1020 0x1040
Block 2 @ 0x1020 ->
Block 3 @ 0x1040 ->
```
