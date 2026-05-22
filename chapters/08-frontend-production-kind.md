# Chapter 8 (alternative): Production Dockerfile + Skaffold profiles — kind variant

> Drop-in replacement for [08-frontend-production.md](08-frontend-production.md). The only kind-specific bit is the `-d localhost:5001` registry flag in the Skaffold commands; the rest (production Dockerfile, nginx config, Skaffold profile patches) is identical to the original.

## 1. Why a second Dockerfile

The `Dockerfile.dev` you built in chapter 2 runs the **webpack-dev-server** — Vue CLI's hot-reload dev server. Great for development, terrible for production:

* Webpack-dev-server holds the entire source tree in memory and re-bundles on every request.
* It opens debugging endpoints (`/sockjs-node`, etc.) that leak source maps.
* The resulting image is huge (node + npm cache + source).
* It's slower than serving static files.

For production we want the inverse:

1. Compile the Vue source down to a static bundle (`dist/` folder: HTML + JS + CSS).
2. Serve those static files via a tiny webserver (nginx).
3. Throw away the build toolchain — the final image only needs nginx + the compiled assets.

This is a textbook **multi-stage Docker build**: stage 1 = build, stage 2 = serve. The output image is ~30MB instead of ~1GB.

## 2. Create the production Dockerfile

### Create `frontend/docker/Dockerfile` (new file)

```Dockerfile
# stage 1: build
FROM node:20-alpine as build-stage
LABEL org.opencontainers.image.authors="tutorial"

RUN npm install -g @vue/cli

ENV NODE_OPTIONS="--max-old-space-size=8192"
ADD package.json package-lock.json* /source/
WORKDIR /source
RUN npm install
ADD . /source
WORKDIR /source
RUN npm install && cp .env.k8s .env && npm run build

# stage 2: serve
FROM nginx:stable-alpine as production-stage
ADD docker/default.conf /etc/nginx/conf.d/default.conf
COPY --from=build-stage /source/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Walk-through:

* `FROM ... as build-stage` — names the first stage so the second can copy from it.
* The build stage is essentially the same as `Dockerfile.dev`, but ends with `npm run build` (Vue CLI's "produce static dist/" command) instead of `npm run serve`.
* `FROM nginx:stable-alpine as production-stage` — fresh, minimal image. No node, no npm, no source.
* `COPY --from=build-stage /source/dist /usr/share/nginx/html` — the only thing that travels from stage 1 to stage 2: the compiled assets.
* `CMD ["nginx", "-g", "daemon off;"]` — nginx in foreground (containers need PID 1 to stay alive).

### Create `frontend/docker/default.conf` (new file)

Nginx needs a config telling it how to serve the SPA. Single-page apps have one HTML file; client-side routing handles the rest. The trick is the `try_files` line — for any URL nginx doesn't recognize as a real file, it falls back to `index.html` so the Vue Router can take over.

```nginx
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

### Quick local test (optional, just to see the prod image work)

From `frontend/`:

```shell
docker build -t myfrontend-prod -f docker/Dockerfile .
```

```shell
docker run --rm -p 8888:80 myfrontend-prod
```

Browse `http://localhost:8888`. Same UI as dev mode, served by nginx. Stop with Ctrl-C.

## 3. Point Skaffold at the production Dockerfile

Open `/home/dev/skaffold-helm-tutorial/skaffold.yaml`. Find the `frontend` artifact and change its `dockerfile:` line:

**Before:**
```yaml
    - image: frontend
      context: frontend
      docker:
        dockerfile: docker/Dockerfile.dev
```

**After:**
```yaml
    - image: frontend
      context: frontend
      docker:
        dockerfile: docker/Dockerfile
```

Now `skaffold run -d localhost:5001` builds the production image. Live-sync of `.vue` files won't work anymore (the nginx image has no node, no webpack-dev-server) — you'd need a full rebuild on every change. That's why you wouldn't run `skaffold dev` against a production image during active development.

## 4. Introduce a Skaffold profile for development

You want **both modes available** without editing `skaffold.yaml` every time. Skaffold's solution is **profiles**: named overrides applied on top of the base config when you pass `-p <name>`.

Append to the end of `skaffold.yaml`:

```yaml
profiles:
  - name: dev
    patches:
      - op: replace
        path: /build/artifacts/1/docker/dockerfile
        value: docker/Dockerfile.dev
```

What this says:

* `name: dev` — the profile name. Activated with `skaffold run -p dev` (or `skaffold dev -p dev`).
* `patches:` — JSON-Patch operations applied to the base config.
* `op: replace` — overwrite a field.
* `path: /build/artifacts/1/docker/dockerfile` — JSON Pointer into the config. `/build/artifacts/1` = the **second** artifact (zero-indexed; `0` = `api`, `1` = `frontend`). `/docker/dockerfile` drills into that artifact's `docker.dockerfile` setting.
* `value: docker/Dockerfile.dev` — what to set it to.

End result: with no profile, you build the prod Dockerfile. With `-p dev`, the same Skaffold config switches to `Dockerfile.dev`.

## 5. Use the modes

Production-style build and deploy:

```shell
skaffold run -d localhost:5001
```

Development build (dev server + hot reload) — same chart, dev Dockerfile:

```shell
skaffold run -d localhost:5001 -p dev
```

Live mode in dev:

```shell
skaffold dev -d localhost:5001 -p dev
```

When you change a `.vue` file in `frontend/src/`, Skaffold syncs it into the running container, webpack-dev-server hot-reloads, the browser refreshes. Same loop as chapter 7, except now flipping `-p dev` on/off chooses which container architecture is running underneath.

## 6. Why profiles instead of separate `skaffold-*.yaml` files

You *could* have two complete files (`skaffold.yaml`, `skaffold-prod.yaml`) and pass `--filename`. Profiles are usually nicer because:

* **DRY** — 95% of the config is shared, the diff is tiny.
* Multiple profiles compose. You can have `dev`, `staging`, `prod`, `prod-eu`, layering JSON-Patch ops.
* Single source of truth: one file in git, profile names listed in the README.

Patches feel awkward at first (the JSON Pointer syntax especially). For changes that touch many fields, profiles also support `patchesStrategicMerge`-style override blocks, but the JSON-Patch form is more precise for small surgical edits like this one.

## 7. Cleanup before chapter 9

```shell
skaffold delete -d localhost:5001
```

Or `skaffold delete -d localhost:5001 -p dev` if you ended on the dev profile — pass the same flags as the run that created the release, so Skaffold knows which release name it owns.

## Review questions

### 1. The dev image is ~1 GB, the prod image is ~30 MB. Where does the size difference come from?

<details>
  <summary>Answer</summary>

The dev image carries everything needed to **build and serve**:

* `node:20-alpine` base (~150 MB)
* `npm install -g @vue/cli` (~400 MB of dependencies for the CLI)
* `npm install` of project dependencies (~600 MB in `node_modules/`)
* All source code (`src/`, `public/`, etc.)
* Webpack-dev-server kept resident in memory at runtime

The prod image carries only what's needed to **serve** static files:

* `nginx:stable-alpine` base (~25 MB)
* The compiled `dist/` folder (your `src/` minified and bundled, typically a few MB for a small app)
* A 1-line nginx config

The build toolchain stays in stage 1, which Docker discards once stage 2 is complete. Multi-stage = "use a big builder, ship a tiny runner."

</details>

### 2. Why does `try_files $uri $uri/ /index.html;` in the nginx config exist? What breaks without it?

<details>
  <summary>Answer</summary>

Single-page apps put **all routing in the browser**. When the user navigates to `/about`, no `/about.html` file exists on the server — Vue Router intercepts the URL and renders the right component using JavaScript.

But the first request to `/about` (typing the URL in the address bar, or hitting refresh while on that page) goes to the **server**. Without `try_files`, nginx looks for a file named `about` in `/usr/share/nginx/html`, doesn't find one, and returns `404 Not Found`. The user sees a broken page.

`try_files $uri $uri/ /index.html;` says:

1. First try `$uri` (e.g. `/about`) as a file.
2. If not found, try `$uri/` (treating it as a directory looking for an index).
3. If still not found, **fall back to `/index.html`**.

Now any unknown path returns `index.html`, the Vue app boots, the Vue Router reads the URL, and renders `/about` client-side.

Same trick is used by every SPA framework's nginx recipe (React, Angular, Svelte, …). Slight variations exist for sub-path deployments.

</details>

### 3. The JSON Pointer `/build/artifacts/1/docker/dockerfile` looks fragile — what happens if you reorder the artifacts in your skaffold.yaml?

<details>
  <summary>Answer</summary>

The patch breaks. `/build/artifacts/1` literally means "the second item in the array." Reorder so `frontend` is first → the patch now points at `api`'s Dockerfile and silently corrupts your `api` artifact instead.

This is a real footgun. Two safer alternatives:

1. **Use `patchesStrategicMerge`-style blocks** instead of JSON-Patch. They reference artifacts by name, not index:
   ```yaml
   profiles:
     - name: dev
       build:
         artifacts:
           - image: frontend
             docker:
               dockerfile: docker/Dockerfile.dev
   ```
   Slightly more verbose but order-independent and self-documenting.

2. **Add a comment to the array** saying "do not reorder, profile patches reference indices." Defensive but ugly.

For tiny configs with two artifacts, the JSON-Patch form is fine. For real projects with five+ artifacts and multiple profiles, the strategic-merge form is safer.

</details>

### 4. You ran `skaffold run -p dev`, then later `skaffold delete -d localhost:5001` without `-p dev`. Did it clean up?

<details>
  <summary>Answer</summary>

**Probably yes**, but it's a footgun worth understanding.

Profiles don't usually change the Helm release **name** — that's set in `deploy.helm.releases[].name`. So `skaffold delete` looks for a release called `myapp` (the name in your base config) and uninstalls it regardless of which profile is active.

But profiles **can** patch the release name. If a profile changes `deploy.helm.releases[0].name` to `myapp-dev`, then deleting without `-p dev` would look for `myapp`, not find it, and silently do nothing — leaving `myapp-dev` orphaned. To catch this:

```
helm list -a
```

Shows all releases regardless of which Skaffold profile was active when they were installed.

Habit: pass the same `-p` flag(s) to `skaffold delete` as you used for `skaffold run`. Or just use `helm uninstall <release>` directly if Skaffold's profile bookkeeping gets confusing.

</details>

### 5. You're adding a third profile for "staging" — same as prod but with a different chart value (`replicas: 2`). What's the minimal change?

<details>
  <summary>Answer</summary>

Add the profile to the bottom of `skaffold.yaml`:

```yaml
profiles:
  - name: dev
    patches:
      - op: replace
        path: /build/artifacts/1/docker/dockerfile
        value: docker/Dockerfile.dev

  - name: staging
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
            setValues:
              frontend.replicas: "2"
```

Profile patches **overlay** the base config — fields you don't mention stay as-is. So this profile inherits the build section (prod Dockerfile) and only changes the Helm `setValues`.

In `myapp/values.yaml` you'd need a corresponding `frontend.replicas: 1` default and the Deployment template would need to use `replicas: {{ .Values.frontend.replicas }}`. Plumb through whatever's actually configurable.

For a real prod/staging split you'd also use different namespaces, different registries, different image-tag policies, etc. — that's why some teams have multiple `skaffold-<env>.yaml` files instead of profiles.

</details>

### 6. The dev image runs `npm run serve` as PID 1. The prod image runs nginx as PID 1. Why does that matter?

<details>
  <summary>Answer</summary>

Linux containers have a special contract for PID 1: **PID 1 is the init process**. It's responsible for:

* Receiving signals (SIGTERM, SIGINT) and forwarding them to children.
* Reaping zombie child processes.
* Exiting cleanly when shutdown is requested.

The container's *whole lifecycle* depends on PID 1 behaving correctly. If PID 1 ignores SIGTERM, `docker stop` and `kubectl delete pod` will hang until the 30-second timeout, then SIGKILL.

* **nginx** is a well-behaved init process when given `-g "daemon off;"`. It catches signals, reaps children, exits cleanly. Production-grade.
* **Webpack-dev-server** is *not* especially well-behaved as PID 1 — it has signal-handling quirks. Fine in dev where you Ctrl-C interactively, but in production you don't want pods taking 30+ seconds to terminate during a rollout.

Other common offenders: shell scripts (`CMD npm run serve` literally runs through `sh -c`, and shell signal handling is iffy), Python apps without explicit signal handlers, Java apps started via wrapper scripts. The fix is often a small init process like `tini` (`tini -- node server.js`) which docker provides via `--init` or `tini` baked into base images.

For our tutorial it's mostly cosmetic — kind clusters tear down whole pods fast. In production it matters a lot.

</details>
