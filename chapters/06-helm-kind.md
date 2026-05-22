# Chapter 6 (alternative): Creating a Helm chart — kind variant

> Drop-in replacement for [06-helm.md](06-helm.md). Differences from the original: image references use `localhost:5001/...`, no `imagePullSecrets` block, no `kubectl create secret docker-registry registry-creds` step. The Helm concepts are otherwise byte-for-byte the same.

## 1. What a Helm chart is

So far you've been writing raw YAML files and `kubectl apply`-ing them one at a time. That works for two or three objects; it starts hurting around ten. A **Helm chart** is:

* a folder of YAML **templates** (each a Kubernetes resource manifest with placeholders for values),
* a `values.yaml` with the default settings those placeholders draw from,
* a `Chart.yaml` with metadata (chart name, version, app version, dependencies).

`helm install` renders the templates with the values, produces ordinary YAML, and submits it to the cluster as a **release**. Helm remembers the release in the cluster (as a Secret), so a later `helm upgrade` knows what was installed last time and can diff against your new templates.

In one picture: **Helm = templating engine + release tracker on top of `kubectl apply`**.

## 2. Bootstrapping a chart

From the tutorial root (`/home/dev/skaffold-helm-tutorial` or wherever you cloned it):

```shell
helm create myapp
```

This drops a `myapp/` folder loaded with example files for a generic web app. Most of it is noise for our purposes. Clean it up:

* Empty `myapp/values.yaml` (delete every line — keep the file).
* Delete every file under `myapp/templates/` but **keep the `templates/` folder itself**.
* Leave `myapp/Chart.yaml` alone.

Now create a **new file** `myapp/templates/frontend.yaml`. Paste the content below into it — this is the same Deployment YAML you used in chapter 4 (with the image ref already adapted for kind). For now leave it un-templated; we'll templatize the image in §3:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: localhost:5001/myfrontend
```

(No `imagePullSecrets` block — our registry is anonymous HTTP.)

Try installing it:

```shell
helm install myapp-deployment-1 myapp
```

`myapp-deployment-1` = the **release name** (your choice). `myapp` = the chart path (the folder).

Verify:

```shell
helm list
```

```shell
kubectl get deployment
```

Wipe it:

```shell
helm uninstall myapp-deployment-1
```

`helm list` is namespace-scoped. If you put releases in a non-default namespace, pass `-n <namespace>`.

## 3. Make the image configurable

Hardcoding `localhost:5001/myfrontend` in the template defeats the point of Helm. Make it a value.

First, uninstall the release from §2 so you don't fight a stale install:

```shell
helm uninstall myapp-deployment-1
```

### Edit `myapp/values.yaml`

The file is empty (you emptied it in §2). Paste the content below into it — order doesn't matter since the file is empty, just save these four lines:

```yaml
frontend:
  image:
    repository: localhost:5001/myfrontend
    tag: null
```

The structure is free-form YAML. Helm reads `values.yaml` into a single Go map; templates reference it as `.Values`.

### Edit `myapp/templates/frontend.yaml`

Open the file. Find this line near the bottom (line 16 if you pasted §2's content verbatim):

```yaml
          image: localhost:5001/myfrontend
```

Replace **only that one line** with:

```yaml
          image: "{{ .Values.frontend.image.repository }}:{{ default .Chart.AppVersion .Values.frontend.image.tag }}"
```

Indentation must stay the same (10 spaces). Everything else in the file is untouched.

<details>
  <summary>Click for the full file as a sanity check</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: "{{ .Values.frontend.image.repository }}:{{ default .Chart.AppVersion .Values.frontend.image.tag }}"
```

</details>

### What the templating does

Two things inside the `{{ ... }}` curlies:

* `{{ .Values.frontend.image.repository }}` → the string from `values.yaml`.
* `{{ default .Chart.AppVersion .Values.frontend.image.tag }}` → Helm's `default` function. Returns `.Values.frontend.image.tag` **unless** it's nil/empty/null, in which case it falls back to `.Chart.AppVersion` from `Chart.yaml`.

The idea: when the tutorial author releases a new version of the chart, they bump `appVersion` in `Chart.yaml` and re-publish. Consumers don't need to override anything to get the new image — the default updates itself. Power users can still override the tag at install time with `--set`.

Try it:

```shell
helm install myapp-deployment-1 myapp --set frontend.image.repository=some.invalid/image --set frontend.image.tag=latest
```

Watch what kubernetes does with the (invalid) image — `kubectl get pod`, `kubectl describe pod` to see `ImagePullBackOff`. Then clean up and reinstall with proper values:

```shell
helm uninstall myapp-deployment-1
```

```shell
helm install myapp-deployment-1 myapp
```

## 4. Add a Service for the frontend

Open `myapp/templates/frontend.yaml`. At the **end of the file**, append the block below. The `---` separator is YAML's way of putting multiple documents in one file; Helm just submits each one to the API:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  ports:
    - port: 80
      targetPort: 80
      name: frontend
  selector:
    app: frontend
```

Re-deploy:

```shell
helm upgrade --install myapp-deployment-1 myapp
```

`helm upgrade --install` is the idiomatic "create if missing, update if exists" pattern. Use it everywhere — no need to manually track which call gets `install` vs `upgrade`.

## 5. Deploy the API

### Edit `myapp/values.yaml`

Right now the file holds the frontend block from §3. Add a backend block **alongside** the frontend one (order doesn't matter — Helm reads it all into one map). The file should end up containing both:

```yaml
frontend:
  image:
    repository: localhost:5001/myfrontend
    tag: null

backend:
  image:
    repository: localhost:5001/myapi
    tag: null
```

### Create `myapp/templates/api.yaml` (new file)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  ports:
    - port: 80
      targetPort: 80
      name: http
  selector:
    app: backend
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: api
  labels:
    app: api
spec:
  serviceName: api
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          imagePullPolicy: Always
          image: "{{ .Values.api.image.repository }}:{{ default .Chart.AppVersion .Values.api.image.tag }}"
```

Try to install:

```shell
helm upgrade --install myapp-deployment-1 myapp
```

Inspect carefully. **There are two bugs** in this `api.yaml`. The chart may install but the resulting workload won't function. Hunt them.

<details>
  <summary>Click to reveal the two bugs (try first!)</summary>

1. The template references `.Values.api.image.*` but `values.yaml` defines `backend.image.*`. One of the two names is wrong — pick which one you want to canonicalize on, then fix both spots so they agree. (The original tutorial inherits "backend" from chapter 4; sticking with that is a sensible choice.)
2. The Service `selector: { app: backend }` doesn't match the StatefulSet's pod label `app: api`. Either change the Service selector to `app: api`, or change the pod label to `app: backend`. They have to agree, or the Service has zero endpoints.

Both are common real-world mistakes, exactly the kind that a `helm template` dry-run wouldn't catch — the YAML is valid, the meaning is wrong.

</details>

Once it's fixed, redeploy. Check:

```shell
kubectl get pod
```

```shell
kubectl logs deployment/frontend
```

```shell
kubectl get endpoints api
```

If `endpoints/api` shows the api pod's IP, the Service is wired correctly.

## 6. Add the database to the chart

The chapter-4 ConfigMap, StatefulSet, and Service for postgres also belong in the chart. We're going to put all three in one file, **and** wrap them in a Helm conditional so the database can be turned off via a value (useful in production where you'd connect to an external managed Postgres instead).

### Edit `myapp/values.yaml`

Add a `db:` block at the end (alongside `frontend:` and `backend:`, order doesn't matter):

```yaml
db:
  enabled: true
```

### Create `myapp/templates/db.yaml` (new file)

Paste this whole file in. It contains ConfigMap + StatefulSet + Service, all wrapped in `{{- if .Values.db.enabled }} ... {{- end }}`:

```yaml
{{- if .Values.db.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgresql-initdb-config
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS counter (
      counterId SERIAL PRIMARY KEY,
      api TEXT NOT NULL,
      counter INTEGER NOT NULL default 0
    );

    INSERT INTO counter (api) VALUES ('myapi');
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
spec:
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
  serviceName: postgres-db
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
        - name: postgresql-db
          image: postgres:16
          volumeMounts:
            - name: postgresql-db-disk
              mountPath: /data
            - name: postgresql-initdb
              mountPath: /docker-entrypoint-initdb.d
          env:
            - name: POSTGRES_PASSWORD
              value: astrongdatabasepassword
            - name: PGDATA
              value: /data/pgdata
      volumes:
        - name: postgresql-initdb
          configMap:
            name: postgresql-initdb-config
  volumeClaimTemplates:
    - metadata:
        name: postgresql-db-disk
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
spec:
  selector:
    app: postgresql-db
  ports:
    - port: 5432
{{- end }}
```

The `{{- if }}` / `{{- end }}` are Helm template directives, not YAML. They wrap the **entire file**. If `db.enabled` is `false`, Helm renders nothing for this file and none of the three resources get created.

Now edit `myapp/templates/db.yaml`. Wrap the **entire content** of the file with a Helm conditional — one directive line at the very top, one at the very bottom:

```yaml
{{- if .Values.db.enabled }}
apiVersion: v1
kind: ConfigMap
# ... your existing ConfigMap content
---
apiVersion: apps/v1
kind: StatefulSet
# ... your existing StatefulSet content
---
apiVersion: v1
kind: Service
# ... your existing Service content
{{- end }}
```

(The `{{- if ... }}` / `{{- end }}` are template directives, not YAML. They wrap the whole file.)

Now `helm upgrade --install myapp-deployment-1 myapp --set db.enabled=false` will skip the database entirely. Useful in production where you'd connect to an external RDS / CrunchyDB cluster instead of an in-chart postgres.

## 7. How Helm actually works under the hood

`helm install` does three things:

1. **Reads the chart**: Chart.yaml, values.yaml, templates/.
2. **Renders templates**: substitutes `.Values`, `.Chart`, `.Release`, etc. into placeholder slots. Output = plain YAML.
3. **Submits to the API server**: same as `kubectl apply -f`.

You can run step 2 alone:

```shell
helm template myapp-installation myapp > myapp-rendered.yaml
```

```shell
cat myapp-rendered.yaml | less
```

Excellent debugging tool. If you can't tell whether Helm is reading your values correctly, render and look at the output.

Helm also stores **release history** in the cluster (as Secrets in the release's namespace, named `sh.helm.release.v1.<release>.v<N>`). Each `helm upgrade` creates a new entry. You can roll back:

```shell
helm history myapp-deployment-1
```

```shell
helm rollback myapp-deployment-1 1
```

> Helm only knows about *what it created*. If you `kubectl apply` an extra Service on top of a Helm release, Helm won't see it on the next upgrade and won't touch it. Conversely, if you `kubectl edit` a Deployment that Helm owns, Helm will overwrite your edit on the next upgrade. Stay disciplined about which tool owns which objects.

![helm-chart](../imgs/helm-chart.png)

![helm-render](../imgs/helm-render.png)

## 8. Troubleshooting

### "Another operation is in progress"

If you `Ctrl-C` Helm mid-install, the release ends up in a half-state stored in those release-secret. Next `helm install` complains.

```shell
helm list -a
```

`-a` shows all releases including failed/pending ones. Look for your release. Then either roll back:

```shell
helm rollback myapp-deployment-1
```

…or, if this is a first install that nothing depends on:

```shell
helm uninstall myapp-deployment-1
```

…and retry.

## Review questions

### 1. Why split the image into `repository` and `tag` in `values.yaml`? Wouldn't a single `image: "localhost:5001/myfrontend:0.0.1"` string be simpler?

<details>
  <summary>Answer</summary>

Both work, but the split is more flexible **because consumers rarely want to override the whole image, only the tag**.

With a split:

```
helm install myapp-deployment-1 myapp --set frontend.image.tag=v2.3.0
```

The repository defaults to the chart's value (`localhost:5001/myfrontend`); the user only specifies what changed.

With a monolithic string:

```
helm install myapp-deployment-1 myapp --set frontend.image="localhost:5001/myfrontend:v2.3.0"
```

The user has to repeat the repository every time. Worse, if they forget the repo, they typo it, or copy from an older example, the override breaks silently.

Coupling the **default tag** to `.Chart.AppVersion` (which the chart author bumps with each release) is the second half of the trick: end users get the new image automatically when they upgrade the chart, with no `--set` needed.

</details>

### 2. When does `helm upgrade --install` actually call `install` vs `upgrade`?

<details>
  <summary>Answer</summary>

Helm checks whether a release with that name already exists in that namespace:

* **Doesn't exist** → behaves like `helm install`: creates the release-secret, renders the templates, submits to the API server.
* **Exists** → behaves like `helm upgrade`: renders new templates, computes a diff against the *last applied* release, and patches what changed. Bumps the release revision (`v2`, `v3`, …).

The flag is idempotent: re-running with no changes is a no-op (well, it creates a new revision marker but doesn't touch any resources). It's the safe default for scripts, CI, and Skaffold (chapter 7 wires Skaffold to call exactly this).

</details>

### 3. What does `helm template` actually produce, and when should you reach for it instead of `helm install`?

<details>
  <summary>Answer</summary>

`helm template` runs only the **rendering** step — substitutes values into templates and emits the resulting YAML to stdout. It **never touches the cluster**.

Use cases:

* **Debugging values**: "did my override actually land?" → render and look.
* **Reviewing changes before apply**: render, `diff` against the previous render, eyeball the change.
* **Generating manifests for GitOps tools** that prefer raw YAML over Helm releases (Argo CD's "render then apply" mode, or Flux's Kustomization layered on top of `helm template`).
* **CI pipelines** that want to lint the YAML with kubeval or kube-linter before deploying.
* **Air-gapped clusters** where Helm can't reach the API server but you still want chart-driven manifests — render locally, ship the YAML to the operator who applies it.

What it loses vs `helm install`:

* No release history. Can't `helm rollback` something that was applied via `helm template | kubectl apply`.
* Helm doesn't track ownership, so cleanup is manual (`kubectl delete -f rendered.yaml`).

For learning Helm, `helm template` is the single best debugging tool. Use it liberally.

</details>

### 4. You change `replicas: 1` to `replicas: 3` in your chart and run `helm upgrade`. Do all three pods restart, or just the two new ones spin up?

<details>
  <summary>Answer</summary>

Just the two new ones spin up. The existing pod is **untouched**.

Helm computes a strategic merge patch: the `spec.replicas` field changes from `1` to `3`. The Deployment controller sees that, creates two more pods to reach the new desired count, and leaves the first one running.

If you'd changed something inside the pod template (image, env, command), the Deployment's rolling-update strategy would replace pods one at a time (or N at a time, configurable via `maxSurge` / `maxUnavailable`).

Field-level surgery, not "restart everything." That's why `helm upgrade` is generally safe — it only disturbs what actually changed in the rendered YAML.

</details>

### 5. You run `kubectl edit deployment frontend` and change the image. Then later `helm upgrade --install myapp-deployment-1 myapp`. What happens to your manual edit?

<details>
  <summary>Answer</summary>

**Overwritten.** Helm re-renders the templates from the chart, computes a diff against the previous Helm-stored manifest, and applies the patch. Your manual edit is not in the chart, so Helm has no reason to keep it.

Worse: Helm doesn't *know* your edit existed, so it didn't even try to merge. The image quietly reverts to whatever the chart says.

This is one of the most common "but I changed that yesterday and it's gone today!" surprises in k8s. Rule: **either Helm owns the object, or you do — never both.**

If you legitimately need a per-environment override (different image in staging vs production), the right answer is:

* Pass `--set` or `-f values-production.yaml` to Helm so the override is part of the chart input.
* Or wrap the chart in another tool (Helmfile, Argo CD, Flux) that records overrides as code.

Server-Side Apply (newer, `--server-side`) does a smarter 3-way merge that can sometimes preserve manual edits to fields the chart doesn't manage — but it's an advanced topic.

</details>

### 6. Helm stores release history as Secrets in the namespace. What happens if you `kubectl delete` those secrets directly?

<details>
  <summary>Answer</summary>

Helm "forgets" the release. The deployed resources (Deployment, Service, etc.) remain in the cluster — they're real Kubernetes objects, not owned by Helm — but `helm list` won't show the release anymore.

Side effects:

* You can't `helm rollback` to a previous revision (the history is gone).
* You can't `helm uninstall` to clean up (Helm doesn't know what to delete). You'd have to `kubectl delete` each object by hand.
* You **can** still `helm install` a fresh release with the same name — Helm sees no conflict, since from its perspective nothing exists.

Practical use: this is one (heavy-handed) way to recover from a corrupt release state, when `helm history`/`helm rollback` themselves are broken. Don't reach for it unless `helm history --max=N` and `helm rollback` have failed.

A gentler equivalent: `helm secrets list -n <ns>` and inspect what's there before deleting.

</details>
