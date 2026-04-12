# Affine: An Incentivized Reinforcement Learning Environment for Open Reasoning

---

## 1. Abstract

Static benchmarks are structurally inadequate for evaluating and training agentic intelligence: fixed task sets saturate through memorization and contamination, single-turn formats miss interactive capabilities, and one-off evaluation scripts do not scale to continuous training loops. Affine is a decentralized evaluation and training system that addresses these limitations through three integrated contributions. First, a **scoring mechanism** built on Pareto dominance filtering, ELO-based temporal ratings, and a two-signal anti-copy detector that incentivizes genuine model improvement on the Bittensor network while resisting plagiarism and gaming. Second, a **container-orchestration infrastructure layer** (Affinetes) that packages evaluation environments as reproducible, isolated Docker services with automatic lifecycle management, SSH-tunneled communication, and multi-instance load balancing — cleanly separating environment execution from GPU-backed model inference. Third, a **family of five interactive evaluation environments** spanning software engineering (SWE-Infinite: continuously generated, multi-language task instances from auto-discovered GitHub repositories), browser-grounded web interaction (LiveWeb Arena: 197 million task configurations with real-time ground truth), agent memory management (MemoryGym: selective storage and maintenance under write-budget constraints), tool-mediated planning (QQR: six MCP tools over 10,000+ renewable travel planning tasks), and strategic game-playing (OpenSpiel: 22 games across seven quality tiers with trajectory diversity exceeding 10^60). Beyond evaluation, the system integrates a KL-divergence distillation environment (DISTILL) that provides per-token training signals from teacher rollouts, enabling direct policy improvement alongside evaluation. All environments provide deterministic seeding, step-level reward signals, and renewable task generation — making them usable as both evaluation benchmarks and reinforcement learning training substrates.

---

## 2. Introduction

The dominant paradigm for evaluating large language models relies on static benchmark suites: fixed collections of question-answer pairs scored by exact match, multiple choice accuracy, or automated code execution. Benchmarks such as MMLU, HumanEval, and GSM8K have driven rapid progress in model development, but they share structural limitations that become acute as the field moves toward agentic systems — models that must act, plan, use tools, and interact with dynamic environments over extended horizons.

**Static benchmarks saturate.** A fixed test set of finite size is vulnerable to contamination through training data overlap, deliberate or accidental memorization, and benchmark-specific optimization. SWE-bench Pro, one of the most rigorous software engineering benchmarks, contains approximately 2,300 instances. Once a model has been trained on or tuned against a dataset of this scale, marginal score improvements may reflect overfitting rather than genuine capability gains. The problem compounds in open networks where model weights are public: any participant can download a leading model, make superficial modifications, and claim credit for equivalent performance. [affine-swe-infinite/docs/en/01-HIGH-LEVEL-DESIGN.md]

**Static benchmarks miss agentic capability.** Recent interactive benchmarks — WebArena for browser navigation, AgentBench for multi-domain tool use, Mind2Web for web interaction trajectories, ToolBench for API orchestration — have begun to address this gap. However, most rely on fixed website snapshots, human-authored trajectories, or small task pools that are themselves subject to saturation. A model that scores well on closed-form reasoning may fail at multi-step tool orchestration, real-time web navigation, or long-horizon memory management under budget pressure. What is needed is not merely interactive evaluation, but interactive evaluation with a *renewable* task supply. [affinetes/environments/qqr/README.md]

**Fixed task sets cannot serve as training substrates.** For reinforcement learning to work, the training environment must supply a continuous stream of diverse tasks. Even SWE-bench Pro's 2,300 instances are exhausted within a handful of training epochs. What is needed is not a larger static dataset but a generative process that produces fresh, validated tasks continuously — tasks diverse enough to prevent overfitting and structured enough to provide meaningful reward signals.

**Infrastructure is an underappreciated bottleneck.** Even when interactive environments exist, deploying them at scale introduces engineering challenges that benchmark papers rarely address: containerization, reproducibility across machines, concurrent evaluation of multiple models, secure communication, and efficient resource management. Without a robust infrastructure layer, interactive environments remain research prototypes rather than operational systems.

Affine addresses these limitations through a system-level approach built on **Bittensor**, a decentralized network in which independent participants (miners) compete to provide useful computational work and are rewarded with token emissions proportional to their demonstrated quality. Bittensor organizes work into subnets, each governed by validators who score miner contributions and set on-chain weights. Affine operates as Subnet 120, directing this incentive structure toward the specific goal of improving agentic reasoning models.

The system combines four contributions:

1. **A decentralized evaluation and scoring mechanism** that incentivizes genuine model improvement while resisting plagiarism, overfitting, and gaming. The mechanism uses Pareto dominance filtering with statistical significance thresholds, ELO-based temporal ratings, and a two-signal anti-copy detector that analyzes model internals. Miners submit models that are evaluated across the full environment suite, with emissions allocated proportionally to demonstrated capability.

2. **A container-orchestration infrastructure layer** (Affinetes) that packages evaluation environments as reproducible, isolated Docker services with automatic lifecycle management, secure SSH-tunneled communication, and multi-instance load balancing. Environment developers write only business logic; the framework handles deployment, scaling, and inter-process communication. This separation allows environments to be developed independently and deployed across distributed clusters.

3. **A family of interactive evaluation environments** targeting capabilities that static benchmarks miss:

   - **SWE-Infinite**: a continuously expanding software engineering environment that auto-discovers repositories, generates validated task instances across multiple programming languages, and applies multi-layer quality filtering — producing a renewable task supply that resists saturation.
   - **LiveWeb Arena**: a real-time web interaction environment where agents navigate live websites (financial data, weather services, cryptocurrency exchanges, blockchain explorers), with a combinatorial task space exceeding 197 million unique task identifiers and dynamic ground truth that changes with real-world data.
   - **MemoryGym**: a memory management benchmark that tests selective storage, incremental updates, and reasoning under write-budget constraints, using correction events that dynamically alter world state — defeating shortcut strategies through nine verified simulation protocols.
   - **NavWorld / QQR**: a tool-mediated travel planning environment requiring multi-step tool orchestration (POI search, navigation, weather, flight and train queries), with 10,000+ renewable tasks generated from city-type-difficulty combinations and weekly salt rotation.
   - **GAME (OpenSpiel)**: a strategic reasoning environment built on DeepMind's OpenSpiel framework, spanning 22 games organized into seven quality tiers and selected for trajectory diversity (minimum 100 distinct trajectories per configuration), from imperfect-information card games to deterministic board games with effectively infinite game trees.

4. **Training-friendly design choices** throughout the system: deterministic seeding for reproducibility, step-level reward signals for policy learning, geometric mean scoring that forces broad-spectrum competence, and task rotation mechanisms that prevent environmental overfitting.

The remainder of this paper is organized as follows. Section 3 describes the Affine mechanism — the evaluation pipeline, scoring logic, and anti-plagiarism defenses that govern the incentive network. Section 4 presents the system architecture, focusing on how Affinetes and Chutes separate environment execution from model inference. Section 5 details each evaluation environment, explaining its design, task generation logic, renewal properties, and training utility.

---

## 3. The Affine Mechanism

### 3.1 System Participants and Roles

Affine operates as a decentralized evaluation network built on Bittensor (Subnet 120). The system comprises four principal roles:

**Miners** are independent participants who train, host, and submit machine learning models. A miner's workflow proceeds through four steps: (1) obtain a base model from the network via `af pull`, (2) improve the model through reinforcement learning or other training methods, (3) deploy the model as a serverless inference endpoint on Chutes, and (4) commit the deployment metadata — including the Hugging Face repository, revision hash, and Chute identifier — to the Bittensor blockchain. Each hotkey may hold exactly one active commitment at any time. To ensure fair comparison, all submitted models must conform to the Qwen3-32B architecture (hidden_size=5120, num_hidden_layers=64), and quantized models are rejected. [affine-cortex/docs/MINER.md; affine-cortex/skill/references/rl-training-guide.md]

**Validators** are blockchain-registered nodes responsible for setting on-chain weights. In Affine's current architecture, validators do not perform evaluation or scoring directly. Instead, they fetch pre-computed normalized weights from the backend API and submit them to the Bittensor chain. This design offloads computationally intensive scoring to centralized backend infrastructure while preserving the decentralized weight-setting protocol. [affine-cortex/docs/VALIDATOR.md]

**The Executor** is a backend service that orchestrates task evaluation. It operates as a main process that spawns one worker subprocess per environment, each running a pool of concurrent execution workers (default: 5 per environment). The executor fetches tasks from a centralized task pool, dispatches them to miner inference endpoints, collects results, and submits scored samples back to the system. All results are cryptographically signed with the executor's wallet hotkey to ensure authenticity. [affine-cortex/affine/src/executor/main.py; worker.py]

**Environments** are the evaluation substrates — containerized services deployed via Affinetes that present tasks to models and return scores. Affine defines a library of canonical environments (currently eighteen, including deduction, abduction, code generation, code editing, logic puzzles, game-theoretic problems, browser-based web interaction, navigation, memory, and knowledge evaluation), of which a dynamically configured subset is enabled for scoring at any given time. Each environment exposes a standard interface: either a single-turn evaluate call or a multi-turn OpenEnv protocol (reset → step → stop). The active environment set is retrieved at scoring time from a centralized system configuration endpoint, allowing the suite to expand without changes to the scoring pipeline. [affine-cortex/affine/core/environments.py; affine-cortex/affine/src/scorer/main.py]

### 3.2 Task Lifecycle

The evaluation pipeline is driven by a pull-based task pool architecture backed by DynamoDB.

**Task creation.** The system generates evaluation tasks for each active miner across all environments. Each task is identified by a unique UUID and keyed by miner hotkey, model revision, environment, and task ID. Tasks are created with `pending` status and a three-day time-to-live. [affine-cortex/affine/database/schema.py]

**Task fetch.** Each executor worker subprocess runs a continuous fetch loop, pulling batches of up to 20 pending tasks from the `/tasks/fetch` API endpoint. Fetching is demand-driven: the worker only requests new tasks when its internal queue falls below the concurrency limit. [affine-cortex/affine/src/executor/worker.py]

**Task execution.** Execution workers pull tasks from an internal asyncio queue and invoke the target miner's inference endpoint through the environment SDK. The environment returns a `Result` object containing the score (which may be negative for some environments), latency, success status, and optional metadata. Failed executions are recorded with a score of 0.0 and an error description. [affine-cortex/affine/src/executor/worker.py]

**Result submission.** Each result is packaged into a cryptographically signed `SampleSubmission` (task UUID, score, latency, metadata) and posted to the `/tasks/submit` endpoint. Submitted samples are persisted in the `sample_results` table with a 30-day TTL. The executor's main process monitors worker liveness and automatically restarts dead subprocesses. [affine-cortex/affine/core/models.py; affine-cortex/affine/database/schema.py]

### 3.3 Evaluation Loop and Sampling Strategy

Affine's evaluation loop is designed to resist two failure modes common in static benchmarks: task saturation (where models memorize fixed test sets) and sampling bias (where some miners receive systematically easier or harder tasks).

**Sampling rotation.** The system maintains a sampling list per environment that rotates tasks over time. New tasks are added at the end of the list while consumed tasks are removed from the front, preserving discovery order. A safety check prevents any sampling window from consuming more than 80% of the available dataset, ensuring that tasks remain renewable and that no miner can overfit to a narrow task distribution. [affine-cortex/affine/core/sampling_list.py]

**Completeness validation.** Before a miner's scores enter the scoring pipeline, the system verifies that the miner has completed at least 90% of assigned tasks in each environment. Miners below this threshold are excluded from the current scoring round, preventing partial evaluation from distorting rankings. [affine-cortex/affine/src/scorer/config.py]

**Common-task alignment.** When comparing two miners, the scoring pipeline restricts comparison to the intersection of tasks that both miners have completed. This ensures that score differences reflect capability differences, not task-assignment differences. [affine-cortex/affine/src/scorer/stage2_pareto.py]

### 3.4 The Scoring Pipeline

Affine computes miner weights through a four-stage pipeline that transforms raw per-task scores into normalized blockchain weights.

#### Stage 1: Data Collection

The collector parses raw evaluation data from the backend API, computing per-miner, per-environment average scores. Environment-specific score ranges are normalized (e.g., SciWorld scores from -100 to 100 are mapped to [0, 1]). Only miners meeting the 90% completeness threshold proceed to subsequent stages. The output is a set of `MinerData` objects, each containing a map of environment scores with averages, sample counts, and per-task score vectors. [affine-cortex/affine/src/scorer/stage1_collector.py]

#### Stage 2: Pareto Frontier Filtering

This stage identifies and removes miners whose performance is statistically dominated by an earlier submission — the primary defense against model plagiarism at the scoring level.

For each pair of miners (A, B) where A committed to the blockchain first, the system tests whether A dominates B across all environments. Dominance is determined by a statistical threshold derived from a binomial confidence interval:

> SE = sqrt(p * (1 - p) / n)
> gap = z * SE
> gap = clamp(gap, MIN_IMPROVEMENT, MAX_IMPROVEMENT)
> threshold = min(prior_score + gap, 1.0)

where `p` is miner A's score, `n` is the number of common tasks, and `z` is the z-score (default 2.0, corresponding to approximately 95.4% confidence). The gap is bounded between a 2% floor and a 10% ceiling. Miner A dominates B only if B fails to exceed A's threshold in *every* environment in the evaluated subset. This ensures that a copy — which will perform nearly identically to the original — is filtered out, while a genuinely improved model that exceeds the statistical threshold survives. [affine-cortex/affine/src/scorer/stage2_pareto.py; utils.py]

The first-commit advantage is enforced by sorting miners by `first_block` before applying dominance tests. An earlier submission can only be dominated by a later one that demonstrates statistically significant improvement, not by random variance.

#### Stage 3: ELO Rating and Weight Distribution

Surviving miners are ranked within each scoring round using a geometric mean of their environment scores:

> GM = ((v1 + epsilon) * (v2 + epsilon) * ... * (vn + epsilon))^(1/n) - epsilon

where epsilon = 0.1 provides smoothing to prevent a single zero-score environment from collapsing the aggregate. This design choice is deliberate: it forces miners to maintain competence across *all* environments rather than specializing narrowly.

Rankings feed into an ELO rating system that tracks miner performance across rounds. Key parameters:

- **Base rating**: 1200 (below average, to prevent new-key spam from immediately competing)
- **K-factor**: 96 for provisional miners (first 48 rounds, approximately one day at 30-minute intervals), decaying to 32 for established miners. The higher provisional K-factor allows genuinely strong new entrants to converge quickly.
- **Absence decay**: Miners that do not participate in a scoring round (due to incomplete sampling, Pareto filtering, or being offline) experience accelerating rating decay: `rating = BASE + excess * (0.95^(rounds^1.4))`. A two-round grace period (approximately one hour) prevents brief outages from triggering decay. This mechanism guarantees that stale ratings converge to the base within approximately 18 hours, preventing inactive miners from occupying weight slots indefinitely.

Weight distribution follows a rank-based decay model: the miner ranked first receives a base weight, and each subsequent rank receives 50% of the previous rank's weight (`weight = 0.5^(rank - 1)`). This includes all miners with at least one ELO round played, not only those participating in the current round. [affine-cortex/affine/src/scorer/stage3_subset.py; config.py]

#### Stage 4: Weight Normalization

The final stage accumulates subset weight contributions per miner, applies a 1% minimum weight threshold, and normalizes the remaining weights to sum to 1.0. Miners below the threshold have their weight redistributed to UID 0 — a mechanism the team also uses deliberately to burn emissions during periods of infrastructure instability, security incidents, or scoring corrections. [affine-cortex/affine/src/scorer/stage4_weights.py]

### 3.5 Anti-Plagiarism Mechanisms

Model copying is the central adversarial threat in decentralized training markets: a rational but unscrupulous participant can download a leading model, make trivial modifications, and redeploy it to capture emissions without contributing genuine improvement. Affine addresses this through three complementary mechanisms.

**Pareto dominance filtering** (described in Section 3.4, Stage 2) operates at the scoring level. Because copies perform nearly identically to the original on common tasks, they cannot exceed the statistical improvement threshold and are filtered as dominated.

**The anti-copy detector** operates as an independent periodic background service (default: 24-hour cycle) that analyzes model internals rather than output scores alone. Unlike Pareto filtering, which runs inline during each scoring round, the anti-copy detector stores its results in a separate audit table for review and policy enforcement. It uses a two-signal voting system:

1. **Hidden-state signal**: For each shared task, the detector computes the cosine similarity of the models' internal hidden-state representations. If the median cosine similarity across all common tasks exceeds 0.99, the signal votes "copy." A norm-deviation gate rejects pairs where the per-task L2 norm ratio deviates more than 5% from unity — a pattern characteristic of fine-tuned models, which exhibit significant norm drift even when cosine similarity remains high.

2. **Logprob signal**: The detector compares the top-3 token log-probability distributions up to the point where the models' generated tokens diverge. If the median cosine similarity of these distributions exceeds 0.99, the signal votes "copy."

A miner is flagged as a copy only if *both* signals vote "copy" and at least 30 shared tasks are available for comparison. This dual-signal requirement sharply reduces false positives: fine-tunes are caught by the norm gate, and models with similar outputs but different internals are caught by the hidden-state signal. Detection results are persisted to the `anti_copy_results` table (with 30-day TTL) but do not automatically feed into the scoring pipeline; the Pareto filter in Stage 2 remains the sole automatic anti-plagiarism mechanism during scoring. The anti-copy detector serves as a deeper forensic layer whose findings can inform manual review or future automated blacklisting. [affine-cortex/affine/src/anticopy/detector.py; affine-cortex/affine/src/anticopy/main.py]

**First-commit advantage** provides a temporal tiebreaker. When two models produce statistically indistinguishable scores, the model committed to the blockchain earlier is treated as the original. This creates a race-to-innovate incentive: the fastest path to emissions is genuine improvement, not replication.

### 3.6 Design Rationale

Several design choices in the Affine mechanism warrant explicit justification.

**Why geometric mean aggregation?** A simple arithmetic mean allows a miner to compensate for poor performance in one environment with strong performance in another. The geometric mean — which collapses toward zero when any input approaches zero — prevents this trade-off. A miner who scores 0.95 in six environments but 0.0 in one receives a near-zero aggregate, not a respectable 0.81. This property is essential for driving broad-spectrum capability improvement rather than narrow specialization.

**Why ELO rather than direct score ranking?** ELO ratings accumulate information across scoring rounds, providing temporal smoothing that a single-round ranking cannot. A miner that performs well consistently over many rounds will maintain a high rating even if a single round is anomalous. The provisional/established K-factor split further balances responsiveness (new miners converge quickly) with stability (established miners are not dislodged by noise).

**Why absence decay?** Without decay, a miner could submit a strong model, go offline, and continue collecting emissions from its historical rating. Absence decay ensures that only actively serving models retain weight, aligning incentives with ongoing availability.

**Why burn emissions to UID 0?** The ability to redirect emissions to UID 0 provides an operational safety valve. During security incidents (such as the discovery of a Chutes routing vulnerability that allowed requests to be redirected to GPT-4o), infrastructure transitions, or scoring mechanism updates, burning emissions prevents exploiters from profiting while the system is in a known-inconsistent state. [affine-cortex/docs/FAQ.md, Q12]

**Why a fixed model architecture?** Requiring all miners to use Qwen3-32B eliminates architectural confounds from evaluation. Score differences reflect training quality, not model capacity or quantization artifacts. This constraint also simplifies the anti-copy detector, which can compare hidden states and logprobs directly without accounting for architectural differences.

The mechanism described above governs *what* is evaluated and *how* scores translate to incentives. The next section describes the infrastructure that makes this evaluation operationally feasible at scale.

---

## 4. System Architecture

Affine's operational architecture separates two concerns that most benchmark systems conflate: **environment execution** (running tasks, computing scores) and **model inference** (generating predictions from neural networks). This separation is not merely organizational — it enables independent scaling, resource-appropriate hardware allocation, and clean substitution of infrastructure components as the system evolves.

### 4.1 Architectural Overview

The system comprises three layers:

1. **The environment layer** (Affinetes) packages evaluation environments as isolated, containerized services. Each environment runs in its own Docker container, exposes a standardized HTTP interface, and is managed through a unified orchestration API. Affinetes handles image building, container lifecycle, inter-process communication, load balancing, and automatic cleanup.

2. **The inference layer** (currently Chutes) serves model predictions via OpenAI-compatible HTTP endpoints. Miners deploy their models as serverless inference endpoints on Chutes, which handles GPU allocation, request routing, and auto-scaling. The environment layer communicates with the inference layer exclusively through HTTP, with no shared state or tight coupling.

3. **The coordination layer** (affine-cortex) runs the scoring pipeline, task pool management, and executor processes described in Section 3. It orchestrates evaluation by dispatching tasks from the task pool to environment instances, which in turn invoke miner models through the inference layer.

This three-layer design means that replacing the inference backend — for example, migrating from Chutes to Targon-backed or Affine-operated GPU clusters — requires no changes to environment code, scoring logic, or task management. The only modification is the base URL passed to environment instances. [affinetes/README.md; affinetes/examples/basilica-sdk/; affinetes/examples/targon-sdk/]

### 4.2 The Environment Layer: Affinetes

Affinetes is a lightweight container orchestration framework designed specifically for evaluation environments. Its core design principle is **zero burden on environment developers**: an environment author writes only an `env.py` file containing business logic, and the framework handles everything else — Docker image construction, HTTP server injection, deployment, networking, and cleanup.

#### Environment Definition

Affinetes supports two environment styles:

**Function-based environments** (the recommended pattern) define an `Actor` class with async methods. The developer provides the class and a Dockerfile specifying the base image and dependencies. During image build, Affinetes performs a **two-stage process**: it first builds the developer's base image, then wraps it in a second layer that injects a FastAPI HTTP server. This server auto-discovers all public methods on the Actor class and exposes them as HTTP endpoints — both in an RPC-style format (`POST /call` with method name and arguments) and as RESTful routes (`POST /{method_name}`). The injected server also provides `/health` and `/methods` endpoints for lifecycle management and introspection. [affinetes/affinetes/infrastructure/image_builder.py; affinetes/affinetes/templates/http_server.py]

**HTTP-based environments** are existing FastAPI applications that manage their own server. Affinetes detects this style by parsing `env.py` for a FastAPI app variable and skips HTTP injection, using the developer's server directly. This mode supports environments with custom routing, middleware, or streaming requirements.

The environment type is detected automatically — no configuration flags are needed. [affinetes/affinetes/infrastructure/env_detector.py]

#### Deployment Modes

Affinetes provides three deployment backends, selectable at load time:

**Docker mode** (default) manages container lifecycle through the Docker SDK. For local deployment, it connects directly to the Docker socket. For remote deployment, it establishes SSH connections to remote Docker daemons, then creates **SSH port-forwarding tunnels** using Paramiko so that the orchestrator can communicate with remote containers without exposing any ports on the remote host. All traffic between the orchestrator and remote containers flows through encrypted SSH tunnels. [affinetes/affinetes/backends/local.py; affinetes/affinetes/infrastructure/ssh_tunnel.py]

**URL mode** connects to user-managed services at arbitrary HTTP endpoints. The backend auto-detects whether the remote service is function-based or HTTP-based by probing for `/methods` or `/openapi.json` endpoints. This mode requires no Docker or SSH — it is a pure HTTP proxy, suitable for environments already deployed on managed infrastructure. [affinetes/affinetes/backends/url.py]

**Basilica mode** creates temporary Kubernetes pods per evaluation session. Each `call_method()` invocation provisions a fresh pod with specified CPU and memory resources, executes the method, and cleans up via TTL-based auto-deletion. A concurrency limiter prevents resource exhaustion. This mode supports the transition toward cloud-native deployment where environments run on managed compute rather than dedicated machines. [affinetes/affinetes/backends/basilica.py]

#### Multi-Instance Scaling and Load Balancing

A single `load_env()` call can deploy multiple replicas of an environment across a heterogeneous set of hosts (local Docker, remote SSH, or mixed). An instance pool manages all replicas behind a load balancer that supports round-robin and random distribution strategies. Each instance tracks its request count independently, and the load balancer routes incoming evaluation calls to the selected instance. This enables horizontal scaling of compute-intensive environments — such as SWE-Infinite, which requires Docker-in-Docker builds — without changes to calling code. [affinetes/affinetes/core/instance_pool.py; affinetes/affinetes/core/load_balancer.py]

#### Lifecycle Management

Affinetes maintains a global environment registry (thread-safe, singleton) that tracks all active environments. On process exit — whether normal or due to an unhandled exception — the registry automatically cleans up all containers via `atexit` hooks. Individual environments can also be cleaned up explicitly through async `cleanup()` calls. Health checks run periodically to detect container restarts or failures, with automatic reconnection when possible. [affinetes/affinetes/core/registry.py]

### 4.3 The Inference Layer: Chutes and Beyond

The current inference layer is **Chutes** (Bittensor Subnet 64), a serverless GPU inference platform. Miners deploy vLLM-backed model endpoints on Chutes, which exposes them as OpenAI-compatible HTTP APIs at `https://llm.chutes.ai/v1`. Environments invoke models by issuing standard chat completion requests to this endpoint, passing the miner's model identifier and API key.

This design achieves clean separation: environments contain no model-loading code, GPU management, or inference optimization. They issue HTTP requests and receive text responses. The inference layer handles batching, quantization (where permitted), GPU scheduling, and auto-scaling independently.

**Why this separation matters:**

- **Resource allocation**: Environments are CPU-bound (running Docker containers, executing scoring logic, managing state); inference is GPU-bound (running forward passes through large neural networks). Separating them allows each layer to run on appropriate hardware.
- **Independent scaling**: The number of environment instances and the number of inference replicas can scale independently based on their respective bottlenecks.
- **Backend substitution**: The same environment code works with any OpenAI-compatible inference endpoint. Basilica SDK examples demonstrate this with vLLM on Basilica GPU pods; Targon SDK examples demonstrate it with Targon's serverless function deployment, which provides automatic HTTPS URL exposure without manual port management.

**Future evolution.** The Basilica and Targon SDK integrations in the Affinetes examples directory demonstrate that the system is designed to support multiple inference backends. The path toward Affine-operated inference infrastructure — using Targon machines or other GPU providers — requires only deploying models to a new endpoint and updating the base URL. No environment code, scoring logic, or orchestration changes are needed. [affinetes/examples/targon-sdk/README.md; affinetes/examples/basilica-sdk/]

### 4.4 The OpenEnv Protocol: A Standardized Training Interface

Beyond single-turn evaluation, Affinetes defines **OpenEnv** — a standardized multi-turn interaction protocol that makes environments usable as reinforcement learning training substrates.

The protocol follows a gym-like pattern:

```
session = await env.openenv().reset(task_id=N, seed=S)
response = await session.step(action)       # returns observation, reward, done, info
await session.stop()                        # cleanup episode
```

Each `reset()` call creates an episode bound to a unique episode ID. The session object automatically injects this ID into all subsequent `step()` calls, maintaining episode state on the environment side without requiring the caller to manage session identifiers. Multiple episodes can run concurrently against the same environment instance.

This interface separates **episode lifecycle** from **environment instance lifecycle**: an environment container persists across many episodes, amortizing startup cost, while each episode maintains independent state. For training loops that require thousands of episodes, this design eliminates the overhead of container creation per episode. [affinetes/affinetes/core/openenv_client.py]

Beyond reward signals, the system supports **logprob collection** across multiple environments: when enabled via the `collect_logprobs` parameter, each inference call returns per-token log-probability distributions alongside the generated text. These distributions power a dedicated **DISTILL** environment (detailed in Section 5.6) that measures distributional alignment between student models and teacher rollouts via KL divergence, producing a continuous training signal complementary to the interactive environments' task-level rewards. The same logprob data also feeds the anti-copy detector's second signal (Section 3.5), creating a dual-purpose data flow. [affine-cortex/affine/database/system_config.json]

### 4.5 Execution Flow Across Layers

A complete evaluation cycle proceeds as follows:

1. The **coordinator** (affine-cortex executor) selects a batch of pending tasks from the task pool, each specifying a miner, environment, and task ID.

2. For each task, the executor invokes the target **environment instance** (managed by Affinetes) via HTTP, passing the miner's model identifier, inference endpoint URL, and task parameters.

3. The **environment** executes its evaluation logic — which may involve multi-turn interaction with the model. For each model query, the environment issues an HTTP request to the **inference layer** (Chutes), receives the model's response, and continues execution.

4. The environment computes a score based on the model's behavior and returns a `Result` to the executor.

5. The executor packages the result into a signed `SampleSubmission` and posts it to the backend API for storage and subsequent scoring.

At no point does the executor directly communicate with the inference layer; all model interaction is mediated through the environment. This ensures that scoring logic is encapsulated within the environment and that the executor remains environment-agnostic.

### 4.6 Scalability and Reproducibility

The architecture provides several properties essential for a production evaluation system:

**Reproducibility.** Environments accept deterministic seeds, and Affinetes ensures identical Docker images across deployments. Given the same seed, task ID, and model, an environment produces identical evaluation trajectories — a property required for both fair scoring and training reproducibility.

**Horizontal scalability.** The executor runs one worker subprocess per environment, each capable of concurrent task execution. Affinetes scales environments across multiple hosts via SSH tunneling and load balancing. The inference layer scales independently through Chutes' serverless auto-scaling. These three scaling dimensions are orthogonal.

**Fault tolerance.** Worker subprocesses are monitored and auto-restarted. Environment containers are health-checked and reconnected. The task pool uses DynamoDB with TTL-based expiration, ensuring that failed or abandoned tasks do not accumulate indefinitely.

**Security.** SSH tunneling ensures that environment containers are never directly exposed to the network. Cryptographic signatures on sample submissions ensure that results can only be submitted by authorized executors. The fixed model architecture constraint prevents inference-time attacks that exploit model heterogeneity.

---

## 5. Evaluation Environments

Affine's environment suite is designed around a central thesis: meaningful evaluation of agentic intelligence requires interactive, renewable, and training-friendly environments — not static test sets. Each environment targets a distinct capability axis, generates tasks through a renewable process, and provides reward signals structured for reinforcement learning. Five interactive environments evaluate behavioral capabilities across complementary axes; a sixth environment (DISTILL) evaluates distributional alignment at the token level, providing a complementary signal that targets internal representation quality rather than task-completion behavior. The table below summarizes the six environments; the subsections that follow detail each one.

| Environment | Capability Target | Task Space | Renewal Mechanism | Training Signal |
|---|---|---|---|---|
| **SWE-Infinite** | Software debugging | Unbounded (continuous) | Auto-discovery from GitHub PRs | FAIL_TO_PASS / PASS_TO_PASS tests |
| **LiveWeb Arena** | Browser-grounded agency | 197M+ configurations | Dynamic real-time data + combinatorial templates | Step rewards (exploration, efficiency) + terminal |
| **MemoryGym** | Memory management | 600+ entities × 10 domains × seeds | Seed-based generation + eval_salt perturbation | 4-axis scoring; shaped RL rewards |
| **QQR (NavWorld)** | Tool-mediated planning | 10,000+ tasks | 7 types × 3 difficulties × 71 cities × weekly salt | Per-step tool quality + 100-pt final |
| **GAME (OpenSpiel)** | Strategic reasoning | >10^60 trajectories | 22 games × config variants × seeds | Normalized game outcome [0,1] |
| **DISTILL** | Distributional alignment | Self-expanding (continuous) | Teacher rollout pipeline from raw corpus | exp(−KL) continuous score [0,1] |

### 5.1 SWE-Infinite: Renewable Software Engineering Tasks

#### Capability Target

SWE-Infinite evaluates a model's ability to understand, diagnose, and repair real software bugs — the core loop of professional software engineering. Unlike simple code generation benchmarks that test function-level synthesis in isolation, SWE-Infinite presents models with full repository contexts, real test suites, and genuine bug-fixing patches extracted from merged pull requests.

#### Why Existing Benchmarks Are Insufficient

SWE-bench Pro, the most rigorous existing software engineering benchmark, contains approximately 2,300 instances drawn from a fixed set of roughly 340 repositories. This fixed dataset creates three problems for ongoing evaluation and training:

1. **Saturation**: A finite task set is exhausted within a few training epochs. Models can overfit to the specific patterns, repositories, and failure modes in the dataset without developing generalizable debugging capability.
2. **Contamination**: In an open network where model weights are public, training data overlap with the evaluation set is difficult to prevent and impossible to verify.
3. **Limited language coverage**: SWE-bench Pro focuses primarily on Python, leaving software engineering capability in other languages unmeasured.

SWE-Infinite addresses these limitations through a fully automated pipeline that continuously discovers new repositories, extracts validated task instances from merged pull requests, and supports multiple programming languages. [affine-swe-infinite/docs/en/01-HIGH-LEVEL-DESIGN.md]

#### Environment Design

The SWE-Infinite pipeline operates in four stages, each with explicit quality targets:

**Stage 0: Repository and PR Discovery.** The system continuously discovers candidate repositories through two channels: GitHub Search API queries segmented by star ranges (in steps of 100, 1,000, and 10,000 to overcome the API's 1,000-result limit) and package registry top-download lists (e.g., PyPI's top 2,000 packages). Discovered repositories are filtered by quantitative admission criteria: minimum 500 stars, at least 50 forks, active CI configuration, and activity within the past 365 days. Pull requests within qualifying repositories are further filtered: 2–8 files changed, 5–500 lines modified, merged within 365 days, and authored by a human (not a bot). A semantic filter based on PR labels and title keywords retains bug fixes, feature additions, and regression repairs while rejecting refactoring, documentation, and performance-only changes. Approximately 40% of candidate PRs survive all Stage 0 filters. [affine-swe-infinite/docs/en/01-HIGH-LEVEL-DESIGN.md; affine-swe-infinite/src/discovery/repo_discovery.py]

**Stage 1: Environment Building (target: 80% pass rate).** For each surviving PR, the system generates a Dockerfile using heuristic language-version detection (checking `.nvmrc`, `pyproject.toml`, GitHub Actions workflows, and other configuration files) followed by build-side dependency resolution that parses PEP 735 and PEP 508 dependency specifications before building. Supported languages include Python (3.6–3.12), JavaScript/TypeScript (Node 16–22), Go (1.13–1.23), Rust (1.40–1.88), and Ruby (2.5–3.3). Built images are pushed to a public Docker registry for cross-machine reuse. [affine-swe-infinite/src/builders/dockerfile_gen.py]

**Stage 2: Test Validation (target: 90% pass rate).** The system applies a patch-splitting algorithm that separates each PR's diff into a `test_patch` (test file changes) and a `solution_patch` (source file changes) using filename-pattern matching. Validation proceeds in three steps: (1) apply only the test patch to the base commit and run tests — new tests should fail against old code; (2) apply the full patch and run tests — all tests should pass; (3) identify the FAIL_TO_PASS set (tests that failed in step 1 but passed in step 2). A task instance must contain at least one FAIL_TO_PASS test to be accepted. [affine-swe-infinite/src/processors/patch_separator.py; affine-swe-infinite/docs/en/01-HIGH-LEVEL-DESIGN.md]

**Stage 3: Patch Quality Verification (target: 85% pass rate).** The system verifies that the solution patch contains actual source code changes (not just test modifications), producing a cumulative end-to-end pass rate of approximately 60% (0.80 × 0.90 × 0.85). [affine-swe-infinite/docs/en/01-HIGH-LEVEL-DESIGN.md]

#### Task Generation and Renewability

SWE-Infinite's task supply is renewable because the discovery pipeline operates continuously rather than against a fixed list. A cursor-based daily advancement mechanism processes merged PRs one day at a time, with persistent state stored in Cloudflare R2. Each supported language maintains an independent cursor, and the system tracks processed tasks in DynamoDB to prevent cross-machine duplication. Failed tasks are retried on language-appropriate schedules: build failures after 30 days (allowing Dockerfile generator improvements to take effect), transient errors after 7 days, while semantic rejections and tasks with no FAIL_TO_PASS tests are permanently skipped.

At a processing rate exceeding 10 tasks per hour per machine, a cluster of five machines produces over 1,200 validated task instances per day — a rate that far exceeds the consumption rate of any single evaluation round. [affine-swe-infinite/docs/en/01-HIGH-LEVEL-DESIGN.md, 442-492]

#### Training Utility

The patch-splitting strategy directly enables reinforcement learning: the FAIL_TO_PASS tests provide a verifiable reward signal (does the model's patch make the failing tests pass?), while the PASS_TO_PASS tests penalize regressions (does the patch break existing functionality?). The output format is fully compatible with SWE-bench Pro, allowing integration into existing training pipelines via a unified task loader. [affine-swe-infinite/docs/en/01-HIGH-LEVEL-DESIGN.md]

#### Key Advantages

- **Unbounded supply**: Continuous discovery from the full GitHub ecosystem, not a fixed repository list
- **Multi-language**: Five language families with language-specific Dockerfile generation and dependency resolution
- **Quality-controlled**: Four-stage validation with explicit pass rate targets and monitoring
- **Format-compatible**: Drop-in replacement for SWE-bench Pro instances in existing evaluation harnesses
- **Training-ready**: Clean FAIL_TO_PASS / PASS_TO_PASS separation enables direct use as RL reward

#### Current Limitations

Production monitoring (Cycle 036, 2026-03-15) reveals uneven quality across languages: Go, Rust, Ruby, and JavaScript achieve near-100% smoke test pass rates, while Python's deep conftest dependency chains reduce its end-to-end pass rate significantly. The heuristic Dockerfile generator does not yet handle all build configurations, and LLM-assisted fallback generation remains a planned enhancement. [affine-swe-infinite/monitor_reports/cycle_036_20260315_0950.md]

---

### 5.2 LiveWeb Arena: Real-Time Browser Agent Evaluation

#### Capability Target

LiveWeb Arena evaluates a model's ability to act as a browser agent: navigating real websites, extracting information from dynamic pages, comparing data across multiple domains, and performing computations over retrieved facts. This measures a compound capability — tool use, planning, information extraction, and grounded reasoning — that static question-answering benchmarks do not address.

#### Why Existing Benchmarks Are Insufficient

Existing browser-agent benchmarks such as WebArena and Mind2Web rely on fixed website snapshots and human-created task-trajectory pairs. These designs share three weaknesses:

1. **Static ground truth**: Answers are frozen at collection time. A model that memorizes the dataset achieves perfect scores without learning to navigate.
2. **Small task spaces**: Fixed question pools are exhausted rapidly during training and are vulnerable to contamination.
3. **Trajectory dependence**: Human-authored trajectories define a single "correct" path, penalizing alternative valid strategies.

LiveWeb Arena addresses these weaknesses through real-time data, a combinatorial task space exceeding 197 million configurations, and page-bound ground truth that adapts to the agent's actual browsing behavior. [liveweb-arena/TASK_TOPOLOGY.md]

#### Environment Design

The system comprises three components:

**Plugins** wrap real-world websites and provide structured data extraction. Five plugins are currently implemented:
- **Weather** (wttr.in): weather data for 51 cities across five global regions
- **Stooq** (stooq.com): stock, forex, index, and commodity prices for 45 financial instruments
- **CoinGecko** (coingecko.com): cryptocurrency data for 39 digital assets including prices, volumes, market caps, and performance metrics
- **Taostats** (taostats.io): Bittensor blockchain subnet metrics for 50+ dynamically tracked subnets
- **Hybrid**: cross-domain tasks combining CoinGecko and Stooq, requiring agents to navigate multiple websites within a single evaluation

Each plugin defines allowed domains, provides a `fetch_api_data(url)` method for structured data extraction, and can block direct API access to force agents to navigate the website's user interface rather than calling APIs directly. [liveweb-arena/liveweb_arena/plugins/base.py; liveweb-arena/README.md]

**Templates** define task generation logic. Thirty-four templates span three difficulty tiers:
- **Easy** (9 templates): single-hop, direct URL extraction (e.g., "What is Bitcoin's current price?")
- **Medium** (13 templates): multi-page navigation or computation (e.g., "Which cryptocurrency had the largest 24-hour price increase?")
- **Hard** (12 templates): cross-site comparison or multi-step reasoning (e.g., "Rank these four assets by 24-hour performance: BTC, ETH, AAPL, MSFT")

[liveweb-arena/TASK_TOPOLOGY.md]

**The evaluation harness** launches a Playwright browser session for each evaluation, generates tasks from templates using deterministic seeds, tracks which pages the agent visits, collects ground truth from those pages, and scores the agent's final response.

#### Task Generation and the 197-Million Task Space

The combinatorial task space arises from three multiplicative factors:

1. **Template combinations**: C(34,1) + C(34,2) + C(34,3) = 6,579 unique combinations of 1–3 templates per evaluation
2. **Variation seeds**: 10,000 seeds per combination, each selecting different entities (cities, coins, stocks) from the entity pools
3. **Sub-task count**: Each seed determines 2, 3, or 4 sub-tasks

This yields 65,790,000 base task IDs. With the sub-task multiplier, the system supports approximately 197 million unique evaluation configurations. The effective question space — accounting for per-template entity and metric permutations — exceeds one billion. [liveweb-arena/TASK_TOPOLOGY.md; liveweb-arena/liveweb_arena/core/task_registry.py]

#### Anti-Memorization Properties

Five mechanisms prevent models from bypassing genuine web navigation:

1. **Dynamic data**: Cryptocurrency prices, stock quotes, and weather conditions change continuously. The same question ("What is Bitcoin's price?") produces different correct answers on different days.
2. **Large entity pools**: 51 cities, 39 cryptocurrencies, and 45 financial instruments create exponential combination spaces (e.g., C(51,2) = 1,275 city pairs for weather comparison alone).
3. **Computation requirements**: Many templates require derived metrics (volatility, percentage change, rankings) that cannot be pre-memorized.
4. **Cross-site exploration**: Hybrid templates require navigating multiple websites and comparing real-time data across domains.
5. **Seed-based determinism**: Each seed produces a unique entity selection, but results are reproducible for a given seed and data snapshot.

A mandatory red-team review protocol verifies that each template cannot be solved by an LLM using world knowledge alone (>60% accuracy without browsing), has an effective variant space exceeding 500, and does not collapse across parameter variations. [liveweb-arena/CLAUDE.md; liveweb-arena/TASK_TOPOLOGY.md]

#### Ground Truth Collection

LiveWeb Arena uses **page-bound ground truth** — a mechanism that ties correct answers to the specific pages the agent actually visits, rather than to a pre-computed answer key. When the agent navigates to a URL, the system simultaneously fetches structured data (via the plugin's API extraction) and caches the page content. Ground truth is then extracted from this collected pool using priority rules: detail page visits overwrite earlier data (they are more authoritative), while list page visits add new entities without overwriting existing ones. Different websites maintain independent caches.

This design means that if an agent visits a cryptocurrency's detail page, the ground truth reflects that page's data at the time of the visit — not a stale pre-cached value. If the agent never visits the required pages, the ground truth collection returns `DATA_NOT_COLLECTED` rather than a default value, preventing the system from scoring ungrounded answers. [liveweb-arena/liveweb_arena/core/gt_collector.py; liveweb-arena/CLAUDE.md]

#### Step-Level Reward Signals

LiveWeb Arena provides dense reward signals suitable for reinforcement learning, not just a binary pass/fail at the end of each episode:

**Exploration rewards**: +0.05 for visiting a new domain, +0.06 per new asset collected, +0.10 per target asset found, +0.15 when all targets are collected, +0.03 for detail page visits.

**Efficiency rewards**: Up to +0.08 scaled by how many steps remain unused when the agent submits its answer.

**Penalties**: -0.04 for revisiting pages, -0.06 for hitting blocked URLs (e.g., direct API calls), -0.02 for failed actions, -0.08 for parse errors.

**Terminal rewards**: +2.00 for achieving 80%+ validation accuracy, score × 0.70 for partial success (30–80%), -0.25 for exhausting maximum steps.

Cumulative step rewards are capped at 1.5, below the terminal success bonus of 2.0, to ensure that agents cannot achieve high scores through exploration alone without answering correctly. [liveweb-arena/liveweb_arena/core/reward.py]

#### Key Advantages

- **Live data**: Ground truth changes with real-world events, eliminating data staleness
- **Massive task space**: 197M+ configurations resist memorization and contamination
- **Dense RL signals**: Per-step exploration and efficiency rewards, not just terminal scores
- **Multi-domain**: Hybrid templates test cross-site reasoning, a capability absent from single-site benchmarks
- **Reproducible**: Deterministic seeding with 72-hour page cache TTL enables consistent evaluation within a window while ensuring long-term freshness

#### Current Limitations

The page cache (72-hour TTL) trades off between reproducibility and data freshness — scores collected on the same task ID may differ slightly across days due to price movements. Some templates rely on third-party API availability (stooq.com, coingecko.com), introducing an external dependency. The current plugin set covers financial and weather data but does not yet extend to e-commerce, social media, or enterprise application domains.

---

### 5.3 MemoryGym: Agent Memory Under Operational Constraints

#### Capability Target

MemoryGym evaluates a model's ability to manage memory as an operational resource: selectively storing information under budget constraints, updating stored memories when corrections arrive, and reasoning over stored data to answer questions. This targets a capability chain that existing benchmarks largely ignore — not whether a model can retrieve a fact from its context window, but whether it can decide *what to remember*, *when to update*, and *what it doesn't know*.

#### Why Existing Benchmarks Are Insufficient

Memory-focused benchmarks such as LoCoMo, MemoryAgentBench, and LongMemEval primarily test **retrieval**: given a long context or conversation history, can the model find the right information? This framing misses three critical aspects of operational memory:

1. **Storage decisions under pressure**: Real agents face information overload and cannot store everything. A customer support agent processing 500 tickets per day must triage — which tickets deserve persistent memory? Existing benchmarks provide unlimited storage, eliminating the triage problem entirely.
2. **Memory maintenance**: Information changes. A stored fact that was correct yesterday may be wrong today. No existing benchmark tests whether an agent can update its memory when corrections arrive, or whether stale memories degrade downstream reasoning.
3. **Metacognition**: An agent that confidently answers questions about information it never stored is worse than one that says "I don't know." Existing benchmarks rarely measure a model's ability to assess the boundaries of its own knowledge.

MemoryGym addresses all three through a write-budget constraint, correction events that mutate world state mid-evaluation, and an explicit abstention axis. [MemoryGym/docs/STATUS_REPORT.md]

#### Environment Design

The MemoryGym pipeline proceeds through six phases:

1. **World generation**: A seed determines a domain template (ten domains: company, research, city, hospital, sport, movie, university, codebase, project, agentteam) and generates N entities with 21–23 attributes each, drawn from six data types (int, float, text, enum, date, list_float). Entity importance is scored based on relationship degree, attribute completeness, and value extremeness. [MemoryGym/memorygym/worlds/base.py]

2. **Document rendering**: Each entity is rendered as a natural-language document in either compact (tabular) or narrative (prose with embedded distractors) format. Distractors use randomized direction (50/50 higher/lower) to prevent "always pick the larger value" attacks. [MemoryGym/memorygym/worlds/base.py]

3. **Stream interleaving**: Documents arrive in batches (default: 10 entities per batch), with correction events, noise documents, and questions interleaved throughout the stream. Approximately 40% of questions are emitted mid-ingest, creating uncertainty pressure — the agent cannot adopt a "store everything first, answer later" strategy because questions arrive before all entities have been seen. [MemoryGym/memorygym/worlds/events.py]

4. **Agent interaction**: The agent receives events one at a time and uses four tools: **Write** (store information, consumes one write from the budget), **Edit** (modify stored information, budget-free during corrections), **Read** (retrieve stored content, free), and **memory_search** (semantic search over stored content, free). The write budget creates a 2:1 entity-to-write ratio at standard tier (60 entities, 30 writes) and a 4:1 ratio at hard tier (120 entities, 30 writes), forcing selective storage. [MemoryGym/memorygym/agents/_tool_helpers.py; MemoryGym/README.md]

5. **Correction events**: Mid-stream, the system mutates entity attributes (numeric values shift by 10–50%, enums switch categories, dates shift by 30–365 days) and issues correction notices. The agent must search its memory for the affected entity and update the stored value. Ground truth for all subsequent questions reflects the corrected state. Correction rates are domain-specific: hospital (15%) has the highest rate, reflecting rapid status changes; city (5%) has the lowest. [MemoryGym/memorygym/worlds/events.py]

6. **Adaptive questioning**: Questions are generated *after* the agent's storage decisions, using importance-weighted entity sampling. The system generates questions across four categories (retrieval, comprehension, update, abstention) using twenty reasoning competencies — including aggregation, comparison, multi-hop reasoning, counterfactual reasoning, relationship chains, temporal trends, and more. Crucially, update questions use identical wording to retrieval questions, making them indistinguishable by surface form. [MemoryGym/memorygym/worlds/base.py; MemoryGym/memorygym/protocol.py]

#### Four-Axis Scoring

MemoryGym scores agents on four orthogonal axes:

- **Storage Breadth (30%)**: Accuracy on retrieval questions — did the agent store this entity and can it retrieve the correct value?
- **Memory Maintenance (25%)**: Accuracy on update questions, gated by storage coverage — the agent must store at least 50% of entities to receive any maintenance credit. This prevents a degenerate strategy of storing few entities and claiming perfect update accuracy.
- **Reasoning (25%)**: Accuracy across twenty reasoning competency types, from basic aggregation to relationship chains and temporal extremes.
- **Efficiency (20%)**: Correct answers per write unit, capped at 1.0 — rewarding agents that pack more useful information into fewer writes.

The composite score is: `0.30 × breadth + 0.25 × maintenance + 0.25 × reasoning + 0.20 × efficiency`. Abstention accuracy (does the agent correctly say "I don't know" for unstored entities?) is reported as a separate diagnostic but does not enter the composite. [MemoryGym/memorygym/protocol.py]

#### Anti-Cheating Verification

Nine deterministic simulation strategies validate the scoring system's integrity before every release:

- **perfect** (store all, apply all updates): must achieve 100%, proving questions are answerable
- **guesser** (store nothing, no guessing): must achieve 0%, proving exact-match requirements prevent lucky guesses
- **smart_guesser** (store nothing, use midpoint/median heuristics): must achieve ≤5%, proving sophisticated statistical guessing fails
- **abstainer** (store all, always answer "I don't know"): must achieve <15%, proving blanket abstention has a hard ceiling
- **strategic > naive + 10%**: the gap between a strategy that applies corrections and one that ignores them must exceed 10 percentage points, proving that memory maintenance is necessary for high scores

These invariants are checked across all ten domain templates and multiple seeds, producing 437+ passing tests. [MemoryGym/memorygym/simulation.py; MemoryGym/CLAUDE.md]

#### Training Utility

MemoryGym provides a gym-compatible RL environment (`MemoryEnv`) with two reward modes: **binary** (correct=+1, wrong=0) and **shaped** (store write=+0.3, correction flow=+0.5, correct answer=+1.0). An `eval_salt` parameter perturbs entity attribute values between training and evaluation, preventing RL agents from memorizing specific entity data while preserving structural patterns. SFT trajectory data (160 perfect + 160 strategic trajectories) and GRPO training code are included. [MemoryGym/memorygym/training/env.py]

#### Key Advantages

- **Tests storage, not just retrieval**: The write budget forces triage decisions that existing memory benchmarks skip
- **Tests maintenance**: Correction events verify that agents update stale memories
- **Anti-gaming verified**: Nine simulation strategies with deterministic invariant checking
- **Domain diversity**: Ten domain templates with domain-specific correction rates, question weights, and entity structures
- **RL-ready**: Gymnasium-compatible environment with shaped rewards and eval_salt perturbation

#### Current Limitations

Empirical results across 123 evaluations show that all tested models (Qwen3.5-397B, Qwen3-235B, MiniMax-M2.5, Kimi-K2.5, GLM-5) achieve composite scores between 10–18%, with breadth averaging only 10.8%. The cascade bottleneck is storage: models store far too few entities, causing downstream failures in maintenance and reasoning. No model currently packs multiple entities per write — an untapped optimization. The scoring system relies on exact or near-exact matching for numeric answers, which may undercount partial understanding. [MemoryGym/docs/STATUS_REPORT.md]

---

### 5.4 NavWorld / QQR: Tool-Mediated Travel Planning

#### Capability Target

The QQR environment evaluates end-to-end agent capability for completing complex, open-ended tasks that require autonomous tool orchestration: selecting appropriate tools, constructing correct parameters, integrating results from multiple sources, and generating structured outputs grounded in tool-provided evidence. Travel planning serves as the evaluation proxy because it naturally demands all of these capabilities — an agent must query points of interest, compute routes, check weather, search transportation options, and synthesize a coherent plan.

#### Why Existing Benchmarks Are Insufficient

Mainstream benchmarks (MMLU, HumanEval, GSM8K) evaluate static knowledge or closed-form reasoning. They do not measure whether a model can autonomously decide *which* external tool to call, *in what order*, with *what parameters*, and produce an output that is factually grounded in tool responses rather than hallucinated. Tool-use benchmarks that do exist (e.g., ToolBench) typically test individual tool calls in isolation, not multi-step orchestration across heterogeneous tools under real-world constraints. [affinetes/environments/qqr/README.md]

#### Environment Design

QQR provides six MCP (Model Context Protocol) tools spanning two categories:

**Real-data tools** (backed by AMap API):
- `poi_search`: Search points of interest (attractions, hotels, restaurants)
- `around_search`: Radius-based nearby search from coordinates
- `direction`: Multi-modal route planning (driving, walking, cycling, transit)
- `weather`: Weather forecasts for Chinese cities

**Deterministic mock tools** (SHA256-seeded):
- `search_flights`: Flight search between city pairs on specified dates
- `search_train_tickets`: Train ticket search with identical determinism

Transport data is generated deterministically: `SHA256(epoch_salt | date | from_city | to_city)` produces identical flights and trains for the same inputs within an epoch. The epoch salt rotates weekly, preventing memorization across evaluation windows while maintaining reproducibility within each window. [affinetes/environments/qqr/env.py; affinetes/environments/qqr/README.md]

#### Task Generation

Tasks are generated from three parameters: **type** (7 categories: intercity transport, multi-day trip, hybrid, single POI, food tour, business travel, family/study trip), **difficulty** (3 levels with escalating constraint tightness, conflict count, and time pressure), and **city** (71 Chinese cities including major metropolitan areas and tourist destinations). Problem type is determined by `SHA256(task_id | epoch_salt | "type")` rather than simple modular arithmetic, decoupling task_id ordering from problem type — preventing sequential memorization. With weekly salt rotation, this produces over 10,000 distinct task configurations. [affinetes/environments/qqr/problem_generator.py; affinetes/environments/qqr/config.py]

#### Scoring System

QQR uses a 100-point scoring system split evenly between code-verifiable metrics (50 points) and LLM-judged quality metrics (50 points), with bidirectional coupling to prevent gaming:

**Code score (50 pts)**: Information Consistency (25 pts, measuring how much output content traces to real tool data across 10 fact categories — flight numbers, train numbers, POI names, weather, distances, times, prices, etc.) plus Completeness (25 pts, measuring coverage of required planning dimensions with proximity-based grounding — a fact mentioned near a tool result scores higher than one mentioned in isolation).

**LLM score (50 pts)**: Practicality, analysis depth, logic, user experience, and factual grounding.

**Bidirectional coupling**: The LLM score is constrained by the code score (`llm_adjusted = llm_raw × min(1.0, code_total / 30)`), preventing high LLM ratings for fluent but ungrounded outputs. Conversely, the code score retains at least 70% of its value regardless of LLM rating, ensuring that factual grounding is always rewarded.

**Hard constraints** apply multiplicative penalties: missing required tool calls (×0.5), unverified POI names (×0.7), fabricated transport data (×0.3–1.0 on a graduated scale), and format violations (×0.15). Multiple failures compound. [affinetes/environments/qqr/scorer.py; affinetes/environments/qqr/README.md]

#### Step-Level Rewards

Each tool call within an episode receives an immediate reward: `0.4 × tool_selection + 0.3 × argument_quality + 0.3 × result_usefulness`. Tool selection rewards calling required tools (+0.6) and penalizes repetitive calls at late stages. Argument quality checks parameter validity (e.g., whether coordinates are present for routing queries). Result usefulness scores by response length and content quality. Per-tool call limits (e.g., 10 direction calls, 3 weather calls) prevent wasteful repetition. Maximum episode length is 15 tool steps. [affinetes/environments/qqr/env.py]

#### Key Advantages

- **Real tool integration**: Four tools backed by live AMap APIs ensure that agents interact with actual geographic data
- **Deterministic reproducibility**: SHA256-seeded transport + epoch salt rotation enable consistent evaluation within windows
- **Anti-hallucination scoring**: Ten fact categories with exact-match verification detect fabricated information
- **Training-friendly**: Per-step rewards guide tool-calling policy learning; bidirectional coupling prevents dimensional gaming

#### Current Limitations

Geographic scope is limited to mainland China (71 cities) due to AMap API coverage. The LLM-based scoring component (50 points) requires an external LLM API call, adding cost and potential variance. Transport mock data, while deterministic, may not reflect real pricing structures. The complex scoring rubric (10+ hard constraints, proximity-based grounding, fact extraction) can be noisy for edge cases.

---

### 5.5 GAME / OpenSpiel: Interactive Strategic Reasoning

#### Capability Target

The OpenSpiel environment evaluates a model's ability to engage in multi-step strategic reasoning: assessing positions, planning ahead, handling imperfect information, estimating probabilities, and adapting to opponent behavior. Games provide a controlled setting for these capabilities because they have well-defined rules, clear win/loss signals, and known complexity properties — making it possible to isolate strategic reasoning from confounding factors like knowledge retrieval or tool use.

#### Why Existing Benchmarks Are Insufficient

Static benchmarks rarely capture interactive strategic reasoning. A model that can solve chess puzzles (a static task) may fail at actual chess play (a dynamic, multi-step, adversarial task). Question-answering benchmarks test factual recall; game-playing tests the ability to maintain a strategy over many decisions, each of which depends on the opponent's prior actions. This sequential, interactive structure is precisely what agentic systems must handle in real-world settings — from negotiation to multi-step planning under uncertainty.

#### Environment Design

The environment wraps **DeepMind's OpenSpiel** framework, providing access to 22 games selected for two properties: **trajectory diversity** (each task_id + seed combination must produce at least 100 distinct game trajectories) and **strategy non-memorability** (no game can be solved by memorizing a single optimal strategy across all configurations).

The 22 games are organized into seven tiers by evaluation quality:

**Tier 1–2: Core evaluation games (8)**: Games with excellent trajectory diversity and proven evaluation quality. These include Goofspiel (bidding strategy), Liar's Dice (probability reasoning, 36–60 million trajectories), Leduc Poker (~120 trajectories per configuration), Gin Rummy (~10^60 trajectories), Othello (spatial reasoning), Backgammon (>10^50 trajectories), Hex (path planning), and Clobber (capture tactics).

**Tier 3–4: Complex strategy games (7)**: Multi-player and high-complexity games requiring advanced reasoning. These include Hearts (4-player, 8 rule combinations), Euchre (trump-based card strategy), Dots and Boxes (spatial control), Go (3 board sizes × 3 komi values), Chess, Checkers, and Quoridor (path blocking).

**Tier 5: Imperfect information games (2)**: Blackjack (probability reasoning against a dealer) and Phantom Tic-Tac-Toe (hidden opponent moves, retained specifically for imperfect-information testing despite the small grid).

**Tier 6: Single-player games (2)**: 2048 (spatial planning over 50–200 steps) and Solitaire (card sequencing with high shuffle diversity), added specifically for testing sequential reasoning without opponent modeling.

**Tier 7: Advanced strategy games (3)**: Bridge (4-player bidding and trick-taking under imperfect information), Amazons (territorial control with queen-like movement and arrow placement), and Oware (Mancala-variant seed-counting strategy).

Five games were explicitly removed during curation: Breakthrough (100% success rate, no discrimination between models), Pig (luck-dominant with limited strategic depth), Cribbage (0% success rate — rules too complex for current LLMs), Battleship (979,000 average tokens per game, economically prohibitive despite 100% success), and Phantom Tic-Tac-Toe 3×3 was initially considered for removal but retained for its imperfect-information properties. [affinetes/environments/openspiel/game_config.py; affinetes/environments/openspiel/README.md]

#### Multi-Player Evaluation

The LLM plays one position in each game, determined by `seed % num_players`. Remaining positions are filled by built-in bots (random or MCTS). For a four-player game like Hearts, the LLM might play as player 2 while players 0, 1, and 3 are bots, each with independently seeded random number generators. This ensures that the same task_id + seed always produces the identical game, while different seeds expose the LLM to different positions and opponent behaviors. [affinetes/environments/openspiel/env.py]

#### Task ID Encoding

Task IDs use a 12-digit integer format: `GGGGCCCCCCCC`, where the first four digits select the game (via circular indexing over the 22-game list) and the remaining eight digits select the configuration variant (board size, rule combination, player count). This yields a vast configuration space that, combined with the seed-driven randomness per game, produces a trajectory space exceeding 10^60 for the full suite. [affinetes/environments/openspiel/game_config.py]

#### Scoring and Training Interface

Each game returns a normalized score in [0, 1] based on the game outcome. The environment supports both one-shot evaluation (`evaluate()`) and the OpenEnv training interface (`reset()` → `step()` → `stop()`), allowing it to serve as both a benchmark and an RL training substrate. The LLM bot maintains full conversation history within each game for context-aware decision-making, with a retry mechanism for action parsing failures. [affinetes/environments/openspiel/env.py; affinetes/environments/openspiel/llm_bot.py]

#### Key Advantages

- **Trajectory diversity**: 22 games across seven quality tiers with ≥100 trajectories per configuration; total space >10^60
- **Anti-memorization**: Parameter variants (board sizes, rule combinations) require different strategies; five memorizable or impractical games removed through explicit curation
- **Multi-player**: Supports 1–4 player games with configurable opponent types
- **Controlled complexity**: Well-defined rules and outcomes enable precise capability measurement
- **Reproducible**: Deterministic seeding for game state, opponent behavior, and LLM player assignment

#### Current Limitations

Token consumption varies dramatically across games: Backgammon requires approximately 347,000 tokens per game, Chess approximately 287,000, and Gin Rummy approximately 168,000 — making large-scale evaluation expensive. Some games require the LLM to understand intricate rules (e.g., Bridge bidding conventions, Hearts passing rules) that may not be well-represented in training data. The default opponent type is random bots, which may not stress strategic reasoning as effectively as stronger opponents; MCTS opponents are available but computationally expensive.

---

### 5.6 DISTILL: Distributional Alignment via KL Divergence

#### Capability Target

DISTILL evaluates a fundamentally different dimension of model quality than the five interactive environments above. Rather than measuring task-completion capability — whether a model can fix a bug, navigate a website, or win a game — DISTILL measures **distributional alignment**: how closely a student model's internal token-level predictions match those of a stronger teacher model across diverse natural-language continuations. This targets the quality of the model's learned representations, not merely its behavioral outputs.

#### Why Behavioral Scoring Alone Is Insufficient

Interactive environments provide rich task-level reward signals, but they share a limitation: two models can achieve identical task scores through very different internal strategies — one through robust understanding, another through brittle pattern matching. A model that produces the right bug fix for the wrong reasons will score identically to one that genuinely understands the codebase. Distributional alignment provides a complementary signal that penetrates beyond behavioral equivalence. A student whose per-token probability distribution closely matches a stronger teacher's has likely learned similar internal representations, not just similar surface outputs. This signal is also smoother than task rewards: incremental improvements in distributional alignment are detectable even when they do not yet produce measurable gains on pass/fail evaluation.

#### Architecture: A Three-Stage Pipeline

DISTILL operates through a three-stage pipeline that separates teacher rollout generation, storage promotion, and student evaluation.

**Stage 1: Teacher rollout generation.** An independent teacher worker process runs alongside the main executor. It samples task IDs from the Corpus-Eval environment — a prompt source backed by `karpathy/climbmix-400b-shuffle` that provides deterministic raw-corpus prompts rather than curated benchmark questions, eliminating contamination risk. For each sampled task, the teacher model (e.g., Qwen3-235B) generates a 512-token continuation with `collect_logprobs=True`, producing per-position top-20 token probability distributions. The resulting rollout — containing the full conversation text, token positions, and teacher logprob dictionaries — is uploaded to a private R2 bucket (`pending/{env}/{epoch_ms}.json`). To minimize storage I/O, the teacher worker walks segment-aligned task IDs (64-task segments matching Corpus-Eval's LRU shard boundaries), allowing bursts of rollouts to reuse cached corpus shards rather than triggering fresh ~92 MB downloads per rollout. [affine-cortex/affine/src/executor/teacher_worker.py]

**Stage 2: Rollout promotion.** A separate teacher mover process periodically promotes rollouts from the private to a public R2 bucket. On each tick, it reads the DISTILL configuration from SystemConfig (DynamoDB), lists pending rollouts across all source environments, samples a configurable number of candidates (default: 3 per rotation, hourly), copies them to sequentially numbered public files (`task_{next_id:011d}.json`), and updates `metadata.json` with the new `completed_up_to` counter. This counter drives dynamic dataset range expansion: the DISTILL sampling configuration auto-expands its task range by fetching the latest `completed_up_to` from the public metadata endpoint, ensuring the evaluation pool grows continuously without code changes. Promotion cadence is operator-tunable from a single DynamoDB entry. [affine-cortex/affine/src/executor/teacher_mover.py]

**Stage 3: Student evaluation.** The DISTILL environment itself is a containerized service (`affinefoundation/distill:latest`) deployed via Affinetes. For each evaluation task, it loads the corresponding teacher rollout from R2 (with local filesystem caching to avoid repeated downloads), then performs a student forward pass using the vLLM completions API with `echo=True` and `logprobs=20` — recovering the student's top-20 token distribution at every position in the teacher's continuation. The environment then computes per-position KL divergence between teacher and student distributions and returns a scalar score. [affinetes/environments/distill/env.py]

#### KL Divergence Scoring

The scoring algorithm implements two paths depending on the rollout format:

**Top-K path (primary).** For each position *i* in the teacher continuation where the teacher provides a top-20 distribution:

1. Extract the teacher's top-20 tokens and their log-probabilities.
2. Extract the student's top-20 log-probabilities at the same position.
3. For tokens in the teacher's support set but absent from the student's top-20, assign a conservative fallback: the minimum of the student's top-20 log-probabilities.
4. Renormalize both distributions onto the teacher's support set via log-softmax.
5. Compute KL divergence on this restricted simplex: KL_i = Σ_t exp(p_t) · (p_t − q_t), where p and q are the renormalized teacher and student log-probabilities.
6. Clip KL_i to [0, 10.0] to guard against outliers.

**Legacy path (single-sample fallback).** For older rollouts that store only the chosen token's log-probability, the environment uses Schulman's k3 estimator — a single-sample, unbiased, non-negative KL estimate: r = exp(s_lp − t_lp); kl_i = (r − 1) − log(r).

The final score maps average clipped KL to a [0, 1] range via the exponential: **score = exp(−avg_kl)**. A perfect distributional match yields 1.0; an average KL of 1.0 yields approximately 0.37; scores approach 0 as divergence increases. This exponential mapping provides a smooth, continuous reward signal where every incremental improvement in distributional alignment produces a detectable score increase. [affinetes/environments/distill/env.py, lines 220–328]

#### Integration with the Scoring Pipeline

DISTILL scores enter the standard scoring pipeline through Stage 1 collection, where they are treated as any other environment score — contributing to the geometric mean that determines miner rankings. With `min_completeness=0.6` (lower than the 90% threshold for interactive environments, reflecting the pipeline's younger rollout supply), DISTILL scores factor into Pareto filtering, ELO ratings, and weight distribution. Because the geometric mean collapses toward zero when any environment score is near zero, a miner cannot achieve high overall weight without maintaining reasonable distributional alignment alongside strong task performance.

The logprob data collected during DISTILL evaluation also feeds into the anti-copy detector's logprob signal (Section 3.5), creating a dual-purpose data flow: the same per-token distributions used for scoring also power forensic plagiarism detection. [affine-cortex/affine/database/system_config.json]

#### Key Advantages

- **Complementary signal**: Measures internal distributional quality, not just behavioral task success — catches models that pass tasks through brittle strategies
- **Smooth gradient**: Exponential KL mapping provides continuous reward where task-level signals are sparse or binary
- **Contamination-resistant**: Teacher rollouts sourced from raw corpus data (climbmix-400b-shuffle), not curated benchmarks
- **Self-expanding**: Dynamic dataset range auto-grows as the teacher pipeline produces new rollouts, without code changes
- **Dual-purpose data**: Logprob distributions serve both scoring and anti-copy detection

#### Current Limitations

The pipeline depends on a specific teacher model (currently Qwen3-235B) whose quality upper-bounds the training signal — a student that surpasses the teacher receives no further guidance. The top-20 support restriction means that probability mass outside both models' top-20 tokens is unaccounted for, introducing a systematic underestimate of true KL divergence. The legacy single-sample path (k3 estimator) has higher variance than the top-K path, creating measurement inconsistency across rollout formats during the transition period. The lower completeness threshold (60% vs. 90%) means DISTILL scores can enter the pipeline with fewer data points than other environments, potentially increasing scoring noise.

---

## 6. Conclusion and Future Work

This paper has presented Affine, a decentralized system for evaluating and training agentic intelligence through interactive, renewable environments. The system makes three primary contributions: a scoring mechanism that resists plagiarism and gaming through Pareto dominance filtering, ELO-based temporal ratings, and a dual-signal anti-copy detector; a container-orchestration infrastructure (Affinetes) that cleanly separates environment execution from model inference; and a suite of five interactive evaluation environments — SWE-Infinite, LiveWeb Arena, MemoryGym, QQR, and OpenSpiel — each targeting a distinct capability axis that static benchmarks fail to measure, supplemented by a KL-divergence distillation environment (DISTILL) that provides per-token training signals from teacher rollouts.

Several properties distinguish this approach from conventional benchmark design. Task supplies are renewable rather than fixed, resisting saturation and contamination. Environments are interactive rather than single-turn, capturing the sequential decision-making that defines agentic behavior. Infrastructure is treated as a first-class concern, enabling reproducible evaluation at scale rather than one-off script execution. And the system is explicitly designed for dual use: every environment serves as both an evaluation benchmark and a reinforcement learning training substrate, with deterministic seeding, step-level reward signals, and standardized interfaces (OpenEnv) that integrate directly into training loops.

### Limitations

The current system has notable constraints. The fixed model architecture requirement (Qwen3-32B) ensures fair comparison but limits the diversity of models that can participate. The centralized executor and backend scoring pipeline, while operationally necessary, represent a trust assumption that partially offsets the decentralized weight-setting protocol. Several environments have geographic or domain restrictions: QQR is limited to mainland China, LiveWeb Arena covers financial and weather domains but not e-commerce or social media, and MemoryGym's empirical results show that current models achieve only 10–18% composite scores — indicating either that the benchmark is well-calibrated for future capability growth or that the task design needs refinement as models improve.

### Future Directions

**Inference infrastructure evolution.** The path from Chutes (Subnet 64) to Affine-operated GPU clusters — via Targon or equivalent serverless platforms — would reduce external dependencies and give the system direct control over inference scheduling, latency, and cost. The architecture's three-layer separation makes this transition a backend substitution rather than a system redesign.

**Environment expansion.** The Affinetes framework supports arbitrary environment addition without changes to the scoring pipeline. A knowledge evaluation environment spanning GPQA-Diamond, MMLU-Pro, HLE, and IFEval — with cross-question distractor pools and virtual task IDs for anti-contamination — is built and deployed but not yet included in the scoring configuration. Planned directions include e-commerce interaction (extending LiveWeb Arena), multi-agent coordination tasks (extending MemoryGym's agentteam domain), and longer-horizon software engineering tasks that span multiple files and test suites (extending SWE-Infinite). [affine-cortex/affine/core/environments.py]

**Scoring mechanism refinement.** The anti-copy detector currently operates independently from the scoring pipeline, with results stored for manual review. Integrating detection signals directly into Pareto filtering — automatically excluding flagged copies from scoring rounds — would close the loop between forensic analysis and incentive enforcement. Absence decay parameters and the ELO K-factor schedule may also benefit from adaptive tuning as the miner population grows.

**Cross-environment training.** The geometric mean scoring already incentivizes broad-spectrum capability, but the system does not yet provide tools for curriculum learning across environments — for example, scheduling training on easier environments before harder ones, or weighting training samples by environment-specific capability gaps. Such tooling would make the platform more directly useful for RL researchers.

Affine's central bet is that the right incentive structure, combined with renewable environments and production-grade infrastructure, can turn open participation into sustained intelligence improvement. The system described here is the current instantiation of that bet. Whether it succeeds will be measured not by benchmark scores on a fixed test set, but by whether the models that emerge from this process are genuinely more capable agents than those trained without it.
