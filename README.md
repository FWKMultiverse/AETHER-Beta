# AETHER

> **PRIVATE REPOSITORY**  
> Development history only. No license is granted by access to this repository.

---

## Overview

AETHER is an AI system built around a different architectural premise than current mainstream approaches.

This is not a research paper, a demo, or a product announcement. It is a working codebase under active development. The core architecture is complete. Training has not yet begun — by design, not by accident. There are specific reasons for that order of operations that anyone who has worked on systems of this kind will recognize.

A beta will be available when it's ready. Based on current progress, that is estimated around early 2027. Timelines for systems of this complexity are inherently uncertain, and this one will not ship before it is ready.

There will be no demo before then. That is not a limitation — it is a deliberate decision. A system where every component is interdependent cannot be meaningfully demonstrated in parts. The behavior that makes it different only exists when the whole runs together.

---

## What This Is Built to Address

These are not novel observations. They are well-documented limitations that affect every major AI system available today, and they stem from decisions made at the architectural level — which is why they cannot be fixed through fine-tuning or prompt engineering alone.

**Confidence without calibration.** Models generate responses with consistent confidence regardless of whether they actually know the answer. This is a structural property of how current systems are built, not a tuning issue. A model that has never seen a fact will generate it with the same fluency as one it has seen ten thousand times.

**Structural blindness.** Language models predict tokens based on surface patterns. They do not reason about structure — the relationships between components, how parts depend on and constrain each other. This is why generated code often fails: the model learned what code looks like, not how it works. Structural reasoning requires a different kind of representation entirely.

**Specialization through fragmentation.** The dominant approach to domain-specific capability is separate systems: separate models, separate pipelines, separate tools. This creates coordination overhead and makes genuine cross-domain reasoning structurally impossible. A system that is split cannot integrate the insight that a technical question has an emotional dimension, or that a financial decision has a security implication.

**Static behavior after training.** Standard systems do not improve from experience at inference time. What is learned during training is fixed. Interactions that go well and interactions that go poorly leave no trace that changes future behavior.

AETHER was designed to address these at the architectural level. Not as features added on top — as properties of how the system is built from the ground up.

---

## How the System Works

### From Input to Response

Every input AETHER receives is processed through multiple simultaneous reasoning paths before a single token of output is generated. These paths are not independent modules that produce separate outputs and hand them off — they influence each other and are integrated into the language model's running state before generation begins.

```
Input (text, code, or image)
  ↓
[Graph construction — input is converted to a token relationship graph on the fly]
  ↓
[Seven parallel GNN paths — each reasons about a different dimension simultaneously]
    ├── Code structure         (HGT)
    ├── Security signals       (GATv2)
    ├── Semantic relationships (GIN)
    ├── Emotional context      (GIN, Tanh-bounded)
    ├── Social context         (GATv2)
    ├── General text           (GIN, GELU)
    └── Visual structure       (VisionGNN — active when image is present)
  ↓
[Internal cross-path attention — GNN paths attend to each other before injection]
  ↓
[ContextRouterGNN — learns how much each path matters for this specific input]
  ↓
[Each GNN embedding injected into LLM hidden state
 via dedicated 8-head cross-attention + residual connection, sequentially]
  ↓
[LLM generates output from a hidden state continuously shaped by all GNN paths]
  ↓
Response
```

The critical distinction from published approaches: the language model does not receive a pre-processed summary from the GNNs and then generate. Each GNN embedding is injected into the LLM's running hidden state through dedicated cross-attention layers with residual connections — meaning the LLM's internal representation is modified by structural reasoning during processing, not before or after. The injection is sequential and cumulative: each path builds on the state left by the previous one. Seven injection points in total, each with its own normalization layer.

The ordering, weighting, and normalization of this injection process is part of the protected implementation.

### Graph Construction from Arbitrary Input

One of the core challenges in applying GNNs to language tasks is that language is not inherently graph-structured. The closest published work (GL-Fusion) sidesteps this by requiring pre-existing graph input. AETHER does not.

Tokens become nodes. An adaptive sliding window over the token sequence defines edges — the window size adjusts based on sequence length to capture local context without overwhelming longer inputs. This graph is constructed on the fly from any input: a plain question, a codebase, an ambiguous instruction, a conversation in progress. The GNN paths run on it regardless. This is the problem the published literature has not solved in a general-purpose system.

---

## Graph Neural Network Architecture

AETHER uses five GNN types across seven specialized paths. Each was selected for what it is theoretically and empirically best at.

**HGT — Heterogeneous Graph Transformer** (WWW 2020, Hu et al.)  
Designed for graphs where nodes and edges are of different types — which is exactly what code structure looks like. An abstract syntax tree has function nodes, variable nodes, call edges, assignment edges. HGT was built for this. In benchmarks on graphs with up to 179 million nodes, HGT outperformed all prior GNN baselines by 9–21% on heterogeneous tasks. In AETHER, HGT processes code and project structure on CPU, leveraging large cache architectures, then passes its structural embedding to the injection layer.

**GATv2 — Graph Attention Network v2** (ICLR 2022, Brody et al.)  
Standard GAT computes static attention — the same weights regardless of which node is querying, making it unable to represent "A matters more when asked from B than from C." GATv2 fixes this with fully dynamic attention conditioned on both nodes. In AETHER, GATv2 runs two paths: one that identifies structural security signals in input (adversarial patterns, injection attempts, policy violations), and one that models social and relational dynamics between concepts in conversation.

**GIN — Graph Isomorphism Network** (ICLR 2019, Xu et al.)  
Theoretically the most expressive standard message-passing GNN — proven to be at least as powerful as the Weisfeiler-Leman isomorphism test, meaning it can distinguish graph structures that other GNN families cannot. AETHER uses GIN across three paths, each with activation functions chosen for their purpose:
- **Semantic** — ReLU, standard gradient propagation
- **Emotional** — Tanh, bounded output appropriate for sentiment representation
- **General text** — GELU, smoother gradient curve for longer reasoning chains

**VisionGNN** — GATv2-based. Processes image patches as graph nodes with spatial relationships as edges. Integrates visual input into the same GNN-LLM fusion pipeline, without a separate vision encoder or a separate model for multimodal tasks.

**ContextRouterGNN** — GATv2-based meta-model. Does not process user input directly. It takes the output embeddings of all other GNN paths as its input and learns to produce a weighted representation of what the current context requires. This is the component that dynamically adjusts how much emotional context matters versus security signals versus code structure — learned from patterns across inputs, not configured by a fixed rule.

Before any GNN embedding reaches the LLM injection layer, the five text paths first interact through an internal 8-head cross-attention mechanism within TextContextGNN. Each path can adjust its output based on what the others found. The attention-attended and concatenation-fused outputs are combined before projection into LLM hidden dimension space. Each path then enters the LLM through its own dedicated cross-attention + residual + LayerNorm block.

---

## Agent Architecture and Routing

AETHER operates through specialized agents that share the same underlying model and GNN-augmented reasoning but are optimized for different domains. These are not separate models — they are reasoning modes within one system, each with domain-specific behavior, tools, and response patterns.

| Domain | Scope |
|--------|-------|
| Software Engineering | Code generation, review, debugging, refactoring, migration |
| Quality Assurance | Code review, test generation, bug identification |
| Research & Science | Synthesis, scientific and mathematical reasoning |
| System Architecture | ML pipeline design, architectural planning |
| Security | Vulnerability detection, security audit, structural input analysis |
| Financial | Real-time market analysis and advanced quantitative methods |
| Data & ML | Data science, model development, statistical analysis |
| Infrastructure | Cloud, DevOps, CI/CD, deployment, database, API |
| Mobile | iOS and Android development |
| Game Development | Logic, systems design, real-time state analysis, reinforcement learning |
| Creative | Writing, content strategy, long-form creative work |
| Education | Adaptive explanation, exercise generation, comprehension tracking |
| Mental Health | Crisis-aware, emotionally calibrated supportive conversation |
| Career | Resume, job search, professional development |
| Daily Life | Health, scheduling, decision support |
| General Conversation | Context-rich open-ended dialogue |
| Web Search | Real-time information retrieval integrated into responses |
| Explainability | Reasoning transparency on demand |

**MixtureOfExpertsRouter and LearnedNeuralRouter**

Agent selection uses two mechanisms in combination. The primary is a learned neural router (LearnedNeuralRouter) that is itself a GNN — GATv2 for attention-based modeling of agent relationships, and GIN for learning interaction patterns. It takes the current input embedding and conversation context, and produces a ranked permutation of relevant agents ordered by relevance score — not a fixed priority. The same input asked in different conversation contexts can route to different agents, or to the same agents in a different order.

The secondary mechanism is a keyword fallback. If the neural router produces low-confidence output, the keyword system provides a deterministic backup. Systems that depend entirely on learned routing in production are brittle under distribution shift. AETHER is built to be used, not only to be evaluated on benchmarks.

Conflict resolution when multiple agents are invoked simultaneously, and the coordination mechanism between them, are part of the protected implementation.

---

## On Existing Research

There is published research on combining GNNs with language models. This work is known and has been studied carefully. The three most relevant approaches:

**GNN-RAG** (2024) — GNN retrieves reasoning paths from a knowledge graph and passes them as text to an LLM. The systems remain completely separate. Integration happens at the prompt level before the LLM runs. The LLM receives a text summary of what the GNN found — it does not interact with the GNN during inference.

**GL-Fusion** (arXiv Dec 2024, submitted ICLR 2025) — The closest published work to deep integration, and worth being precise about. GL-Fusion injects GNN message-passing directly into LLM transformer layers at specific positions (layers 0, 4, 8, 12, 16, 20, 24, 28 of a LLaMA-3-8B backbone), uses Graph-Text Cross-Attention at alternating layers, and runs a Twin Predictor for both GNN and LLM prediction paths simultaneously. It achieves state-of-the-art on OGBN-Arxiv and OGBG-Code2. This is serious, well-executed work. The constraint is scope: GL-Fusion requires text-attributed graph input — graphs that already have explicit node and edge structure. Its benchmarks are graph datasets and graph tasks. A plain conversation, a block of code without pre-existing graph structure, or an image are not valid inputs without external graph construction preprocessing. GL-Fusion is deep integration for graph data. AETHER is deep integration for anything.

**Hybrid-LLM-GNN** (2024) — GNN and LLM produce embeddings independently, then combine them for a final prediction. Demonstrates accuracy improvements on domain-specific tasks. Integration happens after both systems finish processing independently.

The consistent pattern: either integration happens at the boundaries, or it is deep but constrained to pre-structured graph input. Neither approach addresses integration of structural reasoning into LLM inference over unrestricted general text and code.

---

## On the Training Methodology

**Where the field is**

The dominant trend has moved from standard PPO toward GRPO (Group Relative Policy Optimization) — which eliminates the value model and evaluates outputs relative to each other within a batch. DeepSeek-R1 used this. It is computationally efficient and performs well for tasks with clear correct answers.

Constitutional AI uses a critique-revise loop against predefined principles — effective for alignment, but revision happens at the output level after generation is complete.

Both follow the same basic loop: generate → score → update. The model learns from what it produced.

**What AETHER does differently**

The loop is: retrieve relevant experience → generate → critique → refine → score across three layers → store with priority → update policy with adaptive clipping.

The model learns from refined output, not raw generation. The reward is three-layered: code quality evaluation, a comprehensive scoring component that weights quality, novelty, and measurable improvement separately, and a human-signal component. The GNN structural embedding enters the loss function directly — not as annotation after the fact, but as part of what the training signal is computed from.

Prioritized replay weights experiences by KL divergence between old and new policy — not only by TD error — so interactions where the model's behavior changed most significantly are revisited during training. Adaptive epsilon clipping adjusts the clip range based on live KL divergence: tightening when policy shifts are large, relaxing when they are small.

The specific coefficients, smoothing values, and implementation structure are part of the protected implementation.

The short version: standard RL trains on what the model generated. AETHER trains on what the model generated after being critiqued and given the opportunity to improve — scored structurally, weighted toward the experiences that carry the most learning signal. Whether this produces meaningfully better results than GRPO is an empirical question. The architecture is built to find out.

---

## Why the Code Is Not Public

The system is a single codebase. This is an architectural requirement, not a convention. Every component has dependencies that only exist in the context of the full running system. The routing decision affects GNN path weighting. The GNN paths modify the LLM's hidden state before generation. The training loop depends on outputs from a system where both GNN and LLM are active simultaneously. There is no meaningful subset.

Running any component in isolation produces either a standard model or nothing useful. A partial demonstration would be actively misleading about what the system is. There is no demo, no isolated benchmark, and no public code — not because there is nothing to show, but because showing a part would misrepresent the whole.

The code will remain private until training is complete and the results can speak directly.

---

## Development Status

Development started May 2025.

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

The following are claimed as original work, first developed and implemented beginning May 2025:

- The method of constructing graphs from unrestricted text and code input and integrating multi-path GNN reasoning into LLM hidden state via sequential cross-attention injection at inference time, in a general-purpose architecture
- The seven-path parallel GNN system with per-path dedicated cross-attention, residual injection, and LayerNorm into a running LLM hidden state
- The ContextRouterGNN — a GNN-based meta-router that dynamically weights GNN path contributions based on learned input context
- The MixtureOfExpertsRouter with GNN-based LearnedNeuralRouter incorporating full conversation trajectory for agent selection
- The training methodology: critique-refine loop, three-layer reward, GNN embedding in loss computation, KL-weighted prioritized replay, and adaptive epsilon clipping as implemented

This repository is private. Access does not grant any license to use, reproduce, modify, or build upon any part of this system. Unauthorized use of these methods will be treated as infringement.

---

*The commit history is the record. The results will speak when training is complete.*
