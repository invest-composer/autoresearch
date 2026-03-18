# Composer Autoresearch Plan

## 1. Goal

Adapt upstream `autoresearch` into a Composer-specific research harness that helps an agent search for strong Symphony DSL candidates without starting from a blank page.

The initial version should stay close to the upstream shape:

- `prepare.py` prepares the fixed research environment and corpus-derived assets
- `train.py` runs a barebones strategy search loop
- `program.md` tells the agent how to improve `train.py` over time

The main difference from upstream is the executable loop. Upstream runs short LLM training experiments. Composer should begin with a corpus-seeded strategy search loop that backtests Symphony candidates, feeds the resulting stats into a fixed OOS prediction model, and then ranks the candidates.

## 2. Decision Summary

The updated plan is:

1. Keep the repo small and use the upstream file layout as the initial interface.
2. Use the existing Symphony corpus as the starting point for search, not free-form generation from scratch.
3. Put corpus ingestion, deduping, seeding, and fixed evaluation-set creation into `prepare.py`.
4. Put a minimal search / score / shortlist / checkpoint loop into `train.py`.
5. Put agent operating instructions into `program.md`, with the expectation that the agent mostly edits `train.py`.
6. Treat the OOS prediction model as a fixed reward model during any one optimizer-improvement cycle.
7. Keep backtest gating mandatory for top candidates.
8. Do the initial build and experimentation locally, and defer GCP until the local loop is proven useful.

## 3. What We Are Building

### Core problem

Given:

- a Symphony DSL program
- a large historical corpus of existing Symphonies and their metrics
- a learned model that predicts OOS alpha and related quality signals

Find:

- the highest-quality valid Symphony under a constrained objective
- then a diversified set of high-quality Symphonies
- then, later, recursive compositions of those Symphonies

### Working objective

Because the OOS prediction model depends on backtest outputs, there is no true pre-backtest score. The minimal ranking loop is:

1. validate / compile the Symphony
2. run the backtest
3. compute the backtest-stat feature vector
4. compute `predicted_oos_alpha`
5. reject candidates that violate hard constraints
6. apply a small number of extra penalties if they add value
7. rank the survivors

So the working objective is:

`score(S) = predicted_oos_alpha(S) - extra_penalties(S)`

subject to:

- valid DSL / successful compile
- minimum backtest history
- basic risk and quality constraints

For portfolio construction later:

`portfolio_score(S, P) = predicted_oos_alpha(S) - corr_penalty(S, P) - extra_penalties(S)`

Where `P` is the current selected set.

### Score function design

The scoring function should not begin as a large hand-weighted mixture of backtest metrics. Many of those metrics are already inputs to the OOS prediction model, so using them again as soft penalties risks double-counting.

The initial design should be:

- use `predicted_oos_alpha` as the main score
- use hard constraints for obviously unacceptable candidates
- add only a small number of extra penalties for things the OOS model likely does not capture well or that reflect explicit product preferences

Examples of likely hard constraints:

- compile or backtest failure
- too few backtest days
- extreme annualized turnover
- extreme Herfindahl concentration
- extreme max drawdown
- extreme return concentration, such as too much dependence on a small number of days

Examples of likely extra penalties:

- complexity penalty derived from DSL structure
- out-of-distribution penalty in backtest-stat feature space
- near-duplicate penalty against the existing corpus
- portfolio-correlation penalty later, when selecting a basket instead of a single strategy

Examples to defer initially:

- liquidity penalty, unless we compute it from actual realized trades using daily holdings plus OHLCV
- large weighted mixtures of Sharpe, Sortino, Calmar, skew, kurtosis, and related diagnostics

The weights on any extra penalties should not be treated as fixed truths. Once we score the full historical corpus and have predicted OOS alpha for all feasible candidates, we should study the highest-scoring failures and use that empirical ranking behavior to decide which extra penalties are worth adding.

In practice, that means:

- start with `predicted_oos_alpha` plus hard constraints
- inspect the ranked corpus for bad but highly scored candidates
- add one penalty at a time
- keep a penalty only if it improves historical selection quality on unseen data

### Fragility of returns

Return fragility should be derived from the backtest return series, not guessed from the DSL.

The initial plan should treat fragility as a hard constraint candidate, not a carefully weighted soft term. Good first measures are:

- concentration of total positive performance in the best few days
- sensitivity of annualized return or Sharpe to removing the best few days
- dependence on a single month or quarter

The existing `train_top_ten_percent_day_contribution` stat is a useful starting point, but it should probably be supplemented later with a jackknife-style sensitivity metric from the full return series.

### Non-goals for v1

- no direct live trading from agent-generated strategies
- no automatic deployment to customer accounts
- no recursive composition in the first release
- no trust in the reward model without backtest gating
- no requirement that the LLM invent strategies from scratch

## 4. How This Relates To Upstream Autoresearch

Upstream `autoresearch` fixes the evaluation harness and lets the agent iteratively improve `train.py`.

Composer should mirror that pattern:

- humans define the fixed setup in `prepare.py`
- humans define the research policy in `program.md`
- the agent starts from a barebones `train.py` and improves it over time

The key adaptation is that Composer's `train.py` is not training an LLM. It is running a strategy search loop over Symphony candidates.

### Stage 1 meaning

In the first Composer version, `train.py` should:

- load a prepared seed pool from the existing Symphony corpus
- generate or mutate candidates from those seeds
- backtest the candidates
- score them with a fixed reward model that consumes backtest stats
- rank and checkpoint the results

At this stage, the agent is improving the search procedure, not training a generator model.

### Later stages

Only after the search loop is working should we consider:

- recursive strategy composition
- training a generator model to propose better candidates
- letting the agent modify more than `train.py`

## 5. Role Of The Existing Symphony Corpus

The existing corpus is the most important advantage we have over a blank-slate setup.

The corpus should be treated as:

- a warm-start seed bank
- a source of canonical valid strategies
- a source of benchmark strategies for evaluation
- a source of negative examples, duplicates, and weird edge cases

The initial system should not ask the LLM to invent candidate Symphonies from nothing. It should begin by retrieving, mutating, and recombining existing strategies from the corpus.

### What `prepare.py` should extract from the corpus

At minimum:

- a canonicalized strategy representation
- deduped or near-deduped strategy rows
- a seed set for search
- a fixed benchmark set for evaluating search-loop changes
- a holdout set for checking reward-model drift or search overfitting

The corpus almost certainly contains many copies or small variants. That is fine, but `prepare.py` should reduce the operational dataset to something closer to unique strategy structures so the search loop is not dominated by duplicates.

## 6. Proposed Repository Shape

For the first Composer adaptation, keep the repo close to upstream:

```text
COMPOSER_PLAN.md
prepare.py
train.py
program.md
README.md
```

Expected responsibilities:

- `prepare.py`
  - load exported Symphony corpus data
  - canonicalize and dedupe enough for practical search
  - materialize seed, benchmark, and holdout artifacts
  - expose fixed helper functions used by `train.py`
- `train.py`
  - run the minimal strategy search loop
  - backtest candidates and then score them with the fixed reward model
  - checkpoint and print summary metrics
- `program.md`
  - tell the agent how to evaluate changes to `train.py`
  - tell the agent to start from the corpus-derived assets
  - define what is fixed versus editable

Only after this works should we consider breaking Composer-specific logic into more files or packages.

## 7. System Design For The Minimal Version

### 7.1 `prepare.py`

`prepare.py` becomes the fixed environment builder.

Its job is to:

- ingest a local export or cloud snapshot of the Symphony corpus
- build a canonical representation for each strategy
- suppress obvious exact duplicates and optionally simple near-duplicates
- write out a seed pool
- write out a benchmark set
- write out a holdout set
- provide helper loaders for `train.py`

`prepare.py` should also define fixed evaluation boundaries, the same way upstream fixes `evaluate_bpb`, including the fixed benchmark and holdout sets used to judge whether scoring changes are helping.

### 7.2 `train.py`

The first Composer `train.py` should be intentionally simple.

It should:

- load the prepared seed pool
- pick an initial batch of candidates from the corpus
- apply simple mutations or recombinations
- backtest each feasible candidate
- compute `predicted_oos_alpha` from the backtest outputs
- apply hard constraints and any enabled extra penalties
- keep the best results under a fixed run budget
- print a small fixed summary at the end

This gives the agent something concrete to improve without requiring a big framework first.

### 7.3 `program.md`

`program.md` should be rewritten for the Composer problem.

It should tell the agent:

- what `prepare.py` is responsible for and that it is fixed
- that the corpus already exists and should be treated as the starting prior
- that `train.py` is the main editable surface
- what metrics matter and which ones are constraints versus soft penalties
- when a change should be kept or discarded
- that candidates must be backtested before predicted OOS alpha can even be computed

### 7.4 Fixed metrics for the agent

The agent needs a stable notion of improvement.

For the initial loop, improvement should be measured on a fixed benchmark set derived from the corpus and a fixed reward-model snapshot, plus the quality of the resulting ranked selection after constraints and any extra penalties are applied.

The metric must not drift inside the same optimizer-improvement cycle.

## 8. Recommended Phases

### Phase 0: bootstrap

- fork the repo into `invest-composer`
- document the plan
- identify the corpus export format
- decide the first fixed metrics and artifact paths

Exit criteria:

- repo exists under Composer ownership
- plan document exists
- the initial corpus input format is chosen

### Phase 1: corpus preparation

Teach `prepare.py` to build the fixed research assets.

Scope:

- load corpus snapshot
- canonicalize strategies
- suppress duplicates enough for useful seeding
- write seed / benchmark / holdout artifacts

Exit criteria:

- `prepare.py` can run end-to-end on a representative corpus snapshot
- the resulting seed pool is materially smaller than the raw copied corpus
- the benchmark and holdout sets are fixed and reproducible

### Phase 2: barebones search loop

Build the first Composer `train.py`.

Scope:

- load prepared assets
- sample candidates from the seed pool
- apply a minimal mutation / recombination strategy
- backtest feasible candidates
- compute predicted OOS alpha from the backtest stats
- rank using hard constraints plus a minimal extra-penalty set
- emit a fixed summary and checkpoints

Exit criteria:

- a single run completes end-to-end
- top candidates come from corpus-seeded search, not blank generation
- results are reproducible from a saved config

### Phase 3: agent-facing `program.md`

Rewrite `program.md` so the agent can begin improving `train.py`.

Scope:

- describe the Composer objective
- define what is fixed and what can change
- define the experiment loop and output format
- instruct the agent to use corpus-derived assets, not fresh invention

Exit criteria:

- a coding agent can read `program.md` and run one valid experiment cycle
- the agent has an unambiguous success metric

### Phase 4: backtest-gated ranking

Tighten candidate ranking beyond the first proof of concept.

Scope:

- compare reward-model ranking versus backtest ranking
- compare plain predicted-OOS ranking versus constrained / penalized ranking
- track disagreement cases
- refine shortlist rules

Exit criteria:

- the system keeps only candidates that survive backtest validation
- backtest outcomes are logged as a first-class output

### Phase 5: recursive composition and richer search

Only after the first loop is stable.

Scope:

- recursive strategy composition
- deeper mutation policies
- more structured recombination
- possibly generator-model training later

Exit criteria:

- recursive search adds value beyond flat corpus-seeded search

## 9. Local-First Execution Plan

## 9.1 Principles

- start with the smallest setup that supports an end-to-end run on one developer machine
- keep the benchmark sets and prepared corpus artifacts fixed for each experiment cycle
- prefer simplicity and debuggability over parallelism
- separate one-time preparation work from repeated search runs
- defer cloud complexity until local runs are clearly bottlenecked

## 9.2 Minimal local shape

Start with one repo and one local worker process.

Recommended components:

- local filesystem for corpus exports, prepared artifacts, checkpoints, and logs
- one local Python environment managed by `uv`
- one local run directory per experiment
- lightweight structured logs written to files

Delay cloud storage, remote workers, and job orchestration until we actually need them.

## 9.3 Where `prepare.py` runs

`prepare.py` should run locally first. It is mainly data preparation and artifact construction.

Recommended approach:

- run `prepare.py` on CPU if the corpus processing path fits comfortably there
- only use local GPU acceleration if reward-model inference during preparation is expensive
- write prepared artifacts to a deterministic local directory so `train.py` runs can reuse them

## 9.4 Where `train.py` runs

`train.py` should also run locally first. It is the first likely GPU consumer because it may need repeated reward-model inference and repeated backtest triage.

Recommended approach:

- start on the current local machine, even if it is slower than ideal
- keep the first loop small enough that it can complete locally
- checkpoint to local disk
- only introduce remote execution once local debugging becomes the bottleneck

## 9.5 Local compute strategy

Upstream `autoresearch` was tested on a single H100 for LLM training. That should not drive the first Composer implementation.

Implications:

- keep `prepare.py` and the first `train.py` loop small enough for local iteration
- reduce search breadth, top-`K`, or run budget if needed to fit the local machine
- optimize first for correctness and inspectability, not throughput

If local runs prove too slow, that is the point to measure the bottleneck and decide whether cloud GPU is actually necessary.

## 9.6 Local storage layout

Use a dedicated local artifact directory, for example:

```text
./artifacts/
  corpus/
  prepared/
  checkpoints/
  logs/
  results/
  backtests/
  models/
```

Use it for:

- raw or exported corpus snapshots
- prepared seed / benchmark / holdout artifacts
- run checkpoints
- backtest outputs
- reward-model snapshots
- logs and run summaries

These directories can later be mirrored to cloud storage if local-first execution works.

## 9.7 Runtime environment

For the first local worker:

- `uv` for dependency management
- one reproducible local environment
- simple local scripts for setup and execution

Practical recommendation:

- validate the local environment once
- keep dependencies pinned
- only containerize after the local workflow is stable

## 9.8 Observability and cost controls

Track at minimum:

- successful versus failed runs
- checkpoint age
- candidates scored per run
- shortlist size and backtest pass rate
- reward-model latency
- local runtime per useful run

Controls:

- keep one run at a time until the loop is stable
- cap local artifact growth
- record per-run runtime so we know when local execution stops being practical

## 9.9 Deferred cloud plan

If local execution proves useful but too slow, move in this order:

1. local artifacts mirrored to cloud storage
2. one manually managed remote worker
3. reproducible remote image or container
4. batch-style unattended runs

Cloud is a scaling phase, not a prerequisite for the first milestone.

## 10. Data And Evaluation Plan

The reward model remains the main technical risk, and the ranking function around it is the second one.

Requirements:

- train it on Composer-relevant strategy data
- keep strict temporal holdouts
- compare predicted ranking with realized backtest quality
- audit for reward hacking and distribution shift

The corpus should help in three ways:

- it provides the seed pool for the search loop
- it provides the fixed benchmark and holdout sets used to evaluate changes to `train.py`
- it lets us inspect the full ranked distribution of predicted OOS alpha and empirically decide which extra penalties are actually useful

Key principles:

Do not change the reward model and the search metric at the same time. Inside one optimizer-improvement cycle, freeze the reward-model version and evaluate all `train.py` changes against the same prepared benchmark set.

Do not introduce extra penalties just because they sound reasonable. Add them only after they improve historical selection quality or encode an explicit business preference.

## 11. Risks

### Reward hacking

The search may find strategies that score well because of blind spots in the OOS prediction model.

Mitigation:

- fixed reward-model snapshot per experiment cycle
- backtest gating
- holdout benchmark set
- periodic human review

### Duplicate-heavy corpus

The raw corpus may be dominated by copied strategies or tiny variants.

Mitigation:

- canonicalization in `prepare.py`
- duplicate suppression before seeding
- benchmark sets that are not copy-heavy

### Search collapse

The system may stay too close to the corpus and fail to explore useful variants.

Mitigation:

- allow mutation and recombination in `train.py`
- periodically widen the search radius
- track whether shortlisted candidates are genuinely novel

### Recursive strategy explosion

Recursive composition can become unreadable and fragile.

Mitigation:

- defer recursion until the flat loop works
- enforce depth and complexity limits later

### Compute waste

Local iteration may be too slow to support useful search.

Mitigation:

- shrink the search loop until it is debuggable locally
- profile the real bottleneck before scaling out
- only introduce cloud compute after we can justify it with measurements

### Arbitrary score shaping

The ranking function may become an unprincipled collection of hand-picked penalty weights.

Mitigation:

- start from predicted OOS alpha plus hard constraints
- derive candidate penalty thresholds from the empirical corpus distribution
- add penalties one at a time
- keep them only if they improve unseen historical ranking quality or reflect a clear product policy

## 12. Immediate Next Steps

### Repo work

1. Rewrite `prepare.py` around Composer corpus preparation and fixed research artifacts.
2. Rewrite `program.md` around the Composer objective and fixed/editable boundaries.
3. Replace `train.py` with a barebones corpus-seeded search loop.
4. Keep the initial implementation deliberately small enough for an agent to iterate on.

### Product and research work

1. Choose the initial corpus export format and storage location.
2. Define the reward-model API and output schema.
3. Define the shortlist backtest interface and the return-series outputs needed for fragility calculations.
4. Decide the first fixed benchmark and holdout split policy.
5. Evaluate the plain predicted-OOS ranking on the historical corpus before adding extra penalties.

### Infra work

1. Choose a local artifact directory layout for corpus snapshots, prepared artifacts, checkpoints, and results.
2. Validate one reproducible local environment for `prepare.py` and `train.py`.
3. Add simple local scripts or commands for setup and execution.
4. Revisit cloud only after the first local loop is stable.

## 13. Recommended First Milestone

The first milestone should be:

"Run one reproducible corpus-seeded Composer search job where:

- `prepare.py` ingests the Symphony corpus and writes fixed seed / benchmark / holdout artifacts
- `train.py` loads those artifacts and runs a minimal search loop
- the reward model scores candidates under a fixed version after backtests run
- the first ranking pass uses predicted OOS alpha plus hard constraints, with at most a very small extra-penalty set
- the top `K` ranked candidates are surfaced for review and comparison
- the run writes checkpoints and a ranked summary to local artifacts"

If that milestone is not solid, the more ambitious recursive story is premature.
