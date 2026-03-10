# A Content-Addressed Adaptive Knowledge Substrate for Distributed Epistemic Coordination

**N. Joven** &middot; [ORCID 0009-0008-0679-0812](https://orcid.org/0009-0008-0679-0812) &middot; February 2026 &middot; *Preprint*

---

RAG systems chunk documents, embed them as vectors, and retrieve by similarity. This destroys relational structure, loses provenance, and resets state every session. We propose replacing the retrieval layer with a **content-addressed Merkle DAG** where every node's identity is computed from its content and dependencies.

The result: provenance is structural (not reconstructed from logs), consistency is demand-driven (cold nodes cost nothing), and reasoning terminates when the root hash stabilizes — a deterministic fixed-point convergence criterion.

This is a systems-layer intervention. We don't modify model internals; we externalize memory, provenance, and traversal control into a substrate that any model can use.

### Key ideas

| Concept | Section |
|---|---|
| Content-addressed typed nodes in a Merkle DAG | §2.1 |
| Continuous depth scoring (recency, centrality, info gain) | §2.2 |
| Tiered compute: free hash checks → bounded recompute → expensive LLM calls | §2.3 |
| Delta chain for audit, diff, and temporal attention | §2.4 |
| Fixed-point convergence via root hash stabilization | §2.5, §3 |
| Structural provenance as a first-class property | §4 |
| Demand-driven consistency and epistemic thermodynamics | §8 |
| Activation decay and localized entropy as traversal scoring primitives | §10 |

## Repository contents

| File | Description |
|---|---|
| [`joven_knowledge_substrate.md`](joven_knowledge_substrate.md) | Full paper (Markdown) |
| [`joven_knowledge_substrate.pdf`](joven_knowledge_substrate.pdf) | PDF rendering |
| [`hello_world.ipynb`](hello_world.ipynb) | Pedagogical notebook (~200 lines of Python) demonstrating typed nodes, Merkle DAG, delta chain, scoped recomputation, and fixed-point convergence |
| [`architecture.mmd`](architecture.mmd) | Mermaid diagram of the tiered operation flow |
| [`ket_decay_entropy_spec.md`](ket_decay_entropy_spec.md) | Production change specification for the Ket Rust workspace: decay function interface, localized amplitude entropy scoring, WQS/Lagrangian implications, and MCP breaking changes |
| [`decay_entropy.ipynb`](decay_entropy.ipynb) | Demonstration notebook: activation decay dynamics, localized amplitude entropy, and contested-node detection in a directed epistemic subgraph |

## Production implementation

[**Ket**](https://github.com/nickjoven/ket) is a Rust workspace (10 crates) that implements the full architecture. It provides content-addressed storage (BLAKE3), a typed DAG with lineage tracing, multi-dimensional scoring, a Dolt-backed SQL mirror for the delta chain, multi-agent coordination, and 12 MCP tools for LLM integration.

## License

[CC0 1.0 Universal](LICENSE) — released as prior art.
