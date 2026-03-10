# Architectural Change Proposal: Decay and Structural Entropy Scoring

**Status:** Draft — implementation specification
**Theory home:** [`joven_knowledge_substrate.md §10`](joven_knowledge_substrate.md)
**Implementation home:** [Ket](https://github.com/nickjoven/ket) Rust workspace
**Author:** N. Joven · March 2026

---

## 1. Overview and Motivation

This document specifies two coupled additions to the Ket scoring architecture:

1. **Decay** (§3) — continuous activation decay applied on read, configurable per node. Nodes that have not been activated recently contribute less to traversal priority. This is the primary primitive.

2. **Localized amplitude entropy** (§4) — a structural inconsistency signal computed from the distribution of decay-adjusted activations across a node's neighbors. A node where activation is spread uniformly across many neighbors has no dominant traversal direction; this indicates a structurally contested position in the knowledge subgraph and warrants Tier 2 investigation.

The two features compose: entropy is computed over decay-adjusted activations, so a node with many neighbors that have all decayed to near-floor will correctly read as low-entropy (quiescent, not contested).

---

## 2. Affected Crates

### Primary (structural changes required)

| Crate | Change | Scope |
|---|---|---|
| **ket-score** | Add `amplitude_entropy` and `decay_adjusted_activation` scoring dimensions; integrate decay into score computation | Medium |
| **ket-dag** | Add `DecayConfig` field to node representation | Small |
| **ket-mcp** | Add 2 new MCP tools; extend score response schema (breaking) | Small |
| **ket-opt** | Extend WQS optimizer for non-stationary landscapes; add concavity detection and fallback | Medium |

### Secondary (API compatibility updates only)

| Crate | Change |
|---|---|
| **ket-agent** | Update score struct deserialization to accept 6-dimensional scores |
| **ket-py** | Expose `DecayConfig` and new MCP tools to Python bindings |
| **ket-sql** | Add `decay_half_life` and `last_activation_write` columns to node schema in Dolt mirror |

### Unchanged

`ket-cas`, `ket-cdom` — no structural changes required. BLAKE3 addressing and Tree-sitter parsing are unaffected by decay or entropy scoring.

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

The `KetNode` struct gains one field:

```rust
pub struct KetNode {
    // --- existing fields ---
    pub cid: Blake3Cid,
    pub node_type: NodeType,
    pub activation: f64,           // stored activation (never mutated by decay)
    pub last_written: SystemTime,  // already present; now used by decay computation
    pub parents: Vec<Blake3Cid>,
    // ... other existing fields ...

    // --- new field ---
    /// Per-node decay configuration. None falls back to graph-level defaults.
    pub decay_config: Option<DecayConfig>,
}
```

---

## 4. Localized Amplitude Entropy (`ket-score`)

### 4.1 Motivation

A node that sits at the boundary between structurally distinct subgraph regions will have neighbors whose decay-adjusted activations are roughly equal — no single path dominates. This even distribution is the observable signature of a structurally contested node. A node in a well-structured region will have activation concentrated on the neighbors that form the dominant traversal path.

Localized amplitude entropy measures this directly: it is the normalized Shannon entropy of the decay-adjusted activation distribution across a node's neighbors. It is O(degree) to compute, requires no global graph state, and composes naturally with decay.

### 4.2 Definition

For node `v` with neighbors `N(v)`, let `a_i` be the decay-adjusted activation of neighbor `i`.

```
p_i = a_i / Σ a_j         (normalize to a distribution)

H(v) = -Σ p_i · log(p_i)  (Shannon entropy, nats)

H_norm(v) = H(v) / log(|N(v)|)   (normalized to [0, 1]; 0 for |N(v)| = 1)
```

- `H_norm(v) ≈ 0` — activation concentrated on one or few neighbors; clear dominant traversal direction
- `H_norm(v) ≈ 1` — activation spread uniformly; no dominant direction; structurally contested

Nodes with fewer than 2 neighbors receive `H_norm = 0.0` (no entropy signal possible).

### 4.3 Implementation

```rust
/// Compute normalized localized amplitude entropy for node v.
///
/// `neighbor_activations`: decay-adjusted activations of v's direct neighbors.
/// Returns H_norm ∈ [0.0, 1.0], or 0.0 if fewer than 2 neighbors.
pub fn localized_amplitude_entropy(neighbor_activations: &[f64]) -> f64 {
    let n = neighbor_activations.len();
    if n < 2 {
        return 0.0;
    }

    let total: f64 = neighbor_activations.iter().sum();
    if total < 1e-12 {
        return 0.0;
    }

    let entropy: f64 = neighbor_activations
        .iter()
        .filter(|&&a| a > 1e-12)
        .map(|&a| {
            let p = a / total;
            -p * p.ln()
        })
        .sum();

    let max_entropy = (n as f64).ln();
    if max_entropy < 1e-12 {
        0.0
    } else {
        (entropy / max_entropy).clamp(0.0, 1.0)
    }
}
```

---

## 5. New Scoring Dimensions (`ket-score`)

### 5.1 Current State

`ket-score` implements four scoring dimensions: `correctness`, `efficiency`, `style`, `completeness`, with auto, peer, and human sources.

### 5.2 Required Additions

**New dimension: `amplitude_entropy`**

- **Definition:** `localized_amplitude_entropy(decay_adjusted_neighbor_activations)`
- **Range:** `[0.0, 1.0]` — 0.0 = concentrated, 1.0 = uniform
- **Source:** O(degree) computation at query time
- **Tier 1 action:** If `amplitude_entropy > entropy_threshold` (configurable, default 0.75), flag the node for Tier 2 consistency investigation
- **Interpretation:** High entropy at node v means activation is spread roughly equally across v's neighbors — no dominant traversal direction, structurally contested position

**New dimension: `decay_adjusted_activation`**

- **Definition:** `decayed_activation(node.activation, node.last_written, config)`
- **Range:** `[activation_floor, ∞)` — replaces raw activation as the traversal weight
- **Source:** O(1) computation at query time from stored fields
- **Tier 1 action:** Used as the primary activation signal for depth scoring, replacing stored activation
- **Note:** The stored `activation` field remains unchanged; this dimension is a derived read-time value

**Modified dimension: `structural_centrality`**

- **Current:** Betweenness centrality computed offline over unweighted graph
- **Modified:** Decay-weighted betweenness centrality — shortest paths weighted by `decay_adjusted_activation` of intermediate nodes
- **Rationale:** A high-centrality node that has decayed to near-floor activation should not receive the same traversal priority as a recently active high-centrality node.

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
    pub amplitude_entropy: f64,          // 0.0 if degree < 2; always present
    pub decay_adjusted_activation: f64,  // Always present (= stored activation if no decay)
}

impl NodeScore {
    /// Composite traversal priority: weighted combination of all dimensions.
    /// High amplitude entropy raises investigation priority.
    pub fn traversal_priority(&self) -> f64 {
        let base = 0.3 * self.correctness
            + 0.2 * self.efficiency
            + 0.1 * self.style
            + 0.2 * self.completeness
            + 0.2 * self.decay_adjusted_activation;

        let entropy_modifier = if self.amplitude_entropy > 0.75 {
            1.5
        } else if self.amplitude_entropy > 0.5 {
            1.1
        } else {
            1.0
        };

        base * entropy_modifier
    }
}
```

---

## 6. WQS/Lagrangian Implications for Non-Stationary Landscapes (`ket-opt`)

### 6.1 Core Issue

The existing `ket-opt` WQS binary search finds optimal Lagrange multipliers under the assumption that the value function `f(k)` — expected information gain from k traversal steps — is concave in k. This concavity is guaranteed by the submodularity of information gain over spanning trees.

In a decaying landscape, `f(k, t)` is a function of both the traversal budget `k` and the query time `t`. The concavity property may fail in k at certain times because different nodes decay at different rates, creating a non-monotone information gain profile.

### 6.2 Specific Implications

**1. Time-indexed Lagrange multipliers.** The multiplier λ must be recomputed as the activation landscape shifts due to decay. For half-lives on the order of hours and query latency on the order of seconds, λ is effectively stable within a single query. For half-lives on the order of minutes, λ should be recomputed at the start of each query.

**2. Decay-adjusted edge costs.** The DP formulation computes costs over the spanning tree induced by the DAG. Each edge cost now calls `decayed_activation()` at evaluation time rather than reading a stored weight. This adds an O(1) constant factor per edge per DP evaluation.

**3. Concavity detection under asymmetric decay.** When per-node half-lives are heterogeneous, `f(k, t)` may become non-concave in k. Detection heuristic: if two successive WQS binary search iterations converge to the same k with different λ, the value function is non-concave at that budget level. Fall back to a bounded grid search over `[k_lo, k_hi]`.

**4. Skip-tier allocation under decay.** Tier allocations (Active, Lazy, Skip) must be re-evaluated at query time against current decay-adjusted activations, not cached from prior queries.

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

Concavity of `f(k)` is guaranteed when edge weights are non-negative and fixed — violated under heterogeneous per-node decay. In practice:
- **Homogeneous decay** (all nodes same half-life): concavity is preserved; standard WQS applies.
- **Low-variance heterogeneous decay** (half-lives within 2× of each other): degradation is mild; WQS with concavity check is sufficient.
- **High-variance heterogeneous decay** (half-lives spanning orders of magnitude): concavity fails at multiple budget levels; grid search fallback is required.

Expose `decay_variance` as a diagnostic metric on `DecayAwareOptimizer`. Log a warning when `decay_variance > threshold`.

---

## 7. MCP Interface Changes (`ket-mcp`)

### 7.1 New Tools (Non-Breaking Additions)

Two new tools are added to the MCP stdio JSON-RPC interface. Existing tools are unchanged.

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

---

**Tool: `entropy_check`**

Compute localized amplitude entropy at a target node, returning the neighbor activation distribution.

```json
{
  "name": "entropy_check",
  "description": "Compute localized amplitude entropy at a node. High entropy indicates activation spread evenly across neighbors with no dominant traversal direction — structurally contested position.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "node": { "type": "string", "description": "CID of target node" },
      "depth": { "type": "integer", "description": "Neighbor depth to include", "default": 1 }
    },
    "required": ["node"]
  }
}
```

Response:
```json
{
  "node": "CID string",
  "amplitude_entropy": 0.0,
  "neighbor_count": 4,
  "neighbor_distribution": [
    {
      "node": "CID string",
      "label": "neighbor label",
      "decay_adjusted_activation": 0.0,
      "weight": 0.0
    }
  ],
  "verdict": "concentrated | mixed | contested"
}
```

The `verdict` field:
- `concentrated`: `amplitude_entropy < 0.35`
- `mixed`: `0.35 <= amplitude_entropy < 0.75`
- `contested`: `amplitude_entropy >= 0.75` — Tier 2 investigation recommended

---

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
  "amplitude_entropy": 0.0
}
```

Both new fields are always present (no nullable values in the common path).

**Impact on existing clients:**
- Clients using flexible deserialization (`serde_json::Value`, dict-based) are unaffected.
- Clients deserializing to a fixed 4-field struct will fail with an unknown-field error.
- **Mitigation:** Strict clients should add `#[serde(default)]` on the new fields. Clients built in Python via `ket-py` are unaffected if using dict access.

**No other breaking changes:**
- Transport (stdio), framing (JSON-RPC 2.0), authentication, and tool discovery protocol are unchanged.
- Existing tool input schemas are unchanged.

---

## 8. Migration Path

### Phase 1: Decay infrastructure (non-breaking)
1. Add `DecayConfig` to `KetNode` with `serde(default)` — existing nodes deserialize with `half_life: None` (no decay)
2. Implement `decayed_activation()` in `ket-score`
3. Add `decay_adjusted_activation` to `NodeScore` — existing score consumers receive it as equal to `stored_activation` when no decay is configured
4. Add `decay_status` MCP tool

### Phase 2: Entropy scoring (score schema breaking change)
1. Implement `localized_amplitude_entropy()` in `ket-score`
2. Add `amplitude_entropy` field to `NodeScore`
3. Integrate entropy modifier into `traversal_priority()`
4. Add `entropy_check` MCP tool
5. Update `ket-agent`, `ket-py` deserializers
6. Bump MCP protocol version in tool manifest

### Phase 3: WQS optimizer extension
1. Implement `DecayAwareOptimizer` wrapping existing `WqsOptimizer`
2. Add concavity detection and grid-search fallback
3. Add `decay_variance` diagnostic metric

---

*This document is released under CC0 as prior art alongside the theoretical framework it specifies.*
