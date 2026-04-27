# Chapter 3 (alternative): Installing the cluster with **kind** instead of k3s

> This is a drop-in replacement for [03-install-k3s.md](03-install-k3s.md). Use **this** chapter if you already have other services running on host ports 80/443 (a reverse proxy, a hosted site, anything that listens on the standard HTTP/HTTPS ports) and you want a Kubernetes cluster that doesn't fight them. From chapter 4 onwards everything is identical — Helm and Skaffold don't care which distro is underneath.

## What we're going to build

The end-state is the same as the original chapter 3:

* a real Kubernetes cluster on the local machine
* an **ingress controller** (nginx) so we can later route HTTP traffic
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

* **8080** on the host → 80 inside the cluster (HTTP for nginx-ingress)
* **8443** on the host → 443 inside the cluster (HTTPS for nginx-ingress)

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
    # This label is what nginx-ingress uses to find a node it's allowed to schedule on.
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
kubectl run busybox-test --image=localhost:5001/busybox:test --rm -it --restart=Never --image-pull-policy=Always -- sh -c 'echo "hello from inside the cluster"'
```

If it prints `hello from inside the cluster`, the cluster pulled your image through the redirect successfully. If it errors with `ErrImagePull` or hangs, run `kubectl describe pod busybox-test` and look at the `Events:` section.

---

## 6. Install the nginx ingress controller

Kind has its own preset manifest for nginx-ingress that already knows about the `ingress-ready=true` node label and the host port mapping we set in step 3. So we don't even need helm here:

```shell
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
```

Wait for it to come up:

```shell
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

When that returns, ingress is alive on the host at:

* `http://localhost:8080`
* `https://localhost:8443` (will use a self-signed cert for now)

Quick sanity probe:

```shell
curl -I http://localhost:8080/
# should answer 404 from nginx — that's correct, no ingress rules exist yet
```

You'll also see two `ingress-nginx-admission-*` pods in `Completed` state — that's expected. They're one-shot Kubernetes Jobs that bootstrap the validating webhook and exit. A `Completed` Job is healthy; it should *not* be `Running` long-term.

---

## 7. Install cert-manager + a local CA + a ClusterIssuer

This part is **identical** to the original chapter 3 — it's a Kubernetes concept, not a k3s concept. We need it because:

* later chapters create TLS certificates for ingress hostnames
* chapter 10 demonstrates operators, and cert-manager *is* an operator (CRD + controller) — a great real-world specimen

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
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
kubectl get pods -n ingress-nginx

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

## 9. Differences vs the original chapter — important to know going forward

When you read chapter 4 onwards, replace the following on the fly:

| Original (k3s) | This setup (kind) |
|---|---|
| `registry.kube-public/myfrontend` | `localhost:5001/myfrontend` |
| `registry.kube-public/myapi` | `localhost:5001/myapi` |
| `imagePullSecrets: [registry-creds]` | **omit it** — our registry is plain HTTP, no auth |
| `host: frank-test.duckdns.org` (chapter 9) | `host: tutorial.localhost`, accessed via `curl http://localhost:8080 -H "Host: tutorial.localhost"` |
| `kubectl create secret docker-registry registry-creds ...` | not needed |

Everything else (Deployments, StatefulSets, Services, Helm charts, Skaffold config, Flux gitops) is **byte-for-byte identical**.

When chapter 7 introduces skaffold's `-d registry.kube-public` flag, you'll write:

```
skaffold run -d localhost:5001
```

instead.

---

## 10. Cleanup / restart cheat sheet

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

Before moving on to chapter 4, make sure you can answer:

* What is a **namespace** and why do we have one per software component?
* What does a **ClusterIssuer** do, and how is it different from a `Certificate`?
* What is an **ingress controller** and why doesn't a bare `Ingress` object work without one?
* Why does the cluster need its own way to *pull* images, separate from the way the host *pushes* them?
* What does `kubectl apply` actually do at the API level — and how does that differ from `kubectl create`?

These are the questions the original chapter 3 leaves dangling on purpose. The k3s-vs-kind choice doesn't change any of them.
