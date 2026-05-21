# Chapter 3 (alternative): Installing the cluster with **kind** instead of k3s

> This is a drop-in replacement for [03-install-k3s.md](03-install-k3s.md). Use **this** chapter if you already have other services running on host ports 80/443 (a reverse proxy, a hosted site, anything that listens on the standard HTTP/HTTPS ports) and you want a Kubernetes cluster that doesn't fight them. From chapter 4 onwards everything is identical — Helm and Skaffold don't care which distro is underneath.

## What we're going to build

The end-state is the same as the original chapter 3:

* a real Kubernetes cluster on the local machine
* an **ingress controller** (Traefik) so we can later route HTTP traffic
* a **container registry** so docker images we build are reachable from inside the cluster
* a **certificate manager** with a local CA so we can hand out TLS certs
* the client tools to drive all of this (`kubectl`, `helm`, `skaffold`, `k9s`, `kubectx`)

The only thing that changes is *how the cluster is hosted*:

| | Original (k3s) | This chapter (kind) |
|---|---|---|
| Where it runs | Host-level systemd service | Inside Docker containers |
| Binds host ports 80/443 | Yes (via nginx-ingress) | No (we map to 8080/8443) |
| Conflicts with existing :80/:443 services | Yes | No |
| Image registry | In-cluster `registry.kube-public` over TLS | Sidecar `kind-registry` over plain HTTP |
| Persistent across reboots | Yes (systemd) | No (you re-create the cluster) |

For a learning environment that's a great trade. The cluster is fully throwaway: when something breaks, `kind delete cluster && kind create cluster` and you're back in 30 seconds.

> Why "kind"? It stands for **K**ubernetes **IN** **D**ocker. Each cluster node is just a Docker container running a real kubelet. It's the upstream Kubernetes project's own way to test Kubernetes itself.

---

## 1. Install the client tools

These are the same tools the original chapter installs — they talk to *any* cluster, not just k3s. Skip any you already have.

```shell
# kubectl: the canonical CLI for the Kubernetes API
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin
rm -f kubectl

# helm: package manager for Kubernetes manifests
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

# skaffold: the build/push/deploy glue we'll meet in chapter 7
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/
rm -f skaffold

# k9s: a TUI for Kubernetes
curl -Lo k9s.tgz https://github.com/derailed/k9s/releases/download/v0.32.5/k9s_Linux_amd64.tar.gz
tar -xf k9s.tgz k9s && sudo install k9s /usr/local/bin/
rm -f k9s.tgz k9s

# kubectx: quick context/namespace switcher
curl -Lo kubectx https://github.com/ahmetb/kubectx/releases/download/v0.9.3/kubectx
sudo install kubectx /usr/local/bin/
rm -f kubectx

# kind: the cluster itself
curl -Lo kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
sudo install kind /usr/local/bin/
rm -f kind
```

Skaffold's file-watching mode (`skaffold dev`) opens a *lot* of inotify handles. Bump the kernel limits so we don't hit them later:

```shell
cat << END | sudo tee -a /etc/sysctl.conf
fs.inotify.max_user_watches=1048576
fs.inotify.max_user_instances=1000000
END
sudo sysctl --system
```

Sanity check:

```shell
kubectl version --client
helm version
skaffold version
kind version
```

---

## 2. Decide the cluster's port strategy

This is the one decision that differs from the original chapter.

A kind cluster is a regular Docker container. By default it doesn't publish anything to the host. We have to tell it which container ports to map onto host ports — same rules as `docker run -p`.

For this tutorial we want:

* **8080** on the host → 80 inside the cluster (HTTP for the ingress controller)
* **8443** on the host → 443 inside the cluster (HTTPS for the ingress controller)

Why those numbers? Host ports 80 and 443 are likely already taken on any machine that hosts other web services — a reverse proxy, a website, a media server, anything with an HTTP front-end. Picking the `+8000` variants keeps the two stacks side-by-side without conflict, and they're a common convention for "secondary HTTP listener on this box."

Later on, if you want to expose tutorial workloads publicly, you can have your existing reverse proxy forward selected hostnames into `127.0.0.1:8080`. We'll get to that in chapter 9 if you care about it.

---

## 3. Create a kind config file

Create `~/kind-tutorial.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: tutorial
nodes:
  - role: control-plane
    # This label is what the ingress controller (Traefik) uses to find a node it's allowed to schedule on.
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    # Map host ports onto the node's network namespace.
    # Anything inside the cluster listening on these container ports
    # becomes reachable on the host at 8080 / 8443.
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
      - containerPort: 443
        hostPort: 8443
        protocol: TCP
# Tell containerd (the container runtime inside the kind node) that whenever
# it sees an image starting with `localhost:5001/...`, it should fetch it
# from the registry container we'll create in step 5.
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5001"]
      endpoint = ["http://kind-registry:5000"]
```

A few things to point out:

* `name: tutorial` → this becomes the kubectl context name. You'll see it as `kind-tutorial`.
* The `node-labels: "ingress-ready=true"` line lets us pin the ingress controller pod to this exact node — important because `extraPortMappings` only works for the node that has them.
* The `containerdConfigPatches` block is the magic that lets us pretend the registry is at `localhost:5001` from both host and cluster. We'll set up the actual registry container next.
* You may notice we use port **5001**, not 5000, for the registry. That's a convention from kind's official docs to dodge a conflict on macOS, where Apple's AirPlay Receiver service binds 5000 by default since macOS Monterey. On Linux 5000 would work fine, but we keep 5001 here so this chapter matches what you'll find in upstream kind documentation if you ever need to look something up.

---

## 4. Create the cluster

```shell
kind create cluster --config ~/kind-tutorial.yaml
```

This spins up:

* one Docker container called `tutorial-control-plane` running the API server, scheduler, controller-manager, etcd, and kubelet
* it auto-writes a kubeconfig stanza to `~/.kube/config` and switches your current context to `kind-tutorial`

Verify:

```shell
kubectl cluster-info
kubectl get nodes
```

You should see one node, `tutorial-control-plane`, in `Ready` state.

> **kubeconfig already-exists case:** if you already have a `.kube/config` from another cluster, kind will *merge* its stanza in (good). To hop between clusters, use `kubectx` (`kubectx kind-tutorial`).

---

## 5. Set up the in-cluster image registry

We need somewhere to store the docker images we'll build in chapters 4 and 7. Kind doesn't ship a registry — instead the standard pattern (from kind's own docs) is to run a tiny `registry:2` container next to the cluster.

```shell
# Run a registry, only listening on the host's loopback so it isn't exposed publicly.
# Container name is "kind-registry" so containerd in the cluster can resolve it.
docker run -d --restart=always \
  -p 127.0.0.1:5001:5000 \
  --name kind-registry \
  registry:2

# Connect the registry to the same docker network the kind cluster uses.
# Without this, the kind node container can't reach the registry container.
docker network connect kind kind-registry || true
```

Now both sides can reach the registry through the same name `localhost:5001`:

* **From the host** (when you `docker push`): hits `127.0.0.1:5001` → forwarded to the registry container's port 5000.
* **From inside the cluster** (when containerd pulls): the `containerdConfigPatches` we wrote earlier rewrites `localhost:5001` to `http://kind-registry:5000`, which it resolves through docker's embedded DNS on the kind network.

It looks weird but it's a tested pattern. The end result is that **both sides use the exact same image reference**, e.g. `localhost:5001/myfrontend:latest` — no fiddling with two names.

(Optional, mostly cosmetic: tell tools like Skaffold that this registry exists, by writing the standard `local-registry-hosting` ConfigMap.)

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:5001"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF
```

Test it from the host:

```shell
docker pull busybox:latest
docker tag busybox:latest localhost:5001/busybox:test
docker push localhost:5001/busybox:test
```

If the push prints `test: digest: sha256:... size: ...` you're done. Modern Docker treats `localhost` and `127.0.0.1` as insecure-trusted by default, so plain HTTP just works.

> **If you see** `http: server gave HTTP response to HTTPS client` — you have an older Docker that doesn't auto-trust localhost. Edit `/etc/docker/daemon.json` (create it if missing):
> ```json
> {
>   "insecure-registries": ["localhost:5001"]
> }
> ```
> Then `sudo systemctl restart docker`. Be aware: restarting the Docker daemon briefly stops every container on this host. They'll auto-restart if their compose files use `restart: unless-stopped`, but expect a few seconds of downtime on anything else running on this machine. Schedule it for a quiet moment.

Verify the cluster can pull the image you just pushed (this is the real test of the redirect trick):

```shell
kubectl run busybox-test --image=localhost:5001/busybox:test --rm -it --restart=Never --image-pull-policy=Always -- sh -c 'sleep 1; echo "hello from inside the cluster"'
```

If it prints `hello from inside the cluster`, the cluster pulled your image through the redirect successfully. If it errors with `ErrImagePull` or hangs, run `kubectl describe pod busybox-test` and look at the `Events:` section.

> The `sleep 1;` before the `echo` exists only so `kubectl ... -it` has time to wire up its TTY attach before the container exits. Without it, the command still works but kubectl prints a `couldn't attach... falling back to streaming logs` warning. Cosmetic.

---

## 6. Install the Traefik ingress controller

We use Traefik as the cluster's ingress controller. If you already run Traefik elsewhere as a standalone reverse proxy, it's the **same binary** — only the config source is different: this one reads Kubernetes `Ingress` resources from the API server, where the standalone one reads Docker labels or static files. Two independent processes, no shared state.

Install via helm. We use `helm upgrade --install` instead of plain `helm install` so the command stays re-runnable — if a previous attempt left a failed release behind, this updates it in place instead of erroring with "cannot re-use a name."

```shell
helm repo add traefik https://traefik.github.io/charts && helm repo update
```

```shell
helm upgrade --install traefik traefik/traefik -n traefik --create-namespace --set-string "nodeSelector.ingress-ready=true" --set "tolerations[0].key=node-role.kubernetes.io/control-plane" --set "tolerations[0].operator=Exists" --set "tolerations[0].effect=NoSchedule" --set "ports.web.hostPort=80" --set "ports.websecure.hostPort=443" --set "service.type=ClusterIP"
```

What each `--set` does:

* `--set-string "nodeSelector.ingress-ready=true"` → schedule Traefik on the node we labeled in step 3, the one whose `extraPortMappings` actually publishes ports to the host. **Why `--set-string` instead of `--set`:** helm's `--set` auto-converts unquoted `true`/`false` into Go booleans, but Kubernetes' `nodeSelector` schema requires the value to be a string. `--set-string` forces string interpretation.
* `tolerations[0...]` → kind's single-node cluster has only a control-plane node, and control-plane nodes carry a `NoSchedule` taint that normally blocks workload pods. The toleration tells Traefik "I'm fine landing on a control-plane node."
* `ports.web.hostPort=80` / `ports.websecure.hostPort=443` → bind directly to the kind node container's ports 80/443. Kind's `extraPortMappings` then surfaces those as host `localhost:8080` / `localhost:8443`.
* `service.type=ClusterIP` → don't request a `LoadBalancer` (we have no cloud LB provider). External traffic enters via `hostPort` instead.

Wait for it to come up:

```shell
kubectl wait --namespace traefik --for=condition=ready pod --selector=app.kubernetes.io/name=traefik --timeout=180s
```

When that returns, Traefik listens on:

* `http://localhost:8080`
* `https://localhost:8443` (Traefik's auto-generated self-signed cert; we'll replace it with the cert-manager-issued one in chapter 9)

Sanity probe:

```shell
curl -I http://localhost:8080/
# should answer 404 — correct, no Ingress rules defined yet
```

A 404 here is **good**. It proves the request reached Traefik but Traefik has nothing to route it to. Once chapter 9 creates an `Ingress` resource, the same URL will resolve.

> **Why Traefik over nginx-ingress?** Both are valid `Ingress` controllers. Traefik makes the standard k8s `Ingress` object work without nginx-specific annotations (`nginx.ingress.kubernetes.io/use-regex`, etc.), keeping later chapters' YAML portable across controllers. nginx-ingress remains a fine choice — the swap is one helm install away.

---

## 7. Install cert-manager + a local CA + a ClusterIssuer

### What cert-manager is

**cert-manager** is the Kubernetes-native way to issue and renew TLS certificates automatically. If you've used Let's Encrypt with `certbot` on a regular Linux server, cert-manager is the same idea — turn a *request* for "I need an HTTPS cert for `foo.example.com`" into a real, signed certificate file — but driven by Kubernetes objects instead of cron jobs and shell scripts.

The pieces:

* **`Issuer` / `ClusterIssuer`** — *who* should sign certs. Could be Let's Encrypt over ACME, a self-signed CA you control, HashiCorp Vault, etc. `Issuer` is namespace-scoped; `ClusterIssuer` is cluster-wide. In this tutorial we make one `ClusterIssuer` backed by a local self-signed CA, so every namespace can request certs from it.
* **`Certificate`** — *what* cert you want. Hostname(s), validity, which Issuer to ask. cert-manager reconciles each `Certificate` by asking the referenced Issuer for a signed cert.
* **The result** — cert-manager writes the signed cert + private key into a normal Kubernetes **`Secret`** (`type: kubernetes.io/tls`). Your Ingress / Service / Pod consumes that Secret the usual way. Renewal happens automatically before expiry.

Why it matters for this tutorial:

* Later chapters create TLS certificates for ingress hostnames — cert-manager is what makes that one-line.
* Chapter 10 introduces **operators**, and cert-manager *is* a textbook operator: it ships CRDs (`Certificate`, `Issuer`, `ClusterIssuer`, `CertificateRequest`, `Challenge`, `Order`) plus a controller that reconciles them. Installing it now also means by chapter 10 you already have a real operator running to dissect.

This part is **identical** to the original chapter 3 — cert-manager is a Kubernetes concept, not a k3s concept.

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set crds.enabled=true
```

Wait until the three cert-manager pods are running:

```shell
kubectl get pods -n cert-manager
```

Now the local CA (generates a key pair you'll use to sign certificates issued inside the cluster):

```shell
mkdir -p $HOME/kubeca && cd $HOME/kubeca
[[ -f ca.key ]] || openssl genrsa -out ca.key 2048
[[ -f ca.crt ]] || openssl req -x509 -new -nodes -key ca.key -subj "/CN=local_kind" -days 3650 \
  -reqexts v3_req -extensions v3_ca -out ca.crt
```

Upload the CA key+cert into the cluster as a `tls` secret, then create a `ClusterIssuer` that references it:

```shell
kubectl create secret tls ca-key-pair \
   --cert=ca.crt \
   --key=ca.key \
   --namespace=cert-manager

cat << END | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-ca-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: ca-key-pair
END
```

If you get an "Admission Web Hook" error, the cert-manager pods aren't fully up yet — wait 20 seconds and re-apply.

> Same as the original chapter: this is a *self-signed* CA, so browsers and Docker won't trust it without an extra step. Trust it on the host with:
> ```
> sudo cp ca.crt /usr/local/share/ca-certificates/selfsigned-kind.crt
> sudo update-ca-certificates
> ```

You don't need to restart Docker for this CA (we're using HTTP for the registry, not TLS).

---

## 8. Verify the whole stack

```shell
# every pod across every namespace should be Running or Completed
kubectl get pods -A

# the cluster issuer should report Ready=True
kubectl get clusterissuer

# the ingress controller should be Running
kubectl get pods -n traefik

# the registry container should be reachable
curl http://localhost:5001/v2/_catalog
# should answer: {"repositories":[...]}
```

Open k9s once just to get the feel:

```shell
k9s
```

* `:` then type `pods` → list all pods
* `:` then `ns` → list namespaces, hit enter to pin one
* `0` → show all namespaces
* `?` for the keybindings cheatsheet
* `Ctrl-C` to quit

---

## 9. Cleanup / restart cheat sheet

Things go wrong sometimes. These commands are safe to run any time:

```shell
# nuke the cluster, keep the registry contents
kind delete cluster --name tutorial

# re-create from the same config file
kind create cluster --config ~/kind-tutorial.yaml

# nuke the registry too (also wipes the docker images stored in it)
docker rm -f kind-registry

# bring the registry back
docker run -d --restart=always -p 127.0.0.1:5001:5000 --name kind-registry registry:2
docker network connect kind kind-registry
```

If the host reboots, the kind container won't auto-start. Bring it back with:

```shell
docker start tutorial-control-plane
docker start kind-registry
```

---

## What you should now understand (chapter 3's real learning goals)

Before moving on to chapter 4, make sure you can answer the following. Try first; then peek at the spoiler for a check.

### 1. What is a **namespace** and why do we have one per software component?

<details>
  <summary>Answer</summary>

A **namespace** is a virtual partition inside a single Kubernetes cluster. Resource names (a Deployment called `frontend`, a Service called `db`, etc.) are unique only **within** a namespace, not across the cluster. Most workload-type resources (Pods, Deployments, Services, ConfigMaps, Secrets, …) live inside a namespace; a small set of resources are cluster-scoped (Nodes, ClusterIssuer, ClusterRole, StorageClass, …).

Reasons to put each software component (cert-manager, ingress controller, registry, your app) in its own namespace:

* **No name collisions** — two unrelated charts can both name their main Deployment `controller` without conflict.
* **Scoped cleanup** — `kubectl delete namespace foo` deletes everything in that namespace in one shot. Great for "let me just rip out this experiment."
* **RBAC boundaries** — a Role/RoleBinding can grant access to one namespace only. Devs see their own; ops see all.
* **Quotas and limits** — CPU/memory/PVC quotas attach per namespace.
* **Listing hygiene** — `kubectl get pods -n traefik` shows you only Traefik's pods, not 50 unrelated ones.

You *could* put everything in `default`. You shouldn't; on day 30 you won't know what's what.

</details>

### 2. What does a **ClusterIssuer** do, and how is it different from a `Certificate`?

<details>
  <summary>Answer</summary>

These are two cert-manager CRDs that play different roles.

* **`ClusterIssuer`** describes *who* will sign certificates and *how*. It encapsulates a certificate authority — a self-signed CA backed by a secret (our case), Let's Encrypt over ACME, HashiCorp Vault, etc. It's cluster-scoped, so any namespace can reference it. You usually have one per CA you trust.
* **`Certificate`** describes *what* you want — hostnames, key type, validity, and **which Issuer/ClusterIssuer to ask**. cert-manager reconciles each `Certificate` by asking the named issuer to sign a CSR, and stores the resulting cert+key in a regular `kubernetes.io/tls` Secret.

Analogy: a `ClusterIssuer` is a passport office; a `Certificate` is a passport application that names the office it should be sent to. The output (a passport / a TLS secret) is what other resources consume.

There's also `Issuer` (namespace-scoped variant of `ClusterIssuer`). Same shape, just narrower visibility.

</details>

### 3. What is an **ingress controller** and why doesn't a bare `Ingress` object work without one?

<details>
  <summary>Answer</summary>

An **`Ingress`** resource is *data*: a YAML object stored in etcd that says "route `foo.example.com/api` to the `api` Service on port 80, with this TLS secret." It has no behavior of its own.

An **ingress controller** is *code*: a Pod (often a Deployment) running an actual reverse proxy — nginx, Traefik, HAProxy, Istio gateway, etc. — that:

1. Watches the Kubernetes API for `Ingress` objects (and the Services and Endpoints they reference).
2. Translates them into its own routing configuration on the fly.
3. Listens on real ports (host ports via `hostPort`, or a `LoadBalancer` service) so external traffic can actually arrive.

Without a controller, `Ingress` objects sit in etcd, nobody reads them, no traffic flows. Some k8s distros bundle a controller (k3s ships Traefik); cloud-managed clusters often ship one tied to the cloud's load balancer. On kind you install one yourself, which is exactly what step 6 of this chapter does.

</details>

### 4. Why does the cluster need its own way to *pull* images, separate from the way the host *pushes* them?

<details>
  <summary>Answer</summary>

The host's Docker daemon and the cluster nodes' container runtime (containerd, in kind's case) are **two different processes with two different image caches and two different network views**. They don't share state.

* **Host build**: `docker build` stores the image in the *host* daemon's local cache. Cluster nodes have no idea it exists.
* **Cluster pull**: when a Pod is scheduled, the node's containerd reads the image reference, resolves it to a registry URL, downloads via HTTPS, and stores it in *containerd*'s own cache.

The bridge between the two is a **registry** — a process both sides can reach over HTTP(S). The host *pushes* to it (uploads layers); cluster nodes *pull* from it (download layers).

Why the URLs aren't identical from both sides: in our setup the host reaches the registry through Docker's port-publish mapping (`127.0.0.1:5001` → registry container `:5000`); cluster nodes reach the same registry container directly over the `kind` Docker network (`kind-registry:5000`), bypassing port publishing entirely. The `containerdConfigPatches` block in step 3 rewrites `localhost:5001` → `http://kind-registry:5000` on the cluster side so we can use *one* image reference everywhere.

In a real cloud setup, the registry might be ECR / GCR / a private Harbor; same shape — both sides need network access to the same registry URL, possibly with credentials.

</details>

### 5. What does `kubectl apply` actually do at the API level — and how does that differ from `kubectl create`?

<details>
  <summary>Answer</summary>

Both end up POSTing or PATCHing to the same `/api/...` endpoint, but with very different semantics:

* **`kubectl create`** is **imperative**. It says "make this new object now." If the named object already exists, the call fails (409 Conflict). Good for one-shot operations; bad for "I'm going to re-run this script tomorrow."
* **`kubectl apply`** is **declarative**. It says "make the cluster's state match this YAML." Internally:
  1. It computes a diff between the YAML you passed, the **last-applied configuration** stored as an annotation on the live object (`kubectl.kubernetes.io/last-applied-configuration`), and the current live state.
  2. It builds a strategic merge patch from those three inputs and PATCHes the API.
  3. It updates the last-applied annotation to your new YAML.

Consequences:

* `apply` is **idempotent** — re-run with no changes, no-op. Re-run with changes, patches the diff.
* `apply` is the right verb for everything that lives in git (your helm charts, your flux manifests, your CI scripts). It survives manual edits in the middle reasonably well.
* `create` is great in scripts where you genuinely want to fail if something already exists (e.g. one-shot secret creation).

Server-Side Apply (`--server-side`) is the newer variant that lets multiple actors co-own different fields of the same object — relevant for operators and GitOps tools that touch the same resource.

</details>

These are the questions the original chapter 3 leaves dangling on purpose. The k3s-vs-kind choice doesn't change any of them.
