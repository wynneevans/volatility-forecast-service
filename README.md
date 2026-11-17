# volatility-forecast-service

A toy volatility forecasting service: a LightGBM model that forecasts next-day SPY volatility, served with FastAPI in Docker on a local Kubernetes (kind) cluster.

```
yfinance (SPY OHLC, VIX) ──► features.py ──► train.py ──► artifacts/model.txt
                                                               │ baked into image
             FastAPI (/predict, /healthz, /readyz) in Docker ◄─┘
                                   │
             kind cluster: Deployment (2 replicas, probes, limits) + Service
```

## Model

- **Target:** next-day log Garman-Klass volatility of SPY (open-to-close; ignores the overnight gap).
- **Features (known at day t's close):** HAR terms (today, 5-day mean and 22-day mean log vol) plus log VIX.
- **Validation:** time-ordered split. Train 2005-02 to 2022-07 (4,384 days), test 2022-07 to 2026-11 (1,097 days).
- **Deployed model:** LightGBM refit on all data after evaluation.

| Model | QLIKE ↓ | RMSE (log vol) ↓ |
|---|---|---|
| Naive (tomorrow = today) | 0.612 | 0.470 |
| HAR (linear, 3 features) | 0.395 | 0.393 |
| LightGBM (HAR + VIX) | **0.350** | **0.384** |

QLIKE is the standard volatility loss (Patton, 2011): robust to noise in the volatility proxy and asymmetric, penalising under-forecasts more. LightGBM's larger QLIKE gain suggests it mainly avoids under-forecasting around volatility spikes. HAR coefficients (daily 0.26, weekly 0.43, monthly 0.24) sum to 0.93: volatility is persistent but mean-reverting.

## Serving

`POST /predict` takes the four features and returns the forecast with the model version:

```json
{"log_vol": -4.864, "vol": 0.0077, "model_version": "20261117-133536"}
```

- The model loads once at startup with a warm-up prediction; startup fails if the model's feature list doesn't match the code's.
- `/healthz` backs the liveness probe (restart if failing), `/readyz` the readiness probe (no traffic if failing).
- Multi-stage Docker image: uv builds the venv from `uv.lock`; the runtime image has no build tools and runs as a non-root user.

## Latency

`hey`, 2,000 requests, 20 concurrent, via `kubectl port-forward`, all 200s. With a 500m CPU limit:

| p10 | p50 | p75 | p95 | p99 |
|---|---|---|---|---|
| 9 ms | 15 ms | 80 ms | 86 ms | 90 ms |

The distribution was bimodal (about 10-15 ms and about 80-90 ms). The cause was CPU throttling: the limit allows 50 ms of CPU per 100 ms window, so under load requests stall until the next window. **Removing the CPU limit (keeping the request) removed the slow cluster**, confirming it. The trade-off: without a limit, a busy pod can take CPU from its neighbours, so in a shared cluster you'd size the limit generously or rely on requests instead. Note also that port-forward tunnels through the API server to a single pod, so it adds latency and bypasses Service load balancing.

## Run it

Needs Docker, [kind](https://kind.sigs.k8s.io/), kubectl, [just](https://just.systems/), [uv](https://docs.astral.sh/uv/) and hey. On macOS, LightGBM also needs `brew install libomp`.

```bash
uv sync                                  # install dependencies
just data                                # download and cache SPY + VIX
just train                               # compare models, save artifacts/model.txt
just serve                               # run locally; then `just predict` in another terminal

kind create cluster --name volsvc
just build                               # docker build + load into kind
just deploy                              # apply k8s/ and wait for rollout
just status                              # pods should be 1/1 Running
just forward                             # then `just predict` or `just loadtest`
```

## Limitations and next steps

- **Attribute the gain:** add a linear HAR + VIX baseline to separate the effect of VIX from the effect of trees.
- **Rigour:** walk-forward CV and a Diebold-Mariano test instead of one split; bias-correct the log forecasts before scoring on variance.
- **Latency:** load-test from inside the cluster rather than via port-forward; try multiple uvicorn workers.
- **Deployment:** immutable image tags (e.g. git SHA), one place for the version (Kustomize/Helm), separate training and serving images, model from a registry rather than baked in.
- **Production:** metrics endpoint (Prometheus), HorizontalPodAutoscaler, CI building and pushing images.
