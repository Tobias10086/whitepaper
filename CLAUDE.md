# CLAUDE.md

## Mission

You are helping produce a **publishable whitepaper** for the **Affine** project by working from the **local repositories, local PDFs, local notes, and local codebase** in this workspace.

Your job is **not** to invent a story first and search for support later. Your job is to **extract the system as it actually exists**, then turn that evidence into a strong, coherent whitepaper.

The final paper should read like a serious technical whitepaper: precise, grounded, structured, and persuasive without overstating unsupported claims.

---

## Primary Goal

Draft the following whitepaper sections for the Affine project:

1. **Abstract**
2. **Introduction**
3. **The Affine Mechanism**
4. **System Architecture**
5. **Evaluation Environments**
6. **Scoring Logic**

   * Note: scoring may be integrated with **The Affine Mechanism** if that produces a stronger narrative.

The current emphasis is:

* **Abstract & Introduction**
* **The Affine Mechanism & Scoring**
  Based primarily on `affine-cortex`, explaining how Affine works and how the scoring logic operates.
* **System Architecture**
  Focus on how `affinetes` supports clusterized container deployment for environments, how `chutes` is currently used as the GPU inference cluster, and how the system may later adapt to **Targon** machines for a self-owned inference cluster.
* **Evaluation Environments**
  Focus on five environments:

  * **SWE** — `affine-swe-infinite`
  * **LIVEWEB** — `liveweb-arena`
  * **Memory** — `MemoryGym` and related MemoryBench materials
  * **NavWorld / QQR** — `affinetes/environments/qqr/`
  * **GAME / OpenSpiel** — `affinetes/environments/openspiel/`

---

## Operating Principles

### 1. Evidence first

Always ground claims in the local workspace.

Priority order for evidence:

1. **Source code**
2. **Config files / schemas / interfaces**
3. **READMEs / docs / comments**
4. **Local PDFs / reports / design notes**
5. **Commit history or issues**, only if available and useful

Do **not** make strong claims that cannot be tied back to concrete evidence.

### 2. No fabrication

If a metric, mechanism, pipeline detail, or comparison is not supported by available materials, do one of the following:

* mark it as **TBD**
* frame it as a **hypothesis**
* write it as a **future work direction**
* explicitly say **evidence not yet found in workspace**

Never invent benchmark numbers, architecture details, or design rationale.

### 3. Final prose should be in English

* Internal reasoning notes can be concise.
* The **whitepaper draft itself should be written in polished English**.
* Keep tone formal, technical, and investor/research-reader friendly.

### 4. Explain both *what* and *why*

For every major system component, explain:

* what it is
* how it works
* why it was designed this way
* what capability it improves
* how it compares to existing approaches
* what tradeoffs or limitations exist

### 5. Prefer concrete mechanisms over vague claims

Bad:

* “Affine is highly scalable and powerful.”

Good:

* “Affinetes packages environments into containerized services and supports cluster-oriented deployment, enabling reproducible environment execution and scalable task serving across multiple nodes.”

### 6. Be explicit about boundaries

When comparing against existing work, separate clearly between:

* **verified local evidence**
* **high-confidence synthesis**
* **claims that still require citation or external verification**

Do not blur these together.

---

## Expected Repository Targets

Treat these as likely source anchors and inspect them early:

* `affine-cortex`
* `affinetes`
* `affine-swe-infinite`
* `liveweb-arena`
* `affinetes/environments/qqr/`
* `affinetes/environments/openspiel/`
* local PDFs such as:

  * `MemoryGym(MemoryBench) 分析报告.pdf`
  * `QQR 旅行规划评测环境.pdf`

If file or directory names differ, discover the nearest equivalent and document the mapping.

---

## Whitepaper Narrative Requirements

The paper should present Affine as a **training-and-evaluation oriented agentic intelligence system**, not merely as a static benchmark suite.

The narrative should consistently convey that Affine is about:

* scalable evaluation of agentic models
* environment-grounded capability measurement
* training-friendly and replayable infrastructure
* open-ended or effectively unbounded task generation where possible
* stronger alignment between benchmark design and real-world agent behavior

---

## Section-Specific Guidance

## §1 Abstract

The abstract should answer:

* What is Affine?
* Why do current benchmarks or environments fall short?
* What are Affine’s core contributions?
* What kinds of environments and capabilities does it cover?
* Why is its infrastructure useful for both evaluation and training?

Keep it dense and crisp. Avoid marketing language.

---

## §2 Introduction

The introduction should establish:

* the limitations of static or saturated benchmarks
* why agentic capability requires **interactive environments**
* why infrastructure and environment design matter as much as model size
* why infinite or continuously synthesized task generation matters
* why Affine’s system-level approach is differentiated

It should end by previewing the paper’s main contributions.

Suggested contribution framing:

* a mechanism for agentic evaluation and scoring
* an infrastructure layer for clustered environment deployment
* a family of environments targeting distinct agentic capabilities
* training-friendly design choices for reproducibility and scale

---

## §3 The Affine Mechanism

This section should be written mainly from `affine-cortex`.

It should clearly describe:

* the main entities in the system
* how tasks are sampled
* how models/miners are evaluated
* how environments are invoked
* how scores are computed, aggregated, normalized, or ranked
* what design choices exist for fairness, stability, and anti-gaming
* how this mechanism differs from naive static benchmark execution

Potential substructure:

1. System participants and roles
2. Task lifecycle
3. Environment interaction loop
4. Per-task evaluation
5. Score aggregation and ranking
6. Stability / fairness / anti-copy or anti-overfitting considerations
7. Why the mechanism supports ongoing improvement

Where scoring is deeply intertwined with the mechanism, integrate scoring into this section naturally.

---

## §4 System Architecture

This section should focus on the architecture that makes Affine operational.

Mandatory themes:

* how **Affinetes** supports clusterized container deployment of environments
* how environments are packaged, exposed, orchestrated, and scaled
* how **Chutes** currently serves as the GPU inference cluster
* why this design separates environment execution from inference execution
* future compatibility with **Targon** machines for building a self-owned inference cluster

The section should explain **interfaces, deployment model, and scaling logic**, not just provide a repo tour.

Suggested substructure:

1. Architectural overview
2. Environment layer via Affinetes
3. Inference layer via Chutes
4. Execution flow across components
5. Scalability, reproducibility, and deployment benefits
6. Future evolution toward Affine-operated inference clusters

Highlight the most important architectural insight:
**Affine treats environments as scalable, containerized, reproducible services rather than one-off benchmark scripts.**

---

## §5 Evaluation Environments

This is one of the most important sections.

Write one substantial subsection for each environment. Each subsection should explain:

1. **What capability it measures**
2. **Why existing benchmarks are insufficient**
3. **How the environment is designed**
4. **How tasks are generated**
5. **Why the task supply can be effectively unbounded / renewable / replayable**
6. **How it can improve models during training**
7. **Its key advantages over prior approaches**
8. **Its current limitations or future work**

### §5a SWE — `affine-swe-infinite`

Focus points:

* Affine’s own implementation
* how infinite or continuously synthesized software engineering test data is built from network-retrieved or synthesized resources
* how this improves model performance on SWE-style tasks
* comparison to existing SWE frameworks and datasets
* strengths relative to `swe-bench`, `swe-bench-pro`, synthetic SWE datasets, SWE-smith-like directions, or related paradigms
* why renewable task generation matters for reducing saturation and contamination

### §5b LIVEWEB — `liveweb-arena`

Focus points:

* real-time website interaction as an evaluation environment
* how Affine measures browser-agent capability in a live or semi-live web setting
* comparison to existing browser-agent benchmarks/projects
* key design advantages such as:

  * renewable or effectively unbounded task construction
  * cacheability or replay-friendliness for training
  * closer alignment with real browser-agent workflows
* how this environment strengthens agentic planning, tool use, navigation, and real-world interaction

### §5c Memory — `MemoryGym` / MemoryBench materials

Focus points:

* Affine’s original innovation direction
* comparison to **LoCoMo**, **MemoryAgentBench**, **LongMemEval**, or similar memory-focused evaluation frameworks
* what kind of memory is being measured: retrieval, compression, budgeted storage, long-horizon multi-turn persistence, etc.
* what MemoryGym adds beyond existing memory benchmarks
* use local materials such as `MemoryGym(MemoryBench) 分析报告.pdf` as a primary source where available

### §5d NavWorld / QQR — `affinetes/environments/qqr/`

Focus points:

* explain the environment through the **QQR** framework
* what tool-using, planning, grounding, and decision-making abilities it tests
* how the environment supports renewable or effectively unbounded task generation
* why travel-planning style tasks are a useful proxy for structured agentic reasoning
* what kinds of model capabilities are improved by training on this environment
* use `QQR 旅行规划评测环境.pdf` where relevant

### §5e GAME / OpenSpiel — `affinetes/environments/openspiel/`

Focus points:

* built around **DeepMind OpenSpiel**
* what kinds of strategic, sequential, multi-step, adversarial, or planning abilities it probes
* why game environments matter for agent training and evaluation
* how this environment differs from simple static QA-style tests
* what kinds of reasoning and decision-making capabilities it can strengthen

---

## Required Writing Standard

Every section draft should be:

* **technically grounded**
* **clear to an informed reader outside the immediate team**
* **specific about mechanisms**
* **honest about limitations**
* **consistent in terminology**
* **free of unnecessary hype**

Avoid empty phrases such as:

* “revolutionary”
* “world-leading”
* “next-generation”
* “massively scalable”
  unless immediately justified by concrete evidence.

---

## Source Handling Rules

For each section, maintain a lightweight evidence trail.

At minimum, always keep:

* files inspected
* directories inspected
* key implementation anchors
* unresolved questions
* claims that still need verification

When writing draft text, prefer citing evidence internally using placeholders such as:

* `[Source: affine-cortex/path/to/file.py]`
* `[Source: affinetes/... ]`
* `[Source: MemoryGym(MemoryBench) 分析报告.pdf]`

These placeholders are for drafting discipline and can later be converted into final citations or footnotes.

---

## Mandatory Work Sequence

For any new section, follow this sequence:

1. **Inventory**
   Locate the relevant repos, directories, and PDFs.

2. **Evidence Extraction**
   Read code, docs, configs, and notes. Pull out mechanisms, not just names.

3. **Section Thesis**
   State the core message of the section in 2–4 bullets.

4. **Detailed Outline**
   Build the subsection structure before drafting prose.

5. **Draft**
   Write the section in publication-quality English.

6. **Verification Pass**
   Re-check each strong technical claim against workspace evidence.

7. **Revision Pass**
   Improve clarity, coherence, comparisons, and transitions.

---

## Output Format Per Loop Iteration

In each /loop cycle, produce output in this structure:

### Iteration Goal

State the exact target for this round.

### Sources Inspected

List the key files/repos/PDFs reviewed in this round.

### Extracted Evidence

Summarize the most important technical findings.

### Draft Delta

Show the new or revised prose created in this round.

### Open Questions

List missing evidence, unclear mechanisms, or decisions that still need confirmation.

### Next Iteration Plan

State the most valuable next step.

Keep this concise and execution-focused.

---

## Done Criteria

A section is only considered complete when all of the following are true:

* the section has a clear thesis
* its main claims are grounded in workspace evidence
* terminology is consistent with the codebase
* the prose is coherent and publication-ready
* comparisons to prior work are careful and not overstated
* unresolved assumptions are explicitly marked or removed

---

## What Not To Do

Do **not**:

* invent numbers
* invent architecture details
* claim superiority without concrete reasons
* copy README language uncritically
* produce vague summaries that do not explain mechanisms
* hide uncertainty when evidence is incomplete

---

## Collaboration Reminder

This project is high-context and repository-driven.
When in doubt, **inspect more code, write less hype**.

The goal is to produce a whitepaper that would still look credible to a technical reviewer who checks the repositories line by line.
