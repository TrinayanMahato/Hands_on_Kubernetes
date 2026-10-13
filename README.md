# Hands-On Kubernetes

Kubernetes manifests for deploying a two-tier CRUD application — a React/Vite frontend and
a Node.js backend talking to MongoDB — onto a local cluster. A practical reference for
Deployments, Services, NodePort exposure, and injecting configuration through Secrets.

---

## Table of Contents

- [What's Here](#whats-here)
- [Architecture](#architecture)
- [Operational Notes](#operational-notes)
- [Prerequisites](#prerequisites)
- [Deploying](#deploying)
- [Verifying](#verifying)
- [Configuration](#configuration)
- [Teardown](#teardown)

---

## What's Here

```
├── deployment_backend/
│   └── backend.yaml      # Backend Deployment (2 replicas) + NodePort Service
├── deployment_frontend/
│   └── frontend.yaml     # Frontend Deployment (2 replicas) + NodePort Service
└── secrets/
    ├── backend.yaml      # backend-secrets  — PORT, MONGO_URI
    └── frontend.yaml     # frontend-secrets — VITE_API_BASE_URL
```

Each deployment manifest contains both the `Deployment` and its `Service`, separated by a
`---` document break.

---

## Architecture

Two independently scalable tiers, each exposed on its own NodePort, with configuration
injected from Secrets rather than baked into the images.

```
                         ┌──────────┐
                         │ Browser  │
                         └────┬─────┘
              :30005          │          :30000
        ┌───────────────────┬─┴─┬───────────────────┐
        ▼                   │   │                   ▼
┌───────────────────┐       │   │       ┌───────────────────┐
│ frontend-service  │       │   │       │  backend-service  │
│ NodePort 80→5173  │       │   │       │ NodePort 3000     │
└─────────┬─────────┘       │   │       └─────────┬─────────┘
          │ selector app=frontend     selector app=backend │
          ▼                                                ▼
┌───────────────────┐                   ┌───────────────────┐
│ frontend-deploy   │                   │ backend-deploy    │
│ replicas: 2       │                   │ replicas: 2       │
│ :5173             │                   │ :3000             │
└─────────┬─────────┘                   └─────────┬─────────┘
          │ envFrom                               │ envFrom
          ▼                                       ▼
┌───────────────────┐                   ┌───────────────────┐
│ frontend-secrets  │                   │ backend-secrets   │
│ VITE_API_BASE_URL │                   │ PORT, MONGO_URI   │
└───────────────────┘                   └─────────┬─────────┘
                                                  ▼
                                              MongoDB
```

### Object model

Each manifest file carries two objects separated by `---`: a **Deployment** that owns pods,
and a **Service** that gives them a stable address.

| Object | Purpose | Binds via |
| :--- | :--- | :--- |
| `Deployment` | Declares desired replica count and the pod template | `matchLabels` |
| `Service` | Stable virtual IP + NodePort in front of matching pods | `selector` |
| `Secret` | Key/value configuration consumed as environment variables | `envFrom.secretRef` |

The coupling is entirely by label. `backend-service` finds its pods because its `selector`
is `app: backend` and the pod template carries that label — the Service never references
the Deployment by name.

### Why both tiers use NodePort

An internal `ClusterIP` would be enough for a server-to-server call, but the frontend here
runs **in the browser**. When that JavaScript calls the API, the request originates from
the user's machine, not from inside the cluster, so cluster-internal DNS such as
`backend-service:3000` cannot resolve. Both tiers therefore need an address reachable from
outside, which is what `VITE_API_BASE_URL` points at.

### Configuration flow

Secrets use `stringData`, so values are written in plain text and Kubernetes base64-encodes
them on admission. `envFrom.secretRef` then projects **every** key in the Secret into the
container as an environment variable, which is why the key names in the Secret match the
variable names the application reads.

The ordering constraint follows from this: a Deployment referencing a Secret that does not
exist leaves its pods in `CreateContainerConfigError`, so Secrets are applied first.

## Operational Notes

- **Secrets do not hot-reload.** Values projected through `envFrom` are read once at
  container start. After editing a Secret, restart the Deployment so pods pick it up:
  `kubectl rollout restart deployment/backend-deployment`.
- **`imagePullPolicy: IfNotPresent`** uses a locally built image when one is present, which
  is convenient locally but means a `:latest` tag is not re-pulled when you push a new one.
  Delete the pods or move to versioned tags when that matters.
- **NodePort range** is `30000–32767`; the two static ports here sit at the bottom of it.
- **`VITE_API_BASE_URL` is read at build time by Vite.** Supplying it at runtime only takes
  effect if the container builds on start — a pre-built static bundle carries whatever
  value was present during `npm run build`.

## Prerequisites

- A running Kubernetes cluster — Minikube, kind, Docker Desktop, or k3s
- `kubectl` configured against that cluster
- A reachable MongoDB instance

---

## Deploying

Apply the Secrets **first** — the Deployments reference them via `envFrom`, and pods will
fail to start if the Secrets do not exist yet.

```bash
kubectl apply -f secrets/backend.yaml
kubectl apply -f secrets/frontend.yaml

kubectl apply -f deployment_backend/backend.yaml
kubectl apply -f deployment_frontend/frontend.yaml
```

---

## Verifying

```bash
kubectl get pods -o wide
kubectl get svc
kubectl logs -l app=backend --tail=50
```

Then open the app:

```bash
# Docker Desktop / k3s
open http://localhost:30005

# Minikube
minikube service frontend-service
```

If a pod is stuck in `CreateContainerConfigError`, the Secret it needs is usually missing:

```bash
kubectl describe pod -l app=backend
```

---

## Configuration

Both Secrets use `stringData` rather than `data`, which means **values are written in
plain text and Kubernetes base64-encodes them on admission** — convenient while learning,
since there is nothing to encode by hand.

| Secret | Key | Consumed by |
|---|---|---|
| `backend-secrets` | `PORT` | Backend listen port |
| `backend-secrets` | `MONGO_URI` | MongoDB connection string |
| `frontend-secrets` | `VITE_API_BASE_URL` | Backend API base URL baked into the frontend |

To point the app at your own MongoDB, edit `secrets/backend.yaml` and re-apply, then
restart the backend so it picks up the change:

```bash
kubectl apply -f secrets/backend.yaml
kubectl rollout restart deployment/backend-deployment
```

> **Note:** a `Secret` change is *not* picked up automatically by running pods when it is
> consumed through `envFrom`. The rollout restart above is required.

---

## Teardown

```bash
kubectl delete -f deployment_frontend/frontend.yaml
kubectl delete -f deployment_backend/backend.yaml
kubectl delete -f secrets/frontend.yaml
kubectl delete -f secrets/backend.yaml
```

---
