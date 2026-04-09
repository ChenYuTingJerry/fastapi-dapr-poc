# fastapi-dapr-poc

> **Status: frozen proof of concept.** Built to evaluate Dapr + FastAPI for team adoption. Not production-ready and no longer actively developed.

## Why this exists

I wanted to find out whether [Dapr](https://dapr.io) is a credible foundation for a team that already writes microservices in FastAPI. Specifically: do Dapr's building blocks (service invocation, pub/sub, workflow) actually let me write services that don't know about each other's hosts, ports, or transport, *and* do they hold up when the same code moves from a laptop to Kubernetes? This repo is the smallest end-to-end thing I could build that exercises all three building blocks at once.

## What it shows

Four FastAPI services collaborating on a fake "place an order" flow. Every cross-service hop goes through a Dapr sidecar — there is no direct HTTP between services in the application code.

```
   POST /create
        │
        ▼
  ┌──────────────┐  invoke_method   ┌────────────────┐
  │ order-service│ ───────────────▶ │ payment-service│
  └──────────────┘                  └───────┬────────┘
                                            │ publish_event
                                            ▼
                                  ┌───────────────────┐
                                  │  pub/sub: pubsub  │
                                  │ topic: payment.   │
                                  │       success     │
                                  └─────────┬─────────┘
                                            │ subscribe
                                            ▼
                                  ┌───────────────────┐
                                  │ subscriber-service│
                                  └─────────┬─────────┘
                                            │ invoke_method
                                            ▼
                                  ┌───────────────────┐
                                  │  workflow-service │
                                  │  schedules wf:    │
                                  │  deliver_product  │
                                  │   → notify_user   │
                                  └───────────────────┘
```

## The thing I'm proud of

The same service code runs unchanged locally (`dapr run -f dapr.yaml`) and on a k3d cluster. Dapr abstracts the Redis host, the pub/sub topic plumbing, and service discovery — none of that leaks into the Python.

There is exactly one env-specific seam in the whole repo, and it lives in the makefile, not in the application code:

```make
# components/pubsub.yaml + components/statestore.yaml point at localhost:6379
# for local dev. For k3d we rewrite that to the in-cluster Redis service.
sed 's/value: "localhost:6379"/value: "redis-master.default.svc.cluster.local:6379"/' \
    components/pubsub.yaml > components/pubsub-k8s.yaml
```

That's it. Nothing else changes between environments.

## Quickstart — local

Prereqs: Python ≥ 3.11, [Poetry](https://python-poetry.org), [Dapr CLI](https://docs.dapr.io/getting-started/install-dapr-cli/), Docker, [HTTPie](https://httpie.io) (for the smoke test).

```bash
make install      # poetry install
make run-all      # dapr init + dapr run -f dapr.yaml (all 4 services + sidecars)
make test         # http POST http://0.0.0.0:8001/create
```

You should see the order service log a `/create`, the payment service log a `/pay` and publish a `payment.success` event, the subscriber service log `📨 收到 PubSub 訊息`, and finally the workflow service log `🧭 Workflow started ...` followed by `📦 Delivering product ...` and `📬 Notifying user ...`.

Tear down with `make stop`.

## Quickstart — k3d

Extra prereqs: [k3d](https://k3d.io), `kubectl`, [Helm](https://helm.sh).

```bash
make run-k3d      # one command, ~a few minutes the first time
make test-k3d     # http POST http://localhost:8081/create
```

`make run-k3d` does the following in order:

1. Creates (or starts) a k3d cluster named `dapr-cluster` with `--port 8081:80@loadbalancer`.
2. Installs Dapr into the cluster (`dapr init -k`).
3. Installs Redis via the bitnami Helm chart into `default`.
4. Rewrites the component YAML to point at in-cluster Redis and applies it.
5. Builds images with `docker compose build` and pushes them into the cluster with `k3d image import` (deployments use `imagePullPolicy: Never`, so the import step is required — if you see `ErrImageNeverPull`, that step didn't run).
6. Applies the manifests under `k8s/`.

Clean up with `make k3d-delete-cluster`.

## Architecture notes

| Service              | Local port | Dapr HTTP port | Source                          |
|----------------------|-----------:|---------------:|---------------------------------|
| `order-service`      |       8001 |           3500 | `apps/order_service/app`        |
| `payment-service`    |       8002 |           3501 | `apps/payment_service/app`      |
| `subscriber-service` |       8003 |           3502 | `apps/subscriber_service/app`   |
| `workflow-service`   |       8004 |           3503 | `apps/workflow_service/app`     |

A few things worth knowing if you read the code:

- **All inter-service traffic goes through `DaprClient`.** Services address each other by `appID` + method name (`client.invoke_method("payment-service", "pay", ...)`), never by host/port. Pub/sub events are published via `client.publish_event(pubsub_name="pubsub", topic_name="payment.success", ...)` and consumed declaratively with `@dapr_app.subscribe(pubsub="pubsub", topic="payment.success")`.
- **Dapr components live in `components/`.** Two of them: `pubsub.yaml` (Redis Streams) and `statestore.yaml` (Redis, also used as the workflow backend). The makefile's `deploy-components` target rewrites the Redis host before applying them to k3d.
- **The workflow itself lives in `apps/workflow_service/app/workflow.py`.** It uses `dapr.ext.workflow.WorkflowRuntime` with two activities (`deliver_product`, `notify_user`) chained via `ctx.call_activity`. The HTTP entry point in `routes.py` schedules an instance via `DaprWorkflowClient.schedule_new_workflow`.
- **`dapr.yaml` is the canonical local config.** The individual `make run-<service>` targets exist but have drifted from `dapr.yaml` in a few places (ports, `--resources-path`); use `make run-all` for anything end-to-end.

## What's NOT here

This is a PoC. It does not have, and was never meant to have:

- **No tests.** `make test` and `make test-k3d` are HTTP smoke calls, not a test suite.
- **No auth.** Endpoints are open.
- **No tracing or metrics wiring.** Dapr ships with OpenTelemetry support; this repo doesn't configure it.
- **Minimal secrets handling.** The Redis password is referenced via `secretKeyRef` in the component YAML, but there is no real secret management story.
- **No retry / DLQ tuning** on the pub/sub component.
- **No CI.** No GitHub Actions, no linting gate, no image publishing.
- **`delivery-service` is referenced in old makefile/README leftovers but does not exist in `apps/`.** It's vestigial and should be ignored.

If you wanted to take this somewhere real, those are the gaps to close.

## Repo layout

```
apps/
  order_service/        # POST /create  → invokes payment-service
  payment_service/      # POST /pay     → publishes payment.success
  subscriber_service/   # subscribes payment.success → invokes workflow-service
  workflow_service/     # schedules order_fulfillment_workflow
components/             # Dapr pubsub + statestore (Redis)
k8s/                    # Deployment + Service + Ingress per service
dapr.yaml               # canonical local multi-app run config
docker-compose.yaml     # used by `make run-k3d` to build images
makefile                # all the entry points (run-all, run-k3d, test, ...)
```

## License / contact

No license file. Author: Jerry Chen — `nmjk2000@gmail.com`. Open a GitHub issue if anything in this README is wrong or unclear.
