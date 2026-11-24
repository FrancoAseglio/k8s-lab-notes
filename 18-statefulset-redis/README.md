# A real stateful application - Redis primary and replicas 

**Work from this directory:** `cd 18-statefulset-redis`  

**Goal:** run a replicated database on a StatefulSet and see how the pieces fit: a ConfigMap and a
start-up script that decide each pod's role from its **ordinal**, a headless Service for stable
names, and a volume per pod. Then test what a StatefulSet does for you when things fail, and
what it doesn't.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing of your own in it.
**Ending state:** everything from this session deleted, volumes included.

**The design**

    redis-0  = PRIMARY   (writes)        redis-0.redis  <- stable DNS name
    redis-1  = replica of redis-0 (reads)  each pod has its own volume (AOF persistence)
    redis-2  = replica of redis-0 (reads)

Replication is done by **Redis itself**. The StatefulSet supplies the identity (names, DNS, volumes)
that makes it possible. This is a teaching setup, not a production one (no authentication, no
automatic failover).

Read the top of `redis-config.yml` and `statefulset.yml` before you start.

---

## Golden rule

Before any command that changes something, know **which cluster and which
namespace** you are on:

    kubectl config current-context
    kubectl config get-contexts           # NAMESPACE column; empty = default

Applying to the wrong cluster, or looking in the wrong namespace, are the two
most common real-world mistakes.

## Prerequisites

    docker version
    kind version
    kubectl version --client

Docker must be running. Nothing else is needed: no cloud and no extra tools. StatefulSets
here use kind's built-in storage (local directories on the node containers), as in the
storage lab.

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get storageclass              # standard (default)
    kubectl get sts,pods,pvc              # No resources found

All match: go straight to Step 63.

### Bootstrap (if anything above is missing)

No `lab-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name lab-a --config cluster.yml
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Cluster exists but the namespace default is wrong or missing:

    kubectl config use-context kind-lab-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Leftovers in `lab` from other sessions or experiments (this also removes their
volumes, since the default class deletes a volume with its claim):

    kubectl delete ns lab
    kubectl apply -f namespace.yml

Completely broken cluster: `kind delete cluster --name lab-a`, then the first block above.

---

## Step 63: Deploy and read the roles

**Do**

    kubectl apply -f redis-config.yml -f service.yml
    kubectl apply -f statefulset.yml
    kubectl get pods -w                          # Ctrl+C once redis-2 is 1/1
    kubectl logs redis-0 | head -3
    kubectl logs redis-1 | head -3
    kubectl get pvc
    for i in 0 1 2; do echo "== redis-$i"; kubectl exec redis-$i -- redis-cli info replication | grep -E '^(role|connected_slaves|master_host|master_link_status)'; done

**Expect**

- Pods start in order: `redis-0` (and it must answer `PING` before it counts as Ready), then
  `redis-1`, then `redis-2`.
- The first log line of `redis-0` is **`[redis-0] starting as PRIMARY`**; of `redis-1`,
  **`[redis-1] starting as REPLICA of redis-0.redis`**.
- Three claims `data-redis-0..2`.
- `redis-0`: `role:master` and `connected_slaves:2`. `redis-1` and `redis-2`: `role:slave` (Redis' old word
  for replica), `master_host:redis-0.redis`, `master_link_status:up`.

**Why**

`start.sh` derives the **ordinal from the pod's name** (`${name##*-}`) and chooses the role. This only works
because the name is stable (the pod called `redis-0` is always the primary) and because `redis-0.redis` is
a stable DNS name that survives the primary's restarts. A Deployment can't give a pod either guarantee.
Because the script lives in a **ConfigMap**, you can change it without rebuilding the image
(and, as you learned, pods need a restart to see it).

**Try**

    kubectl get configmap redis-config -o yaml | head -30
    kubectl exec redis-0 -- ls -la /data               # appendonlydir: the persisted write log

---

## Step 64: Replication, read-only replicas and persistence

**Do**

    kubectl exec redis-0 -- redis-cli set greeting "hello from redis-0"
    kubectl exec redis-1 -- redis-cli get greeting
    kubectl exec redis-2 -- redis-cli get greeting
    kubectl exec redis-1 -- redis-cli set x y
    kubectl delete pod redis-1
    kubectl get pods -w                          # Ctrl+C once redis-1 is 1/1 again
    kubectl exec redis-1 -- redis-cli get greeting
    kubectl exec redis-1 -- redis-cli info replication | grep -E '^(role|master_link_status)'

**Expect**

- The value written on the primary is readable on **both replicas**.
- Writing to a replica fails: `(error) READONLY You can't write against a read only replica.`
- After `redis-1` is deleted, it comes back **as the same member** (`redis-1`), as a replica again, with
  its data, and the link `master_link_status:up`.

**Why**

Each replica keeps its own copy on its own volume. When a replica is replaced it rejoins under the same
name, picks its role from its ordinal and reconnects to the primary by name.

---

## Step 65: Failure, and what a StatefulSet does *not* do

**Do: the primary restarts**

    kubectl exec redis-0 -- redis-cli set counter 1
    kubectl delete pod redis-0
    kubectl get pods -w                          # Ctrl+C once redis-0 is 1/1 again
    kubectl exec redis-0 -- redis-cli get counter
    kubectl exec redis-1 -- redis-cli info replication | grep master_link_status

**Do: the primary can't be rescheduled**

    NODE=$(kubectl get pod redis-0 -o jsonpath='{.spec.nodeName}'); echo $NODE
    kubectl cordon $NODE
    kubectl delete pod redis-0
    kubectl get pods
    kubectl describe pod redis-0 | tail -6
    kubectl exec redis-1 -- redis-cli get greeting
    kubectl exec redis-1 -- redis-cli info replication | grep -E '^(role|master_link_status)'
    kubectl exec redis-1 -- redis-cli set z 1
    kubectl uncordon $NODE
    kubectl get pods -w                          # Ctrl+C once redis-0 is 1/1 again
    kubectl exec redis-1 -- redis-cli info replication | grep master_link_status

**Do: scaling**

    kubectl scale sts redis --replicas=5
    kubectl get pods -w                          # Ctrl+C once redis-4 is 1/1
    kubectl exec redis-4 -- redis-cli get greeting
    kubectl exec redis-0 -- redis-cli info replication | grep connected_slaves
    kubectl scale sts redis --replicas=3
    kubectl get pvc

**Expect**

- When the primary restarts, `counter` is **still there** (the AOF log on its own volume) and the replicas
  reconnect within seconds.
- With the node cordoned, `redis-0` stays **`Pending`**: its volume lives on that node and the node
  accepts no pods (events mention an unschedulable node and a volume node affinity conflict).
  Meanwhile `redis-1` **still serves reads**, reports `master_link_status:down`, stays a replica,
  and **refuses writes**. Nobody is promoted: there is no automatic failover. After `uncordon` everything
  heals by itself.
- Scaling to 5: `redis-3` and `redis-4` are created in order, **sync from the primary** and already
  hold `greeting`; the primary then reports `connected_slaves:4`.
- Scaling back to 3 leaves the claims `data-redis-3` and `data-redis-4` behind (`Retain`).

**Why: what the StatefulSet did and didn't do**

| Provided by the StatefulSet | Not provided |
|---|---|
| Stable names and DNS (`redis-0.redis`) | Replication (Redis does it) |
| A private, re-attached volume per pod | **Failover**: promoting a replica when the primary is lost |
| Ordered start (primary first) and ordered updates | Backups, monitoring, upgrades of the data format |
| Same member comes back after a restart | Moving a pod whose **local** volume's node is gone |

For production, people add **Redis Sentinel** or use an **operator** that automates promotion, backups and
upgrades. A StatefulSet is the foundation such operators build on. Also note the local-volume lesson: with
node-local storage a stateful pod is tied to its node; network storage (cloud disks, Ceph) lets it move.

---

## Cleanup for this session

    kubectl delete ns lab
    kubectl apply -f namespace.yml
    kubectl get sts,pods,svc,pvc,pv                  # nothing of yours left
    kubectl get nodes                                # all Ready, none SchedulingDisabled

Deleting the namespace removes the StatefulSet, the ConfigMap, the Service and all claims (and so the
volumes). The last command re-creates the empty `lab` namespace so your context default still works. If a
node still says `SchedulingDisabled`, run `kubectl uncordon <node>`.

---

## End of Session 3: checkpoint

    kubectl get sts,pods,pvc,pv           # No resources found
    kubectl get nodes                     # no SchedulingDisabled

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `redis-1` never starts | `redis-0` isn't Ready: `kubectl logs redis-0`, `kubectl describe pod redis-0` |
| Replica shows `master_link_status:down` | `kubectl logs redis-1`: can it resolve `redis-0.redis`? Is `redis-0` Ready (DNS lists ready pods only)? It retries every second |
| `READONLY` when writing | You wrote to a replica: use `redis-0` |
| `redis-cli: command not found` | You are not exec'ing into a Redis pod |
| `ImagePullBackOff` | No internet, or the tag no longer exists: use another existing tag such as `redis:7.4-alpine` in `statefulset.yml` |
| Pod `CrashLoopBackOff` | `kubectl logs <pod> --previous`: usually a mistake in `redis.conf` or `start.sh` (edit the ConfigMap, then delete the pod) |
| `redis-0` stuck `Pending` after the lab | A node is still cordoned: `kubectl get nodes`, then `kubectl uncordon <node>` |
| Edited the ConfigMap but nothing changed | Mounted files update slowly and the script runs only at start: delete the pod |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] explain how the start-up script picks a role, and why it needs a stable ordinal and name
- [ ] show that replicas hold the data and refuse writes
- [ ] explain why `redis-0` returns with its data (its own volume) and replicas find it again (its DNS name)
- [ ] say what a StatefulSet does **not** do (replication, failover, backups)
- [ ] explain why a pod with a node-local volume can't simply move to another node

## Stretch ideas

- Add Redis Sentinel (a second StatefulSet) and test automatic promotion.
- Add `requirepass` from a Secret (see the ConfigMaps and Secrets lab) and update the probe and script.
- Add anti-affinity or `topologySpreadConstraints` so primary and replicas land on different nodes.
- Set `publishNotReadyAddresses: true` on the headless Service and discuss what changes for peer discovery.
- Replace Redis with another clustered app (for example a 3-node etcd or a small MySQL primary/replica) using the same pattern.
