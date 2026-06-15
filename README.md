# Helm Master Lab — Movie API on Docker → Kubernetes → Helm

A hands-on lab. You are **given** a small Movie API (Python / FastAPI + MongoDB) and you take it through six incremental steps: run it locally → containerize it → deploy it to Kubernetes by hand → add MongoDB in the cluster → package it as a Helm chart → consume MongoDB as a Helm dependency.

**Audience:** you already know basic Docker and Kubernetes. The depth is in **Helm**.

---

## The provided application

You do **not** write the application — it is provided in [`movie-api/`](movie-api/). You only run, containerize, and deploy it.

It is a tiny REST API (FastAPI + the Motor async MongoDB driver):

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Liveness/readiness probe — returns `{ "status": "ok" }` |
| `GET` | `/movies` | List all movies |
| `POST` | `/movies` | Create a movie — body `{ "title", "year", "genre" }` |
| `DELETE` | `/movies/{id}` | Delete a movie by id |
| `GET` | `/docs` | Interactive Swagger UI (provided by FastAPI) |

It is configured **only** through environment variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `3000` | HTTP port |
| `MONGO_URI` | `mongodb://localhost:27017/movies` | MongoDB connection string |

```
movie-api/
├── requirements.txt
└── app/
    ├── __init__.py
    ├── main.py          # FastAPI app + /health + router wiring
    ├── db.py            # Motor connection (reads MONGO_URI)
    ├── models.py        # Pydantic models + serializer
    └── routes/
        ├── __init__.py
        └── movies.py    # the /movies router
```

---

## Prerequisites

| Tool | Version | Check |
|------|---------|-------|
| Python | 3.11+ | `python --version` |
| Docker | recent | `docker version` |
| kubectl | 1.28+ | `kubectl version --client` |
| Local cluster | kind / minikube / Docker Desktop | `kubectl get nodes` |
| Helm | 3.12+ | `helm version` |
| Docker Hub account | — | used to push your image (replace `<dockerhub-user>` everywhere) |

---

## Steps

Do them in order. Each step builds on the previous one and adds a single new layer.

1. [Step 01 — Run the provided app](docs/step-01-run-app.md)
2. [Step 02 — Containerize with Docker](docs/step-02-docker.md)
3. [Step 03 — Deploy to Kubernetes (manual manifests)](docs/step-03-k8s-manual.md)
4. [Step 04 — MongoDB on Kubernetes](docs/step-04-k8s-mongodb.md)
5. [Step 05 — Package as a Helm chart](docs/step-05-helm-basic.md)
6. [Step 06 — MongoDB as a Helm dependency](docs/step-06-helm-mongodb-dependency.md)

| Step | What you add | Key concept |
|------|--------------|-------------|
| 01 | Run app + Mongo container | env-based config (`MONGO_URI`) |
| 02 | Dockerfile + image | build, push, service-name networking |
| 03 | k8s manifests | Deployment/Service/ConfigMap/Secret, probes |
| 04 | Mongo on k8s | StatefulSet + PVC + headless service |
| 05 | Helm chart | values, helpers, conditionals, upgrade/rollback |
| 06 | Helm dependency | subcharts, Bitnami MongoDB |
