# Your Model Is Failing Silently: Drift Monitoring with Evidently, Prometheus, and Grafana

Here's the property of ML systems that should scare anyone who runs them: **a model can be completely wrong while every ops dashboard is green.** 200s across the board, p99 in single digits, error rate zero — and the predictions quietly decaying because the world no longer looks like the training data.

No stack trace. No exception. Just a credit model approving people it shouldn't, for months, until the losses show up in a quarterly review.

Coming from platform engineering, this was the genuinely new problem MLOps handed me. RED metrics (rate, errors, duration) tell you the *service* is healthy. Nothing in classic observability tells you the *answers* are still right. So I built the missing layer as a standalone stack — Evidently for the statistics, Prometheus for alerting, Grafana for the picture — with a traffic simulator that lets you watch a model decay in fast-forward. Code: [model-drift-monitor](https://github.com/KarthikTools/model-drift-monitor).

## What drift actually is (30-second version)

**Data drift**: input distributions shift. Median applicant income drops 20% in a downturn; the model has never seen this population. Detectable immediately, from inputs alone.

**Concept drift**: the input→outcome relationship itself changes. Same income now implies different default risk because the labor market changed. Fully confirmable only when ground truth arrives — which in lending is months later. So input drift is your **leading indicator**, and you treat it accordingly: it doesn't prove the model is wrong, it tells you to look.

The statistics are refreshingly unexotic. Keep a reference sample from training; compare each live feature's distribution against it — Wasserstein distance for numeric features, Jensen–Shannon divergence for categorical; flag a feature when the distance crosses a threshold (0.1 is Evidently's sensible default); alert when the *share* of drifted features crosses yours.

## Architecture: measure, decide, show — separately

```
Scoring API ──feature rows──▶ Monitor service ──/metrics──▶ Prometheus ──▶ Alertmanager
                             (Evidently window                 │
                              vs reference)                    ▼
                                                            Grafana
```

The design rule that matters: **the monitor measures, Prometheus decides, Grafana shows.** The monitoring service computes drift and exposes gauges — it contains zero alerting logic. Thresholds, `for:` durations, severities, routing: all Prometheus rules, reviewable in one file, changeable without touching Python. Every layer is replaceable because the interfaces are boring (HTTP ingest in, exposition format out).

The service itself is ~150 lines of FastAPI: `POST /ingest` receives feature rows from the scoring path into a sliding window (push, not DB-polling — the monitor stays decoupled from the serving stack's storage); `POST /check` recomputes the Evidently report on demand, leaving *cadence* to a scheduler; `/metrics` exposes the results:

```
model_drift_share                        0.6
model_drifted_features                   6
model_feature_drift_score{feature="credit_score"}  0.496
model_feature_drift_score{feature="debt_to_income"} 0.798
monitor_last_check_timestamp             1756702800
```

The per-feature gauges are the triage view. "Drift share 0.6" says something is wrong; "`debt_to_income` at 0.8, `credit_score` at 0.5" tells you it's a macro-credit story and not, say, a broken upstream field — before you've opened a notebook.

## Alerts, including the one people forget

```yaml
- alert: ModelDataDriftHigh        # drift_share > 0.3 for 5m → warn, kick retraining
- alert: ModelDataDriftCritical    # drift_share > 0.6 for 5m → consider fallback rules
- alert: DriftCheckStale           # no check in an hour → the monitor itself is broken
```

Two graduated severities, because the responses differ: at 30% drifted you trigger the continuous-training pipeline and review; at 60% the model's outputs are suspect *now*, and the conversation is whether to fall back to conservative rules while retraining runs.

The third alert is the one that separates monitoring from monitoring theater: **`DriftCheckStale` fires when the drift loop itself stops running.** A silent monitor is strictly worse than no monitor — it manufactures false confidence. Watch the watcher. (`for: 5m` on the drift alerts matters too: one weird traffic batch shouldn't page anyone; sustained drift should.)

## Watching it fail (on purpose)

Reading about drift teaches you little; watching the gauge climb teaches you a lot. The repo ships a simulator that sends stable traffic for five minutes, then ramps in a synthetic downturn — incomes sliding, credit scores compressing, debt ratios inflating:

```bash
docker compose up --build        # monitor + prometheus + grafana, one command
python scripts/simulate_traffic.py --minutes 10
```

On the Grafana "Model Health" board you watch it unfold in order: per-feature distances rise first, `debt_to_income` crosses 0.1, then three more features follow, drift share breaks 30%, the Prometheus alert goes *pending*, holds five minutes, *fires*. That's the incident-response story for a class of incident that throws no exception — and having run the loop end to end, even against simulated traffic, is what turns "I know about drift" into "I've operated drift detection."

## Where I'd take it in production

Prediction-distribution monitoring alongside input drift (a shifting score distribution is another leading indicator); ground-truth backfill jobs to compute *actual* performance when labels arrive; and drift verdicts writing directly to the retraining trigger — which is exactly how the companion [platform repo](https://github.com/KarthikTools/credit-risk-mlops-platform) closes the loop with its nightly CT workflow.

Models decay. That's not a failure of modeling — it's the operating condition. The engineering answer is the same as it's always been in ops: instrument, alert, automate the response, and test the failure path before production tests it for you.

---

*Part of my MLOps portfolio: [end-to-end platform](https://github.com/KarthikTools/credit-risk-mlops-platform) · [Terraform ML infra](https://github.com/KarthikTools/terraform-ml-platform-aws) · [LLMOps evals](https://github.com/KarthikTools/llmops-eval-pipeline). I'm Karthik Marimuthu — [LinkedIn](https://www.linkedin.com/in/karthik-marimuthu-5416091ba/).*
