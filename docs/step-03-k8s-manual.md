# Step 03 — Deploy to Kubernetes (manual manifests)

**Goal:** deploy your image to Kubernetes by hand, with configuration in a ConfigMap and the database URI in a Secret. MongoDB stays **outside** the cluster for now.

You write these manifests yourself. Put them in a `k8s/` folder (anywhere convenient, e.g. `step-03/k8s/`).

---

## 1. ConfigMap — non-secret config

`configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: movie-api-config
data:
  PORT: "3000"
```

## 2. Secret — the database URI

The `MONGO_URI` can contain credentials, so it belongs in a Secret. `stringData` lets you write plain text; Kubernetes base64-encodes it for you.

`secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: movie-api-secret
type: Opaque
stringData:
  # Point this at a Mongo reachable from the cluster.
  # On Docker Desktop / minikube you can reach your host via host.docker.internal.
  MONGO_URI: "mongodb://host.docker.internal:27017/movies"
```

> Make sure a MongoDB is reachable at that address — e.g. the `docker run -d --name mongo -p 27017:27017 mongo:7` container from Step 01, running on your host.

## 3. Deployment — the app

Note `envFrom`: it injects every key from the ConfigMap and the Secret as environment variables. The probes hit `/health`.

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: movie-api
  labels:
    app: movie-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: movie-api
  template:
    metadata:
      labels:
        app: movie-api
    spec:
      containers:
        - name: movie-api
          image: <dockerhub-user>/movie-api:1.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: movie-api-config
            - secretRef:
                name: movie-api-secret
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 5
```

## 4. Service — stable access

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: movie-api
spec:
  type: ClusterIP
  selector:
    app: movie-api
  ports:
    - port: 80
      targetPort: 3000
```

---

## 5. Apply and verify

```bash
kubectl apply -f k8s/

kubectl get pods -l app=movie-api
kubectl logs deploy/movie-api          # should show "[db] connected to ..."

# reach the service from your machine
kubectl port-forward svc/movie-api 8080:80
curl http://localhost:8080/health
```

---

## What you learned

- Config and secrets are externalized from the image: the same image runs anywhere, configured by ConfigMap + Secret via `envFrom`.
- A Service gives the Deployment a stable name and port inside the cluster.

## Next

→ [Step 04 — MongoDB on Kubernetes](step-04-k8s-mongodb.md)
