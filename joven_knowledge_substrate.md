# A Content-Addressed Adaptive Knowledge Substrate for Distributed Epistemic Coordination

**N. Joven** ([ORCID: 0009-0008-0679-0812](https://orcid.org/0009-0008-0679-0812))
February 2026
*Preprint — feedback welcome*

---

## Abstract

Current retrieval-augmented generation (RAG) systems represent knowledge as independently retrievable text fragments selected via embedding similarity. While effective for many tasks, this approach introduces structural limitations: relational information is fragmented across chunks, retrieval is guided by surface semantic similarity rather than typed dependency structure, state is reconstructed ephemerally per session, and provenance — the record of where a conclusion came from and what supports it — is either absent or reconstructed forensically from logs.

We propose a content-addressed, typed knowledge substrate in which facts and relations are represented as nodes in a Merkle directed acyclic graph (DAG). The substrate maintains a persistent, verifiable state identified by a single root hash. Traversal depth during reasoning is governed by a continuously updated scoring mechanism based on recency, structural centrality, and observed information gain. Reasoning cycles terminate when the root hash stabilizes, yielding a fixed-point convergence criterion. Provenance is structural: every conclusion's identity is computed from the identities of its premises, making derivation paths verifiable and tamper-evident by construction.

Consistency in the substrate is demand-driven: nodes carry no propagation cost while unqueried, and staleness is detected structurally at read time via hash comparison. This yields a scaling property in which the cost of consistency grows with query volume, not graph size.

We position this architecture as a systems-layer intervention addressing several failure modes identified in recent analyses of large language model (LLM) reasoning (Song et al., 2026). Models cannot reliably self-correct their own reasoning without external feedback (Huang et al., 2024), further motivating external structural intervention. We do not modify model internals; rather, we externalize memory persistence, provenance, and traversal control into a deterministic substrate. The component technologies are individually established; the contribution is their integration into a unified reasoning infrastructure.

---

## 1. The Problem Is the Substrate

Large language models have demonstrated remarkable reasoning capability. They also hallucinate, contradict themselves, forget context, and cannot update cleanly when the world changes. These failures are routinely attributed to model limitations. We argue they are substrate failures.

Current RAG systems work by chunking documents into text, embedding those chunks as vectors, and retrieving the top-*k* most semantically similar chunks at query time. The model reasons over whatever was retrieved. This architecture has three structural problems that cannot be patched at the model level.

**Chunking destroys relational structure.** A causal relationship between two facts that live in different chunks is invisible to the retriever. The model sees retrieved fragments, not the graph connecting them.

**Embedding similarity is a lossy proxy for relevance.** Two nodes can be semantically distant but causally critical to each other. Vector space encodes surface similarity, not causality, dependency, or logical entailment.

**State is ephemeral.** Every session starts cold. Nothing learned during retrieval informs future retrieval. The substrate has no model of its own uncertainty, no memory of what was recently traversed, and no way to detect that it has already resolved a question confidently.

Fine-tuning addresses none of these. It bakes knowledge into weights opaquely and permanently, making it impossible to update, audit, or inspect without retraining. Attempts at model-level self-correction fare no better: LLMs struggle to correct their own reasoning without external feedback, and their performance can degrade after self-correction attempts (Huang et al., 2024).

The correct fix is not a better model. It is a better substrate.

---

## 2. The Proposed Architecture

### 2.1 Content-Addressed Nodes

The primitive of the substrate is a typed node. Nodes represent facts, entities, relationships, events, derived conclusions, or agent observations. Each node is content-addressed: its identifier is a deterministic hash of its content and the hashes of its dependencies.

This gives us several properties for free. Identity comparison becomes hash comparison — two nodes are identical if and only if their hashes match. Deduplication is automatic and structural: the same fact cannot be stored twice because it would produce the same hash. Verification is free: any node can be checked against its hash without trusting the storage layer.

Edges between nodes are typed and also hashed. The graph is a Merkle DAG. The root hash of the current graph is a single identifier that uniquely and verifiably describes the entire state of the knowledge base at a given moment.

### 2.2 Depth Scoring

Each node carries a live depth score: a scalar representing how deeply the current reasoning cycle should traverse from that node. Depth scores are not static. They are updated continuously by a lightweight scoring process that runs at maximum frequency between expensive reasoning operations.

The scoring function takes into account: **Recency** — nodes whose hashes changed recently score higher. **Information gain** — nodes whose last traversal produced new downstream conclusions score higher. **Structural centrality** — nodes with high betweenness centrality, those that bridge otherwise disconnected subgraphs, score higher independent of recency. **Query relevance** — nodes adjacent to the current query entry point receive a temporary relevance boost that decays as traversal proceeds.

Depth scores govern how far the traversal engine walks from any given node before stopping. High-scoring nodes receive deep multi-hop traversal. Low-scoring nodes receive shallow or no traversal. The scoring process itself is cheap enough to run thousands of times per reasoning cycle, continuously updating the attention topology of the graph.

### 2.3 Tiered Operation Cost Model

Reasoning operations on the substrate are explicitly tiered by cost.

**Tier 1 — trivial, run always:** compare hashes, check if a score crossed a threshold, increment a traversal counter, propagate a delta score to neighbors. These operations are essentially free. The substrate runs them continuously.

**Tier 2 — moderate, run when triggered:** traverse a subgraph to a given depth, run a local consistency check, compute a derived value, merge two candidate nodes. These run when a Tier 1 operation flags something worth examining.

**Tier 3 — expensive, run when earned:** deep multi-hop traversal, invoking a language model to reason over a retrieved subgraph, writing durable state, spawning a new agent. These run only when Tier 2 operations confirm that the expected information gain justifies the compute cost.

This tiering is the substrate's implementation of an anytime algorithm: at any point in a reasoning cycle, the system has a valid current answer derived from whatever depth has been reached so far, and continues deepening only while the marginal value of additional traversal exceeds its cost.

### 2.4 The Delta Chain

Each reasoning cycle that modifies the substrate produces a new root hash. The substrate does not store full snapshots between cycles — it stores the minimal delta: the set of nodes that were added, modified, or removed, plus the new root hash. The delta chain is the complete history of how the substrate evolved.

This gives us several properties that are otherwise expensive to engineer. **Free audit trail:** any state in the history is reconstructible from the delta chain; the substrate always knows what it knew at any prior moment. **Structural diff:** comparing two states is graph traversal until hashes diverge; the unchanged portion of the graph costs nothing to compare. **Temporal attention signal:** nodes whose hashes appear frequently in recent deltas are changing often, and the depth scoring system uses delta frequency as a recency signal without additional instrumentation. **Causal legibility:** because deltas record which operations caused which hash changes, the reasoning history is automatically legible — an external auditor can trace any conclusion back through the delta chain to the observations that produced it.

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

## 4. Provenance as a First-Class Property

In current AI systems, provenance — the complete record of where a conclusion came from, what evidence supports it, which agent produced it, and what transformations were applied — is either absent or reconstructed after the fact from logs. When a model produces a wrong answer, the question "why did it believe this?" has no structural answer. The best available response is to re-run the query with logging enabled and hope the failure reproduces. The substrate makes provenance structural rather than forensic.

Every node in the Merkle DAG encodes its own provenance by construction. A derived conclusion's hash is computed from the hashes of its parent nodes — the facts, observations, and intermediate reasoning steps that produced it. The derivation path is not metadata attached to a conclusion; it is the conclusion's identity. Change any upstream fact and the derived node's hash changes. The conclusion cannot silently persist after its premises have been invalidated.

This has several concrete consequences.

**Auditability without instrumentation.** An external auditor can trace any node in the substrate back through its parent chain to the original observations, data sources, or agent actions that produced it. This trace is verifiable at every step via hash comparison. No logging framework is required; the DAG is the audit log.

**Attribution across agents.** In multi-agent systems, each agent's contributions are identified by the agent field on the nodes it creates. When agents disagree, the disagreement is structurally visible: two nodes with the same semantic intent but different hashes, each traceable to the agent and evidence chain that produced it. Reconciliation becomes graph comparison rather than prompt archaeology.

**Institutional trust boundaries.** In distributed deployments, institutions sign the deltas they contribute. A downstream consumer receiving a subgraph can verify not only that it has not been tampered with (hash verification) but who asserted each fact and when (signature chain). Provenance becomes the mechanism by which trust propagates through a shared knowledge base without requiring universal trust in a central authority.

**Regulatory compliance.** Regulated industries — finance, healthcare, legal, defense — increasingly require explainability and traceability for AI-assisted decisions. The substrate satisfies these requirements by construction: the answer to "why did the system conclude X?" is always a concrete path through the DAG, not a post-hoc rationalization generated by the same model that produced the conclusion.

Provenance in this architecture is not a feature layered on top. It is a consequence of content addressing. Any system that hashes its conclusions from its premises gets provenance for free. The contribution is recognizing this as a design principle and building the reasoning substrate around it.

---

## 5. Interaction with Documented LLM Failure Modes

Song et al. (2026) identify categories of reasoning failures in large language models, including fundamental limitations such as working-memory constraints and proactive interference, application-specific failures including multi-agent termination and verification breakdowns, and robustness failures including order sensitivity and anchoring effects. The proposed substrate interacts with these categories at the systems level.

**Working memory and proactive interference.** In standard RAG systems, updated information competes with prior context within a fixed token window. In the proposed substrate, knowledge is externalized into a persistent content-addressed graph. Updates produce new nodes and new root hashes rather than competing for token attention. Recency-sensitive depth scoring biases traversal toward recently modified regions. This does not remove model-level working memory limits but reduces reliance on transient prompt state for knowledge persistence.

**Order sensitivity and anchoring.** Song et al. document that logically equivalent prompts with different ordering can yield divergent outputs. In text-based retrieval systems, ordering of retrieved chunks introduces uncontrolled variation. In the proposed architecture, retrieved context corresponds to a typed subgraph rather than an arbitrary sequence of chunks. Subgraph serialization can be made deterministic (e.g., canonical topological ordering), reducing variance introduced by retrieval order. Anchoring effects internal to the model may remain, but extraneous ordering degrees of freedom at the systems layer are reduced.

**Multi-agent termination and verification.** Multi-agent systems frequently fail to verify intermediate results or terminate prematurely. Song et al. emphasize structured verification protocols as a mitigation strategy. The proposed architecture introduces a fixed-point convergence criterion: a reasoning cycle terminates when successive cycles produce identical root hashes. Because the entire substrate state is content-addressed, convergence is detectable via hash equality. This provides a substrate-level termination signal independent of agent self-assessment.

**Self-correction limitations.** Huang et al. (2024) demonstrate that LLMs cannot reliably self-correct reasoning without external feedback. The substrate provides exactly this external structure: when a derived conclusion's upstream premises change, the hash mismatch is detected by Tier 1 operations and the affected derivations are flagged for recomputation. Correction is driven by structural invalidity, not by asking the model to evaluate its own outputs.

---

## 6. Relation to Existing Work

This architecture synthesizes ideas from several active research areas that have not previously been unified.

**GraphRAG and adaptive retrieval.** Recent work on graph-based retrieval augmented generation has demonstrated that structured knowledge traversal outperforms vector similarity retrieval for multi-hop reasoning tasks. ARK (Adaptive Retriever of Knowledge, Polonuer et al., 2026) introduces adaptive breadth-depth tradeoffs in knowledge graph traversal without pre-set hop limits. Our depth scoring system extends this to a live substrate property rather than a query-time heuristic, and grounds it in information-theoretic feedback from the delta chain.

**Self-organizing knowledge graphs.** Buehler (2025) demonstrates that LLM-coupled recursive graph expansion produces scale-free networks with emergent hub structure, stable modularity, and sustained growth without saturation. Our architecture provides the durable, content-addressed substrate that such a system requires to persist across sessions and be shared across agents.

**Content-addressed storage.** Git (Torvalds, 2005), IPFS (Benet, 2014), and related systems demonstrate that Merkle DAG structures enable cheap diffing, structural deduplication, and verifiable provenance at scale. These systems address storage and verification. We apply the same primitives to a live reasoning substrate, adding depth scoring and fixed-point dynamics as first-class properties.

**Durable execution.** Systems such as Temporal and Restate demonstrate that distributed stateful workflows can survive crashes, restarts, and time while remaining consistent. Our substrate requires a similar durability guarantee for the delta chain.

**Anytime algorithms.** The tiered operation model is an instance of an anytime algorithm (Dean and Boddy, 1988): the system maintains a valid current answer at all times and improves it continuously until interrupted or until a fixed point is reached.

**LLM reasoning failure taxonomies.** Song et al. (2026) identify reasoning failure modes across LLM architectures, including working-memory constraints, order sensitivity, and multi-agent coordination breakdowns. Section 5 maps our substrate mechanisms to their taxonomy. Huang et al. (2024) demonstrate that models cannot self-correct without external feedback, motivating external structural intervention. Sun et al. (2024) demonstrate that grounding LLM reasoning in knowledge graph traversal improves factual accuracy and reduces hallucination, providing empirical motivation for the graph-structured approach taken here.

The synthesis of these research threads into a unified substrate architecture does not appear to exist in the literature as of this writing.

---

## 7. Reconcilable Truth and Compute Efficiency

The primary argument for this architecture is not efficiency. It is correctness — specifically, the property that the substrate provides a single, reconcilable source of truth that any agent can verify, any auditor can trace, and any update can cleanly propagate through.

Current production systems lack this property. When two agents in a multi-agent pipeline reach different conclusions about the same entity, there is no structural mechanism to detect the inconsistency, determine which conclusion is better supported, or propagate a resolution. The inconsistency persists silently until a downstream failure surfaces it — often to an end user. The substrate eliminates this class of failure: contradictions are visible as divergent hashes on the same entity, each traceable through its provenance chain to the evidence that produced it. Resolution is graph reconciliation, not prompt engineering.

Efficiency is the secondary argument, but it is the one that determines adoption at scale.

Current production AI systems waste a substantial portion of their inference budget on substrate failures rather than reasoning. These failures manifest as concrete operational costs.

**Cold-start overhead.** Every session in a stateless system begins by reconstructing context from scratch. This occurs in practice whenever:

- A user returns to a multi-turn task after a session timeout. The system re-retrieves, re-embeds, and re-reasons over the same material it resolved minutes or hours earlier.
- An agent in a pipeline receives a task that a previous agent already partially resolved. The receiving agent has no access to the prior agent's intermediate state and reconstructs context from zero.
- A nightly batch job processes documents that overlap substantially with the previous run. Each document is chunked, embedded, and retrieved independently, with no memory that the vast majority of the knowledge base is unchanged.
- A model is swapped mid-workflow — for example, routing a subtask to a specialized model. The new model receives a text summary of prior context rather than a verifiable state, losing structure and provenance in the handoff.
- A system scales horizontally by adding agent instances. Each new instance cold-starts independently, duplicating retrieval work already completed by existing instances operating on the same substrate.

In each case, the substrate's delta chain eliminates the cold start. Any agent receiving a root hash and a query can resume from the last fixed point. Only nodes whose hashes have changed since the prior cycle require re-traversal. The unchanged portion of the graph — typically the vast majority — costs nothing to re-verify (Tier 1 hash comparison) and nothing to re-traverse.

**Redundant retrieval.** The same facts are re-embedded, re-retrieved, and re-injected across sessions and agents because the substrate has no memory of prior resolution. Content addressing eliminates this by construction: identical facts hash identically and are never stored or fetched twice.

**Context bloat.** Top-*k* chunk retrieval fills context windows with partially relevant material, forcing the model to spend tokens filtering noise rather than reasoning over signal. Depth scoring replaces top-*k* with structure-bounded traversal, reducing retrieved context to what is actually relevant as determined by recency, centrality, and information gain — not embedding similarity.

**Consistency repair.** Models expend inference compute detecting and resolving contradictions that a structural substrate would surface at write time. In the content-addressed substrate, a fact cannot be written in two inconsistent forms without producing two different hashes. The inconsistency is visible immediately and cheaply, before any model is invoked.

These costs compound multiplicatively in multi-agent systems where each agent maintains its own ephemeral context and performs its own full retrieval. The substrate pays them once — or, in the case of redundant retrieval and consistency repair, not at all.

The structural argument is independent of specific benchmarks: each cost category above is eliminated by the architecture's construction rather than mitigated by tuning. Empirical quantification in deployed systems remains future work. But the directional claim requires no measurement: a system that never re-retrieves identical facts, never re-establishes known context, and never silently harbors contradictions will spend more of its compute budget on reasoning than one that does all three on every session.

---

## 8. Demand-Driven Consistency and Epistemic Thermodynamics

The scaling question for any knowledge substrate is: what happens when the graph grows large? At ten thousand nodes, propagating depth score updates on every write becomes expensive. At a million nodes with high connectivity, it becomes intractable. A naïve consistency model — push updates outward from the point of change — pays costs proportional to graph size on every mutation, regardless of whether the affected nodes are ever queried again.

We observe that the content-addressed structure already contains the solution. Staleness is always detectable at read time for free: a node's hash either matches its recorded dependencies or it does not. There is no need to propagate this information proactively. The query itself is the consistency mechanism.

### 8.1 Consistency as a Read-Time Property

When a query arrives and traversal begins, each visited node performs a Tier 1 hash check against its dependencies. If hashes match expectations, the node's depth score remains valid and traversal continues. If they diverge, the node is stale, and only the affected subgraph — the portion actually being traversed — is recomputed. Everything not touched by the current query pays zero consistency cost, regardless of how stale it may be.

This inverts the conventional model. Consistency is demand-driven, not supply-driven. The substrate pays only for consistency it actively consumes. In a knowledge graph where reads are sparse relative to total graph size — which they almost always are — this is dramatically cheaper than any propagation-based alternative.

The tradeoff is query-time latency when a traversal encounters a deeply stale subgraph. The tiered operation model (§2.3) mitigates this directly: Tier 1 detects staleness instantly, Tier 2 performs bounded recomputation, and if the information gain from deeper reconciliation does not justify Tier 3 cost, the system returns its best answer at current depth and flags the residual uncertainty. This is the anytime property (§2.3) applied to consistency itself.

### 8.2 A Thermodynamic Analogy

We note (carefully) that this consistency model admits a thermodynamic analogy, and that the analogy is structurally meaningful rather than decorative. The mapping is not literal physics, but it is operationally coherent. Information propagates through the substrate in ways that resemble energy flow: localized activation, decay over time, and bounded working sets. Calling this "epistemic thermodynamics" would be grandiose; calling it a useful mental model is accurate.

Nodes have a natural resting state: quiescent. In quiescence, a node consumes no compute. It is neither traversed nor revalidated. A query injects energy into the graph. Traversal activates nodes along its path, triggering consistency checks and depth score updates. When traversal ceases, activation decays. Nodes that are never queried remain dormant indefinitely. This is not a flaw — it is a design property. Staleness is not proactively repaired; it is resolved lazily, on observation.

Within this frame, depth score behaves like temperature. Recently traversed or information-rich nodes are "warm": they are more likely to receive deeper traversal and incur maintenance cost. Untouched nodes cool over time through score decay. The scoring function functions analogously to a cooling law: absent new queries, relevance dissipates; new queries inject energy and raise local temperature.

This perspective clarifies scaling. The substrate does not grow uniformly hotter as it accumulates knowledge. It stratifies. At any moment, only a small active frontier of nodes — often a few hundred within a much larger graph — carries meaningful activation. The remainder are inert structure: stored, content-addressed, and costless until reactivated. Growth increases potential surface area, not steady-state compute. Work concentrates where queries concentrate.

### 8.3 Speed of Truth

The thermodynamic frame also clarifies the fundamental consistency bound. When the rate of incoming epistemic events — new observations, agent commits, world-state changes — exceeds the rate at which updates can propagate through the graph, nodes downstream of those changes make decisions based on stale depth scores. Information moves faster than the speed of truth.

In a propagation-based model, this is a correctness crisis: the system silently operates on invalid state. In the demand-driven model, it is a managed condition. Staleness is not invisible — it is structurally detectable at Tier 1 cost on every read. The system does not claim to be consistent. It knows exactly where it is inconsistent, and it reconciles on demand rather than in advance.

The delta chain (§2.4) quantifies the staleness: the number of unreconciled deltas between a node's last validated state and the current root hash is the divergence metric. A cold node with high delta divergence is not wrong — it is unexamined. The distinction matters. Current RAG systems cannot make this distinction at all.

---

## 9. Open Problems

We have described an architecture. Several significant engineering and research problems remain open.

**Entry-point resolution for novel queries.** When a query arrives with no obvious entry node, the system needs a way to find where in the graph to begin traversal. A lightweight semantic index over node summaries provides a first hop, but the interaction between this index and the content-addressed graph requires careful design to avoid reintroducing the failure modes of pure vector retrieval.

**Schema bootstrapping.** The substrate requires typed nodes and typed edges. Deciding how to represent a new domain structurally before ingesting it is non-trivial. Automated schema induction from existing corpora is an open problem, though recent LLM-based knowledge graph construction work provides partial solutions.

**Perturbation model.** What classes of external events warrant disturbing a fixed point, and how should their arrival be represented in the substrate without unnecessary hash chain fragmentation? A principled perturbation model is needed for production deployment.

**Governance and access control in distributed deployment.** In a multi-institution deployment, who can write deltas, who can read which subgraphs, and how are signing keys managed? The content-addressed structure provides the technical foundation for fine-grained provenance, but the governance layer is a social and institutional problem as much as a technical one.

**Calibration of depth scores.** The demand-driven consistency model (§8) establishes that depth scores function as a cooling equation — relevance decays over time and is restored by query contact. A structural observation partially resolves the computational tractability of calibration: because information gain is concave in traversal budget and the Merkle DAG decomposes into spanning trees rooted at query entry points, optimal tier allocation under multiple budget constraints reduces to a Lagrangian relaxation solvable by WQS binary search (the "Aliens trick" from combinatorial optimization; cf. IOI 2016). For k simultaneous constraints (cost budget, depth bound, tier-3 call limit), the nested binary search finds jointly optimal Lagrange multipliers in O(n · (log V)^k) solver evaluations — each evaluation itself O(n) — compared to O(n · V^k) for grid search at equal precision. A companion notebook ([`wqs_aliens_optimization.ipynb`](wqs_aliens_optimization.ipynb)) provides a proof-of-concept, and the production implementation (ket-opt, §11.2) realizes the full pipeline from DAG-to-spanning-tree conversion through calibration to tier-aware traversal with subtree pruning. What remains open is the *adaptive* problem: online recalibration as query distributions shift, the interaction between optimization lag and the speed-of-truth bound (§8.3), and the degree to which concavity degrades in DAGs with high diamond density versus clean tree structure.

---

## 10. The Decay–Quantum Walk Coupling as an Epistemic Primitive

### 10.1 Motivation: Non-Stationarity as the Normal Operating Condition

The epistemic thermodynamics model (§8) established that node activation decays over time and that unqueried nodes become quiescent, with depth scores functioning analogously to a cooling law. We now examine a deeper structural consequence of this decay regime: when activation values decay continuously and independently of traversal frequency, the landscape over which any search process operates is permanently non-stationary.

This is not a pathological edge case. It is the default condition of any substrate processing real-world knowledge streams, where the relevance of facts evolves, observations accumulate asynchronously, and agents query different regions at different rates. The question is not whether to handle non-stationarity, but which search primitive is adequate to it.

In a static landscape, a classical random walk is analytically sufficient. Given a transition matrix P derived from edge weights, the walk converges to a stationary distribution π in time governed by the spectral gap δ = 1 − |λ₂| of P, where λ₂ is its second-largest eigenvalue (Sinclair and Jerrum, 1989). The Cheeger inequality relates δ to the conductance Φ of the graph: δ/2 ≤ Φ ≤ √(2δ) (Mohar, 1989; Alon, 1986), providing geometric intuition — a graph with good expansion has a large spectral gap and mixes quickly. Under these conditions, classical random walks are well-understood, and their mixing time provides a precise convergence criterion.

But when activation values decay at rate λᵥ per node — a direct consequence of the cooling dynamics of §8 — the transition matrix P(t) changes on every timestep. The stationary distribution toward which the walk converges shifts continuously. Classical mixing theory, which assumes a fixed P, applies only instantaneously. Over any non-trivial trajectory, the classical walk is perpetually misaligned: it converges toward a distribution that has already moved by the time convergence is achieved. This is the entry point for quantum walk behavior in the substrate.

### 10.2 Hypothesis I: The Coupled Decay/Quantum Walk Primitive

**Hypothesis (H-QW).** *In a knowledge substrate where node activation values decay continuously and independently of traversal frequency, discrete-time quantum walks are load-bearing rather than merely useful. The interference pattern across complex amplitude distributions retains structural orientation information about the explored subgraph's topology even as individual node activation values decay. Classical random walks lose this orientation as landscape non-stationarity compounds across steps. Decay and quantum walk behavior therefore constitute a single coupled architectural primitive: removing either renders the other insufficient.*

A continuous-time quantum walk on a graph G = (V, E) with adjacency matrix A is governed by the Schrödinger equation i dψ/dt = Hψ, where H is derived from A (e.g., H = −A or H = L the graph Laplacian). The solution is ψ(t) = e^{−iHt} ψ(0), producing a probability distribution |⟨v|ψ(t)⟩|² over vertices (Farhi and Gutmann, 1998; Kempe, 2003). For discrete-time formulations, a unitary coin-plus-shift operator replaces the continuous evolution, with equivalent topological sensitivity properties (Aharonov, Ambainis, Kempe, and Vazirani, 2001).

The key property is superposition: ψ(t) is a complex linear combination of all paths from the initial position through the graph, weighted by their phase contributions. Constructive interference amplifies paths that are topologically central — paths that appear in many interference terms — and destructive interference suppresses paths through structurally peripheral or contested regions. Crucially, the *phase* of ⟨v|ψ(t)⟩ encodes the topology of the paths from the initial state to v, not their specific activation-weighted lengths.

Uniform rescaling of all activation values — which is precisely what happens under isotropic decay A(v,t) = A(v,0)e^{−λt} — preserves relative phase relationships between paths. The interference pattern survives decay. The classical walk's probability distribution over nodes is directly proportional to activation-weighted path probabilities and changes as activations decay.

More precisely: define the activated transition matrix at time t as P(t)ᵤᵥ = A(v,t)W_{uv} / Σ_w A(w,t)W_{uw}, where W is the edge weight matrix. Under heterogeneous per-node decay (λᵥ differing across nodes), the numerator and denominator decay at different rates for different edges. The classical walk's convergence target shifts in directions determined by the relative decay rates of connected nodes — not by the graph's topology. The quantum walk's amplitude pattern retains phase coherence because phase is topology-dependent and activation-scale-independent.

This defines the coupling: under the epistemic thermodynamics model, decay is not optional — it is the mechanism by which the substrate avoids fixating on stale activations. But a search process that depends on activation magnitudes (classical walk) is destabilized by this necessary feature. A search process that depends on phase relationships (quantum walk) is robust to it. They are not two features that happen to co-exist; they are a coupled primitive in which each entails the other.

### 10.3 Hypothesis II: Interference-as-Coherence

**Hypothesis (H-IC).** *Destructive interference at nodes receiving contradictory high-activation paths from different directions functions as an intrinsic coherence mechanism. The substrate detects internal inconsistency geometrically, without requiring an external oracle, through the measurement statistics of the quantum walk.*

In classical random walks and classical depth scoring, inconsistency between two simultaneously activated regions of the graph is invisible as long as neither region is directly connected to the other. Each region's activation is resolved locally; the global pattern can be self-contradictory without any internal signal. Detection requires an external consistency oracle — an explicit logical inference step or a dedicated contradiction-detection module.

The quantum walk changes this. When two clusters C₁ and C₂ are simultaneously activated at high amplitude and share a common neighbor v (or are connected through a short path via v), the amplitudes they contribute to v carry phase information about their origin. If C₁ and C₂ represent inconsistent epistemic states — regions of the graph assigned mutually contradictory high activation by different query paths — the amplitudes they contribute to v may have opposing phase. The resulting amplitude at v under quantum superposition is reduced or cancels, suppressing v's measurement probability.

Formally: let φ₁ be the amplitude contribution from cluster C₁ to node v, and φ₂ the contribution from C₂. If arg(φ₁) − arg(φ₂) ≈ π, then |φ₁ + φ₂|² < |φ₁|² + |φ₂|². The probability of measuring the walker at v is reduced compared to what either contribution would predict independently. In the substrate's terms: v is scored down not by explicit logical evaluation but by geometric interference.

This mechanism has a structural parallel in Burt's theory of structural holes (Burt, 1992). A structural hole is a gap between two network clusters bridged by a single node. In Burt's social network analysis, the bridging node gains informational advantage from its position between unconnected communities. In the epistemic substrate, a node occupying the bridging position between two inconsistent activation clusters is the locus of maximal tension: it is the point where contradictory evidence meets. Destructive interference at this node is the substrate's geometric signal that a conflict exists.

Attractor dynamics in associative memory (Hopfield, 1982) provide a second parallel. In a Hopfield network, inconsistent patterns are not stable attractors — they lie above attractor basins in the energy landscape. The quantum walk analog: inconsistent activation patterns do not produce stable high-amplitude nodes. The amplitude landscape converges toward configurations where paths from different origins interfere constructively, and suppresses configurations where interference is destructive.

The mechanism's relevance to MCMC-based discovery is also instructive. The Ramanujan Machine (Raayoni et al., 2021) searches a landscape of formal conjectures whose "relevance" shifts as identities are discovered, requiring a search process that navigates under non-stationarity. Lenat's AM (Lenat, 1977) demonstrated that heuristic search over mathematical concept spaces exhibits decay-like attention dynamics — highly explored concepts become worn out and attention must shift to structural frontiers. In both systems, the effective search process must retain structural orientation under a continuously shifting activation landscape. The interference-as-coherence mechanism provides a substrate-level analog: rather than explicit consistency checking at the point of conjecture, the quantum walk's interference statistics implement a distributed, continuous, geometry-based coherence signal that concentrates walker amplitude on activation patterns that are globally self-consistent.

### 10.4 The Emergent Epistemic Immune Function

Together, H-QW and H-IC constitute what we term the *epistemic immune function* of the substrate.

The analogy to biological immunity is structural rather than decorative. The biological immune system distinguishes self from non-self through molecular shape-complementarity — the three-dimensional geometry of receptor-ligand binding — rather than through a symbolic lookup of known pathogens. The mechanism is local, distributed, and requires no central oracle. It produces system-wide coherence from geometric constraints acting at the level of individual molecular interactions.

The quantum walk's coherence mechanism operates analogously. It does not require a global consistency oracle or explicit logical inference. Each node's amplitude is computed as a sum over local phase contributions from its immediate graph neighborhood. The global result — high amplitude at consistent activation patterns, suppressed amplitude at contested nodes — emerges from the local geometry of the graph and current phase relationships, with no additional computational mechanism required.

The two components partition distinct functional roles:

- **H-QW (decay-forced quantum walk)** provides *acquired epistemic memory*: the amplitude pattern encodes the history of exploration — which topological regions have been traversed, which paths were constructively reinforced — in a representation that survives activation decay. This is analogous to immune memory: the system retains orientation about previously encountered patterns even as the immediate activation signal fades.

- **H-IC (interference-as-coherence)** provides *innate epistemic immunity*: a direct, oracle-free response to contradictory activation patterns, suppressing contested nodes and flagging structural inconsistencies as they arise. This is analogous to innate immune response: a rapid, non-specific, geometry-based reaction that does not require prior encounter with the specific contradiction.

The two functions are separable in principle but coupled in practice: H-QW requires that the substrate operate under decay (otherwise classical walks suffice), and H-IC operates through the amplitude patterns produced by the quantum walk (otherwise there are no phase relationships to interfere). The joint architecture — decay regime enabling quantum walk necessity, quantum walk enabling interference-as-coherence — is what constitutes the primitive.

### 10.5 Spectral Gap, Structural Holes, and the Non-Stationary Regime

The Cheeger inequality and spectral gap analysis (Sinclair and Jerrum, 1989; Mohar, 1989) provide a quantitative bridge between graph topology and classical walk behavior. A small spectral gap means slow mixing — the walk takes long to spread across a low-conductance graph. Structural holes (Burt, 1992) are precisely the configurations that produce small effective conductance between clusters: the bridging node is the bottleneck, and the walk's mixing time is governed by the bottleneck capacity.

Under activation decay, the effective conductance of a structural hole can change: if the bridging node's activation decays below threshold while the clusters' activations remain high, the hole becomes effectively impassable for the classical walk. The walk becomes trapped on whichever side it currently occupies. This is a failure mode specific to the decay regime — it does not arise in static landscapes, and it grows more severe as per-node decay rates diverge.

The quantum walk exhibits qualitatively different behavior. The spectral structure of the continuous-time quantum walk is governed by all eigenvalues of H simultaneously, not just the spectral gap. Even when the effective conductance of a structural hole is low — trapping the classical walk — quantum amplitude can tunnel through the hole via off-diagonal phase contributions, maintaining amplitude support on both sides of the gap. This is the formal statement of the orientation-retention claim in the context of structural holes: the quantum walk retains the ability to maintain activation across a structural hole even as individual node activations decay.

A companion notebook (§12.3) demonstrates these dynamics computationally: the spectral gap is computed over time as decay is applied, the classical walk's effective reach is tracked against the instantaneous gap, and the quantum amplitude distribution is shown to retain cross-cluster support as the spectral gap shrinks.

### 10.6 Situating H-QW and H-IC in the Existing Framework

**Relation to compression-as-convergence (§2.5, §3).** Fixed-point convergence in the static substrate is achieved when the root hash stabilizes: all pending operations have been processed, depth scores have calibrated, and no new information emerges from additional traversal. In the decaying substrate, hash-level fixed points are transient — the hash changes continuously as decay modifies activation values. The quantum walk generalization is that fixed-point convergence should be assessed at the level of the interference pattern rather than individual activation values: the substrate has converged when the probability distribution over nodes produced by the quantum walk measurement stabilizes, even as the underlying activation magnitudes continue to decay. Compression, in this regime, means the interference pattern has compressed the space of consistent activation configurations to a stable topological signature.

**Relation to the epistemic thermodynamics model (§8).** The thermodynamic framing of §8 described depth scores as temperatures and introduced the speed-of-truth bound: when epistemic events arrive faster than they propagate, nodes operate on stale state. The decay-forced quantum walk provides a mechanism for operating correctly under this condition: the amplitude pattern's topology-encoding property means the walk can continue to navigate large-scale graph structure even when individual node temperatures have cooled below the threshold of classical walk reliability. The quantum walk effectively extends the useful range of the thermodynamic model into lower-temperature regimes.

**Relation to tiered operations (§2.3).** H-IC introduces a natural Tier 1 operation: coherence measurement at contested nodes. Computing |ψᵥ|² — the probability of observing the walker at node v — is an O(1) operation given the amplitude vector. Nodes where this value is significantly below what their activation score would predict (nodes experiencing destructive interference) are candidates for Tier 2 consistency investigation. This provides a geometry-based filter for expensive Tier 3 logical consistency checking: the quantum walk tells the substrate where to look, and Tier 3 operations verify what it found.

**Relation to the WQS/Lagrangian calibration problem (§9).** The calibration problem identified in §9 — online recalibration as query distributions shift, concavity degradation in high-diamond-density DAGs — acquires new structure in the non-stationary decay regime. The expected information gain from k traversal steps, f(k, t), is now a function of both the traversal budget k and the query time t. When per-node decay rates are heterogeneous, f(k, t) may lose concavity in k at certain time slices, invalidating the WQS binary search at those moments. The architectural change specification ([`ket_decay_quantum_walk_spec.md`](ket_decay_quantum_walk_spec.md)) details the detection heuristic and grid-search fallback required to handle this case in the production implementation.

---

## 11. Conclusion

We have described a substrate architecture in which the entire state of a knowledge base is resolvable from a single content-addressed hash, reasoning depth is governed by a live scoring system continuously calibrated by information gain and structural centrality, provenance is structural rather than forensic, and cycles converge naturally to fixed points that encode the system's current best understanding of the world.

The architecture provides a reconcilable source of truth: contradictions between agents surface as divergent hashes traceable through their respective provenance chains, and resolution propagates structurally through the graph. This is the primary argument for the architecture — efficiency gains are meaningless if the system's conclusions cannot be trusted, traced, and corrected.

The architecture eliminates deduplication overhead, makes consistency checking structural rather than inferential, provides audit history and causal legibility for free via the delta chain, and enables multiple agents to share canonical world state without synchronization overhead. Every conclusion is traceable through the DAG to the observations, agents, and evidence that produced it (§4). The integration of these mechanisms with documented LLM reasoning failure modes (Song et al., 2026) and self-correction limitations (Huang et al., 2024) positions the substrate as a systems-layer intervention: it does not modify model internals, but externalizes memory persistence, provenance, and traversal control into infrastructure that addresses working-memory limitations, order sensitivity, and multi-agent verification breakdowns at the substrate level rather than the prompt level.

The secondary practical consequence is compute efficiency: inference cycles concentrate on reasoning rather than substrate management. Redundant retrieval, context bloat, consistency repair, and cold-start overhead — the dominant sources of wasted compute in current production systems — are eliminated by construction rather than mitigated by tuning (§7). The demand-driven consistency model (§8) ensures these efficiency properties scale: the cost of maintaining consistency grows with query volume, not graph size, because unqueried nodes pay zero consistency cost regardless of staleness. Empirical quantification of these gains remains future work; the structural argument stands independent of specific benchmarks.

The contribution of this paper is the synthesis. The component technologies — Merkle DAGs, depth-bounded graph traversal, anytime algorithms, durable execution patterns — are individually established. Unifying them into a single reasoning substrate that is content-addressed, depth-scored, provenance-preserving, delta-tracked, and convergence-terminated does not appear to exist in the literature. We describe the architecture here, provide implementations at two levels of fidelity (§12), and release this document as prior art under CC0.

---

## 12. Implementations

The architecture described in this paper has been realized at two levels of fidelity.

### 12.1 Pedagogical Reference: [`hello_world.ipynb`](https://github.com/nickjoven/jfk-dsa/blob/main/hello_world.ipynb)

A minimal Python notebook included alongside this paper demonstrates the core substrate mechanics in approximately 200 lines. It implements typed content-addressed nodes (Entity, Relation, Derived), a Merkle DAG with root hash computation, delta chain tracking, and a simulated LLM for transitive reasoning. The notebook illustrates scoped recomputation — when a fact changes, only invalidated derivations (detected via Tier 1 hash comparison) trigger re-execution (Tier 3 LLM call) — and demonstrates fixed-point convergence with explicit cost savings over naive re-derivation.

### 12.2 Production Implementation: Ket $|\psi\rangle$

[Ket](https://github.com/nickjoven/ket) is a Rust workspace of nine crates that implements the full substrate architecture for multi-agent workflows. The mapping from paper concepts to implementation components is:

| Paper Concept | Ket Component | Notes |
|---|---|---|
| Content-addressed nodes (§2.1) | **ket-cas** | BLAKE3 flat-file blob store; identical content = identical CID |
| Merkle DAG, typed edges (§2.1) | **ket-dag** | Parent chains, soft links, lineage tracing, export/import bundles |
| Depth scoring (§2.2) | **ket-score** | Four scoring dimensions (correctness, efficiency, style, completeness) with auto, peer, and human sources |
| Tiered operations (§2.3) | **ket-score** | Auto-scoring via `cargo build/test/clippy` demonstrates the Tier 1 cache check → Tier 3 compute pattern |
| Tier allocation via Lagrangian relaxation (§2.3, §9) | **ket-opt** | WQS binary search (Aliens trick) over nested budget constraints; O(n · (log V)^k) joint calibration of cost, depth, and tier-3 limits. Tier-aware traversal prunes Skip-allocated subtrees at runtime, enforcing the parent-activation constraint. Extended in §10.6 and [`ket_decay_quantum_walk_spec.md`](ket_decay_quantum_walk_spec.md) to handle non-stationary decay. |
| Delta chain (§2.4) | **ket-sql** | Dolt versioned SQL mirror; full commit history with structural diff |
| Fixed-point / single root hash (§2.5, §3) | **ket-dag** | `compute_root_hash()` over all nodes and edges; drift detection compares file hashes to stored CIDs |
| Provenance (§4) | **ket-dag** + **ket-cas** | Parent chains encode derivation history; every node traceable to its evidence |
| Multi-agent coordination (§5) | **ket-agent** + **ket-mcp** | Agent registry, task lifecycle, subprocess spawning; 11 tools exposed over MCP (JSON-RPC) for Claude and other agents |

Ket also introduces capabilities that extend the paper's scope:

- **Code Document Object Model (ket-cdom):** Tree-sitter parsing for Rust and Python symbol extraction, representing code structure as content-addressed typed nodes consistent with the substrate's node philosophy.
- **BLAKE3 parallelization:** Content addressing leverages BLAKE3's tree-hashing mode for parallel hashing of large artifacts, a performance consideration not addressed in the abstract architecture.
- **Queryable SQL mirror:** Dolt provides Git-like versioning semantics over a relational schema, enabling SQL queries against the substrate state — useful for operations, debugging, and governance that the delta chain alone does not conveniently support.
- **Python bindings (ket-py):** PyO3 bindings expose CAS and DAG operations to Python, enabling integration with ML pipelines and notebook-based workflows.

These extensions emerged during implementation and suggest directions for future architectural refinement.

### 12.3 Decay–Quantum Walk Demonstration Notebooks

Two artifacts accompany §10 and the production change specification ([`ket_decay_quantum_walk_spec.md`](ket_decay_quantum_walk_spec.md)):

**[`decay_quantum_walk.ipynb`](decay_quantum_walk.ipynb)** — A Python notebook demonstrating the core algebraic and visual claims of §10. It constructs a weighted directed graph with typed nodes and signed edges, implements both a classical random walk and a continuous-time quantum walk over the same graph, applies per-node decay parametrized by half-life, and shows visually how the classical walk loses structural orientation as the landscape decays while the quantum amplitude pattern retains cross-cluster coherence. A dedicated section demonstrates destructive interference at a node receiving contradictory high-activation paths — the interference-as-coherence mechanism of §10.3 — and the final section computes the spectral gap over time under decay, linking it to mixing time degradation for the classical walk.

**[`decay_quantum_walk_viz.html`](decay_quantum_walk_viz.html)** — A standalone interactive visualization (no server required) allowing real-time manipulation of global and per-node decay rates, walk type (classical vs. quantum), interference sensitivity threshold, and graph topology parameters (node count, cluster density, structural hole presence). The visualization shows a live node activation heatmap, amplitude distribution for the quantum walk, spectral gap, and a coherence score derived from interference cancellation at contested nodes.

---

## References

Aharonov, D., Ambainis, A., Kempe, J., and Vazirani, U. (2001). Quantum walks on graphs. *Proceedings of the 33rd Annual ACM Symposium on Theory of Computing* (STOC 2001), pp. 50–59.

Alon, N. (1986). Eigenvalues and expanders. *Combinatorica*, 6(2), 83–96.

Benet, J. (2014). IPFS — Content addressed, versioned, P2P file system. *arXiv preprint*, arXiv:1407.3561.

Burt, R.S. (1992). *Structural Holes: The Social Structure of Competition*. Harvard University Press.

Buehler, M.J. (2025). Agentic deep graph reasoning yields self-organizing knowledge networks. *Journal of Materials Research*, 40, 2204. arXiv:2502.13025.

Dean, T. and Boddy, M. (1988). An analysis of time-dependent planning. *Proceedings of the AAAI Conference on Artificial Intelligence* (AAAI-88).

Farhi, E. and Gutmann, S. (1998). Quantum computation and decision trees. *Physical Review A*, 58(2), 915–928.

Hopfield, J.J. (1982). Neural networks and physical systems with emergent collective computational abilities. *Proceedings of the National Academy of Sciences*, 79(8), 2554–2558.

Huang, J., Chen, X., Mishra, S., Zheng, H.S., Yu, A.W., Song, X., and Zhou, D. (2024). Large language models cannot self-correct reasoning yet. *International Conference on Learning Representations* (ICLR 2024). arXiv:2310.01798.

Kempe, J. (2003). Quantum random walks: An introductory overview. *Contemporary Physics*, 44(4), 307–327.

Lenat, D.B. (1977). Automated theory formation in mathematics. *Proceedings of the 5th International Joint Conference on Artificial Intelligence* (IJCAI-77), pp. 833–842.

Mohar, B. (1989). Isoperimetric numbers of graphs. *Journal of Combinatorial Theory, Series B*, 47(3), 274–291.

Polonuer, J., Vittor, L., Arango, I., Noori, A., Clifton, D.A., Del Corro, L., and Zitnik, M. (2026). Autonomous knowledge graph exploration with adaptive breadth-depth retrieval. *arXiv preprint*, arXiv:2601.13969.

Raayoni, G., Gottlieb, S., Manor, Y., Pisha, G., Harris, Y., Mendlovic, U., Haviv, D., Hadad, Y., and Kaminer, I. (2021). Generating conjectures on fundamental constants with the Ramanujan Machine. *Nature*, 590, 67–73.

Sinclair, A. and Jerrum, M. (1989). Approximate counting, uniform generation and rapidly mixing Markov chains. *Information and Computation*, 82(1), 93–133.

Szegedy, M. (2004). Quantum speed-up of Markov chain based algorithms. *Proceedings of the 45th Annual IEEE Symposium on Foundations of Computer Science* (FOCS 2004), pp. 32–41.

Song, P., Han, P., and Goodman, N. (2026). Large language model reasoning failures. *arXiv preprint*, arXiv:2602.06176.

Sun, J., Xu, C., Tang, L., Wang, S., Lin, C., Gong, Y., Shum, H.-Y., and Guo, J. (2024). Think-on-Graph: Deep and responsible reasoning of large language model on knowledge graph. *International Conference on Learning Representations* (ICLR 2024). arXiv:2307.07697.

Torvalds, L. (2005). Git: Distributed version control system. [https://git-scm.com](https://git-scm.com).

---

*This document is released under CC0 as prior art.*

*The author acknowledges the use of large language model assistants (Claude, Anthropic; ChatGPT, OpenAI; et al.) during manuscript development for discussion, structural feedback, and argumentation refinement."*

