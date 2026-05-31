# ETA Model — MLOps Lifecycle

```mermaid
flowchart TD
    A[("Data Lake\n(S3 raw events)")]
    B["Feature Store\n(Feast)"]
    C["Experiment Tracking\n(MLflow)"]
    D["Training Pipeline\n(KubeFlow)"]
    E["Model Evaluation\n(frozen holdout)"]
    F["Registry: Staging\neta v1.5.0"]
    G["Registry: Production\neta v1.5.0"]
    H["Registry: Archived\neta v1.4.x"]
    I["Serving Layer\n(Triton, ~200 RPS)"]
    J["Monitoring\n(Evidently AI)"]

    A -->|"dataset_hash: sha256-a3f1\n(auto)"| B
    B -->|"feature_set: eta-features@v47\n(auto)"| C
    C -->|"run_id: mlf-run-9c2d\nhyperparams.yaml\n(auto)"| D
    D -->|"model.onnx + git_sha: c4a7b12\n(auto)"| E

    E -->|"eval_report.json — gates pass\n(auto)"| F
    E -->|"eval_report.json — gates fail\n→ back to experimentation\n(auto)"| C

    F -->|"canary_result: 1% traffic, 24 h\n(MANUAL: ML Lead + SRE)"| G
    F -->|"rejected — back to staging\n(MANUAL)"| C

    G -->|"model_uri: northstar-models/eta/1.5.0\ndeployed_version: eta-prod-v1.5.0\n(auto)"| I
    G -->|"superseded on next promotion\n(auto)"| H

    I -->|"predictions + p95_latency_ms\n(auto)"| J

    J -->|"no drift detected\n(auto, continuous)"| I
    J -->|"drift_signal: PSI > 0.25 on pickup_hour\nor RMSE regression > 10%\n(auto → triggers retraining)"| A
```

## Transition legend

| Arrow | Trigger | Who/What |
|-------|---------|----------|
| Data Lake → Feature Store | New raw-event batch lands | Airflow DAG (auto) |
| Feature Store → MLflow | Engineer kicks off experiment | Engineer (manual) |
| MLflow → Training Pipeline | Experiment selected for full run | Engineer (manual) |
| Training Pipeline → Evaluation | Pipeline completes | KubeFlow (auto) |
| Evaluation → Staging | All gates green (see registry spec) | CI gate (auto) |
| Evaluation → back to MLflow | Any gate fails | CI gate (auto) |
| Staging → Production | Canary passes + dual sign-off | ML Lead + SRE (manual) |
| Staging → back to MLflow | Canary rejected | ML Lead or SRE (manual) |
| Production → Serving | Deployment pipeline executes | CD pipeline (auto) |
| Production → Archived | Newer version promoted to Production | Registry hook (auto) |
| Monitoring → Data Lake | PSI > 0.25 on any key feature OR RMSE regression > 10% vs. champion | Evidently alert (auto) |
