# Step 05 — Package as a Helm chart

**Goal:** turn the raw manifests from Step 04 into a parameterized, reusable **Helm chart**. This is the core of the lab — you will write `values.yaml`, helper templates, and conditionals yourself, then practice install / upgrade / rollback / uninstall.

There is **no copy-paste chart here** — you build it from the requirements and hints. (Reference answer: `solved/step-05/movie-chart/`.)

> **Set up:** `helm create movie-chart` scaffolds a chart. Delete the generated files under `templates/` (keep `_helpers.tpl` as a starting point and the `Chart.yaml`/`values.yaml` to edit). You will end up with this layout:
>
> ```
> movie-chart/
> ├── Chart.yaml
> ├── values.yaml
> ├── values-prod.yaml
> └── templates/
>     ├── _helpers.tpl
>     ├── configmap.yaml
>     ├── secret.yaml
>     ├── deployment.yaml
>     ├── service.yaml
>     ├── mongo-service.yaml
>     ├── mongo-statefulset.yaml
>     └── NOTES.txt
> ```

---

## A. `Chart.yaml`

**Task:** set chart metadata.

*Hints:* `apiVersion: v2`, `name: movie-chart`, `type: application`, a `version:` (the chart version, e.g. `0.1.0`) and an `appVersion:` (the image version, e.g. `"1.0"`). Know the difference: bump `version` when the chart changes, `appVersion` when the app image changes.

## B. `values.yaml` — the knobs

**Goal:** make every setting that varied between environments a *value* you can override later, instead of hard-coding it in a template.

**What to define** — recreate this structure in `values.yaml`:

| Key | Purpose | Example |
| --- | --- | --- |
| `replicaCount` | how many app pods | `2` |
| `image.repository` | image name | `<dockerhub-user>/movie-api` |
| `image.tag` | image version | `"1.0"` |
| `image.pullPolicy` | when to pull | `IfNotPresent` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `service.targetPort` | container port | `3000` |
| `config.PORT` | app config → ConfigMap | `"3000"` |
| `secret.mongoUri` | DB connection → Secret | see hint below |
| `mongo.enabled` | toggle in-cluster Mongo | `true` |
| `mongo.image` | Mongo image | `mongo:7` |
| `mongo.storage` | PVC size | `1Gi` |
| `mongo.port` | Mongo Service port | `27017` |
| `extraEnv.LOG_LEVEL` | extra env vars → `env:` | `"info"` |

**Two things need special handling:**
- **`secret.mongoUri` should contain `{{ .Release.Name }}` *inside the string*** — e.g. a value like `"mongodb://{{ .Release.Name }}-mongo:27017/movies"`. This makes the Mongo host follow the release name (`demo-mongo`, `prod-mongo`, …). A normal `{{ .Values.secret.mongoUri }}` would print that template text literally, so in **step F** you re-render it with `tpl`.
- **`mongo.enabled` is a switch.** In **steps I and J** it gates the entire in-cluster Mongo (Service + StatefulSet) behind an `{{- if }}` so the same chart works with either an in-cluster or an external database.

*Reference answer: `solved/step-05/movie-chart/values.yaml`.*

## C. `values-prod.yaml` — an environment override

**Goal:** a *partial* values file holding **only** what changes in production. You apply it on top of `values.yaml` with `-f` (step L); unspecified keys keep their base defaults because Helm deep-merges this file over `values.yaml`.

**What to override** — just the production-specific keys, for example:
- `replicaCount` → bump it (e.g. `4`)
- `service.type` → `LoadBalancer`
- `extraEnv.LOG_LEVEL` → `warn`

Leave everything else out — do not re-list keys that don't change.

*Reference answer: `solved/step-05/movie-chart/values-prod.yaml`.*

## D. `templates/_helpers.tpl` — named templates

**Goal:** define reusable name/label snippets once, so every resource is named and labelled consistently. You call them from other templates with `include`.

This is your first time writing Helm template *functions*, so before the tasks, here are the building blocks you'll use.

### Concepts you'll need first

- **`{{- define "movie-chart.name" -}}` … `{{- end -}}`** declares a named template (think: a small function). It doesn't output anything on its own — something has to *call* it.
- **`{{ include "movie-chart.name" . }}`** calls a named template. The trailing **`.`** is the argument: it's the current context (the root object that holds `.Values`, `.Release`, `.Chart`, …). **You must pass `.`** — without it the called template can't see any values.
- **The dashes** in `{{-` and `-}}` trim surrounding whitespace/newlines so the rendered YAML doesn't get blank lines. Use them on the `define`/`end` lines.
- **`|` (pipe)** sends the value on the left as the *last argument* to the function on the right. So `printf "..." | trunc 63 | trimSuffix "-"` builds a string, then trims it, then cleans it — left to right.

Functions you'll call (all built into Helm):

| Function | What it does | Why you need it |
| --- | --- | --- |
| `default A B` | returns `B` if it's set, otherwise falls back to `A` | let a user override the name, but default to the chart name |
| `printf "%s-%s" x y` | string formatting, like in C/Go | glue `<release>-<name>` together |
| `trunc 63 X` | cut a string to 63 chars | k8s names have a 63-char limit |
| `trimSuffix "-" X` | drop a trailing `-` if present | truncating at 63 might leave a dangling `-`, which is illegal |

### The five templates to define

#### 1. `movie-chart.name` — the base name
- **Returns:** the chart's name, but allow a user override via `.Values.nameOverride`.
- **How:** `default .Chart.Name .Values.nameOverride`, then pipe through `trunc 63 | trimSuffix "-"`.
- **Why:** every label and other helper builds on this one name, so it lives in one place.

#### 2. `movie-chart.fullname` — the full resource name
- **Returns:** `<release>-<name>`, e.g. `demo-movie-chart`.
- **How:** `printf "%s-%s" .Release.Name (include "movie-chart.name" .)`, then `trunc 63 | trimSuffix "-"`. Note you **call helper #1 from inside #2** with `include "movie-chart.name" .` wrapped in parentheses.
- **Why:** resources need release-specific names so two installs of the chart don't collide.

#### 3. `movie-chart.labels` — the full label set
- **Returns:** three lines of YAML labels:
  - `app:` → `{{ include "movie-chart.name" . }}`
  - `release:` → `{{ .Release.Name }}`
  - `chart:` → `{{ printf "%s-%s" .Chart.Name .Chart.Version }}`
- **Why:** putting the same labels on every object makes the release's resources easy to find and group together.

#### 4. `movie-chart.selectorLabels` — the stable subset
- **Returns:** only the **first two** labels from #3 — `app` and `release`. **Leave out `chart`.**
- **Why (important):** a Deployment's/StatefulSet's `selector` is **immutable** — you can't change it after creation. The `chart` label contains the chart *version*, which changes on every chart bump. If the version were in the selector, your next `helm upgrade` would try to change an immutable field and **fail**. So selectors get only the labels that never change.

#### 5. `movie-chart.mongoName` — the Mongo name
- **Returns:** `<release>-mongo`, e.g. `demo-mongo`.
- **How:** `printf "%s-mongo" .Release.Name`.
- **Why:** the Mongo Service name must match the host inside `secret.mongoUri` (step B), which is also built from the release name — so they always line up.

> **How you'll use these later:** in the other templates you call a helper for a name like `name: {{ include "movie-chart.fullname" . }}`, and for a whole label block like:
> ```yaml
> labels:
>   {{- include "movie-chart.labels" . | nindent 4 }}
> ```
> `nindent 4` re-indents the helper's output by 4 spaces so it nests correctly under `labels:`. (Plain `indent` doesn't add the leading newline; `nindent` does — that's why label blocks use `nindent`.)

*Reference answer: `solved/step-05/movie-chart/templates/_helpers.tpl`.*

---

> **Steps E–K — don't write these from scratch.** You already wrote these exact manifests in your `k8s/` folder back in Steps 03–04. The job here is to **copy each file in and *templatize* it** — replace the hard-coded names, labels, and values with `{{ ... }}` expressions.
>
> For each step below:
> 1. **Copy** the matching file from your `k8s/` folder into `movie-chart/templates/`.
> 2. **Adapt** it by making the changes listed — swap hard-coded values for the helpers and `.Values.*` references.
>
> | This template | Copy from |
> | --- | --- |
> | `configmap.yaml` | `k8s/configmap.yaml` |
> | `secret.yaml` | `k8s/secret.yaml` |
> | `deployment.yaml` | `k8s/deployment.yaml` |
> | `service.yaml` | `k8s/service.yaml` |
> | `mongo-service.yaml` | `k8s/mongo-service.yaml` |
> | `mongo-statefulset.yaml` | `k8s/mongo-statefulset.yaml` |

## E. `templates/configmap.yaml`

**Copy** your `k8s/configmap.yaml`, then **adapt** it to turn every key under `.Values.config` into a ConfigMap entry:

- Replace the hard-coded name with `{{ include "movie-chart.fullname" . }}-config`.
- Replace the `labels:` block with `{{- include "movie-chart.labels" . | nindent 4 }}`.
- Replace the fixed `data:` entries with a loop over `.Values.config`: `{{- range $key, $val := .Values.config }}` … `{{- end }}`, emitting `{{ $key }}: {{ $val | quote }}`.

*Reference answer: `solved/step-05/movie-chart/templates/configmap.yaml`.*

## F. `templates/secret.yaml`

**Copy** your `k8s/secret.yaml`, then **adapt** the `Opaque` Secret so it's named `...-secret` and holds `MONGO_URI` under `stringData`:

- Name → `{{ include "movie-chart.fullname" . }}-secret`; labels → `{{- include "movie-chart.labels" . | nindent 4 }}`.
- The catch (from step B): `secret.mongoUri` *itself* contains `{{ .Release.Name }}`. A plain `{{ .Values.secret.mongoUri }}` would emit that template text verbatim. Pass it through **`tpl`** to render the inner template against the current context — i.e. `tpl .Values.secret.mongoUri .`, then `| quote`.

*Reference answer: `solved/step-05/movie-chart/templates/secret.yaml`.*

## G. `templates/deployment.yaml`

**Copy** your `k8s/deployment.yaml`, then **adapt** it to templated form:

- **Name:** `{{ include "movie-chart.fullname" . }}` — **replicas:** `{{ .Values.replicaCount }}`.
- **Labels/selectors:** `metadata.labels` uses `movie-chart.labels`; both `selector.matchLabels` and the pod-template `metadata.labels` use `movie-chart.selectorLabels`. Mind the indentation: `nindent 4` for the metadata labels, `nindent 6` under `matchLabels`, `nindent 8` under the pod template.
- **Image:** `"{{ .Values.image.repository }}:{{ .Values.image.tag }}"`, `imagePullPolicy: {{ .Values.image.pullPolicy }}`, `containerPort: {{ .Values.service.targetPort }}`.
- **`envFrom`:** point at the templated ConfigMap (`...-config`) and Secret (`...-secret`) names.
- **`extraEnv`:** render the `env:` block only if the map is non-empty — wrap it in `{{- with .Values.extraEnv }}` … `{{- end }}` and `range` inside, emitting `name`/`value` per key.
- **Probes:** liveness/readiness hit `/health` on `{{ .Values.service.targetPort }}`.

*Reference answer: `solved/step-05/movie-chart/templates/deployment.yaml`.*

## H. `templates/service.yaml`

**Copy** your `k8s/service.yaml`, then **adapt** the app Service:

- Name `{{ include "movie-chart.fullname" . }}`, common labels via `movie-chart.labels`.
- `type: {{ .Values.service.type }}`.
- `selector` = `movie-chart.selectorLabels` (`nindent 4`).
- A single port mapping: `port: {{ .Values.service.port }}` → `targetPort: {{ .Values.service.targetPort }}`.

*Reference answer: `solved/step-05/movie-chart/templates/service.yaml`.*

## I. `templates/mongo-service.yaml` (conditional)

**Copy** your `k8s/mongo-service.yaml` (the headless Mongo Service), then **adapt** it — the new twist is that it must only render when in-cluster Mongo is enabled:

- Wrap the **entire file** in `{{- if .Values.mongo.enabled }}` … `{{- end }}`.
- Name it `{{ include "movie-chart.mongoName" . }}`; make it headless (`clusterIP: None`).
- Add a `component: mongo` label to **both** the labels and the selector, so this Service's selector doesn't collide with the app Service's selector (they'd otherwise share the same app+release labels and grab each other's pods).
- Port: `{{ .Values.mongo.port }}` → `targetPort: 27017`.

*Reference answer: `solved/step-05/movie-chart/templates/mongo-service.yaml`.*

## J. `templates/mongo-statefulset.yaml` (conditional)

**Copy** your `k8s/mongo-statefulset.yaml`, then **adapt** it — templated and wrapped in the **same** `{{- if .Values.mongo.enabled }}` guard as step I:

- Name and `serviceName` both = `{{ include "movie-chart.mongoName" . }}`; `replicas: 1`.
- Same `component: mongo` label on metadata, selector, and pod template as in step I (same indentation rules as the Deployment).
- Container image `{{ .Values.mongo.image }}`, container port `27017`, volume mount at `/data/db`.
- A `volumeClaimTemplates` entry with `accessModes: ["ReadWriteOnce"]` and `storage: {{ .Values.mongo.storage }}`.

*Reference answer: `solved/step-05/movie-chart/templates/mongo-statefulset.yaml`.*

## K. `templates/NOTES.txt`

**This one is new** — there's no Step 04 file to copy. Write it from scratch: post-install help, printed by Helm after `install`/`upgrade`.

Requirements:
- A `kubectl port-forward` line built from `{{ include "movie-chart.fullname" . }}` and `{{ .Values.service.port }}`.
- A line that differs depending on `mongo.enabled` — use `{{- if .Values.mongo.enabled }}` / `{{- else }}` / `{{- end }}` (e.g. "in-cluster Mongo enabled" vs. "Mongo is external").

*Reference answer: `solved/step-05/movie-chart/templates/NOTES.txt`.*

---

## L. The Helm workflow

Run these from inside the chart directory (`movie-chart/`). Read the output after each — the point is to *see* what Helm does.

**1. Lint** — catch template/values errors before installing:
```bash
helm lint .
```

**2. Render locally without installing** — inspect the generated YAML:
```bash
helm template demo .
```

**3. Install** the release as `demo`:
```bash
helm install demo .
```

**4. Inspect** what was applied and the running pods:
```bash
helm get manifest demo
kubectl get pods,svc
```

**5. Upgrade with an inline override** (bump replicas without editing files):
```bash
helm upgrade demo . --set replicaCount=3
```

**6. Upgrade using the prod values file** (layered over `values.yaml`):
```bash
helm upgrade demo . -f values-prod.yaml
```

**7. History, then roll back** to the first revision:
```bash
helm history demo
helm rollback demo 1
```

**8. Uninstall:**
```bash
helm uninstall demo
```

> **Verify the conditional works:** render with Mongo disabled and confirm the Mongo Service *and* StatefulSet vanish from the output:
> ```bash
> helm template demo . --set mongo.enabled=false | grep -i mongo
> ```
>
> **Local image note (minikube/kind):** if you built the image locally instead of pushing to Docker Hub, add these to your `install`/`upgrade`/`template` commands so Kubernetes uses the local image:
> ```bash
> --set image.repository=movie-api --set image.tag=1.0 --set image.pullPolicy=Never
> ```

---

## What you learned

- **values.yaml** parameterizes a chart; `--set` and `-f` override per environment.
- **`_helpers.tpl`** named templates (`include`) keep names/labels DRY and consistent.
- **Conditionals** (`{{- if .Values.mongo.enabled }}`) and **range/with** make templates flexible.
- `helm upgrade` / `rollback` / `history` give you versioned, reversible releases.

## Next

→ [Step 06 — MongoDB as a Helm dependency](step-06-helm-mongodb-dependency.md)
