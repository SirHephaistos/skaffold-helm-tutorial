# Chapter 4 (alternative): Going to Kubernetes with our application — kind variant

> Drop-in replacement for [04-kubernetes.md](04-kubernetes.md). Assumes you ran [03-install-kind.md](03-install-kind.md) instead of the k3s install. The two chapters are 95% identical — the only meaningful differences are:
>
> * image references use `localhost:5001/...` (kind's local-registry sidecar)
> * we don't need an `imagePullSecrets` block — our registry has no auth
> * we skip the `kubectl create secret docker-registry registry-creds ...` step entirely

## 1. Pushing our container images to the in-cluster registry

So far the `myapi` and `myfrontend` images you built in chapters 1 and 2 live only in the host's Docker daemon. The kind cluster can't see them. We need to push them to a registry that the cluster *can* reach. We already installed one in chapter 3 — `kind-registry`, addressable as `localhost:5001` from both the host and the cluster (thanks to the containerd redirect).

In Docker, "push to registry X" means **tagging the image so its name starts with X**, then running `docker push`. The image name *is* its registry address.

```shell
docker tag myfrontend localhost:5001/myfrontend
```

```shell
docker push localhost:5001/myfrontend
```

```shell
docker tag myapi localhost:5001/myapi
```

```shell
docker push localhost:5001/myapi
```

Verify both arrived:

```shell
curl http://localhost:5001/v2/_catalog
```

Should return `{"repositories":["busybox","myapi","myfrontend"]}` (busybox is left over from the chapter 3 verification — harmless).

> **Why no `docker login`?** The chapter-3 registry runs without authentication: no htpasswd, no TLS. That's fine on a single-host kind setup that listens only on loopback. In production you'd bolt cert-manager + basic auth on top — covered conceptually in the original chapter 3.

## 2. Creating a Deployment for the frontend

A **Deployment** is the most common k8s workload object. It says "I want N copies of this pod template running, all of the time, and I want a rolling update when the template changes."

Create `frontend-deployment.yaml`:

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

Note what's **not** here compared to the original chapter:

* No `imagePullSecrets:` block. Our registry doesn't require credentials.
* No prior `kubectl create secret docker-registry registry-creds ...` step. Skip it.

Apply it:

```shell
kubectl apply -f frontend-deployment.yaml
```

Check what got created:

```shell
kubectl get deployment frontend
```

```shell
kubectl get pod
```

If the pod is stuck in `ErrImagePull` or `ImagePullBackOff`, run `kubectl describe pod <pod-name>` and look at `Events:`. The most likely cause is that you forgot the `localhost:5001/` prefix when tagging the image, or you typed `localhost:5000` (off by one — the host port is 5001).

You can also browse the deployment in k9s: hit `:` then type `deploy`.

## 3. Creating a StatefulSet for the database

Our backend needs a database to count visits. We'll run a tiny single-instance PostgreSQL. There's a real difference between this and the frontend:

* **Stateless apps** (frontend, api) keep no data on disk. If a pod dies, a replacement pod is identical. Order doesn't matter. Deployments handle them.
* **Stateful apps** (databases) keep data on disk. If pod `db-0` dies, the new pod must be **the same `db-0`**, with the same PersistentVolume reattached, or you've lost data.

For this we use a **StatefulSet**. Key behaviors vs a Deployment:

| | Deployment | StatefulSet |
|---|---|---|
| Pod identity | Random suffixes, interchangeable | Stable: `name-0`, `name-1`, ... |
| Storage | Pods share, or none | Each pod gets its **own** PVC via `volumeClaimTemplates` |
| Update order (during rollout) | New pod created **before** old terminated (zero-downtime preference) | Old terminated **before** new created (preserves at-most-one-writer for DBs) |
| Use case | API servers, web frontends, workers | Databases, brokers, anything that owns a disk |

We also want the database to come up with our `counter` table already created. Postgres reads any `*.sql` files placed in `/docker-entrypoint-initdb.d/` on first start. We'll mount a **ConfigMap** there.

### The init script as a ConfigMap

A **ConfigMap** is just a named bag of key/value pairs stored in etcd. You can mount its keys as files inside pods.

Save as `postgres-initdb-configmap.yaml`:

```yaml
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
```

Apply:

```shell
kubectl apply -f postgres-initdb-configmap.yaml
```

### The StatefulSet itself

Save as `postgres-statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
spec:
  selector:
    matchLabels:
      app: postgresql-db
  replicas: 1
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
```

Apply:

```shell
kubectl apply -f postgres-statefulset.yaml
```

A few details worth understanding:

* `volumeClaimTemplates` (note the *template* — plural-ish): the StatefulSet creates a **new PVC per replica** by following this template. If you set `replicas: 3`, you'd get three PVCs: `postgresql-db-disk-postgresql-db-0`, `-1`, `-2`. Each pod gets its own disk, sticky for its lifetime.
* `accessModes: ["ReadWriteOnce"]` — only one node can mount this volume at a time (correct for a database). Other modes: `ReadOnlyMany`, `ReadWriteMany` (rare, needs a network filesystem like NFS).
* `PGDATA: /data/pgdata` — Postgres writes data here. The mount at `/data` is backed by the PVC, so the data survives pod restarts.
* `image: postgres:16` — pinned major version. The original chapter uses `postgres:latest`; pinning is safer because a major version bump (16 → 17) can require a manual data migration.

### Where the actual data lives (kind-specific)

Kind ships with a `local-path-provisioner` (one of the pods you saw in step 8 of chapter 3, under `local-path-storage` namespace). When a PVC is created, the provisioner picks a folder on the node container's filesystem (under `/var/local-path-provisioner/`) and binds it. So your database's data is a directory inside the `tutorial-control-plane` docker container. `kind delete cluster` = data gone. Fine for tutorial; in production you'd point a `StorageClass` at real cloud or NAS storage.

Verify everything came up:

```shell
kubectl get statefulset
```

```shell
kubectl get pod
```

```shell
kubectl get pvc
```

You should see `postgresql-db-disk-postgresql-db-0` as a `Bound` PVC.

## 4. Wrapping up

You now have a **Deployment** managing one stateless pod (frontend) and a **StatefulSet** managing one stateful pod (postgres), plus a **ConfigMap** holding the database init script. None of these pieces know how to talk to each other yet — Pods only know their own IPs, and those IPs change. That's the job of **Services**, covered in chapter 5.

![frontend-deployment](../imgs/frontend-deployment.png)

## Review questions

### 1. Try to scale the **deployment** up and down — using k9s, then using kubectl. What did you observe?

<details>
  <summary>Answer</summary>

With kubectl:

```shell
kubectl scale deployment frontend --replicas=3
```

```shell
kubectl get pods -w
```

You'll see two new `frontend-...` pods appear in `Pending`, then `ContainerCreating`, then `Running`. Random suffixes. Each pod's name is independent — they're interchangeable.

```shell
kubectl scale deployment frontend --replicas=1
```

Two pods get `Terminating`. The pod that **remains** is the same one that was running before — Deployments try to keep existing pods where possible, but the survivor's name is essentially arbitrary; if you'd scaled back to zero and then back up, you'd get fresh names.

In k9s: navigate with `:deploy`, highlight the row, press `s` (scale), type a number, enter. Same effect, less typing.

The key takeaway: scaling a Deployment is fast and stateless. The replicas don't have identities you can rely on.

If you try the same on the StatefulSet (`kubectl scale statefulset postgresql-db --replicas=3`), the result is different: pods come up **sequentially** (`postgresql-db-0`, then `-1`, then `-2`), each one gets its own PVC, and scaling back down terminates them in reverse order (`-2`, `-1`, …). PVCs are *not* deleted when you scale down — they're kept around in case you scale back up. That's intentional safety for databases.

</details>

### 2. Can you port-forward the frontend deployment to your host so you can browse it?

<details>
  <summary>Answer</summary>

```shell
kubectl port-forward deployment/frontend 8888:80
```

Then visit `http://localhost:8888` from a browser on the same machine, or use VS Code's remote port-forwarding if you're SSH'd in.

Notes:

* The `8888:80` form is `localPort:containerPort`. Container port 80 is what the frontend image listens on (set by the Dockerfile `CMD`).
* Targeting a **deployment** picks one of its pods automatically. You can also target a specific pod (`pod/frontend-abc123`), a service (`service/frontend`), or a statefulset.
* `kubectl port-forward` is a debug tool, not a production exposure mechanism. It runs through your kubeconfig (kube-apiserver tunnels the traffic). Closing the terminal kills the tunnel.
* In k9s: select the pod, press `Shift-F`, choose a port mapping. Same thing.

</details>

### 3. What happens if you delete the postgres pod directly? What if you delete the StatefulSet?

<details>
  <summary>Answer</summary>

```shell
kubectl delete pod postgresql-db-0
```

The StatefulSet controller notices its desired state (one replica) doesn't match the actual state (zero pods), and **immediately re-creates** `postgresql-db-0`. The new pod re-attaches to the same PVC `postgresql-db-disk-postgresql-db-0` — same data. You'll see it briefly go through `Pending` → `ContainerCreating` → `Running`, but the data inside (the `counter` table) is intact.

```shell
kubectl delete statefulset postgresql-db
```

The StatefulSet is gone. The pod terminates. **But the PVC is not deleted** — by default, StatefulSets leave PVCs behind so you don't accidentally lose data. If you re-create the StatefulSet with the same name, it will re-bind to the same PVC and you'll see your old data.

To actually delete the data:

```shell
kubectl delete pvc postgresql-db-disk-postgresql-db-0
```

This is the difference between **persistent** state (lives with the PVC) and **ephemeral** state (lives with the pod). Knowing which is which prevents some really bad days.

</details>

### 4. The original tutorial uses `image: postgres:latest`. This chapter uses `image: postgres:16`. Why?

<details>
  <summary>Answer</summary>

Three reasons to pin to a major version:

1. **Postgres major versions are not data-compatible.** Going from 16.x to 17.x is a manual operation (`pg_upgrade` or a dump/restore). If your StatefulSet was running `postgres:latest` when 16 was current and you re-deploy six months later, you'll suddenly pull 17, the pod will start, refuse to read 16's data files, crash-loop, and your database is offline. Pinning to `16` prevents the surprise.
2. **`latest` is implicit, not explicit.** Two clusters deployed a week apart using `latest` may actually be running different versions. Pinning makes the manifest self-documenting.
3. **Image caching is more predictable.** With `latest`, the `imagePullPolicy` defaults to `Always`, meaning the cluster pulls from the registry on every pod start. With a real tag like `16`, the default is `IfNotPresent`, which is faster.

The exception: in *development* you may genuinely want `latest` for the frontend/backend you're iterating on. Skaffold (chapter 7) handles this by tagging each build uniquely (e.g. `inputDigest`), so neither `latest` nor a fixed tag — every change gets its own tag.

</details>

### 5. Why does the database StatefulSet update its pod by *terminating the old pod before* creating the new one, while Deployments do the opposite?

<details>
  <summary>Answer</summary>

Different invariants. Both behaviors are correct for their target workloads.

**Deployments** assume **stateless, interchangeable, replaceable** workloads. They optimize for **availability**: during a rollout, create the new pod first; only when it's `Ready` do they terminate an old one. This is the rolling-update strategy. End-users see zero downtime because there's always at least one healthy old or new pod serving.

**StatefulSets** assume **stateful, identity-bound, often single-writer** workloads (databases, brokers). They optimize for **safety**: never have two pods with the same identity (same name, same disk) running at the same time, even briefly. So they kill the old `db-0` first, wait until it's fully gone, then bring up the new `db-0`. The cost is brief downtime per pod during a rolling update; the benefit is correctness for software that assumes "I'm the only one writing to this disk right now."

For Postgres specifically, this matters because two postgres processes writing to the same `PGDATA` directory at the same time will corrupt the data. The StatefulSet rollout strategy makes that impossible by design.

</details>

### 6. The frontend pod's image reference is `localhost:5001/myfrontend`. From inside the pod, can you `curl localhost:5001`?

<details>
  <summary>Answer</summary>

**No.** And this is one of the most common k8s gotchas, so worth burning into memory.

Inside the pod, `localhost` refers to **the pod's own network namespace** — i.e. the frontend container itself. There's no registry listening inside the pod, so `curl localhost:5001` errors with connection refused.

The image reference `localhost:5001/myfrontend` is read by **containerd on the kind node**, not by the pod. Containerd is on the node container's network, and the `containerdConfigPatches` redirect (from chapter 3 step 3) rewrites `localhost:5001` → `http://kind-registry:5000`. The pod never sees that URL.

This is the same trap as the chapter 3 puzzle ("why is the redirect rewriting `localhost` to `kind-registry`"). Pattern to internalize: **`localhost` is contextual — always ask "from whose perspective?".**

If you actually wanted to reach the registry from inside a pod, you would `curl kind-registry:5000` (if the pod's DNS is correctly set up — which on kind, by default, it is, because the kind node's CoreDNS forwards through to Docker's embedded DNS).

</details>
