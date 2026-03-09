# AETHER

> **PRIVATE REPOSITORY**  
> Development history only. No license is granted by access to this repository.

---

## Overview

AETHER is an AI system built around a different architectural premise than current mainstream approaches.

This is not a research paper, a demo, or a product announcement. It is a working codebase under active development. The core architecture is complete. Training has not yet begun — by design, not by accident.

A beta will be available when it is ready. Based on current progress, that is estimated around early 2027. Timelines for systems of this complexity are inherently uncertain, and this one will not ship before it is ready.

There will be no demo before then. A system where every component depends on every other cannot be meaningfully demonstrated in parts. Running one component in isolation produces a standard model. The behavior that distinguishes AETHER only exists when the full system runs together.

---

## What This Is Built to Address

These are not fringe observations. They are well-documented limitations that affect every major AI system available today — and they come from decisions made at the architectural level, not from insufficient training data or model size.

**Confidence without calibration.** Current models generate responses with consistent fluency regardless of whether the underlying knowledge is solid or absent. A model that has seen a fact ten thousand times produces it with the same confidence as one it has never encountered. This is structural, not a tuning issue.

**Structural blindness.** Language models learn surface patterns. They approximate structure through repetition — enough examples of how code, logic, or argument structure looks, and the output resembles those patterns. But resemblance is not understanding. The model does not reason about relationships between components; it predicts tokens that are statistically likely given what it has seen. This is why generated code often fails on edge cases, and why outputs that look correct are often not.

**Specialization through fragmentation.** The standard solution to multi-domain AI is multiple systems: separate models, tools, and pipelines that hand off to each other. This works, up to a point. What it cannot do is integrate — recognizing that a technical question carries emotional weight, or that a financial decision has a security implication, requires a system where those reasoning types genuinely interact. Separate pipelines cannot do this.

**No improvement from experience.** After training, standard models are fixed. What is learned is what is learned. Interactions go well or poorly and leave no trace. AETHER is designed so that relevant past interactions influence how future ones are handled — not through naive memorization, but through a structured memory system with multiple retrieval layers.

---

## How the System Works

### The Core Idea

Most AI systems work like this: input arrives, the language model processes it, a response is generated. If additional processing happens — tool use, retrieval, safety checks — it happens before or after the language model runs, as a separate step.

AETHER works differently. Before the language model generates a single token, the input is analyzed through seven parallel graph-based reasoning paths simultaneously. The outputs of those paths are injected directly into the language model's internal state — not as additional context it reads, but as modifications to how it represents and processes the input at inference time. By the time generation begins, the model is reasoning from a hidden state that has been continuously shaped by structural, semantic, emotional, social, security, and contextual analysis of the input.

The language model and the structural reasoning components are not two systems that communicate. They are one system.

### From Input to Response

```
Input (text, code, or image)
  ↓
[Graph construction — token relationship graph built on the fly from any input]
  ↓
[Seven parallel GNN paths — running simultaneously]
    ├── Code structure         (HGT)
    ├── Security signals       (GATv2)
    ├── Semantic relationships (GIN)
    ├── Emotional context      (GIN, Tanh-bounded)
    ├── Social context         (GATv2)
    ├── General text           (GIN, GELU)
    └── Visual structure       (VisionGNN — when image is present)
  ↓
[Cross-path attention — paths attend to each other before injection]
  ↓
[ContextRouterGNN — learns how much each path matters for this input]
  ↓
[Sequential injection into LLM hidden state
 — each path via dedicated 8-head cross-attention + residual + LayerNorm]
  ↓
[Policy head generates output. Value head produces scalar for training.]
  ↓
Response
```

### Why Sequential Injection Matters

Each GNN path's embedding is injected into the LLM hidden state one at a time, in an order determined by context. Each injection modifies the state that the next injection builds on. By the end, the state that the language model reasons from has been shaped by all seven paths — and the contributions are not additive patches, they are integrated modifications with residual connections preserving what was already there.

This is not the same as prompting the model with the outputs of seven analysis tools. The injection happens inside the model's representation of the input, not in the text it receives. The distinction is meaningful: a model told "this input has emotional content" will process that differently than a model whose internal representation of the input has been directly modified by an emotional reasoning component.

### Graph Construction from Any Input

Applying GNNs to language requires converting language to a graph. The closest published work sidesteps this by requiring graph-structured input to begin with. AETHER does not.

Every input — a question, a codebase, a conversation, an image — is converted to a token relationship graph at inference time. Tokens become nodes. An adaptive sliding window defines edges, with window size adjusting based on sequence length. This graph is then processed by all seven GNN paths. The input format is unrestricted. This is the part of the problem that the existing literature has not solved in a general-purpose system.

---

## Graph Neural Network Architecture

Five GNN types across seven specialized paths. Each was selected for specific theoretical and empirical properties.

**HGT — Heterogeneous Graph Transformer** (WWW 2020, Hu et al.)  
Built for graphs where nodes and edges are of different types — which is exactly the structure of code. A codebase has function nodes, class nodes, variable nodes, import edges, call edges, assignment edges. Homogeneous GNNs treat all of these the same. HGT does not. In benchmarks on graphs up to 179 million nodes, HGT outperformed prior baselines by 9–21% on heterogeneous tasks. In AETHER, HGT runs on CPU to leverage large cache architectures, processes code and project structure as a heterogeneous graph, and passes a structural embedding to the injection pipeline.

**GATv2 — Graph Attention Network v2** (ICLR 2022, Brody et al.)  
The original GAT has a known limitation: attention weights are computed statically, depending only on the pair of nodes, not on which node is querying. The formal consequence is that GAT cannot distinguish certain graph structures that a dynamic attention mechanism could. GATv2 fixes this — attention is fully conditioned on both the source and target node at query time. In AETHER, GATv2 runs two separate paths: structural security analysis (adversarial patterns, injection attempts, policy violations in the input) and social context modeling (relational dynamics between concepts in conversation).

**GIN — Graph Isomorphism Network** (ICLR 2019, Xu et al.)  
Proven to be as expressive as the Weisfeiler-Leman graph isomorphism test — the theoretical ceiling for message-passing GNNs. In practice, this means GIN can distinguish graph structures that GCN, GraphSAGE, and standard GAT cannot. AETHER uses GIN for three paths, each with activation functions matched to their function:

| Path | Activation | Reason |
|------|-----------|--------|
| Semantic | ReLU | Standard, effective for relational structure |
| Emotional | Tanh | Bounded output — appropriate for sentiment, prevents unbounded amplification |
| General text | GELU | Smoother gradient, better for longer dependency chains |

**VisionGNN** — GATv2-based. Image patches become nodes; spatial relationships between patches become edges. Visual input enters the same GNN-LLM fusion pipeline as text, without a separate vision encoder or multimodal model. One system handles both.

**ContextRouterGNN** — GATv2-based meta-model. Does not process user input. It takes the output embeddings of all other GNN paths and learns to produce a dynamically weighted composite based on what the current context requires. For a technical debugging conversation, it learns to weight the code structure and semantic paths more. For a distressed user, emotional context more. This weighting is learned, not configured — and it is a live output of the model at inference time, not a static routing rule.

Before any path reaches the LLM, the five text paths run through an internal 8-head cross-path attention layer — each path can adjust based on what the others found. Emotional context can modify what semantic emphasis surfaces; security signals can affect how general understanding is weighted. The attended and concatenation-fused outputs combine before projection to LLM hidden dimension.

---

## Memory Architecture

AETHER maintains four layers of memory operating at different timescales.

**L1 — Redis (hot cache)** Immediate working context. Sub-millisecond retrieval for active session state with TTL-based expiration.

**L2 — Qdrant (semantic search)** Session and medium-term memory. Vector similarity search using cosine distance over 768-dimensional embeddings. Retrieval is semantic — similar past contexts surface regardless of keyword match.

**L3 — MinIO (long-term storage)** Persistent storage with versioning for high-value experiences — currently filtered at reward threshold 8.0+. Designed for experiences the system should be able to draw on across sessions.

**L4 — Local indexed cache** Intelligent indexing by reward score, programming language, code patterns, and complexity. Retrieval combines semantic similarity with reward-boosted scoring — past solutions that worked well surface with higher priority than those that were merely related.

These layers are not independent. The retrieval system combines signals across layers, and the reward system that determines what gets stored to L3 and L4 is the same reward signal used in training. What the system learns to value is what it learns to remember.

---

## Self-Monitoring and Safety Architecture

**HallucinationDetector** — Checks responses before they are returned, using consistency analysis, uncertainty keyword detection, factual claim pattern matching, and cross-verification. The system is designed to recognize when it is making claims it cannot support and to express that uncertainty rather than generate with false confidence.

**ConstitutionalAI** — A critique-and-refine layer that evaluates outputs against constitutional principles and rewrites those that violate them, using the model itself to produce the refinement rather than a rule-based filter.

**DeepSelfAwarenessSystem / SelfAwarenessSystem** — Monitors the model's own outputs for signs of overconfidence, inconsistency, and boundary violations. Tracks whether the model is representing its limitations accurately.

**AdvancedSafetySystem** — Structural safety evaluation at the input and output level, including pattern detection for injection attempts and policy violations. Separate from the security GNN path — this operates on the response side, not the input analysis side.

These components are active in the BaseAgent layer and apply to all agents in the system. They are not optional modules for specific domains — they run on every response.

---

## Agent Architecture and Routing

All agents share the same underlying model and GNN-augmented reasoning. They are reasoning modes within one system, not separate models.

| Domain | Scope |
|--------|-------|
| Software Engineering | Code generation, review, debugging, refactoring, migration |
| Quality Assurance | Code review, test generation, bug identification |
| Research & Science | Synthesis, scientific and mathematical reasoning |
| System Architecture | ML pipeline design, architectural planning |
| Security | Vulnerability detection, audit, structural input analysis |
| Financial | Real-time market analysis and advanced quantitative methods |
| Data & ML | Data science, model development, analysis |
| Infrastructure | Cloud, DevOps, CI/CD, deployment, database, API |
| Mobile | iOS and Android development |
| Game Development | Logic, systems design, real-time state analysis |
| Creative | Writing, content strategy, long-form creative work |
| Education | Adaptive explanation, exercise generation, comprehension |
| Mental Health | Crisis-aware, emotionally calibrated supportive conversation |
| Career | Resume, job search, professional development |
| Daily Life | Health, scheduling, decision support |
| General Conversation | Context-rich open-ended dialogue |
| Web Search | Real-time retrieval integrated into responses |
| Explainability | Reasoning transparency on demand |

**MixtureOfExpertsRouter and LearnedNeuralRouter**

The primary routing mechanism is a learned GNN — GATv2 for modeling agent relationships, GIN for learning interaction patterns. It produces a ranked permutation of relevant agents ordered by relevance score, incorporating full conversation history. The same question asked after a technical conversation routes differently than the same question asked after an emotional one.

The keyword fallback exists because learned routing fails under distribution shift. A production system needs a deterministic path when confidence is low. AETHER has both.

---

## On Existing Research

**GNN-RAG** (2024) — GNN retrieves reasoning chains from a knowledge graph and passes them as text to the LLM. Fully decoupled. Integration at the prompt level. The LLM reads a summary of what the GNN found; it does not interact with the GNN.

**GL-Fusion** (arXiv Dec 2024, submitted ICLR 2025) — The technically closest published work. Injects GNN message-passing into LLM transformer layers at specific positions (layers 0, 4, 8, 12, 16, 20, 24, 28 of LLaMA-3-8B), uses Graph-Text Cross-Attention at alternating layers, and runs a GNN-LLM Twin Predictor simultaneously. State-of-the-art on OGBN-Arxiv and OGBG-Code2. Serious work. The constraint: GL-Fusion requires text-attributed graph input — graphs with explicit node and edge labels. It is evaluated on graph datasets and graph tasks. A plain sentence, a codebase, or a conversation are not valid inputs without external graph construction. GL-Fusion integrates GNN into LLM over structured graph data. AETHER integrates GNN into LLM over anything.

**Hybrid-LLM-GNN** (2024) — GNN and LLM produce embeddings independently; outputs are combined for final prediction. Demonstrates domain-specific accuracy improvements. Integration after both finish processing independently.

The pattern across all three is the same: either the systems remain decoupled, or integration is deep but constrained to pre-existing graph structure. The general-purpose version of the deep integration problem has not been solved in the published literature.

One question worth addressing directly: if this is an unsolved research problem, why would a non-research project solve it first? The honest answer is that research optimizes for novelty and rigor of proof. Engineering optimizes for whether something works. Sometimes a working implementation precedes a formal proof that it is the right approach. Whether AETHER's implementation holds up under training is an open empirical question — but the architecture is complete, and the implementation exists.

---

## On the Training Methodology

**Where the field is**

GRPO (Group Relative Policy Optimization, DeepSeek-R1) eliminates the value model and scores outputs relative to each other within a batch. Computationally efficient, works well for tasks with clear correct answers.

Constitutional AI uses critique-revise loops against predefined principles — effective for alignment, operates at the output level after generation is complete.

Both: generate → score → update. Learn from what was produced.

**What AETHER does differently**

Loop: retrieve relevant past experience → generate → critique → refine → score across three layers → store with priority weighting → update policy.

The model trains on critiqued-and-refined output, not raw generation. Reward is three-layered — code quality, a composite scoring component weighting quality/novelty/measurable improvement separately, and a human signal. The GNN structural embedding enters the loss directly, not as post-hoc annotation.

Prioritized replay is weighted by KL divergence — not only TD error. Experiences where the policy changed most are revisited more. Adaptive epsilon clipping adjusts based on live KL: tightening when shifts are large, relaxing when they are small. CuriosityModule provides an additional intrinsic motivation signal — forward model prediction error — which drives exploration toward unfamiliar input distributions rather than only optimizing on what has been seen.

Specific coefficients and implementation structure are protected.

The short version: standard RL trains on what the model generated. AETHER trains on what the model generated, then critiqued, then improved — scored structurally, with memory-weighted experience retrieval and policy-shift-weighted replay. Whether this outperforms GRPO on real-world tasks is unknown until training completes. The architecture makes a coherent argument for why it should.

---

## Why the Code Is Not Public

The system is a single codebase because it has to be. Every component depends on state that only exists in the context of the full running system. The routing decision affects GNN path weighting. The GNN paths modify the LLM's hidden state before generation. The training loop depends on outputs that require both GNN and LLM to be active simultaneously. Separating any component produces either a standard model or something that does not run.

There is no partial demonstration that would represent the system accurately. Showing one component would be showing something different from what AETHER is.

The code will remain private until training is complete.

---

## Development Status

Started May 2025.

| Milestone | Status |
|-----------|--------|
| Architecture design | Complete |
| Full system implementation | Complete |
| Bug resolution and optimization | In progress |
| Training infrastructure preparation | In progress |
| Training | Not yet started |
| Beta | Estimated early 2027 — subject to delay |

The commit history is the timestamped record of development and the proof of authorship.

---

## Intellectual Property

**© 2025–2026. All rights reserved.**

The following are claimed as original work, first implemented beginning May 2025:

- The method of constructing token relationship graphs from unrestricted text, code, and image input and integrating multi-path GNN reasoning into LLM hidden state via sequential cross-attention injection at inference time, in a general-purpose architecture
- The seven-path parallel GNN system with per-path dedicated cross-attention, residual injection, and LayerNorm into a running LLM hidden state
- The ContextRouterGNN — a GNN-based meta-router that dynamically weights GNN path contributions at inference time based on learned context
- The MixtureOfExpertsRouter with GNN-based LearnedNeuralRouter incorporating full conversation trajectory for agent selection and ranked permutation routing
- The four-layer hierarchical memory system as integrated with the GNN-LLM architecture and reward-weighted retrieval
- The training methodology: critique-refine loop, three-layer reward, GNN embedding in loss computation, KL-weighted prioritized replay, adaptive epsilon clipping, and CuriosityModule integration, as implemented

This repository is private. Access does not grant any license to use, reproduce, modify, or build upon any part of this system. Unauthorized use of these methods will be treated as infringement.

---

*The commit history is the record. The results will speak when training is complete.*
