# Hands-On Kubernetes

Kubernetes manifests for deploying a two-tier CRUD application — a React/Vite frontend and
a Node.js backend talking to MongoDB — onto a local cluster. A practical reference for
Deployments, Services, NodePort exposure, and injecting configuration through Secrets.

---

## Table of Contents

- [What's Here](#whats-here)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Deploying](#deploying)
- [Verifying](#verifying)
- [Configuration](#configuration)
- [Teardown](#teardown)
- [Notes and Gotchas](#notes-and-gotchas)

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

```
  Browser
     │
     ├── :30005 ──► frontend-service ──► frontend pods ×2  (container :5173)
     │                                        │
     │                                        ▼ VITE_API_BASE_URL
     └── :30000 ──► backend-service  ──► backend pods ×2   (container :3000)
                                              │
                                              ▼ MONGO_URI
                                          MongoDB
```

Both tiers are exposed with `NodePort` because the frontend runs **in the browser** — it
needs a backend address reachable from outside the cluster, not an internal DNS name.

| Component | Image | Replicas | Container port | NodePort |
|---|---|---|---|---|
| Frontend | `tiru111/crud-frontend:latest` | 2 | 5173 | **30005** |
| Backend | `tiru111/crud-backend:latest` | 2 | 3000 | **30000** |

---

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

## Notes and Gotchas

- **`imagePullPolicy: IfNotPresent`** on both Deployments means a locally built image is
  used if present. Convenient for local work, but a `:latest` tag will then *not* be
  re-pulled when you push a new one — delete the pods or use a versioned tag.
- **NodePort range** is `30000–32767`. The two static ports here sit at the bottom of it.
- **`VITE_API_BASE_URL` is a build-time variable in Vite.** Supplying it as a runtime env
  var only works if the frontend container reads it at startup; a pre-built static bundle
  will have baked in whatever value was present at `npm run build`. Verify this if the
  frontend cannot reach the API.
- **The committed Secrets contain a hard-coded host IP.** Replace it with your own before
  deploying, and prefer a real secret manager — or at minimum an untracked local override —
  over committing connection strings, since `stringData` is plain text in git.
