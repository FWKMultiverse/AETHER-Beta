# AETHER

> **PRIVATE REPOSITORY**  
> Development history only. No license is granted by access to this repository.

---

## Overview

AETHER is an AI system built around a different architectural premise than current mainstream approaches.

This is not a research paper, a demo, or a product announcement. It is a working codebase under active development. The architecture is complete. Training has not yet begun — by design, not by accident. There are specific reasons for that order of operations that anyone who has worked on systems of this kind will recognize.

A beta will be available when it's ready. Based on current progress, that is estimated around early 2027, though timelines for systems of this complexity are inherently uncertain.

There will be no demo before then. That is not a limitation — it is a deliberate decision. A system that integrates components at this level cannot be meaningfully demonstrated in parts. The whole is what matters.

---

## What This Is Built to Address

These are not novel observations. They are well-documented limitations of current AI architectures:

**Confidence without calibration.** Models generate responses with consistent confidence regardless of actual knowledge. This is an architectural property, not a tuning issue.

**Context without structure.** Current systems process meaning at the surface level. The structural relationships within information — how components relate, depend on, and constrain each other — are not directly reasoned about. They are approximated through pattern matching.

**Specialization through fragmentation.** The dominant approach to domain-specific capability is separate systems, separate pipelines, separate models. This creates coordination overhead and prevents genuine cross-domain reasoning.

**Static behavior after training.** Systems do not meaningfully improve from experience at inference time. What is learned during training is fixed.

AETHER was designed to address these at the architectural level. Not as features added on top of a standard architecture — as properties of how the system is built.

---

## What Is and Isn't Being Claimed

To be direct about what this project is:

The core architectural approach — specifically how structural reasoning and language generation are integrated at the processing level rather than the output level — is novel. Published research acknowledges this as an open problem. Existing work either keeps the two systems decoupled, or solves it narrowly for specific tasks.

AETHER implements this in a general-purpose system. That is a meaningful claim and it is backed by working code.

What this project is not claiming: that the problem of AI understanding, reasoning, or general intelligence is solved. It is not. AETHER is a significant architectural step in a direction that current systems have not taken. The results of training will determine how significant.

The reason there is no benchmark data, no paper, and no public demo is straightforward: the system has not been trained yet. Architectural claims can be evaluated by those with the technical background to read the implementation. Everything else waits for training to complete.

---

## On Existing Research

There is published research on combining graph neural networks with language models. This work is known, has been studied, and some of it is genuinely good. Dismissing it would be dishonest.

The three most relevant approaches in current literature:

**GNN-RAG** (2024) — Uses GNN to retrieve reasoning paths from knowledge graphs, then passes those paths as text input to an LLM. The two systems remain fully separate. GNN runs first, produces text, LLM reads that text. Integration happens at the input prompt level.

**GL-Fusion** (arXiv Dec 2024, submitted to ICLR 2025) — The closest published work to genuine deep integration, and worth understanding precisely. It introduces three innovations: Structure-Aware Transformers that embed GNN message-passing directly into LLM transformer layers (specifically at layers 0, 4, 8, 12, 16, 20, 24, 28 of a LLaMA-3-8B backbone), Graph-Text Cross-Attention that processes full uncompressed text from graph nodes and edges at alternating layers, and a GNN-LLM Twin Predictor that runs both prediction paths simultaneously. It achieves state-of-the-art results on OGBN-Arxiv (node classification) and OGBG-Code2 (function name generation from AST). This is serious, well-executed work. The scope is the limitation: GL-Fusion requires graph-structured input — specifically text-attributed graphs where nodes and edges carry labels. The GNN message-passing happens across explicit graph topology. It is evaluated on graph datasets and graph tasks. There is no general-purpose input path. A plain conversation, a block of code, or an ambiguous question — none of these are graph-structured data. GL-Fusion does not process them as-is.

**Hybrid-LLM-GNN** (2024) — Extracts embeddings from both GNN and LLM separately, then combines them for prediction. Demonstrates meaningful accuracy improvement in domain-specific tasks. The combination happens after both models have finished processing independently.

The pattern across all three — and across the broader field — is consistent: either the two systems remain decoupled and exchange results, or the integration is deep but narrow — built for graph-structured data and specific benchmark tasks, not general-purpose language understanding and generation.

The pattern across all three — and across the broader field — is consistent: either the two systems remain fully separate and pass outputs to each other, or the integration is deep but requires graph-structured input and task-specific training.

GL-Fusion is the most technically advanced of these approaches. Being precise about the difference matters. GL-Fusion integrates GNN into LLM for inputs that are already graphs. AETHER derives structural representations from arbitrary inputs — plain language, code, conversation — and integrates them into language model inference in real time, without requiring the input to be a graph. The input format is unrestricted. The structural reasoning is constructed from whatever arrives. That is a different problem, and it is the one that has not been solved in the published literature.

---

## On the Training Methodology

The training approach in AETHER is also worth addressing directly, because the field has moved in a specific direction and AETHER diverges from it.

**Where the field is now:**

The dominant trend in RL for LLMs has shifted from standard PPO toward GRPO — Group Relative Policy Optimization — which eliminates the value model entirely and instead scores multiple sampled outputs relative to each other. The motivation is computational efficiency. This is what DeepSeek-R1 used. It works well for reasoning tasks with clear right answers.

Constitutional AI takes a different approach — models critique and revise their own outputs based on predefined principles, which is effective for alignment but operates at the output level.

Both approaches share a common structure: generate output, score it externally, update the policy. The reward signal is applied after generation is complete.

**What AETHER does differently:**

The training loop is not generate → score → update. It is retrieve → critique → refine → score across multiple dimensions → store → update — with the graph embedding from the structural reasoning component entering the loss calculation directly, not as a post-hoc evaluation.

This means the model learns from outputs that have already been improved through a critique-refine cycle, not raw generation. The reward is three-layered: code quality evaluation, a comprehensive scoring component that weights quality, improvement, and novelty separately, and a human-signal component. Priority of replay experiences is influenced not only by TD error but also by the KL divergence between old and new policy — so experiences that caused large policy shifts are revisited more.

The specific values, coefficients, and implementation of the adaptive clipping and GAE smoothing are part of the protected implementation.

The short version: standard RLHF trains on what the model produced. AETHER trains on what the model produced after being told what was wrong with it, evaluated structurally, and given a multi-dimensional reward signal that distinguishes between fixing bugs and genuine improvement. Whether that produces meaningfully better results is an empirical question. The architecture is built to find out.

---

## Why the Code Is Not Public

The system is built as a single file containing all components. This is not a style choice — it is an architectural requirement. Every component depends on every other component in ways that make separation meaningless. GNN reasoning paths feed directly into language model inference. The routing system depends on fused state that only exists when both are running together. The training loop depends on outputs that only make sense in the context of the full system.

This also means the system cannot be partially demonstrated. Running one component in isolation produces a standard model. The behavior that makes AETHER different only exists when everything runs together. That is why there is no demo, no benchmark on isolated components, and no public code — not because there is nothing to show, but because a partial showing would be actively misleading.

The code will remain private until the system has been trained and the results speak clearly.

---

## Domains

The system is designed to handle the following without switching models or pipelines:

Software engineering — code generation, review, debugging, architectural reasoning
Research and analysis — synthesis, pattern recognition, structured investigation  
Financial analysis — evaluation and decision support
Writing and strategy — content, ideation, communication
Education — explanation, exercises, comprehension
Emotional support — context-aware conversation
Game development — logic, systems, behavior
General conversation — with genuine contextual understanding

---

## Graph Neural Network Architecture

AETHER uses seven distinct GNN types, each chosen for a specific function. This is not a single GNN applied to all problems — it is a set of specialized graph models that each reason about a different dimension of the input.

**HGT — Heterogeneous Graph Transformer** (WWW 2020, Hu et al.)  
Used for code structure analysis. HGT is designed for graphs where nodes and edges are of different types — exactly what an abstract syntax tree or a project dependency graph looks like. In published benchmarks on graphs with up to 179 million nodes, HGT outperformed all prior GNN baselines by 9–21%. In AETHER, HGT runs on CPU to leverage large cache architectures, processing code as a heterogeneous graph before its structural output is passed to the fusion layer.

**GATv2 — Graph Attention Network v2** (ICLR 2022, Brody et al.)  
Used for security analysis and social context. Standard GAT computes static attention — the same attention rankings regardless of the query node. GATv2 fixes this by making attention dynamic: the ranking changes depending on which node is asking. In AETHER, GATv2 runs two separate paths — one that flags structural security risks in input (adversarial patterns, injection attempts, policy violations), and one that models social relationships between concepts in conversation.

**GIN — Graph Isomorphism Network** (ICLR 2019, Xu et al.)  
Used for semantic understanding, emotional context, and general text. GIN is theoretically the most expressive of the standard message-passing GNNs — capable of distinguishing graph structures that weaker models treat as identical. AETHER uses GIN across three separate paths, each with different activation functions tuned to their purpose: standard ReLU for semantic understanding, Tanh for the emotional range (bounded output is more appropriate for sentiment), and GELU for general text (smoother gradient for longer reasoning chains).

**VisionGNN** — A GATv2-based graph model that processes visual input, representing image patches as graph nodes with spatial relationships as edges. Allows the same GNN-LLM integration to apply to visual inputs without a separate image processing pipeline.

**ContextRouterGNN** — A GATv2-based meta-graph model that does not process user input directly. It processes the outputs of all other GNN paths and learns to weight their relative contribution based on what the input context requires. This is the component that decides, for a given input, how much the emotional path matters versus the security path versus the structural code path.

All five text-processing GNN paths produce independent embeddings that are first projected to LLM hidden dimension space (each with its own projection layer), then fused through two mechanisms: an 8-head cross-attention layer that allows each path to attend to the others, and a concatenation-based fusion layer. The outputs of both mechanisms are combined at a fixed ratio before being injected into the language model's reasoning process.

The specific injection mechanism — how and where the fused GNN state enters the LLM — is part of the protected implementation.

---

## Agent Architecture and Routing

AETHER operates through a set of specialized agents that share a common foundation but are optimized for different types of tasks. These are not separate models. They are reasoning modes within the same system, each with domain-specific knowledge, tools, and behavior patterns.

The system currently defines agents across the following domains:

Software engineering — code generation, review, debugging, refactoring, migration across languages and frameworks  
Quality assurance — code review, bug identification, test generation  
Research and analysis — synthesis, investigation, scientific and mathematical reasoning  
System design — architecture planning, ML pipeline design  
Financial — real-time market analysis and advanced quantitative methods as separate agents  
Creative — writing, content strategy, and long-form creative work  
Education — adaptive explanation and comprehension  
Mental health — crisis-aware supportive conversation  
Career — job search, resume, professional development  
Daily life — health, scheduling, decision support  
Security — vulnerability detection and audit  
Game development — game logic, systems, and real-time state analysis  
Infrastructure — cloud, DevOps, deployment, database, API development  
Mobile — iOS and Android development  
Data science and machine learning — model development and analysis  
General conversation — context-rich open-ended dialogue  
Web search — real-time information retrieval integrated into responses  
Explainability — reasoning transparency on demand

**Routing — MixtureOfExpertsRouter with LearnedNeuralRouter**

Agent selection is handled by a Mixture of Experts routing system that combines two mechanisms: a neural router and a keyword fallback.

The neural router (LearnedNeuralRouter) is itself a GNN — using GATv2 for attention-based agent relationship modeling and GIN for learning interaction patterns between agents. It takes an embedding of the input and the current context, and learns which agent or combination of agents is most appropriate. This is not a static classifier. It learns from routing history and can represent complex relationships between agents — for example, understanding that a security question asked in the middle of a code review session should route differently than the same question in isolation.

The keyword fallback exists for reliability. If the neural router produces low-confidence output or encounters an input type it hasn't seen in sufficient quantity, the keyword system provides a deterministic backup. Production systems should not depend solely on learned routing, and AETHER does not.

When multiple agents are relevant, the router produces a ranked permutation ordered by relevance score — not a fixed priority ordering. This means the same agents may be invoked in different orders depending on context. The specific conflict resolution and coordination mechanism is part of the protected implementation.

---

Started: May 2025

| Milestone | Status |
|-----------|--------|
| Architecture design | Complete |
| Core implementation | Complete |
| Bug resolution and optimization | In progress |
| Training preparation | In progress |
| Training | Not yet started |
| Beta | Estimated early 2027 — subject to delay |

The commit history of this repository is the honest record of progress. It is also the timestamped proof of authorship.

---

## Intellectual Property

**© 2025–2026. All rights reserved.**

The following are claimed as original work, developed and first implemented beginning May 2025:

- The specific method of integrating graph-based structural reasoning into language model inference at the processing level, in a general-purpose architecture
- The multi-path parallel reasoning system and its cross-path fusion approach
- The context-aware routing system incorporating conversation trajectory
- The unified multi-agent architecture as a single coherent system
- The combined training methodology as implemented

This repository is private. Access does not grant any license to use, reproduce, modify, or build upon any part of this system. Unauthorized use of these methods will be treated as infringement.

---

*Development timeline, commit history, and implementation are the record. Nothing else needs to be said until training is complete.*
