# A Content-Addressed Adaptive Knowledge Substrate for Distributed Epistemic Coordination

**N. Joven**  
February 2026  
*Preprint — feedback welcome*

---

## Abstract

Current AI memory architectures treat knowledge as text to be searched. We argue this is a category error. We propose instead a content-addressed, distributed, durable knowledge substrate in which facts, relationships, and derived conclusions are typed nodes in a Merkle DAG. A live depth-scoring system governs how deeply any topic is traversed in a given reasoning cycle, calibrated continuously by information gain, recency, and structural centrality. The entire state of the substrate at any moment is resolvable from a single root hash, enabling trivial diffing, free audit history, structural contradiction detection, and fixed-point convergence as a natural termination signal. We argue this architecture eliminates the root causes of hallucination, context bloat, inconsistency, and cold-start failure in production AI systems — and that deployed at scale, it represents viable infrastructure for large-scale distributed reasoning across many agents and institutions. The component technologies all exist. The synthesis does not. We describe it here and release this document as prior art under CC0.

---

## 1. The Problem Is the Substrate

Large language models have demonstrated remarkable reasoning capability. They also hallucinate, contradict themselves, forget context, and cannot update cleanly when the world changes. These failures are routinely attributed to model limitations. We argue they are substrate failures.

Current retrieval-augmented generation (RAG) systems work by chunking documents into text, embedding those chunks as vectors, and retrieving the top-k most semantically similar chunks at query time. The model reasons over whatever was retrieved. This architecture has three structural problems that cannot be patched at the model level.

**Chunking destroys relational structure.** A causal relationship between two facts that live in different chunks is invisible to the retriever. The model sees retrieved fragments, not the graph connecting them.

**Embedding similarity is a lossy proxy for relevance.** Two nodes can be semantically distant but causally critical to each other. Vector space encodes surface similarity, not causality, dependency, or logical entailment.

**State is ephemeral.** Every session starts cold. Nothing learned during retrieval informs future retrieval. The substrate has no model of its own uncertainty, no memory of what was recently traversed, and no way to detect that it has already resolved a question confidently.

Fine-tuning addresses none of these. It bakes knowledge into weights opaquely and permanently, making it impossible to update, audit, or inspect without retraining.

The correct fix is not a better model. It is a better substrate.

---

## 2. The Proposed Architecture

### 2.1 Content-Addressed Nodes

The primitive of the substrate is a typed node. Nodes represent facts, entities, relationships, events, derived conclusions, or agent observations. Each node is content-addressed: its identifier is a deterministic hash of its content and the hashes of its dependencies.

This gives us several properties for free. Identity comparison becomes hash comparison — two nodes are identical if and only if their hashes match. Deduplication is automatic and structural: the same fact cannot be stored twice because it would produce the same hash. Verification is free: any node can be checked against its hash without trusting the storage layer.

Edges between nodes are typed and also hashed. The graph is a Merkle DAG. The root hash of the current graph is a single identifier that uniquely and verifiably describes the entire state of the knowledge base at a given moment.

### 2.2 Depth Scoring

Each node carries a live depth score: a scalar representing how deeply the current reasoning cycle should traverse from that node. Depth scores are not static. They are updated continuously by a lightweight scoring process that runs at maximum frequency between expensive reasoning operations.

The scoring function takes into account:

- **Recency**: nodes whose hashes changed recently score higher
- **Information gain**: nodes whose last traversal produced new downstream conclusions score higher
- **Structural centrality**: nodes with high betweenness centrality — those that bridge otherwise disconnected subgraphs — score higher independent of recency
- **Query relevance**: nodes adjacent to the current query entry point receive a temporary relevance boost that decays as traversal proceeds

Depth scores govern how far the traversal engine walks from any given node before stopping. High-scoring nodes receive deep multi-hop traversal. Low-scoring nodes receive shallow or no traversal. The scoring process itself is cheap enough to run thousands of times per reasoning cycle, continuously updating the attention topology of the graph.

### 2.3 Tiered Operation Cost Model

Reasoning operations on the substrate are explicitly tiered by cost.

**Tier 1 — trivial, run always**: compare hashes, check if a score crossed a threshold, increment a traversal counter, propagate a delta score to neighbors. These operations are essentially free. The substrate runs them continuously.

**Tier 2 — moderate, run when triggered**: traverse a subgraph to a given depth, run a local consistency check, compute a derived value, merge two candidate nodes. These run when a Tier 1 operation flags something worth examining.

**Tier 3 — expensive, run when earned**: deep multi-hop traversal, invoking a language model to reason over a retrieved subgraph, writing durable state, spawning a new agent. These run only when Tier 2 operations confirm that the expected information gain justifies the compute cost.

This tiering is the substrate's implementation of an anytime algorithm: at any point in a reasoning cycle, the system has a valid current answer derived from whatever depth has been reached so far, and continues deepening only while the marginal value of additional traversal exceeds its cost.

### 2.4 The Delta Chain

Each reasoning cycle that modifies the substrate produces a new root hash. The substrate does not store full snapshots between cycles — it stores the minimal delta: the set of nodes that were added, modified, or removed, plus the new root hash. The delta chain is the complete history of how the substrate evolved.

This gives us several properties that are otherwise expensive to engineer:

**Free audit trail**: any state in the history is reconstructible from the delta chain. The substrate always knows what it knew at any prior moment.

**Structural diff**: comparing two states is graph traversal until hashes diverge. The unchanged portion of the graph costs nothing to compare.

**Temporal attention signal**: nodes whose hashes appear frequently in recent deltas are changing often. The depth scoring system uses delta frequency as a recency signal without any additional instrumentation.

**Causal legibility**: because deltas record which operations caused which hash changes, the reasoning history is automatically legible. An external auditor can trace any conclusion back through the delta chain to the observations that produced it.

### 2.5 Fixed-Point Convergence

A reasoning cycle over the substrate terminates naturally when running another cycle would produce the same root hash. The system has reached a fixed point: all depth scores have stabilized, all pending Tier 1 and Tier 2 operations have been processed, and no new information has emerged from the most recent traversals.

The fixed point is not the end of useful operation — it is a resting state that awaits perturbation. An external event, a new agent observation, a world-state change, or a user query arrives and perturbs the system away from the fixed point. The speed at which the substrate finds the new fixed point is a direct measure of how well the depth scoring is calibrated.

Well-calibrated systems converge fast because their depth scores correctly direct compute toward genuinely uncertain or recently changed regions of the graph. Poorly calibrated systems thrash — spending expensive traversal on stable regions while missing genuine change elsewhere. The delta chain provides the feedback signal needed to improve calibration over time.

---

## 3. Why a Single Hash Is Sufficient

If the substrate is fully deterministic and content-addressed, the entire current state of the knowledge base — every fact, every relationship, every depth score, every traversal history — is uniquely described by a single root hash. This has practical implications that extend beyond storage efficiency.

Any agent at any point in time can reconstruct full context from a hash. There is no context window to stuff, no session state to maintain, no "remind me what we were discussing." An agent receiving a hash and a query has everything it needs to begin traversal.

Multiple agents operating on the same substrate share a canonical world state. When one agent commits a delta, the new root hash is available to all agents immediately. Disagreements between agents are structurally visible as divergent hashes pointing to the same entity — not as implicit inconsistencies buried in separate context windows.

The hash is also the natural unit of trust in a distributed deployment. An institution contributing data to the substrate signs its deltas. A downstream consumer verifies those signatures against the hash chain. The provenance of any fact is a path through the delta chain to the signing institution. This is not an additional feature — it follows directly from the content-addressed structure.

---

## 4. Relation to Existing Work

This architecture synthesizes ideas from several active research areas that have not previously been unified.

**GraphRAG and adaptive retrieval**: Recent work on graph-based retrieval augmented generation has demonstrated that structured knowledge traversal outperforms vector similarity retrieval for multi-hop reasoning tasks. ARK (Adaptive Retriever of Knowledge, Polonuer et al. 2026) introduces adaptive breadth-depth tradeoffs in KG traversal without pre-set hop limits. Our depth scoring system extends this to a live substrate property rather than a query-time heuristic, and grounds it in information-theoretic feedback from the delta chain.

**Self-organizing knowledge graphs**: Buehler (2025) demonstrates that LLM-coupled recursive graph expansion produces scale-free networks with emergent hub structure, stable modularity, and sustained growth without saturation. Our architecture provides the durable, content-addressed substrate that such a system requires to persist across sessions and be shared across agents.

**Content-addressed storage**: Git, IPFS, and related systems demonstrate that Merkle DAG structures enable cheap diffing, structural deduplication, and verifiable provenance at scale. These systems address storage and verification. We apply the same primitives to a live reasoning substrate, adding depth scoring and fixed-point dynamics as first-class properties.

**Durable execution**: Systems such as Temporal and Restate demonstrate that distributed stateful workflows can survive crashes, restarts, and time while remaining consistent. Our substrate requires a similar durability guarantee for the delta chain.

**Anytime algorithms**: The tiered operation model is an instance of an anytime algorithm (Dean and Boddy, 1988): the system maintains a valid current answer at all times and improves it continuously until interrupted or until a fixed point is reached.

The synthesis of these five research threads into a unified substrate architecture does not appear to exist in the literature as of this writing.

---

## 5. The Compute Efficiency Argument

The technical argument above is self-contained. This section makes the broader case for why it matters beyond correctness — specifically, why it matters for compute cost.

Current production AI systems waste the majority of their inference budget on substrate failures rather than reasoning. The dominant cost categories are:

- **Redundant retrieval**: the same facts are re-embedded, re-retrieved, and re-injected into context repeatedly across sessions and agents, because the substrate has no memory of prior resolution
- **Context bloat**: top-k chunk retrieval fills context windows with partially relevant material, forcing the model to spend tokens filtering noise rather than reasoning over signal
- **Consistency repair**: models expend compute detecting and resolving contradictions that a structural substrate would surface for free at write time
- **Cold-start overhead**: every session re-establishes context from scratch, paying full retrieval cost regardless of how recently the same ground was covered

Each of these is a direct consequence of treating knowledge as text to be searched rather than structure to be traversed. They compound at scale: a multi-agent system in which each agent maintains its own ephemeral context and performs its own full retrieval pays these costs multiplicatively rather than once.

The content-addressed substrate eliminates redundant retrieval by construction — identical facts hash identically and are never fetched twice. Depth scoring replaces top-k with structure-bounded traversal, reducing context to what is actually relevant. Contradiction detection happens at write time with no model involvement. The delta chain means any agent can resume from a prior fixed point rather than starting cold.

The result is that inference compute concentrates on reasoning rather than substrate management. At scale, across many agents operating over a shared substrate, the reduction in wasted cycles is the primary practical argument for this architecture — not correctness alone, but the ratio of useful work to total compute.

---

## 6. Open Problems

We have described an architecture. Several significant engineering and research problems remain open.

**Entry-point resolution for novel queries**: When a query arrives with no obvious entry node, the system needs a way to find where in the graph to begin traversal. A lightweight semantic index over node summaries provides a first hop, but the interaction between this index and the content-addressed graph requires careful design to avoid reintroducing the failure modes of pure vector retrieval.

**Schema bootstrapping**: The substrate requires typed nodes and typed edges. Deciding how to represent a new domain structurally before ingesting it is non-trivial. Automated schema induction from existing corpora is an open problem, though recent LLM-based knowledge graph construction work provides partial solutions.

**Perturbation model**: What classes of external events warrant disturbing a fixed point, and how should their arrival be represented in the substrate without unnecessary hash chain fragmentation? A principled perturbation model is needed for production deployment.

**Governance and access control in distributed deployment**: In a multi-institution deployment, who can write deltas, who can read which subgraphs, and how are signing keys managed? The content-addressed structure provides the technical foundation for fine-grained provenance, but the governance layer is a social and institutional problem as much as a technical one.

**Calibration of depth scores**: The feedback loop between traversal outcomes and depth score updates is described in principle here. The specific update rules, learning rates, and decay functions require empirical work.

---

## 7. Conclusion

We have described a substrate architecture in which the entire state of a knowledge base is resolvable from a single content-addressed hash, reasoning depth is governed by a live scoring system continuously calibrated by information gain and structural centrality, and cycles converge naturally to fixed points that encode the system's current best understanding of the world.

The architecture eliminates deduplication overhead, makes consistency checking structural rather than inferential, provides audit history and causal legibility for free via the delta chain, and enables multiple agents to share canonical world state without synchronization overhead.

The primary practical consequence is compute efficiency: inference cycles concentrate on reasoning rather than substrate management. Redundant retrieval, context bloat, consistency repair, and cold-start overhead — the dominant sources of wasted compute in current production systems — are eliminated by construction rather than mitigated by tuning.

This document is released under CC0 as prior art.

---

## References

- Buehler, M.J. (2025). Agentic deep graph reasoning yields self-organizing knowledge networks. *Journal of Materials Research*.
- Dean, T. and Boddy, M. (1988). An analysis of time-dependent planning. *AAAI*.
- Polonuer, J. et al. (2026). Autonomous knowledge graph exploration with adaptive breadth-depth retrieval. *arXiv:2601.13969*.
- Sun, J. et al. (2024). Think-on-Graph: Deep and responsible reasoning of large language model on knowledge graph. *ICLR 2025*.
- Protocol Labs (2015). IPFS: Content addressed, versioned, P2P file system. *arXiv:1407.3561*.
- Torvalds, L. (2005). Git: distributed version control system. *git-scm.com*.
