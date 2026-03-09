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

## Development Status

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
