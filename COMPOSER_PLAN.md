# Composer Autoresearch Plan

## 1. Goal

Build a Composer-owned research system that searches over valid Symphony DSL programs, scores them with a learned out-of-sample alpha model, validates promising candidates with backtests, and eventually composes a diversified portfolio of strategies.

This fork should not be treated as a drop-in deployment of upstream `autoresearch`. The upstream repo demonstrates a pattern:

- keep the codebase small
- give the agent a measurable objective
- run repeated fixed-budget experiments
- keep changes only when the metric improves

That pattern is useful for Composer, but the actual workload is different. Upstream optimizes an LLM training loop. Composer needs to optimize a strategy search loop.

## 2. Decision Summary

The plan is:

1. Preserve the upstream repo as the reference implementation of the `autoresearch` pattern.
2. Add Composer-specific research code next to it instead of forcing the Composer problem into the existing `train.py`.
3. Start with offline flat strategy search.
4. Require backtest-based validation before any candidate is treated as real.
5. Delay recursive strategy-of-strategy composition until the flat pipeline is trustworthy.
6. Use GCP because local H100-class compute is unavailable.
7. Use H100 only where it is actually needed, primarily reward-model training or heavy batched inference. Do not assume every part of the system needs H100.

## 3. What We Are Building

### Core problem

Given:

- a Symphony DSL program
- a validator/compiler for that DSL
- a learned model that predicts OOS alpha and related risk signals

Find:

- the highest-quality valid Symphony under a constrained objective
- then a diversified set of high-quality Symphonies
- then, later, recursive compositions of those Symphonies

### Working objective

For a single candidate strategy `S`, use an objective like:

`score(S) = predicted_alpha(S) - penalties(S)`

Where penalties include:

- invalid DSL or compile failure
- excessive complexity
- excessive turnover
- concentration
- liquidity and capacity issues
- out-of-distribution uncertainty

For portfolio construction, use:

`portfolio_score(S, P) = predicted_alpha(S) - corr_penalty(S, P) - penalties(S)`

Where `P` is the current selected set.

### Non-goals for v1

- no direct live trading from agent-generated strategies
- no automatic deployment to customer accounts
- no recursive composition in the first release
- no trust in the reward model without backtest gating
- no assumption that LLM free-form generation is enough on its own

## 4. How This Relates To Upstream Autoresearch

Upstream `autoresearch` is a meta-optimization loop. It edits the code that performs training, runs a short experiment, and keeps the change only if the metric improves.

Composer should borrow that pattern in two stages.

### Stage 1: build the search system

Implement a fixed search loop that can:

- generate Symphony candidates
- validate them
- score them
- backtest top candidates
- rank survivors

At this stage, the agent is optimizing Symphony candidates, not the optimizer itself.

### Stage 2: let the agent improve the optimizer

Once the search loop exists, use the `autoresearch` pattern on the search code itself:

- mutation operators
- seed generation
- candidate selection
- objective weights
- deduping logic
- portfolio assembly rules
- backtest triage thresholds

That is the point where this genuinely becomes a Composer adaptation of `autoresearch`, rather than a standard search system.

## 5. Proposed Repository Shape

The upstream repo is intentionally minimal. We should keep that quality, but not overload `train.py` with Composer-specific logic.

Recommended additions:

```text
COMPOSER_PLAN.md
composer/
  search.py
  generate.py
  mutate.py
  validate.py
  score.py
  backtest_gate.py
  portfolio.py
  checkpoints.py
programs/
  composer-search.md
  composer-optimizer.md
infra/
  gcp/
    batch/
    compute/
    startup/
docs/
  architecture.md
  operations.md
```

Repository conventions:

- keep upstream files readable and mostly untouched
- keep Composer logic under a dedicated `composer/` package
- keep agent instructions in separate `programs/` files
- keep infrastructure configs under `infra/gcp/`

## 6. System Architecture

### 6.1 Search controller

This is the orchestration layer. It owns one research run.

Responsibilities:

- load the experiment config
- generate or load seed strategies
- submit candidate batches for scoring
- trigger backtests for shortlisted candidates
- write results and checkpoints
- decide which candidates advance

### 6.2 Candidate generator

This should not rely only on free-form LLM output.

It should support:

- hand-built Composer strategy templates
- grammar-aware mutations
- parameter perturbations
- subtree substitution
- template recombination
- optional LLM generation constrained by DSL rules

### 6.3 Validator and canonicalizer

This is mandatory.

Responsibilities:

- reject invalid DSL
- compile to canonical form
- normalize equivalent strategies to the same representation
- compute complexity features
- dedupe near-identical strategies

Without canonicalization, the search will waste time on syntactic variants of the same strategy.

### 6.4 Reward model service

The reward model should return more than one number.

At minimum:

- predicted OOS alpha
- confidence or uncertainty
- predicted turnover or cost proxy
- predicted concentration or exposure proxy
- out-of-distribution score

The search objective should penalize low-confidence or out-of-distribution candidates.

### 6.5 Backtest gate

Backtests are not optional in practice.

Use the reward model to filter cheaply, then backtest the top `K` candidates from each batch. Backtests should produce:

- holdout return series
- Sharpe and drawdown metrics
- turnover and transaction-cost sensitivity
- exposure diagnostics
- benchmark-relative behavior

### 6.6 Portfolio constructor

After finding good standalone candidates, build a diversified set using:

- return correlation
- factor or embedding similarity
- turnover overlap
- shared failure mode heuristics

Do not begin with recursive composition. First build a strong diversified set of flat strategies.

### 6.7 Experiment tracker

Each run should store:

- commit hash
- run config
- seeds used
- candidate specs
- reward-model outputs
- backtest summaries
- selected survivors
- artifacts and logs

This should live outside git. Use cloud storage and a queryable table.

## 7. Recommended Phases

### Phase 0: bootstrap

- fork the repo into `invest-composer`
- create a working branch for the first planning/documentation pass
- keep the upstream layout intact
- document architecture and operating model

Exit criteria:

- repo exists under Composer ownership
- plan document exists
- GCP target architecture is chosen

### Phase 1: flat offline search

Build a single-run controller that searches only flat strategies.

Scope:

- input: strategy templates and mutation rules
- scoring: reward model only
- validation: DSL compile plus canonicalization
- selection: top `N` by score
- checkpointing: required

Exit criteria:

- system can complete a full offline search run
- invalid strategies are filtered correctly
- duplicate candidates are suppressed
- results are reproducible from a saved config

### Phase 2: backtest-gated ranking

Add a backtest validation step for top candidates.

Scope:

- backtest the top `K`
- compute real diversification metrics
- rank by robust holdout metrics, not just predicted alpha

Exit criteria:

- shortlist contains only candidates that survive backtest validation
- reward-model ranking and backtest ranking can be compared quantitatively
- failure cases are logged for reward-model retraining

### Phase 3: portfolio search

Build a diversified portfolio from validated candidates.

Scope:

- portfolio-level objective
- correlation penalty
- exposure diversity
- strategy clustering

Exit criteria:

- system can return a set of candidates, not just a single winner
- portfolio selection is measurably more diversified than naive top-`N`

### Phase 4: recursive composition

Only after flat search works.

Scope:

- allow strategies to reference previously selected strategies
- enforce acyclic dependency checks
- cap depth and node count
- require readability and complexity limits

Exit criteria:

- recursive strategies compile reliably
- search does not collapse into unreadable meta-trees
- backtests show incremental value beyond flat ensemble selection

### Phase 5: self-improving optimizer

This is the true `autoresearch` stage.

Scope:

- agent edits the search code and prompt/program files
- each run is evaluated on a fixed benchmark set
- optimizer changes are kept only if aggregate run quality improves

Exit criteria:

- search-code changes are benchmarked automatically
- the optimizer gets better over time, not just the current candidate set

## 8. GCP Deployment Plan

## 8.1 Principles

- separate control-plane services from GPU-heavy worker jobs
- checkpoint aggressively because the most attractive 1x H100 A3 shapes are Spot or Flex-start only
- keep all generated strategies and metrics offline until reviewed
- use managed services for logs, secrets, and artifact storage

## 8.2 Recommended GCP components

Use:

- `Compute Engine` for initial interactive GPU setup and debugging
- `Batch` for repeatable GPU jobs once the runner is stable
- `Cloud Storage` for checkpoints, logs, and experiment artifacts
- `Artifact Registry` for versioned worker containers
- `Secret Manager` for API keys and internal credentials
- `Cloud Logging` and `Cloud Monitoring` for observability
- `BigQuery` or Cloud SQL for structured experiment metadata
- `Cloud Run` or a small CPU VM for the orchestration API if needed

## 8.3 GPU choice

Upstream `autoresearch` says it was tested on a single H100. Google Cloud currently offers `a3-highgpu-1g` with 1x H100 80GB, 26 vCPU, and 234 GB RAM. Google also documents that `a3-highgpu-1g`, `a3-highgpu-2g`, and `a3-highgpu-4g` must be created as `Spot` or `Flex-start` VMs.

Implications:

- use `a3-highgpu-1g` for reward-model training or very heavy batched inference
- do not bind the full system to H100 if cheaper machines are enough for orchestration and search
- expect preemption or delayed starts and design for resume

## 8.4 Initial deployment recommendation

Start with two layers.

### Layer A: controller

Run on CPU-only infrastructure:

- a small `Cloud Run` service, or
- a small `e2-standard` / `c3-standard` Compute Engine VM

Responsibilities:

- create experiment configs
- enqueue search runs
- monitor job state
- collect outputs
- write metadata rows

### Layer B: worker

Run as a GPU job:

- bootstrap option: single `Compute Engine` VM for manual iteration
- production option: `Batch` jobs backed by GPU VMs

Responsibilities:

- fetch repo revision and config
- load the reward model
- run search batches
- checkpoint progress to GCS
- upload logs and metrics

## 8.5 Machine recommendations

Use the cheapest machine that fits the task.

Recommended defaults:

- reward-model training: `a3-highgpu-1g`
- heavy reward-model inference or large batched scoring: `a3-highgpu-1g`
- ordinary orchestration and metadata work: CPU-only VM or Cloud Run
- if the reward model can score on smaller GPUs, evaluate `g2`/L4 separately before standardizing on H100

The likely mistake here is overprovisioning. Composer should only pay H100 prices where H100 materially improves throughput or model quality.

## 8.6 OS and runtime

For the first GPU worker image:

- use Ubuntu 22.04 or 24.04
- use `uv` for dependency management
- build a pinned container image in Artifact Registry

Google Cloud currently recommends CUDA `12.2.2` or later for A3 H100 VMs, and its GPU driver install docs describe the supported Linux images and startup-script flow.

Practical recommendation:

- create a custom image or container that already has the right driver and CUDA setup validated
- avoid relying on ad hoc manual setup for repeatable jobs

## 8.7 Storage layout

Create a dedicated GCS bucket, for example:

```text
gs://composer-autoresearch-dev/
  configs/
  checkpoints/
  logs/
  results/
  backtests/
  models/
```

Use GCS for:

- search checkpoints
- serialized candidate batches
- backtest outputs
- reward-model snapshots
- worker logs that should survive preemption

Use BigQuery or Cloud SQL for:

- experiment metadata
- run status
- candidate summaries
- aggregate metrics

## 8.8 Batch job model

When the worker code stabilizes, move GPU execution to Batch.

Why Batch:

- it provisions VMs per job
- it is a better fit for repeatable offline runs than a hand-managed long-lived VM
- it supports GPU jobs and driver installation options

Recommended job behavior:

- one Batch job per experiment run
- config passed via GCS path or environment
- checkpoints written every few minutes or every `N` evaluated candidates
- idempotent resume from the latest checkpoint
- explicit max runtime and retry count

## 8.9 Handling Spot or Flex-start constraints

Because the most useful small A3 High shapes are not normal on-demand VMs, assume interruptions.

Design requirements:

- checkpoint frequently
- persist logs outside the VM
- make each run resumable
- make job submission idempotent
- use a controller that can relaunch from saved state

If job start latency becomes a problem, keep a single manually managed debugging VM for developer work and use Batch only for unattended runs.

## 8.10 Security

Use a dedicated service account with least privilege:

- read from the experiment bucket
- write checkpoints and results
- read secrets needed for model or API access

Store in Secret Manager:

- model API keys if applicable
- GitHub token if CI automation needs it
- internal Composer credentials

Do not store secrets in repo files, shell history, or job configs committed to git.

## 8.11 Observability

Minimum required signals:

- job success/failure counts
- average candidates scored per run
- reward-model latency
- backtest queue depth
- checkpoint age
- preemption/retry counts
- cost per completed experiment

Alert on:

- repeated worker crashes
- missing checkpoints for active jobs
- large drift between reward-model ranking and backtest ranking

## 8.12 Cost controls

Add hard controls from day one:

- GPU job concurrency cap
- automatic TTL for idle debug VMs
- per-run budget limit
- nightly or weekly budget alarms
- separate dev and prod projects or budgets

This project will otherwise spend money very quickly with little accountability.

## 9. Data and Evaluation Plan

The reward model is the main technical risk.

Requirements:

- train on Composer-relevant strategy data
- keep strict temporal holdouts
- test for reward hacking and distribution shift
- log disagreement between predicted alpha and realized backtest quality

Key principle:

The search should optimize a model that is constantly audited, not blindly trusted.

Recommended evaluation sets:

- a fixed benchmark set of known strategies
- a held-out temporal slice
- an adversarial set of weird but valid DSL strategies
- a simplicity-biased slice to discourage unreadable strategies

## 10. Risks

### Reward hacking

The search may find strategies that score well because of model blind spots.

Mitigation:

- uncertainty penalty
- backtest gating
- adversarial test set
- periodic human review

### Search collapse

The system may rediscover the same family repeatedly.

Mitigation:

- canonicalization
- clustering
- diversity-aware selection
- seed variety

### Recursive strategy explosion

Recursive composition can become unreadable and fragile.

Mitigation:

- defer recursion
- enforce max depth
- enforce node-count limits
- add strong complexity penalties

### Compute waste

H100-backed jobs can become expensive quickly.

Mitigation:

- use H100 only where justified
- checkpoint and resume
- cap concurrency
- track cost per useful experiment

## 11. Immediate Next Steps

### Repo work

1. Commit this document.
2. Add a `docs/architecture.md` that turns this plan into a component diagram.
3. Add a minimal `composer/` package with stub modules and typed interfaces.
4. Add `programs/composer-search.md` for the first fixed-search loop.

### Product and research work

1. Define the exact DSL validation and canonicalization contract.
2. Define the reward-model API and output schema.
3. Choose the shortlist backtest interface.
4. Build a benchmark set of strategies for regression testing.

### Infra work

1. Create a dedicated GCP project or environment for autoresearch.
2. Set up the GCS bucket, service account, and budget alerts.
3. Bring up one manual GPU worker VM for debugging.
4. Containerize the worker and migrate unattended runs to Batch.

## 12. Recommended First Milestone

The first milestone should not be "recursive agentic optimization."

It should be:

"Run one reproducible offline search job that:

- starts from a fixed seed set
- generates and validates flat Symphony candidates
- scores them with the reward model
- backtests the top `K`
- writes a ranked results table and checkpoints to GCS"

If that milestone is not solid, the recursive story is premature.
