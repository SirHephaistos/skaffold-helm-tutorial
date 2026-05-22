# Chapter 9 (alternative): Ingress on Traefik — kind variant

> Drop-in replacement for [09-ingress.md](09-ingress.md). The original uses nginx-ingress with nginx-specific regex annotations. This variant uses Traefik (installed in chapter 3) with the **standard Kubernetes `Ingress` resource** — no controller-specific annotations, fully portable across ingress controllers. The HelloWorld.vue URL gotcha at the end is the same in both versions.

## 1. What an Ingress is, again

So far you've been reaching the frontend and backend via `kubectl port-forward` (or via Skaffold's `portForward:` block). That works for one developer on one laptop. It doesn't scale to "give my colleague a URL."

An **`Ingress`** object is a routing rule: "for requests with hostname X and path Y, send to Service Z." The ingress controller (Traefik, in our setup) reads `Ingress` objects from the k8s API and rewires its proxy config accordingly.

Recap from chapter 3:

```
Browser
   ↓
host port 8080 (mapped to kind node :80 via extraPortMappings)
   ↓
Traefik pod inside cluster
   ↓ inspects Host header + path, looks up Ingress rules
Service (frontend or api)
   ↓
Pod
```

Until you create an `Ingress`, Traefik has no rules → answers 404 to everything (you saw this in chapter 3 §6 verification).

## 2. Pick a hostname

For a real public ingress you'd point a real DNS name at the cluster (e.g. via duckdns, route53, cloudflare). For tutorial purposes that's overkill — we'll use a hostname that resolves to loopback by convention.

`*.localhost` is reserved by RFC 6761 and is supposed to always resolve to `127.0.0.1` on every machine. Most modern Linux distros honor this via NSS; if yours doesn't, you can always add an `/etc/hosts` entry. We'll use `tutorial.localhost`.

Quick check:

```shell
getent hosts tutorial.localhost
```

If it returns `127.0.0.1 tutorial.localhost` you're set. If it returns nothing, add the entry:

```shell
echo '127.0.0.1 tutorial.localhost' | sudo tee -a /etc/hosts
```

## 3. Pre-flight: make sure the cluster is up

Before creating the Ingress, you need the app actually running. From chapter 7/8:

```shell
cd /home/dev/skaffold-helm-tutorial && skaffold run -d localhost:5001 -p dev
```

(Use `-p dev` if you want hot-reload via webpack-dev-server. Without it you run the production nginx-served build from chapter 8.)

Wait for `kubectl get pod` to show `frontend-...`, `api-0`, `postgresql-db-0` all `Running`.

## 4. Create the Ingress yaml

The frontend lives at `/` (everything that's not an API call). The API answers on three paths: `/time`, `/counter`, `/settings`. Each path has to be a separate rule with `pathType: Prefix` (Traefik supports `Prefix`, `Exact`, and `ImplementationSpecific`; we stick with the portable `Prefix`).

Create a new file `ingress.yaml` anywhere convenient (tutorial root or `/tmp`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: traefik
  rules:
    - host: tutorial.localhost
      http:
        paths:
          - path: /time
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /counter
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /settings
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

Apply:

```shell
kubectl apply -f ingress.yaml
```

A few things to call out:

* **`ingressClassName: traefik`** — replaces the deprecated annotation `kubernetes.io/ingress.class: "nginx"` (or `"traefik"`) used in the original tutorial. The IngressClass approach has been the recommended way since k8s 1.18.
* **No nginx annotations** — the original `nginx.ingress.kubernetes.io/use-regex: "true"` is gone. Traefik doesn't understand it. We use prefix matching instead.
* **Path order matters when paths overlap** — Traefik (and most controllers) match the longest-prefix first, so `/time` beats `/` even though both technically match `/time`. Documented behaviour, no special config needed.

## 5. Test it

The host port mapping from chapter 3 was 8080 → 80 (inside the cluster node container). So the URL the browser sees is `http://tutorial.localhost:8080`. Quick test:

```shell
curl -s http://tutorial.localhost:8080/time
```

Should print something like `{"current":"22/05/2026 11:32:14"}`. The request hit Traefik on `:8080`, Traefik matched `Host: tutorial.localhost` + path `/time` → forwarded to the `api` Service → which load-balances to the `api-0` pod.

```shell
curl -sI http://tutorial.localhost:8080/
```

Should answer `HTTP/1.1 200 OK` with `Content-Type: text/html` — the frontend's index.html.

In a browser, `http://tutorial.localhost:8080/` should load the Vue app. (VS Code Remote-SSH forwards 8080 to your laptop automatically.)

## 6. The HelloWorld.vue URL trap

Click the "request time" button in the browser. Probably doesn't work.

Open `frontend/src/components/HelloWorld.vue`. You'll see:

```javascript
axios.get('http://localhost:9999/time')
```

The frontend hardcodes `localhost:9999` for the API. That URL is correct when using `skaffold dev`'s port-forward block, but not when accessing via the Ingress at `tutorial.localhost:8080`.

The "obvious" fix is to change it to `http://tutorial.localhost:8080/time`. That works, but it's brittle — every developer's local hostname becomes a code edit. The **portable** fix is to use a **relative URL**:

```javascript
axios.get('/time')
```

Now the browser makes the request relative to the page it loaded. If the page loaded from `http://tutorial.localhost:8080`, the API call goes to `http://tutorial.localhost:8080/time`. If it later loaded from `https://app.example.com`, the call goes to `https://app.example.com/time`. No URL is baked into the JS bundle.

Apply that change in `HelloWorld.vue` for both `getTime` and `getCounter` calls (drop the `http://localhost:9999` prefix from both).

```shell
skaffold dev -d localhost:5001 -p dev
```

…lets webpack-dev-server hot-reload the change. Click the button again → time should now show.

## 7. Wait, why doesn't templating the URL in Helm fix it?

A natural next thought: "I'll make `axios.get('http://{{ .Values.ingress.host }}/time')` and override it per environment."

It doesn't work, and the reason is fundamental to how front-end deployment differs from back-end deployment.

Helm templating runs at **`helm install` time**. It substitutes values into YAML manifests and ships them to Kubernetes. **Helm never touches your Vue source code.** Your `HelloWorld.vue` is baked into the frontend docker image at **`docker build` time** — long before Helm has anything to say about deployment.

So:

```
Build time      ─── Vue source compiled into static JS bundle
                    (URL strings are now baked in)
                          ↓
Push time       ─── Image pushed to registry
                          ↓
Helm render     ─── Templates → YAML (operating on Pod specs, not JS code)
                          ↓
Cluster apply   ─── Pods scheduled, frontend serves the pre-built JS
                          ↓
Runtime         ─── Browser executes the JS, sees the URL strings
                    that were baked in at build time
```

**The cluster never sees JS source code.** The Helm chart can't possibly templatize something that doesn't exist by the time it runs.

Workarounds (each with trade-offs):

* **Relative URLs** (what we did) — drop the host entirely, browser figures it out.
* **Runtime config injection** — have the frontend fetch a tiny `/config.json` from the server at startup; nginx serves it from a ConfigMap mounted as a file. Decouples build from runtime config, costs an extra HTTP round-trip on page load.
* **window.\_\_ENV\_\_ injection at HTML render** — the page template gets a small `<script>` block injected by the server with environment-specific values. Slightly clever, slightly cursed.
* **Just rebuild the image per environment** — different docker image per stage. The simplest if you have CI/CD.

Relative URLs work for ~90% of SPAs talking to a sibling API. Reach for the others only if you genuinely need cross-origin or have multiple backends to swap between.

## 8. Make the host configurable in the chart (optional, bonus)

You probably noticed `ingress.yaml` is currently a standalone file, not part of the Helm chart. Real apps put the Ingress in the chart.

### Edit `myapp/values.yaml`

Add an `ingress:` block alongside the existing top-level keys:

```yaml
ingress:
  enabled: true
  host: tutorial.localhost
```

### Create `myapp/templates/ingress.yaml`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  ingressClassName: traefik
  rules:
    - host: {{ .Values.ingress.host | quote }}
      http:
        paths:
          - path: /time
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /counter
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /settings
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
{{- end }}
```

Delete the standalone ingress.yaml (kubectl will remove it):

```shell
kubectl delete -f ingress.yaml
```

Re-run skaffold to deploy the chart-managed Ingress:

```shell
skaffold run -d localhost:5001 -p dev
```

Now `tutorial.localhost:8080` works AND you can override the host via `--set ingress.host=...` for staging/prod. (Frontend will still need relative URLs, since the hostname can't sneak into the JS bundle.)

## 9. Bonus: TLS termination with cert-manager

You installed cert-manager + a self-signed `ClusterIssuer` in chapter 3 §7. To make the Ingress serve HTTPS:

Add to `myapp/templates/ingress.yaml`, inside `spec:`:

```yaml
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - {{ .Values.ingress.host | quote }}
      secretName: tutorial-tls
  rules:
    # ... unchanged
```

And inside `metadata:`:

```yaml
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-ca-issuer
```

cert-manager sees the annotation, generates a cert signed by your local CA, stores it in the Secret `tutorial-tls` in the same namespace. Traefik reads the Secret and serves HTTPS on `:8443`.

```shell
curl -k https://tutorial.localhost:8443/time
```

`-k` because the self-signed CA isn't in `curl`'s trust store (you trusted it on the host in chapter 3, but `curl` uses its own bundle). For browsers, you'd want to install the CA in the browser's trust store.

In production, replace `selfsigned-ca-issuer` with a `letsencrypt-prod` ClusterIssuer (cert-manager's [getting-started docs](https://cert-manager.io/docs/configuration/acme/) cover it in detail) — same Ingress yaml, real public certs, zero extra config.

## 10. Cleanup before chapter 10

```shell
skaffold delete -d localhost:5001 -p dev
```

Or just `helm uninstall myapp`. The PVC for postgres will linger by design — `kubectl delete pvc postgresql-db-disk-postgresql-db-0` if you want a clean slate.

## Review questions

### 1. The Ingress, the Service, and the Pods it forwards to — must they all be in the same namespace? Why or why not?

<details>
  <summary>Answer</summary>

**Ingress and Service: yes, same namespace.** An Ingress rule's `backend.service.name` resolves only within the Ingress's own namespace. Cross-namespace Service references aren't supported in the standard `Ingress` resource (cross-namespace routing requires the newer Gateway API or controller-specific extensions like Traefik's `IngressRoute` CRD).

**Service and Pods: yes, same namespace.** A Service selects Pods by label match, and that selection only applies within the Service's namespace.

**Practical implication:** if you put cert-manager in `cert-manager`, your app in `default`, and you want a Certificate, the `Certificate` resource lives in `default` (where the Ingress lives), references the `ClusterIssuer` (cluster-scoped, accessible from anywhere), and cert-manager writes the resulting Secret in `default`.

</details>

### 2. Why doesn't path order matter in the YAML, even though we have `/time`, `/counter`, `/settings`, and `/` in that order?

<details>
  <summary>Answer</summary>

Most ingress controllers (including Traefik and nginx-ingress) match `pathType: Prefix` rules by **longest prefix wins**, regardless of where they appear in the YAML. So `/time` always beats `/` for a request to `/time/now`, even though `/` also matches.

This is documented behaviour in the [k8s Ingress spec](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types). YAML order is irrelevant. (Different from regex routing in `nginx.ingress.kubernetes.io/use-regex` where order *does* matter and a too-greedy regex can swallow everything.)

If two rules tie on prefix length (e.g. `/api/v1` and `/api/v1` both literally), behaviour is implementation-specific — don't rely on it. Use distinct paths.

</details>

### 3. Why does TLS termination usually happen at the Ingress, not at the pod?

<details>
  <summary>Answer</summary>

Three reasons:

1. **Cert management is centralized.** One place (the Ingress) gets a real cert via cert-manager; every backend pod stays plain HTTP internally. Renewing 50 certs per Service vs renewing 1 at the Ingress is a real time saving.
2. **Pods don't need TLS libraries.** Your FastAPI app doesn't open port 443, doesn't load OpenSSL, doesn't speak ALPN. Smaller image, less attack surface, simpler code.
3. **Inspection at the edge.** Traefik (or nginx-ingress) can inspect the request — for logging, WAF rules, rate limiting, auth — only if it has the decrypted bytes. Terminating TLS at the edge enables those features.

The trade-off: traffic from the Ingress to the backend pod is plain HTTP (within the cluster network). On a single-tenant cluster behind a firewall that's fine. On a multi-tenant cluster you'd add a service mesh (Istio, Linkerd) for pod-to-pod mTLS so even internal traffic is encrypted. Different problem, different solution.

</details>

### 4. The `HelloWorld.vue` URL is hardcoded. Why doesn't templating it in Helm fix it? What's the right fix?

<details>
  <summary>Answer</summary>

Helm runs at **`helm install` time**, operating on YAML manifests sent to the Kubernetes API server. The Vue source code is compiled into a static JS bundle at **`docker build` time** — long before Helm exists in the conversation. The browser executes that pre-built JS at **runtime**, by which point all the URL strings are frozen.

So Helm can never see, never modify, the strings inside your built JS bundle. Three stages of compilation, three different contexts.

Right fixes:

* **Relative URLs in the JS** — `axios.get('/time')` instead of `axios.get('http://X:Y/time')`. The browser uses the page's origin. Cost: zero. Works for ~90% of SPAs.
* **Runtime config injection** — the page fetches `/config.json` on load, which nginx serves from a ConfigMap mounted as a file. Helm renders the ConfigMap content; the frontend reads it. Cost: one extra HTTP round trip.
* **window.\_\_ENV\_\_ at HTML render** — server-side rendering inserts a small `<script>` with env-specific values into `index.html`. Requires the frontend to be served by something smarter than static nginx.

The general lesson: **anything compiled into your frontend bundle is configured at *build time*, not at *deploy time***. Treat Helm values as backend-only.

</details>

### 5. What is `IngressClass` and why does it matter? What happens if you remove `ingressClassName: traefik`?

<details>
  <summary>Answer</summary>

An **`IngressClass`** is a cluster-scoped resource that names an ingress controller. When you install Traefik via its helm chart (chapter 3), it creates an `IngressClass` named `traefik`. Same for nginx-ingress, HAProxy, etc.

`ingressClassName` on an `Ingress` says **"only this controller should handle me."** If you have multiple controllers in the cluster (e.g. nginx for internal, Traefik for public), the field disambiguates which one watches the resource.

What if you remove it:

* If exactly one IngressClass has `ingressclass.kubernetes.io/is-default-class: "true"` set → that one picks up the Ingress. Some helm charts set this by default.
* If no default class exists → the Ingress is ignored by all controllers, no traffic flows, no error message anywhere. Frustrating to debug.
* If multiple controllers fight for the default → undefined behaviour.

Always set `ingressClassName` explicitly in production. (The old `kubernetes.io/ingress.class` annotation is deprecated but still recognized for backwards compat.)

Inspect:

```
kubectl get ingressclass
```

</details>

### 6. Could you swap from Traefik to nginx-ingress without changing your `Ingress` YAML? What would need to change?

<details>
  <summary>Answer</summary>

**Mostly yes** — the `Ingress` resource is standardized, and the YAML you wrote in §4 has zero Traefik-specific bits. To swap:

1. `helm uninstall traefik -n traefik` (or leave it running alongside, ingress controllers don't conflict if they have different IngressClass names).
2. `helm install nginx-ingress ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace` (with similar `nodeSelector` / `tolerations` patches as the Traefik install).
3. Change `ingressClassName: traefik` → `ingressClassName: nginx` in your Ingress yaml.

That's it. The path rules, backend Services, TLS block all stay the same.

What you'd *lose*: any Traefik-native features used via `IngressRoute` CRDs (middlewares, TCP/UDP routing, advanced load-balancing). Conversely, any nginx-only annotations you'd written would have to be ported to Traefik equivalents.

The standard `Ingress` resource as a portability surface is the design lesson the kind variant is built around. The original tutorial's reliance on `nginx.ingress.kubernetes.io/use-regex` and similar annotations breaks this portability — that's exactly why this variant exists.

</details>
