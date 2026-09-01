# From DevOps to MLOps: The Mental Model That Made It Click

I've spent over a decade in platform engineering — Kubernetes, OpenShift, Terraform, CI/CD pipelines that ship microservices for banks. More recently I've been building agent orchestration platforms for production AI systems. But for a long time, "MLOps" sat in my head as a separate discipline I hadn't touched.

Then I sat down and actually built the full loop — training, registry, serving, monitoring, retraining — and the honest realization was: **MLOps is DevOps with one extra axis of change.** If you've operated production services, you already know 70% of it. This post is the mapping that made the other 30% click, written for platform engineers standing where I stood.

## The one-sentence difference

In classic DevOps, your system changes when **code** changes. In MLOps, it changes when code changes, when **data** changes, or when the **world** changes — and the last one happens without anyone pushing a commit.

That single fact generates almost everything that's genuinely new: experiment tracking, model registries, drift monitoring, continuous training. Everything else — containers, pipelines, rollouts, observability, IaC — is the DevOps you already know, pointed at a slightly weirder artifact.

## The mapping table

| You already know... | The MLOps equivalent | What's actually different |
|---|---|---|
| Artifact registry (JARs, images) | Model registry (MLflow, SageMaker) | Artifacts carry *metrics*, not just versions. "v14" means nothing; "v14, AUC 0.83 on holdout" is the identity. |
| `git log` for code lineage | Experiment tracking | You must reconstruct *how* the artifact was made: data window, params, seed, metrics — or you can't debug it, and in a bank, you can't ship it. |
| CI: build + test on PR | CI + **training smoke test** | Tests now include "does training converge and clear a quality bar?" |
| CD: deploy on merge | CD + **promotion gate** | A challenger model ships only if it beats the champion on holdout data. Deploying is a *comparison*, not a copy. |
| — (no equivalent) | **CT: continuous training** | The genuinely new pipeline. Nightly drift check → retrain → same gate → same rollout. The trigger isn't a commit; it's the world changing. |
| Prometheus alerts on error rate, latency | All of that **plus drift metrics** | Your service can be 200-OK and sub-10ms while giving increasingly wrong answers. You need statistical monitoring on inputs and predictions, not just RED metrics. |
| Blue/green, canary deploys | Same, for models | Identical machinery — the health signal adds model quality, not just HTTP errors. |

Two things on that list have no DevOps ancestor: **drift monitoring** and **the CT loop**. Those are where I focused my learning, and where I'd tell any platform engineer to focus.

## Drift: the failure mode with no stack trace

A deployed model is a snapshot of the world at training time. The world moves — a rate cycle turns, customer behavior shifts, an upstream system changes a field's encoding — and prediction quality decays with **no exception, no 500, no log line**.

The standard answer is statistical: keep a reference sample of training data, compare the live feature distributions against it on a schedule (Wasserstein distance for numeric features, Jensen–Shannon for categorical — libraries like Evidently give you this out of the box), and alert when too many features have drifted. That alert is your new "pager goes off" — except the runbook ends in *retrain the model*, not *restart the pod*.

The subtlety worth knowing at interview depth: **data drift** (inputs shifted) is detectable immediately, while **concept drift** (the input→outcome relationship itself changed) is only fully confirmable once ground truth arrives — which for credit risk means months later, when loans actually default or don't. So mature setups monitor input drift *and* prediction distributions as leading indicators, then backfill true performance when labels land.

## CT: the pipeline that triggers itself

Continuous training closes the loop: drift detected → retrain on a fresh window → evaluate → **gate** → promote → rollout. The part most people get wrong (and the part I made the centerpiece of my [platform repo](https://github.com/KarthikTools/credit-risk-mlops-platform)) is the gate: retraining because of drift does **not** guarantee the new model is better. Bad windows happen. The challenger must beat the champion on a fair holdout, or you keep the champion and page a human. A worse model never ships just because the scheduler fired.

If you internalize one design principle from MLOps, make it this: **training and promoting are separate steps with a comparison in between.**

## What surprised me coming from platform land

**The registry is the control plane.** Deployment configs don't point at image tags so much as registry aliases — `models:/credit-risk-classifier@champion`. Promote by moving the alias; roll back the same way. It's `kubectl set image`, but the source of truth is the model registry.

**Reproducibility is compliance, not hygiene.** In regulated environments, "which data trained the model that declined this customer, and can you rebuild it?" is a question regulators actually ask. Seeds, data snapshots, logged params — the boring discipline is the audit trail.

**Serving is just a service.** FastAPI, pydantic validation, health probes, HPA, non-root containers, `/metrics` — every hardening pattern transfers untouched. The only new tenant is the model file. (Low-latency serving does add its own fun — batch-of-one inference cost, model warmup — but that's optimization, not a new paradigm.)

## How I'd learn it, in order

1. **Build the loop small, but build all of it.** One dataset, two candidate models, MLflow, FastAPI, a drift check, a retrain trigger. The value is in the *connections*, not any single tool.
2. **Make the promotion gate fail on purpose.** Train a worse challenger and watch the pipeline refuse it. Now you understand the gate viscerally.
3. **Simulate drift and watch your monitor catch it.** Shift the population, see per-feature scores climb past thresholds, see the alert fire.
4. **Then read the managed offerings** (SageMaker Pipelines, Vertex, Azure ML) — they'll map neatly onto the loop you built by hand, instead of being a wall of product names.

That's exactly the sequence behind my portfolio repos: an [end-to-end platform](https://github.com/KarthikTools/credit-risk-mlops-platform), a [drift monitoring stack](https://github.com/KarthikTools/model-drift-monitor), the [Terraform for bank-grade ML infra](https://github.com/KarthikTools/terraform-ml-platform-aws), and the same lifecycle discipline [applied to LLM prompts](https://github.com/KarthikTools/llmops-eval-pipeline).

If you're a platform engineer eyeing MLOps: you're closer than you think. The extra axis of change is the whole trick. Build the loop once and the vocabulary stops being a moat.

---

*I'm Karthik Marimuthu — platform engineer working on production AI systems. Find me on [LinkedIn](https://www.linkedin.com/in/karthik-marimuthu-5416091ba/) or [GitHub](https://github.com/KarthikTools).*
