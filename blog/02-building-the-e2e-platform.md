# Anatomy of an End-to-End MLOps Platform: CI, CD, and the CT Loop Everyone Skips

Most MLOps tutorials stop at "we logged the model to MLflow 🎉". The interesting engineering starts *after* that: how does a better model reach production without a human copying files, and how does the system notice when the deployed model goes stale?

I built a complete reference implementation to answer both questions concretely — a credit-risk scoring platform with three pipelines (CI, CD, CT), a model registry with promotion gates, hardened Kubernetes serving, and drift-triggered retraining. It runs entirely locally, zero cloud bill. Code is here: [credit-risk-mlops-platform](https://github.com/KarthikTools/credit-risk-mlops-platform). This post walks the architecture and the design decisions I'd defend in a review.

## The shape of the system

Three pipelines with three different triggers:

- **CI** — trigger: pull request. Lint, unit tests, and a *training smoke test*: train the model end to end on a small budget and fail the PR if it can't converge past a quality floor. Model code that can't train is broken code, and you want to know before merge.
- **CD** — trigger: merge to main. Train candidates, evaluate, **gate**, register, build the serving image, deploy with a manual approval in front of production.
- **CT** — trigger: *the world changing*. A nightly drift check compares production traffic against the training reference; drift kicks off retraining, which flows through the exact same gate and rollout as CD.

The symmetry in that last sentence is the whole design. Retrained models don't get a shortcut to production — same bar, same audit trail, whether a human or a scheduler initiated them.

## The promotion gate: deploying is a comparison

The single most load-bearing file in the repo is the evaluation gate (~80 lines). Logic:

1. Score the newest registered version (**challenger**) on a fixed holdout set.
2. It must clear an **absolute bar** (ROC-AUC ≥ 0.75 here) — protects against the "champion silently degraded and the challenger is merely less bad" trap.
3. It must **beat the current champion** on the same data.
4. Only then does the `@champion` alias move. Otherwise the pipeline exits non-zero and CI goes red — a worse model failing to ship should be *loud*.

MLflow's registry aliases (which replaced the old stage-transition API) make promotion atomic: serving configs reference `models:/credit-risk-classifier@champion`, and promotion/rollback are just the alias moving between immutable versions. If you've done blue/green with load balancer target swaps, this is the same move, one level up the stack.

## Serving: a boring service on purpose

The scoring API is FastAPI: pydantic contracts that reject malformed applications at the boundary with a 422, single (`/predict`) and batch (`/predict/batch`) paths, separate liveness and readiness probes — a pod that's *alive* but hasn't loaded its model must report *not ready*, or Kubernetes routes traffic into a 500 factory — and a Prometheus `/metrics` endpoint counting predictions, errors, latency, and how many requests land in the HIGH-risk band. That last one is a *model* health metric disguised as a service metric: if 40% of applications suddenly score high-risk, either the world changed or your model broke; both are pageable.

Deployment hardening is standard platform practice, applied unchanged: multi-stage image, non-root user, read-only root filesystem, dropped capabilities, `maxUnavailable: 0` rollouts, HPA, and a default-deny NetworkPolicy. One deliberate choice worth defending: **the model is baked into the image** rather than pulled from the registry at startup. Pulling is more flexible; baking means an image digest *is* a model version — immutable, scannable, and exactly reproducible — which is the trade a regulated environment wants. (The registry-pull variant is one env var away; the repo shows both.)

## The CT loop, concretely

Nightly, a workflow runs an Evidently `DataDriftPreset` comparing a window of recent traffic against the training reference frame. The drift script talks to the pipeline through an exit code: **42 means drift detected** (share of drifted features ≥ 30%). Crude? Absolutely. Also: greppable, language-agnostic, and impossible to misparse — sometimes the unix way is the right way.

Exit 42 triggers the retrain job: fresh training run, registration, and the *same promotion gate*. Drift explains *why* we retrained; it doesn't lower the bar for what ships. The run also opens a tracking issue automatically, so every retrain event has an audit artifact with the Evidently report attached.

Because the demo dataset is synthetic (deterministic generator, seeded), the repo has a knob real tutorials can't offer: `drift_severity`. Crank it to 0.8 — simulating a macro downturn shifting incomes, credit scores, and debt ratios — and watch 6 of 10 features flag, the workflow branch into retraining, and the gate either promote or refuse the result. Reproducible drift makes the monitoring *testable*, which is the difference between "we have drift detection" and "we've watched it work."

## Decisions I'd defend in review

- **Train ≠ promote.** Training always logs; only the gate moves the alias. That separation is where human approval (GitHub environments) slots in without blocking experimentation.
- **Hermetic CI.** Synthetic data means no dataset downloads, no flaky network, bit-identical reruns. CI you don't trust is CI you ignore.
- **Two candidate models, always.** Logistic regression as the interpretable baseline, gradient boosting as the challenger. The day boosting stops beating a linear model on your data, you want the pipeline — not a person — to notice.
- **Plain-text metrics over a client library.** The Prometheus exposition format is trivial to emit; the *contract* (counter vs gauge semantics) is what matters. One less dependency in the serving image.

## What I'd add next

Feast for online/offline feature parity (training-serving skew is the bug class the current design doesn't defend against), a gRPC path beside REST, and SHAP reason codes on the response — in credit, "declined, because..." isn't UX polish, it's an adverse-action requirement.

---

*Part of my MLOps portfolio: [drift monitoring stack](https://github.com/KarthikTools/model-drift-monitor) · [bank-grade ML infra in Terraform](https://github.com/KarthikTools/terraform-ml-platform-aws) · [LLMOps eval pipeline](https://github.com/KarthikTools/llmops-eval-pipeline). I'm Karthik Marimuthu — [LinkedIn](https://www.linkedin.com/in/karthik-marimuthu-5416091ba/).*
