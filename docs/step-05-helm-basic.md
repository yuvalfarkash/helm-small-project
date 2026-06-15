# Step 05 — Package as a Helm chart

**Goal:** turn the raw manifests from Step 04 into a parameterized, reusable **Helm chart**. This is the core of the lab — you will use values, helper templates, and conditionals, then practice install / upgrade / rollback / uninstall.

Create a chart folder `movie-chart/` with this layout:

```
movie-chart/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
└── templates/
    ├── _helpers.tpl
    ├── configmap.yaml
    ├── secret.yaml
    ├── deployment.yaml
    ├── service.yaml
    ├── mongo-service.yaml
    ├── mongo-statefulset.yaml
    └── NOTES.txt
```

> Tip: `helm create movie-chart` scaffolds a chart for you. Delete the generated `templates/*` (keep `_helpers.tpl` as a starting point) and replace them with the files below.

---

## 1. `Chart.yaml`

```yaml
apiVersion: v2
name: movie-chart
description: Movie API (FastAPI) with an in-cluster MongoDB
type: application
version: 0.1.0          # chart version
appVersion: "1.0"       # app (image) version
```

## 2. `values.yaml` — the knobs

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

# Application config (becomes a ConfigMap)
config:
  PORT: "3000"

# Sensitive config (becomes a Secret)
secret:
  mongoUri: "mongodb://{{ .Release.Name }}-mongo:27017/movies"

# In-cluster MongoDB. Toggle off to use an external Mongo via secret.mongoUri.
mongo:
  enabled: true
  image: mongo:7
  storage: 1Gi
  port: 27017

# Extra environment variables, rendered with range
extraEnv:
  LOG_LEVEL: "info"
```

## 3. `values-prod.yaml` — an environment override

Applied on top of `values.yaml` with `-f`:

```yaml
replicaCount: 4

image:
  tag: "1.0"

service:
  type: LoadBalancer

extraEnv:
  LOG_LEVEL: "warn"
```

## 4. `templates/_helpers.tpl` — named templates

These keep names and labels consistent across every resource.

```yaml
{{/* Base name, overridable */}}
{{- define "movie-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/* Fully qualified app name: <release>-<chart> */}}
{{- define "movie-chart.fullname" -}}
{{- printf "%s-%s" .Release.Name (include "movie-chart.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/* Common labels applied to every object */}}
{{- define "movie-chart.labels" -}}
app.kubernetes.io/name: {{ include "movie-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{- end -}}

{{/* Selector labels (stable subset — never include version here) */}}
{{- define "movie-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "movie-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{/* Mongo service name, derived from the release */}}
{{- define "movie-chart.mongoName" -}}
{{- printf "%s-mongo" .Release.Name -}}
{{- end -}}
```

## 5. `templates/configmap.yaml`

`range` turns every key under `.Values.config` into a ConfigMap entry.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "movie-chart.fullname" . }}-config
  labels:
    {{- include "movie-chart.labels" . | nindent 4 }}
data:
  {{- range $key, $val := .Values.config }}
  {{ $key }}: {{ $val | quote }}
  {{- end }}
```

## 6. `templates/secret.yaml`

`tpl` re-renders the `mongoUri` string so the `{{ .Release.Name }}` placeholder inside `values.yaml` resolves.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "movie-chart.fullname" . }}-secret
  labels:
    {{- include "movie-chart.labels" . | nindent 4 }}
type: Opaque
stringData:
  MONGO_URI: {{ tpl .Values.secret.mongoUri . | quote }}
```

## 7. `templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "movie-chart.fullname" . }}
  labels:
    {{- include "movie-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "movie-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "movie-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: movie-api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          envFrom:
            - configMapRef:
                name: {{ include "movie-chart.fullname" . }}-config
            - secretRef:
                name: {{ include "movie-chart.fullname" . }}-secret
          {{- with .Values.extraEnv }}
          env:
            {{- range $key, $val := . }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
          {{- end }}
          livenessProbe:
            httpGet:
              path: /health
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 3
            periodSeconds: 5
```

## 8. `templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "movie-chart.fullname" . }}
  labels:
    {{- include "movie-chart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "movie-chart.selectorLabels" . | nindent 4 }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
```

## 9. `templates/mongo-service.yaml` (conditional)

The whole MongoDB stack only renders when `mongo.enabled` is true.

```yaml
{{- if .Values.mongo.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "movie-chart.mongoName" . }}
  labels:
    {{- include "movie-chart.labels" . | nindent 4 }}
    app.kubernetes.io/component: mongo
spec:
  clusterIP: None
  selector:
    {{- include "movie-chart.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: mongo
  ports:
    - port: {{ .Values.mongo.port }}
      targetPort: 27017
{{- end }}
```

## 10. `templates/mongo-statefulset.yaml` (conditional)

```yaml
{{- if .Values.mongo.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "movie-chart.mongoName" . }}
  labels:
    {{- include "movie-chart.labels" . | nindent 4 }}
    app.kubernetes.io/component: mongo
spec:
  serviceName: {{ include "movie-chart.mongoName" . }}
  replicas: 1
  selector:
    matchLabels:
      {{- include "movie-chart.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: mongo
  template:
    metadata:
      labels:
        {{- include "movie-chart.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: mongo
    spec:
      containers:
        - name: mongo
          image: {{ .Values.mongo.image }}
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
            storage: {{ .Values.mongo.storage }}
{{- end }}
```

## 11. `templates/NOTES.txt`

Printed after `helm install` / `helm upgrade`.

```txt
Movie API has been deployed as release "{{ .Release.Name }}".

Get the application URL:
  kubectl port-forward svc/{{ include "movie-chart.fullname" . }} 8080:{{ .Values.service.port }}
  curl http://localhost:8080/health
  open  http://localhost:8080/docs

{{- if .Values.mongo.enabled }}
In-cluster MongoDB is enabled (service: {{ include "movie-chart.mongoName" . }}).
{{- else }}
MongoDB is external. MONGO_URI is taken from the chart secret.
{{- end }}
```

---

## 12. The Helm workflow

```bash
# Validate the chart
helm lint ./movie-chart

# Render templates locally without installing (great for debugging)
helm template demo ./movie-chart

# Install
helm install demo ./movie-chart
helm get manifest demo
kubectl get pods

# Upgrade with an inline override
helm upgrade demo ./movie-chart --set replicaCount=3

# Upgrade with a production values file
helm upgrade demo ./movie-chart -f ./movie-chart/values-prod.yaml

# Release history + rollback
helm history demo
helm rollback demo 1

# Remove everything
helm uninstall demo
```

---

## What you learned

- **values.yaml** parameterizes a chart; `--set` and `-f` override per environment.
- **_helpers.tpl** named templates (`include`) keep names/labels DRY and consistent.
- **Conditionals** (`{{- if .Values.mongo.enabled }}`) and **range/with** make templates flexible.
- `helm upgrade` / `rollback` / `history` give you versioned, reversible releases.

## Next

→ [Step 06 — MongoDB as a Helm dependency](step-06-helm-mongodb-dependency.md)
