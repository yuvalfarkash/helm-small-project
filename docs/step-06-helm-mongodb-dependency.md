# Step 06 — MongoDB as a Helm dependency

**Goal:** stop maintaining your own MongoDB manifests and consume the community **Bitnami MongoDB** chart as a dependency (subchart).

Start from a copy of the `movie-chart/` you built in Step 05.

---

## 1. Remove the local Mongo templates

Delete these two files — the subchart replaces them:

```
templates/mongo-service.yaml
templates/mongo-statefulset.yaml
```

You can also drop the `movie-chart.mongoName` helper from `_helpers.tpl`, since the Mongo Service name now comes from the subchart (`<release>-mongodb`).

## 2. Declare the dependency in `Chart.yaml`

```yaml
apiVersion: v2
name: movie-chart
description: Movie API (FastAPI) with MongoDB as a Helm dependency
type: application
version: 0.2.0
appVersion: "1.0"

dependencies:
  - name: mongodb
    version: "15.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: mongodb.enabled
```

## 3. Move Mongo settings under the subchart key in `values.yaml`

Anything under the top-level `mongodb:` key is passed straight into the subchart. The `condition` in `Chart.yaml` reads `mongodb.enabled`.

```yaml
replicaCount: 2

image:
  repository: <dockerhub-user>/movie-api
  tag: "1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

config:
  PORT: "3000"

secret:
  # The Bitnami chart creates a service named "<release>-mongodb".
  # We connect as the root user (authSource=admin). The "movies" database is
  # created automatically by MongoDB on the first write.
  mongoUri: "mongodb://root:secretpass@{{ .Release.Name }}-mongodb:27017/movies?authSource=admin"

extraEnv:
  LOG_LEVEL: "info"

# ---- Subchart values (Bitnami MongoDB) ----
mongodb:
  enabled: true
  auth:
    rootPassword: "secretpass"
  architecture: standalone
  persistence:
    enabled: true
    size: 1Gi
```

> The `mongoUri` uses Bitnami's `root` user with `authSource=admin`. Keep the password consistent with `mongodb.auth.rootPassword`. In production, source the password from an existing Secret instead of hardcoding it.
>
> **Do not** add `auth.databases` (or `auth.usernames`) on their own. The Bitnami chart requires `auth.usernames` **and** `auth.databases` to be set *together and with equal length*; setting only one fails `helm lint`/`install` with a values-validation error. We keep it simple here by using only the root credentials — the app creates the `movies` database on first write.

The remaining templates (`configmap.yaml`, `secret.yaml`, `deployment.yaml`, `service.yaml`, `NOTES.txt`) are **unchanged from Step 05**. Optionally update `NOTES.txt` to mention the subchart:

```txt
{{- if .Values.mongodb.enabled }}
MongoDB (Bitnami subchart) service: {{ .Release.Name }}-mongodb
{{- else }}
MongoDB is external. MONGO_URI is taken from the chart secret.
{{- end }}
```

---

## 4. Pull the dependency and deploy

```bash
# Add the Bitnami repo once
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Download the dependency into charts/ and write Chart.lock
helm dependency update ./movie-chart
#   (later, use `helm dependency build` to install exactly what Chart.lock pins)

helm lint ./movie-chart
helm template demo ./movie-chart      # confirm both the API and mongodb render

# Install — brings up the API AND the MongoDB subchart
helm install demo ./movie-chart
kubectl get pods
```

> **Troubleshooting — `helm repo update` / `helm dependency update` times out.**
> The Bitnami index (`https://charts.bitnami.com/bitnami`) is very large and is sometimes slow or rate-limited, so you may see `context deadline exceeded`. If that happens:
> - retry `helm repo update bitnami` (often succeeds on the second try), or
> - pin an exact chart version in `Chart.yaml` (e.g. `version: 15.6.26`) and run `helm dependency build` instead of `update` — `build` fetches just that one chart using `Chart.lock` and skips the full index refresh.

## 5. Verify

```bash
# the API should connect to the subchart's MongoDB
kubectl logs deploy/$(kubectl get deploy -l app.kubernetes.io/instance=demo -o jsonpath='{.items[0].metadata.name}') | grep "connected to"

kubectl port-forward svc/demo-movie-chart 8080:80
curl -X POST http://localhost:8080/movies \
  -H "Content-Type: application/json" \
  -d '{"title":"Arrival","year":2016,"genre":"Sci-Fi"}'
curl http://localhost:8080/movies
```

## 6. Upgrade / rollback / uninstall

```bash
helm upgrade demo ./movie-chart --set replicaCount=3
helm history demo
helm rollback demo 1
helm uninstall demo
```

---

## What you learned

- A chart can depend on other charts via `Chart.yaml` `dependencies`; `helm dependency update` vendors them into `charts/` and pins them in `Chart.lock`.
- Subchart configuration lives under a top-level key named after the subchart (`mongodb:`), and a `condition` lets you toggle the whole dependency on or off.
- You can now ship one chart: flip `mongodb.enabled` off to point at a managed MongoDB, or leave it on for a self-contained deployment.

## Done

You have taken a provided app from local run → container → manual Kubernetes → in-cluster database → a Helm chart → a chart with a managed dependency. 🎉
