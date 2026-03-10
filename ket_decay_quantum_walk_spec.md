# Architectural Change Proposal: Decay–Quantum Walk Coupling in Ket

**Status:** Draft — theoretical specification
**Theory home:** [`joven_knowledge_substrate.md §10`](joven_knowledge_substrate.md)
**Implementation home:** [Ket](https://github.com/nickjoven/ket) Rust workspace
**Author:** N. Joven · March 2026

---

## 1. Overview and Motivation

Section §10 of the theoretical framework establishes two coupled hypotheses:

- **H-QW**: In a substrate where node activations decay continuously and independently of traversal frequency, quantum walks are load-bearing (not merely useful) because their interference pattern retains topological orientation information even as individual activation magnitudes decay. Classical random walks lose this orientation as the landscape shifts under decay.

- **H-IC**: Destructive interference at nodes receiving contradictory high-activation paths from different directions functions as an intrinsic coherence mechanism — the substrate detects inconsistency geometrically, without an external oracle.

Together these constitute a single coupled architectural primitive: **decay + quantum walk behavior**. Removing either renders the other insufficient.

This document specifies the concrete changes required to realize this primitive in the Ket Rust workspace. The framing throughout treats decay and quantum walk amplitude tracking as one feature with two surfaces, not two independent additions.

---

## 2. Affected Crates

### Primary (structural changes required)

| Crate | Change | Scope |
|---|---|---|
| **ket-score** | Add `decay_adjusted_activation` and `quantum_coherence` scoring dimensions; integrate decay into score computation | Medium |
| **ket-dag** | Add `DecayConfig` field to node representation; add optional `QuantumAmplitude` runtime field | Medium |
| **ket-mcp** | Add 4 new MCP tools; extend score response schema (breaking) | Medium |
| **ket-opt** | Extend WQS optimizer for non-stationary landscapes; add concavity detection and fallback | Large |

### Secondary (API compatibility updates only)

| Crate | Change |
|---|---|
| **ket-agent** | Update score struct deserialization to accept 6-dimensional scores |
| **ket-py** | Expose `DecayConfig`, `QuantumAmplitude`, and new MCP tools to Python bindings |
| **ket-sql** | Add `decay_half_life` and `last_activation_write` columns to node schema in Dolt mirror |

### Unchanged

`ket-cas`, `ket-cdom` — no structural changes required. BLAKE3 addressing and Tree-sitter parsing are unaffected by decay or amplitude tracking.

---

## 3. Decay Function Interface (`ket-dag` + `ket-score`)

### 3.1 Design Principles

Decay is applied **on query, not on write** — consistent with the demand-driven consistency model of §8.1. The stored activation value is never modified; only the computed (decayed) activation changes at read time. This preserves the Merkle DAG's content-addressing invariant: the stored CID reflects the node's activation at write time, not a continuously drifting value. The decayed value is an ephemeral computation applied during scoring.

An `activation_floor` prevents decay to zero. Quiescent nodes remain findable; they just receive minimal traversal depth.

### 3.2 Core Types

```rust
/// Per-node decay configuration, stored as part of the node's persistent record.
/// Serialized into the Merkle DAG node payload alongside activation and node type.
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct DecayConfig {
    /// Half-life in seconds. None = no decay (backward-compatible default).
    pub half_life: Option<f64>,

    /// Minimum activation floor — prevents decay below this value.
    /// Default: 0.01 (1% of original activation).
    pub activation_floor: f64,
}

impl Default for DecayConfig {
    fn default() -> Self {
        Self { half_life: None, activation_floor: 0.01 }
    }
}

/// Graph-level decay defaults, held by the DAG root and overridden per node.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GraphDecayConfig {
    /// Default half-life applied to all nodes without explicit per-node config.
    pub default_half_life: Option<f64>,
    pub default_floor: f64,
}
```

### 3.3 Decay Computation

```rust
use std::f64::consts::LN_2;
use std::time::SystemTime;

/// Compute decay-adjusted activation at query time.
/// Called during Tier 1 scoring; O(1) per node.
pub fn decayed_activation(
    stored_activation: f64,
    last_written: SystemTime,
    config: &DecayConfig,
) -> f64 {
    match config.half_life {
        None => stored_activation,
        Some(half_life) => {
            debug_assert!(half_life > 0.0, "half_life must be positive");
            let elapsed = last_written
                .elapsed()
                .unwrap_or_default()
                .as_secs_f64();
            let decay_factor = (-elapsed * LN_2 / half_life).exp();
            (stored_activation * decay_factor).max(config.activation_floor)
        }
    }
}
```

### 3.4 Node Schema Extension (`ket-dag`)

The `KetNode` struct gains two fields:

```rust
pub struct KetNode {
    // --- existing fields ---
    pub cid: Blake3Cid,
    pub node_type: NodeType,
    pub activation: f64,           // stored activation (never mutated by decay)
    pub last_written: SystemTime,  // already present; now used by decay computation
    pub parents: Vec<Blake3Cid>,
    // ... other existing fields ...

    // --- new fields ---
    /// Per-node decay configuration. None falls back to graph-level defaults.
    pub decay_config: Option<DecayConfig>,

    /// Quantum walk amplitude — ephemeral runtime state, not persisted to CAS.
    /// Populated by the walk engine during an active walk session; None otherwise.
    #[serde(skip)]
    pub quantum_amplitude: Option<QuantumAmplitude>,
}
```

The `quantum_amplitude` field is tagged `#[serde(skip)]` — it is never hashed into the CID and never written to the CAS blob store. It is purely ephemeral runtime state owned by the walk engine. Walk snapshots (amplitude vectors at a given timestep) can be serialized separately as CAS blobs for reproducibility, but they are not part of the node's persistent identity.

---

## 4. Quantum Walk Amplitude Representation (`ket-dag` + walk engine)

### 4.1 Design

The quantum walk operates over a subgraph extracted from the DAG at query time. The walk engine holds the amplitude vector externally (not inside individual nodes) and writes computed amplitudes into `node.quantum_amplitude` during a walk session for scoring access.

```rust
/// Complex amplitude for a single node during a quantum walk session.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QuantumAmplitude {
    /// Complex amplitude components: (real, imaginary)
    pub amplitude: (f64, f64),

    /// Walk session identifier — allows distinguishing stale amplitudes
    /// from prior sessions.
    pub session_id: u64,

    /// Step number within the current session when this amplitude was last set.
    pub step: u32,
}

impl QuantumAmplitude {
    /// Measurement probability: |amplitude|^2
    pub fn probability(&self) -> f64 {
        self.amplitude.0.powi(2) + self.amplitude.1.powi(2)
    }

    /// Phase angle in radians: atan2(im, re)
    pub fn phase(&self) -> f64 {
        self.amplitude.1.atan2(self.amplitude.0)
    }

    /// Magnitude: sqrt(re^2 + im^2)
    pub fn magnitude(&self) -> f64 {
        self.probability().sqrt()
    }

    /// Whether this amplitude belongs to the current session.
    pub fn is_current(&self, current_session: u64) -> bool {
        self.session_id == current_session
    }
}
```

### 4.2 Walk Engine

```rust
/// Continuous-time quantum walk engine over a Ket subgraph.
///
/// Evolution: ψ(t + Δt) = e^{-iHΔt} ψ(t)
/// where H is the decay-weighted adjacency matrix of the subgraph.
pub struct QuantumWalkEngine {
    /// Current amplitude vector: index corresponds to node position in `nodes`.
    pub psi: Vec<(f64, f64)>,  // (re, im) per node

    /// Node ordering for this session (indices into psi).
    pub nodes: Vec<Blake3Cid>,

    /// Decay-weighted Hamiltonian matrix H (n × n, Hermitian).
    /// H_{ij} = decay_adjusted_activation(j) * edge_weight(i,j) / norm
    pub hamiltonian: Vec<Vec<f64>>,  // Real-valued for undirected/symmetrized graph

    pub session_id: u64,
    pub current_step: u32,
}

impl QuantumWalkEngine {
    /// Advance the walk by one step using first-order unitary approximation.
    /// For large Δt, use matrix exponentiation (batch mode).
    ///
    /// ψ_new ≈ (I - iHΔt)ψ, then renormalize.
    pub fn step(&mut self, delta_t: f64) {
        let n = self.psi.len();
        let mut psi_new = vec![(0.0_f64, 0.0_f64); n];

        for i in 0..n {
            // Identity term: psi[i]
            psi_new[i].0 += self.psi[i].0;
            psi_new[i].1 += self.psi[i].1;

            // -iHΔt term: for each j, -i * H[i][j] * Δt * psi[j]
            // -i * (re + i*im) = im - i*re
            for j in 0..n {
                let h = self.hamiltonian[i][j] * delta_t;
                let (re_j, im_j) = self.psi[j];
                // -i * h * (re_j + i*im_j) = h*im_j - i*h*re_j
                psi_new[i].0 += h * im_j;
                psi_new[i].1 -= h * re_j;
            }
        }

        // Renormalize
        let norm: f64 = psi_new.iter().map(|(r, m)| r*r + m*m).sum::<f64>().sqrt();
        if norm > 1e-12 {
            for (r, m) in psi_new.iter_mut() {
                *r /= norm;
                *m /= norm;
            }
        }

        self.psi = psi_new;
        self.current_step += 1;
    }

    /// Measurement probability at node index i.
    pub fn probability(&self, i: usize) -> f64 {
        let (r, m) = self.psi[i];
        r*r + m*m
    }

    /// Interference suppression ratio at node i relative to activation-predicted baseline.
    /// Values near 0 = full constructive; values near 1 = full destructive interference.
    pub fn interference_suppression(&self, i: usize, baseline_probability: f64) -> f64 {
        if baseline_probability < 1e-12 {
            return 0.0;
        }
        let measured = self.probability(i);
        ((baseline_probability - measured) / baseline_probability).clamp(0.0, 1.0)
    }
}
```

### 4.3 Hamiltonian Construction

The Hamiltonian is built from the decay-adjusted adjacency matrix of the subgraph at query time:

```rust
pub fn build_hamiltonian(
    subgraph: &[(Blake3Cid, Vec<(Blake3Cid, f64)>)],  // (node, [(neighbor, weight)])
    node_activations: &[(Blake3Cid, f64)],             // decay-adjusted
) -> Vec<Vec<f64>> {
    let n = subgraph.len();
    let mut h = vec![vec![0.0_f64; n]; n];
    let act_map: std::collections::HashMap<_, _> = node_activations.iter().cloned().collect();

    for (i, (_, neighbors)) in subgraph.iter().enumerate() {
        for (nb_cid, base_weight) in neighbors {
            if let Some(j) = subgraph.iter().position(|(c, _)| c == nb_cid) {
                let activation = act_map.get(nb_cid).copied().unwrap_or(1.0);
                // Symmetrize for Hermitian H: H[i][j] = H[j][i]
                let w = base_weight * activation;
                h[i][j] += w;
                h[j][i] += w;
            }
        }
    }

    // Normalize by max edge weight to keep eigenvalues O(1)
    let max_w = h.iter().flat_map(|row| row.iter()).cloned().fold(0.0_f64, f64::max);
    if max_w > 1e-12 {
        for row in h.iter_mut() {
            for v in row.iter_mut() {
                *v /= max_w;
            }
        }
    }
    h
}
```

---

## 5. New Scoring Dimensions (`ket-score`)

### 5.1 Current State

`ket-score` implements four scoring dimensions: `correctness`, `efficiency`, `style`, `completeness`, with auto, peer, and human sources.

### 5.2 Required Additions

**New dimension: `quantum_coherence`**

- **Definition:** `1.0 - interference_suppression_ratio(v)`
- **Range:** `[0.0, 1.0]` — 1.0 = fully coherent, 0.0 = fully destructively interfered
- **Source:** auto-computed from `QuantumWalkEngine::interference_suppression()`
- **Tier 1 action:** If `quantum_coherence < coherence_threshold` (configurable, default 0.3), flag the node for Tier 2 consistency investigation
- **Interpretation:** A low coherence score at node v means v is receiving contradictory high-activation paths from different graph regions — H-IC signal of structural inconsistency

**New dimension: `decay_adjusted_activation`**

- **Definition:** `decayed_activation(node.activation, node.last_written, config)`
- **Range:** `[activation_floor, ∞)` — replaces raw activation as the walk weight
- **Source:** O(1) computation at query time from stored fields
- **Tier 1 action:** Used as the primary activation signal for depth scoring, replacing stored activation
- **Note:** The stored `activation` field remains unchanged; this dimension is a derived read-time value

**Modified dimension: `structural_centrality`**

- **Current:** Betweenness centrality computed offline over unweighted graph
- **Modified:** Decay-weighted betweenness centrality — shortest paths weighted by `decay_adjusted_activation` of intermediate nodes
- **Rationale:** A high-centrality node that has decayed to near-floor activation should not receive the same traversal priority as a recently active high-centrality node. The decay-weighted version naturally suppresses dormant hubs.

### 5.3 Updated Score Struct

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NodeScore {
    // Existing dimensions (unchanged semantics)
    pub correctness: f64,
    pub efficiency: f64,
    pub style: f64,
    pub completeness: f64,

    // New dimensions
    pub quantum_coherence: Option<f64>,         // None if no active walk session
    pub decay_adjusted_activation: f64,         // Always present (= stored activation if no decay)
}

impl NodeScore {
    /// Composite traversal priority: weighted combination of all dimensions.
    /// quantum_coherence gates traversal — low coherence raises priority for investigation.
    pub fn traversal_priority(&self) -> f64 {
        let base = 0.3 * self.correctness
            + 0.2 * self.efficiency
            + 0.1 * self.style
            + 0.2 * self.completeness
            + 0.2 * self.decay_adjusted_activation;

        // Low coherence = potentially inconsistent = higher investigation priority
        let coherence_modifier = match self.quantum_coherence {
            Some(c) if c < 0.3 => 1.5,  // destructive interference: boost priority
            Some(c) if c < 0.6 => 1.1,  // mild interference: slight boost
            _ => 1.0,
        };

        base * coherence_modifier
    }
}
```

---

## 6. WQS/Lagrangian Implications for Non-Stationary Landscapes (`ket-opt`)

### 6.1 Core Issue

The existing `ket-opt` WQS binary search finds optimal Lagrange multipliers under the assumption that the value function `f(k)` — expected information gain from k traversal steps — is concave in k (a condition guaranteed by the submodularity of information gain over spanning trees). This concavity is what makes the Aliens trick valid: the binary search over the single multiplier λ finds the globally optimal budget allocation.

In a decaying landscape, `f(k, t)` is a function of both the traversal budget `k` and the query time `t`. The concavity property may fail in k at certain times because different nodes decay at different rates, creating a non-monotone information gain profile: spending one more traversal step may lead to a freshly activated high-value node at time t₁ but only to a near-floor decayed node at time t₂.

### 6.2 Specific Implications

**1. Time-indexed Lagrange multipliers.** The multiplier λ must be recomputed as the activation landscape shifts due to decay. For a substrate with half-lives on the order of hours and query latency on the order of seconds, λ is effectively stable within a single query. For half-lives on the order of minutes (aggressive decay), λ should be recomputed at the start of each query against the current decay-adjusted activations.

**2. Decay-adjusted edge costs.** The DP formulation in the Aliens trick computes costs over the spanning tree induced by the DAG. Each edge cost now calls `decayed_activation()` at evaluation time rather than reading a stored weight. This adds an O(1) constant factor per edge per DP evaluation — negligible for the traversal sizes ket-opt currently targets.

**3. Concavity detection under asymmetric decay.** When per-node half-lives are heterogeneous, the value function `f(k, t)` may become non-concave in k at specific budget levels. Detection heuristic: if two successive WQS binary search iterations converge to the same k with different λ, the value function is non-concave at that budget level. In this case, fall back to a bounded grid search over the budget range `[k_lo, k_hi]` identified by the binary search bounds.

**4. Skip-tier allocation under decay.** Current ket-opt assigns tier allocations (Active, Lazy, Skip) statically per query. Under decay, a previously Active node may have decayed to Skip-eligible between queries. Tier allocations must be re-evaluated at query time against current decay-adjusted activations, not cached from prior queries.

### 6.3 `DecayAwareOptimizer` Interface

```rust
/// Wraps the existing WqsOptimizer with decay-aware recalibration.
pub struct DecayAwareOptimizer {
    inner: WqsOptimizer,
    graph_decay: GraphDecayConfig,
    /// How frequently to recompute λ (set based on minimum half-life in graph)
    recalibration_interval: std::time::Duration,
    last_calibrated: SystemTime,
    /// Cached Lagrange multipliers from last calibration
    cached_lambdas: Vec<f64>,
}

impl DecayAwareOptimizer {
    /// Compute tier allocations at the given query time.
    /// Uses decay-adjusted activations; falls back to grid search if concavity fails.
    pub fn allocate_tiers_at(
        &self,
        dag: &KetDag,
        query_time: SystemTime,
        budget_constraints: &BudgetConstraints,
    ) -> Result<TierAllocation, OptError> {
        let decayed = self.compute_decayed_weights(dag, query_time);
        match self.inner.wqs_search_with_weights(&decayed, budget_constraints) {
            Ok(alloc) => Ok(alloc),
            Err(OptError::ConcavityFailure { k_lo, k_hi }) => {
                // Non-concave value function detected; fall back to grid search
                self.grid_search_fallback(&decayed, budget_constraints, k_lo, k_hi)
            }
            Err(e) => Err(e),
        }
    }

    /// Whether cached λ values are stale given elapsed time and current activation decay.
    pub fn needs_recalibration(&self) -> bool {
        self.last_calibrated
            .elapsed()
            .map(|e| e >= self.recalibration_interval)
            .unwrap_or(true)
    }
}
```

### 6.4 Concavity Analysis

The concavity of `f(k)` is guaranteed when:
1. Information gain is submodular (always holds for spanning-tree DP formulations)
2. Edge weights are non-negative and fixed (violated under heterogeneous per-node decay)

Condition 2 is violated precisely when different nodes have different decay rates. The degree of violation is proportional to the variance of decay rates across nodes in the active subgraph. In practice:
- **Homogeneous decay** (all nodes same half-life): concavity is preserved; standard WQS applies.
- **Low-variance heterogeneous decay** (half-lives within 2× of each other): concavity degradation is mild; standard WQS with concavity check is sufficient.
- **High-variance heterogeneous decay** (half-lives spanning orders of magnitude): concavity fails at multiple budget levels; grid search fallback is required.

Recommendation: expose `decay_variance` as a diagnostic metric on `DecayAwareOptimizer`. Log a warning when `decay_variance > threshold` to signal that the grid search fallback will be invoked frequently.

---

## 7. MCP Interface Changes (`ket-mcp`)

### 7.1 New Tools (Non-Breaking Additions)

Four new tools are added to the MCP stdio JSON-RPC interface. Existing tools are unchanged.

---

**Tool: `walk_quantum`**

Execute a continuous-time quantum walk from a start node for N steps.

```json
{
  "name": "walk_quantum",
  "description": "Execute a quantum walk from a start node. Returns amplitude vector and interference scores.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "start_node": { "type": "string", "description": "CID of the walk origin node" },
      "steps": { "type": "integer", "description": "Number of walk steps", "default": 10 },
      "delta_t": { "type": "number", "description": "Time increment per step", "default": 0.1 },
      "max_subgraph_nodes": { "type": "integer", "description": "Limit subgraph size", "default": 50 },
      "decay_override": {
        "type": "object",
        "description": "Override decay config for this walk only",
        "properties": {
          "half_life": { "type": "number" },
          "activation_floor": { "type": "number" }
        }
      }
    },
    "required": ["start_node"]
  }
}
```

Response:
```json
{
  "session_id": "u64",
  "steps_completed": "u32",
  "amplitudes": [
    {
      "node": "CID string",
      "label": "node label",
      "real": 0.0,
      "imaginary": 0.0,
      "probability": 0.0,
      "phase_radians": 0.0,
      "decay_adjusted_activation": 0.0,
      "interference_suppression": 0.0,
      "quantum_coherence": 0.0
    }
  ],
  "global_coherence_score": 0.0,
  "spectral_gap": 0.0
}
```

---

**Tool: `walk_classical`**

Execute a classical random walk for comparison against `walk_quantum`.

```json
{
  "name": "walk_classical",
  "description": "Execute a classical random walk under current decay-adjusted activations.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "start_node": { "type": "string" },
      "steps": { "type": "integer", "default": 10 },
      "decay_override": { "type": "object" }
    },
    "required": ["start_node"]
  }
}
```

Response:
```json
{
  "steps_completed": "u32",
  "distribution": [
    {
      "node": "CID string",
      "label": "node label",
      "probability": 0.0,
      "decay_adjusted_activation": 0.0
    }
  ],
  "effective_spectral_gap": 0.0
}
```

---

**Tool: `coherence_check`**

Compute interference suppression at a target node, returning the contributing paths.

```json
{
  "name": "coherence_check",
  "description": "Compute interference suppression ratio at a node. Low coherence signals contradictory activation paths (H-IC, §10.3).",
  "inputSchema": {
    "type": "object",
    "properties": {
      "node": { "type": "string", "description": "CID of target node" },
      "walk_session_id": { "type": "integer", "description": "Use amplitudes from this walk session" },
      "depth": { "type": "integer", "description": "Path depth to trace", "default": 3 }
    },
    "required": ["node"]
  }
}
```

Response:
```json
{
  "node": "CID string",
  "quantum_coherence": 0.0,
  "interference_suppression": 0.0,
  "baseline_probability": 0.0,
  "measured_probability": 0.0,
  "contributing_paths": [
    {
      "path": ["CID", "CID", "CID"],
      "path_labels": ["label", "label", "label"],
      "amplitude_real": 0.0,
      "amplitude_imaginary": 0.0,
      "phase_radians": 0.0,
      "constructive": true
    }
  ],
  "verdict": "coherent | contested | inconsistent"
}
```

The `verdict` field maps coherence score to a human-readable label:
- `coherent`: `quantum_coherence >= 0.7`
- `contested`: `0.3 <= quantum_coherence < 0.7`
- `inconsistent`: `quantum_coherence < 0.3` — Tier 2 investigation recommended

---

**Tool: `decay_status`**

Report current decay-adjusted activations for a set of nodes.

```json
{
  "name": "decay_status",
  "description": "Return current decay-adjusted activations. Shows how much activation has decayed since last write.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "nodes": {
        "type": "array",
        "items": { "type": "string" },
        "description": "CIDs of nodes to query"
      }
    },
    "required": ["nodes"]
  }
}
```

Response:
```json
{
  "query_time": "ISO 8601 timestamp",
  "nodes": [
    {
      "node": "CID string",
      "label": "node label",
      "stored_activation": 0.0,
      "decay_adjusted_activation": 0.0,
      "decay_fraction_remaining": 0.85,
      "half_life_seconds": 3600.0,
      "seconds_since_write": 600.0,
      "tier_recommendation": "Active | Lazy | Skip"
    }
  ]
}
```

### 7.2 Breaking Changes

**Score response schema extension (additive breaking change):**

The `score` object in responses from `node_get`, `dag_traverse`, and `score_node` currently contains four fields:
```json
{ "correctness": 0.0, "efficiency": 0.0, "style": 0.0, "completeness": 0.0 }
```

After this change, it contains six:
```json
{
  "correctness": 0.0,
  "efficiency": 0.0,
  "style": 0.0,
  "completeness": 0.0,
  "decay_adjusted_activation": 0.0,
  "quantum_coherence": null
}
```

`quantum_coherence` is `null` when no active walk session exists (the common case for existing callers).

**Impact on existing clients:**
- Clients using flexible deserialization (`serde_json::Value`, dict-based) are unaffected.
- Clients deserializing to a fixed 4-field struct will fail with an unknown-field error.
- **Mitigation:** Add `#[serde(deny_unknown_fields)]` stricter clients should use `#[serde(default)]` on the new fields. Clients built in Python via `ket-py` are unaffected if using dict access.

**No other breaking changes:**
- Transport (stdio), framing (JSON-RPC 2.0), authentication, and tool discovery protocol are unchanged.
- Existing tool input schemas are unchanged.
- The new `walk_type` parameter discussed in §7.1 is optional with backward-compatible default (`classical`).

---

## 8. Migration Path

### Phase 1: Decay infrastructure (non-breaking)
1. Add `DecayConfig` to `KetNode` with `serde(default)` — existing nodes deserialize with `half_life: None` (no decay)
2. Implement `decayed_activation()` in `ket-score`
3. Add `decay_adjusted_activation` to `NodeScore` — existing score consumers receive it as equal to `stored_activation` when no decay is configured
4. Add `decay_status` MCP tool

### Phase 2: Quantum walk engine (non-breaking)
1. Implement `QuantumWalkEngine` and `QuantumAmplitude` in a new `ket-walk` crate or as a module within `ket-dag`
2. Add `walk_quantum` and `walk_classical` MCP tools
3. Add `coherence_check` MCP tool
4. `quantum_coherence` remains `null` in `NodeScore` until an active walk session writes to nodes

### Phase 3: Scoring integration (score schema breaking change)
1. Add `quantum_coherence` field to `NodeScore`
2. Integrate coherence into `traversal_priority()`
3. Update `ket-agent`, `ket-py` deserializers
4. Bump MCP protocol version in tool manifest

### Phase 4: WQS optimizer extension
1. Implement `DecayAwareOptimizer` wrapping existing `WqsOptimizer`
2. Add concavity detection and grid-search fallback
3. Add `decay_variance` diagnostic metric

---

*This document is released under CC0 as prior art alongside the theoretical framework it specifies.*
