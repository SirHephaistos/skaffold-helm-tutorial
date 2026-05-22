# Chapter 7 (alternative): Skaffold — kind variant

> Drop-in replacement for [07-skaffold.md](07-skaffold.md). The only meaningful change vs the original is the registry flag: `-d localhost:5001` instead of `-d registry.kube-public`. The `skaffold.yaml` itself is generic — it doesn't hardcode the registry.

## 1. Why Skaffold

After chapter 6 you have a working Helm chart, but every code change still requires:

1. `docker build` for whichever app changed
2. `docker tag localhost:5001/<image>`
3. `docker push localhost:5001/<image>`
4. `helm upgrade --install myapp-deployment-1 myapp --set <something>.image.tag=...`

Four steps. Annoying after the third time, infuriating after the thirtieth. **Skaffold** is the glue layer that runs all four for you when you save a file. It doesn't replace Docker, Helm, or the cluster — it just chains them together.

What Skaffold does, end to end:

1. Watches your source code (only the directories you tell it).
2. On change → rebuilds the affected Docker image(s).
3. Tags each image with a content-addressable digest (so identical source = identical tag = cache hit next time).
4. Pushes to the configured registry.
5. Updates the Helm release with the new image tag(s) — patches whatever values the chart needs.
6. Auto port-forwards Services to your host so you can hit them in a browser.

Optional bonus: live-sync. For some file types (Python, JS, Vue), Skaffold can copy the changed file directly into the running container instead of rebuilding the image. Faster than a full rebuild when the runtime supports it (Python's `uvicorn --reload`, Vue's webpack-dev-server hot reload).

## 2. Clean up the manual Helm release

Skaffold will take ownership of the Helm release. Wipe the one you installed by hand in chapter 6 so there's no conflict:

```shell
helm uninstall myapp-deployment-1
```

```shell
kubectl get pod
```

All `frontend`, `api`, `postgresql-db` pods should be `Terminating` or gone.

## 3. Create `skaffold.yaml`

Skaffold reads its config from `skaffold.yaml` at the project root. Create `/home/dev/skaffold-helm-tutorial/skaffold.yaml` (new file) with this content:

```yaml
apiVersion: skaffold/v4beta10
kind: Config
metadata:
  name: myapp
build:
  tagPolicy:
    inputDigest: {}
  local:
    concurrency: 0
  artifacts:
    - image: api
      context: myapi
      docker:
        dockerfile: docker/Dockerfile
      sync:
        infer:
          - "*.py"
          - "**/*.py"
          - "**/*.html"
          - "**/*~"
    - image: frontend
      context: frontend
      docker:
        dockerfile: docker/Dockerfile.dev
      sync:
        infer:
          - "*.js"
          - "*.html"
          - "*.vue"
          - "**/*.vue"
          - "**/*.js"
          - "**/*~"

deploy:
  helm:
    releases:
      - name: myapp
        chartPath: myapp
        setValueTemplates:
          frontend.image.repository: "{{.IMAGE_REPO_frontend}}"
          frontend.image.tag: "{{.IMAGE_TAG_frontend}}"
          backend.image.repository: "{{.IMAGE_REPO_api}}"
          backend.image.tag: "{{.IMAGE_TAG_api}}"

portForward:
  - resourceType: service
    resourceName: frontend
    port: 80
    localPort: 8080
  - resourceType: service
    resourceName: api
    port: 80
    localPort: 9999
```

Note: `setValueTemplates` references `backend.image.*` (not `api.image.*`) — matches the values.yaml structure you canonicalized on in chapter 6. If you picked `api.image.*` instead, adapt accordingly.

### What each section does

* **`build.tagPolicy.inputDigest`** — Skaffold tags each image with a hash of its build context. Identical inputs = identical tag = no rebuild, no push, no helm update. Other policies exist (`gitCommit`, `envTemplate`, `dateTime`).
* **`build.local.concurrency: 0`** — Build all artifacts in parallel (0 = unlimited).
* **`build.artifacts`** — each block describes one image: name, context dir, Dockerfile, file-sync patterns.
* **`deploy.helm`** — wraps `helm upgrade --install`. Skaffold calls it for you.
* **`setValueTemplates`** — wires Skaffold's per-image variables (`IMAGE_REPO_*`, `IMAGE_TAG_*`) into the chart's value paths. After building image `frontend`, Skaffold sets `frontend.image.repository = <whatever-registry>/frontend` and `frontend.image.tag = <digest>`.
* **`portForward`** — auto-forward cluster Services to your host while `skaffold dev` runs. Same as `kubectl port-forward`, just declarative.

## 4. Build and push (without deploying)

```shell
cd /home/dev/skaffold-helm-tutorial
```

```shell
skaffold build -d localhost:5001
```

`-d localhost:5001` = the default registry. Skaffold combines this with the artifact name → `localhost:5001/frontend:<digest>` and `localhost:5001/api:<digest>`. Watch the output — it builds both, pushes both, prints the tags.

Verify they landed:

```shell
curl http://localhost:5001/v2/_catalog
```

You should see `frontend` and `api` repos with new digest tags (alongside the manually-pushed `myfrontend` / `myapi` from earlier).

> Note: Skaffold builds the image with name `api` and `frontend`, then publishes as `localhost:5001/api` / `localhost:5001/frontend` — different names from the `myapi` / `myfrontend` you pushed by hand in chapters 4 and 6. Skaffold owns these images now.

## 5. Build, push, and deploy

```shell
skaffold run -d localhost:5001
```

This runs the full pipeline: build → push → `helm upgrade --install`. When it exits, the Helm release `myapp` is on the cluster with the freshly-built images.

```shell
kubectl get pod
```

You should see `frontend-...`, `api-0`, `postgresql-db-0` all `Running`.

Open `http://localhost:8080` in a browser (matches the `portForward` block in `skaffold.yaml`). The frontend should load and the "request time" button should hit the API at `localhost:9999`.

## 6. Live mode: `skaffold dev`

```shell
skaffold dev -d localhost:5001
```

This stays running. It watches your source files. Change anything matched by the `sync.infer` patterns in `skaffold.yaml` and:

* For matched file types → **sync**: copy the new file into the running container. Vue's webpack-dev-server hot-reloads the browser. Uvicorn's `--reload` restarts the FastAPI process. Sub-second feedback loop.
* For files not matched (e.g. `Dockerfile`, `setup.py`, `package.json`) → **rebuild**: full image rebuild + push + Helm patch + pod rolling update. Slower, ~30–60s.

Try it: open `frontend/src/components/HelloWorld.vue`, change the heading text, save. The change should appear in the browser within a couple seconds.

Stop with `Ctrl-C`. Skaffold then runs the cleanup: it `helm uninstall`s the release.

> If `skaffold dev` exits without cleaning up (`kill -9`, crash), the release stays. Use `helm uninstall myapp` to remove it.

## 7. Clean up before chapter 8

```shell
skaffold delete -d localhost:5001
```

Runs `helm uninstall myapp` for you. After this, `kubectl get pod` should show no `frontend`/`api`/`postgresql-db` pods.

## Review questions

### 1. What's the difference between `skaffold build`, `skaffold run`, `skaffold dev`, and `skaffold delete`?

<details>
  <summary>Answer</summary>

| Command | What it does | When to use |
|---|---|---|
| `skaffold build` | Builds + pushes images. **Doesn't deploy.** Prints the resulting tags to stdout. | CI pipelines that build artifacts and hand off to a separate deploy step. |
| `skaffold run` | Build + push + deploy via Helm. One-shot, then exits. The release stays. | Manual "deploy what I have right now" — like a heavier `helm upgrade --install`. |
| `skaffold dev` | Build + push + deploy + **watch source** + auto-rebuild on change + auto-cleanup on Ctrl-C. | Your default development loop. The headline feature. |
| `skaffold delete` | Run the cleanup step alone — `helm uninstall` the release Skaffold created. | When you finished `skaffold dev` cleanly but later want a manual cleanup, or when `skaffold run` left a release behind. |
| `skaffold debug` (bonus) | Like `dev` but configures language-specific debug ports (Java JDWP, Node inspector, etc.). | Attaching a debugger to a pod from your IDE. |

`dev` = the bread and butter. The others are for specific moments.

</details>

### 2. What is `setValueTemplates` actually doing, and what's the difference between `IMAGE_REPO_frontend` and `IMAGE_TAG_frontend`?

<details>
  <summary>Answer</summary>

`setValueTemplates` is Skaffold's wiring between **what it just built** and **what the Helm chart expects**.

After building an image, Skaffold knows two things about it:

* `IMAGE_REPO_<artifact-name>` — the full registry+repo string, e.g. `localhost:5001/frontend`
* `IMAGE_TAG_<artifact-name>` — the tag Skaffold assigned, e.g. `f3a9c2b...` (the inputDigest)

`setValueTemplates` says: "for the Helm release, set these chart value paths to these Skaffold variables." So this block:

```yaml
setValueTemplates:
  frontend.image.repository: "{{.IMAGE_REPO_frontend}}"
  frontend.image.tag: "{{.IMAGE_TAG_frontend}}"
```

…is equivalent to running:

```
helm upgrade --install myapp myapp \
  --set frontend.image.repository=localhost:5001/frontend \
  --set frontend.image.tag=f3a9c2b...
```

…where `localhost:5001/frontend` and `f3a9c2b...` change every build.

This is why the chart's `values.yaml` can have `tag: null` as a placeholder — Skaffold fills it in. The `default .Chart.AppVersion` fallback you wrote in chapter 6 only matters when Skaffold isn't driving the deploy (e.g. someone installs the chart by hand).

</details>

### 3. The `tagPolicy: inputDigest` means images are tagged with a hash of the build context. Why is that better than tagging with `latest` or with the git SHA?

<details>
  <summary>Answer</summary>

* **`latest`** is the worst option. Two builds with different code can share the same tag. The cluster sees `latest` already pulled and reuses the cached image, never picking up your new code. Also `imagePullPolicy` defaults to `Always` with `latest`, eating bandwidth even when nothing changed.
* **Git SHA** is fine for CI builds but bad locally. Every commit gets a unique tag, but a *change without a commit* (you edited a file, didn't commit yet) gets the same SHA as before. So during active development the cluster keeps pulling the same tag and seeing the same old image.
* **`inputDigest`** = hash of the actual build context. Edit a `.py` file → digest changes → new tag → new pull → new pod with new code. Don't edit anything → digest unchanged → no rebuild, no push, no pod restart. **Maps exactly onto "did the code actually change?"**

inputDigest is the right policy for development loops. For production release pipelines, a deterministic policy tied to a release tag (`envTemplate` driven by `$VERSION`) is more readable in registry catalogs.

</details>

### 4. Why doesn't `skaffold.yaml` mention `localhost:5001` anywhere? It uses bare names like `image: api`.

<details>
  <summary>Answer</summary>

The artifact `name` in `skaffold.yaml` is the **logical** image name. The actual registry prefix comes from the `-d <registry>` flag at command time (or from a `default-repo` setting in `~/.skaffold/config`).

This separation is intentional:

* The same `skaffold.yaml` works against multiple registries — `localhost:5001` for kind, `your-team.azurecr.io` for staging, `myteam-prod.amazonaws.com` for production.
* Skaffold combines `default-repo + artifact-name` to compute the full image ref: `localhost:5001/api`, `your-team.azurecr.io/api`, etc.
* Helm chart values get the full ref from `{{.IMAGE_REPO_*}}` — no chart-side config needed for "which registry are we using today."

Want to bake the registry in permanently? Run once:

```
skaffold config set default-repo localhost:5001
```

Then you can drop the `-d` flag — Skaffold reads it from `~/.skaffold/config`. Fine for one-developer setups.

</details>

### 5. `skaffold dev` says it does "live sync" for matching file types. What's the difference between sync and rebuild, and why is sync sometimes wrong?

<details>
  <summary>Answer</summary>

* **Rebuild** = full Docker build → push → Helm upgrade → rolling update. The cluster gets a new image. Always correct but slow (30–60s for our chart).
* **Sync** = Skaffold copies the changed file directly into the running container via `kubectl cp` (or its equivalent). No image rebuild, no push, no Helm upgrade. The container's filesystem now has the new file. Whether the *running process* notices depends on the app:
  * Vue webpack-dev-server watches its filesystem → reloads page automatically.
  * `uvicorn --reload` watches Python files → restarts the FastAPI process.
  * A statically compiled binary (Go, Rust, C++) → **no effect**; the binary in memory is unchanged. You'd need a rebuild.

When is sync wrong? When the change matters at build time:

* `package.json` / `setup.py` — installed deps depend on these; only a rebuild picks up new deps.
* Dockerfile changes — obviously rebuild.
* Config files that the app only reads at startup — sync the file, but the app still uses the old version until restarted.

Skaffold's `sync.infer` pattern list controls which files trigger sync. Files not matching → rebuild. The default patterns in our `skaffold.yaml` are conservative on purpose.

If you suspect a sync gave you stale behavior, `kubectl delete pod <pod>` forces a fresh pull and is the quick sanity check.

</details>

### 6. What happens if you Ctrl-C `skaffold dev` vs `kill -9` it?

<details>
  <summary>Answer</summary>

* **Ctrl-C** = SIGINT. Skaffold's signal handler runs: it calls `helm uninstall myapp`, removes the port-forward, exits clean. Cluster goes back to pre-`skaffold dev` state.
* **kill -9** = SIGKILL. Process dies instantly, no signal handler runs. The Helm release stays. Port-forwards die (they were owned by Skaffold), but the deployed resources remain. Next `skaffold dev` will see the leftover release and try to upgrade it — usually fine, but occasionally causes "release in progress" lockouts.

Recovery if `kill -9` left a mess:

```
helm list -a
helm uninstall myapp
```

…and try `skaffold dev` again. If `helm list -a` shows the release in `pending-upgrade` state, you may need `helm rollback myapp` first.

</details>
