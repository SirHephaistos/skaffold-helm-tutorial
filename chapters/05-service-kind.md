# Chapter 5 (alternative): Services — kind variant

> Drop-in replacement for [05-service.md](05-service.md). The actual content is **near-identical** — Services are a pure Kubernetes concept, untouched by which distro you run. This chapter exists in the kind variant only to keep the chapter numbering parallel and to add spoiler-answer review questions at the end.

## 1. The problem Services solve

After chapter 4 you have:

* a **Deployment** running `frontend`
* a **StatefulSet** running `postgresql-db` (one replica)
* matching pods, each with its own IP address

The frontend container would like to reach the database. It needs an **address** — a hostname or IP. The pod's IP looks like it should work:

```shell
kubectl get pod postgresql-db-0 --template '{{printf "%s\n" .status.podIP}}'
```

Use that IP to connect from inside the frontend pod. Open a shell:

```shell
kubectl exec -it deployment/frontend -- /bin/sh
```

Then inside the shell:

```shell
apk add postgresql-client
```

```shell
PGPASSWORD=astrongdatabasepassword psql -h <db-pod-ip> -U postgres -c '\dt'
```

You should see the `counter` table from the init script. So pod-to-pod IP routing works.

### Now the problem

Delete the postgres pod:

```shell
kubectl delete pod postgresql-db-0
```

The StatefulSet immediately replaces it. Look at the new pod's IP:

```shell
kubectl get pod postgresql-db-0 --template '{{printf "%s\n" .status.podIP}}'
```

**Different IP.** The frontend now can't find the database — its hardcoded IP is stale.

This isn't a bug, it's how pods work. Every pod lifecycle event (delete, evict, reschedule, node restart) gets a fresh IP. Building anything on top of pod IPs directly = broken design.

![statefulset](../imgs/statefulset.png)

## 2. The Service object

A **Service** gives a group of pods a stable name + stable virtual IP. It also load-balances across them if there are multiple matches. Save as `postgres-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
spec:
  selector:
    app: postgresql-db
  ports:
    - port: 5432
```

```shell
kubectl apply -f postgres-service.yaml
```

Three things going on:

* `selector: { app: postgresql-db }` — find every pod with the label `app: postgresql-db` (set by our StatefulSet in chapter 4). Live tracked: pods that come and go get added/removed automatically.
* `ports[].port: 5432` — the Service listens on TCP 5432. With no `targetPort` set, traffic forwards to **the same port** on each matching pod. (You can override with `targetPort: 5433` if the pod listens elsewhere.)
* Service type defaults to **`ClusterIP`** — reachable only from inside the cluster. The Service gets a virtual IP from a cluster-internal range. CoreDNS auto-creates a DNS record `postgres-db.<namespace>.svc.cluster.local`, abbreviated as just `postgres-db` from pods in the same namespace.

![statefulset-with-service](../imgs/statefulset-with-service.png)

## 3. Use it

Re-open a shell in the frontend pod:

```shell
kubectl exec -it deployment/frontend -- /bin/sh
```

```shell
apk add postgresql-client
```

```shell
PGPASSWORD=astrongdatabasepassword psql -h postgres-db -U postgres -c '\dt'
```

It works. Now delete the database pod again:

```shell
kubectl delete pod postgresql-db-0
```

Wait a few seconds, retry the `psql` call. **Still works.** The Service tracks the new pod's IP automatically.

This is the right way for components to find each other in k8s: **by Service name, never by Pod IP.**

## 4. Cleanup before chapter 6

In chapter 6 we re-create everything via Helm. Wipe the manual yaml objects first:

```shell
kubectl delete deployment frontend
```

```shell
kubectl delete statefulset postgresql-db
```

```shell
kubectl delete configmap postgresql-initdb-config
```

```shell
kubectl delete service postgres-db
```

Don't delete the PVC unless you want to wipe the database data:

```shell
kubectl get pvc
```

If you want a fully fresh start for chapter 6:

```shell
kubectl delete pvc postgresql-db-disk-postgresql-db-0
```

## Review questions

### 1. Why does a pod get a new IP every time it restarts? Couldn't Kubernetes just keep the old one?

<details>
  <summary>Answer</summary>

Two reasons rooted in how the networking is implemented:

1. **Pod IPs come from a CNI plugin's address pool.** When a pod is destroyed, its IP goes back to the pool. When a new pod is created, the next IP in the pool is handed out — there's no "remember which pod had which IP" logic, because pods are designed to be ephemeral.
2. **Pods can be rescheduled to different nodes.** Each node typically gets its own CIDR slice of the cluster's pod IP range (e.g. node A has `10.244.0.0/24`, node B has `10.244.1.0/24`). A pod that lands on a different node *must* get an IP from that node's slice. Keeping the old IP across nodes would require routing tricks no CNI does by default.

The whole design assumes IPs are throwaway. That's exactly why Services exist — they layer a stable virtual IP + DNS name on top of the ephemeral pod IPs.

</details>

### 2. The Service has no IP of its own when you create it — kubectl shows a `CLUSTER-IP` like `10.96.123.45`. Where does that come from, and what listens on it?

<details>
  <summary>Answer</summary>

When a `Service` of type `ClusterIP` is created, the API server allocates one IP from the cluster's `service-cluster-ip-range` (set by the cluster admin, e.g. `10.96.0.0/12`). This is a **virtual** IP — no network interface has it.

The IP becomes reachable because **kube-proxy on every node** programs iptables (or IPVS) rules saying:

> "Any packet to `10.96.123.45:5432` should be DNAT'd to one of these pod IPs: `[10.244.0.5:5432, 10.244.1.7:5432, ...]`"

Pick-one logic = simple round-robin (or hash, in IPVS mode). When a pod is added or removed from the Service's endpoint set (because a pod was created/deleted/became Ready/became NotReady), kube-proxy rewrites the iptables rules within seconds.

So nothing actually *listens* on the Service IP. It's a virtual address that the kernel rewrites at packet time. That's why it's blazing fast and survives any number of pod restarts.

You can see the underlying pod IPs the Service forwards to:

```
kubectl get endpoints postgres-db
```

Or its newer name:

```
kubectl get endpointslices -l kubernetes.io/service-name=postgres-db
```

</details>

### 3. There are four Service types: `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`. Which one would you use to expose a development tool to your team over the office network?

<details>
  <summary>Answer</summary>

Most likely **`LoadBalancer`** (cloud) or **`NodePort`** (bare metal), depending on where the cluster runs.

Recap:

| Type | What it does | Reachable from |
|---|---|---|
| `ClusterIP` (default) | Virtual IP inside the cluster only | Pods, host with `kubectl port-forward` |
| `NodePort` | Opens a port (30000–32767 by default) on **every node** | Anyone who can reach a node's IP on that port |
| `LoadBalancer` | Asks the cloud provider for an external LB → routes to NodePorts | Public internet (or VPC, depending on annotations) |
| `ExternalName` | DNS CNAME to an outside host | Pods (resolves to an external DNS name, no proxying) |

For a team tool on the office network:

* On a cloud cluster → `LoadBalancer` is one line, gets you a real external IP/DNS, locked down via security groups.
* On bare-metal → `NodePort` exposes a high port on every node. Hand out `http://any-node-ip:31234/`. Or install MetalLB which lets `LoadBalancer` work on bare metal.
* In **this tutorial's** kind setup → neither works directly. We use `hostPort` on the ingress controller (chapter 3 step 3) and route through Traefik, which serves the same purpose.

`ClusterIP` is wrong here — it's intentionally invisible to anyone outside the cluster.

`ExternalName` is for a different problem: making an external host (e.g. `db.amazonaws.com`) look like an in-cluster Service so pods can use a friendly name.

</details>

### 4. A "headless Service" is created by setting `clusterIP: None`. When is that useful?

<details>
  <summary>Answer</summary>

Two main cases.

**Stable per-pod DNS for a StatefulSet.**

A headless Service has no virtual IP and no kube-proxy magic. Instead, CoreDNS returns the **set of pod IPs** directly to the client when it resolves the Service name. Combined with a StatefulSet, you also get one DNS A record *per pod*: `postgresql-db-0.postgres-db.default.svc.cluster.local`, `postgresql-db-1.postgres-db...`, etc.

Why care? Some software needs to talk to a specific replica (e.g. write to the primary postgres, read replicas separately; or each node of a clustered cache needs to be addressed individually). A normal Service load-balances; a headless Service preserves identity.

The Postgres operator (chapter 10) uses headless Services for exactly this reason.

**Client-side load balancing.**

If the app embeds its own load-balancing logic (typical for gRPC, where channels keep persistent connections), you don't want kube-proxy interfering — you want a list of all backend IPs and the client picks. A headless Service exposes the raw list via DNS.

</details>

### 5. What happens if a Service's selector matches **zero** pods? Or **pods that aren't Ready**?

<details>
  <summary>Answer</summary>

**Zero matches** → the Service is created and gets a Cluster IP, but its endpoints object is empty. Any packet sent to the Service IP is dropped (connection refused). DNS still resolves, traffic just goes nowhere.

This is a common bug source. Diagnosis:

```
kubectl get endpoints <svc-name>
```

If `ENDPOINTS` column shows `<none>`, your selector doesn't match what you think it matches. Common causes:

* typo in label key/value (`app: postgresql-db` vs `app: postgresql_db`)
* selector pointing at the wrong namespace (Services and the Pods they select must be in the same namespace)
* pods exist but in `Pending` (never got scheduled) — they have no IP yet
* pods exist but all `NotReady` (failing their readiness probe) — by default, they're excluded from the endpoint set

**Pods Running but NotReady** → they appear in `endpoints` (under `notReadyAddresses`), but kube-proxy still excludes them. Traffic only goes to `Ready` pods. This is intentional: readiness probes let your app tell k8s "I'm alive but not yet ready to serve" (e.g. while warming a cache, loading config). The Service stops sending traffic until you're Ready again.

If you want traffic to also reach not-ready pods (rare), set `spec.publishNotReadyAddresses: true` on the Service.

</details>

### 6. Why doesn't a Service need a port mapping like `8080:80`? It just says `port: 5432`.

<details>
  <summary>Answer</summary>

Because **the Service IP is not the host's IP** — there's no host-vs-container port translation to do.

When you `docker run -p 8888:80`, you're mapping a *host* port (the host has many other ports) to a *container* port (the container has its own private namespace). That mapping is needed because both sides have port number scarcity to worry about.

A `Service` is just a name + virtual IP that maps to pod IPs. The Service can listen on **any port** without conflicting with the pods' ports or anything else — the Service IPs are a private range carved out for this purpose. If the pod's app happens to listen on 5432, you can expose the Service on 5432 (`port: 5432`, `targetPort` defaults to the same), or on any other port:

```yaml
ports:
  - port: 80           # Service listens on :80
    targetPort: 5432   # forwards to pod's :5432
```

This is useful when you want a clean front-door port (`80` for everything) regardless of what the pods actually run on internally. Different problem from `docker -p`.

</details>
