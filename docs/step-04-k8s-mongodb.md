# Step 04 — MongoDB on Kubernetes

**Goal:** move MongoDB *inside* the cluster with persistent storage, and repoint the app at it. You add two new manifests and change one line in the Secret.

Keep working in your `k8s/` folder from Step 03.

---

## 1. Headless Service for MongoDB

A headless Service (`clusterIP: None`) gives the StatefulSet pod a stable DNS name (`mongo-0.mongo`) and lets other pods reach it as `mongo`.

`mongo-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    app: mongo
spec:
  clusterIP: None
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```

## 2. StatefulSet for MongoDB (with persistent storage)

`volumeClaimTemplates` creates a PersistentVolumeClaim so the data survives pod restarts.

`mongo-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:7
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

## 3. Repoint the app's Secret at the in-cluster Mongo

Change `MONGO_URI` in `secret.yaml` to use the `mongo` Service name as the host:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: movie-api-secret
type: Opaque
stringData:
  # in-cluster Mongo: <service>:<port>. The headless service is "mongo".
  MONGO_URI: "mongodb://mongo:27017/movies"
```

> `configmap.yaml`, `deployment.yaml`, and `service.yaml` are unchanged from Step 03.

---

## 4. Apply in the right order

Bring up MongoDB first, then the app, so the app finds the database on startup.

```bash
# 1) MongoDB
kubectl apply -f k8s/mongo-service.yaml
kubectl apply -f k8s/mongo-statefulset.yaml
kubectl rollout status statefulset/mongo

# 2) the app (and its config/secret)
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

---

## 5. Verify

```bash
kubectl logs deploy/movie-api | grep "connected to"
# [db] connected to mongodb://mongo:27017/movies

kubectl port-forward svc/movie-api 8080:80
curl -X POST http://localhost:8080/movies \
  -H "Content-Type: application/json" \
  -d '{"title":"Dune","year":2021,"genre":"Sci-Fi"}'
curl http://localhost:8080/movies
```

---

## What you learned

- StatefulSet + headless Service + PVC is the standard pattern for stateful workloads like databases on Kubernetes.
- The app didn't change at all — only the `MONGO_URI` in the Secret. Same image, new database target.

## Next

→ [Step 05 — Package as a Helm chart](step-05-helm-basic.md)
