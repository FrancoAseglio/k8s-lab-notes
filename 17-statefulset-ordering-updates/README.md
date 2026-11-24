# Ordering, scaling and updates

**Work from this directory:** `cd 17-statefulset-ordering-updates`

**Goal:** understand the *rules of order*: how a StatefulSet creates, scales and updates its
pods, and how to take control: canary updates with `partition`, manual updates with `OnDelete`,
and the `Parallel` policy that drops the ordering.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing of your own in it.
**Ending state:** everything from this session deleted, volumes included.

**How the sample app works.** Each pod serves its own name and version from its volume.
Its **readiness probe** requires the file `/data/ready`, which doesn't exist on a new volume, so you
act as a "slow application": a pod stays `0/1` until you run `kubectl exec <pod> -- touch /data/ready`.
That makes the ordering rules visible. The file lives on the pod's *volume*, so it survives
restarts and updates.

Read the top of `sts-ordered.yml` and `sts-parallel.yml` before you start.

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

All match: go straight to Step 59.

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

## Step 59: Ordered creation: nothing starts until the previous pod is Ready

**Do: terminal 1**

    kubectl apply -f services.yml -f sts-ordered.yml
    kubectl get pods -w

**Do: terminal 2** (run each command, one after the other, watching terminal 1)

    kubectl get sts ordered
    kubectl describe sts ordered | tail -8
    kubectl exec ordered-0 -- touch /data/ready
    kubectl exec ordered-1 -- touch /data/ready
    kubectl exec ordered-2 -- touch /data/ready
    kubectl get pods,pvc

**Expect**

- Only **`ordered-0`** exists, `Running` but **`0/1`**. Wait 20 seconds: still no `ordered-1`.
  `get sts` shows `READY 0/3`.
- Right after `touch /data/ready` on `ordered-0`, it turns `1/1` and **`ordered-1`** is created. The
  same again for `ordered-2` (you can `exec` into a pod as soon as it is `Running`, even if not ready).
- At the end three pods `1/1` and three claims `data-ordered-0..2`.

**Why**

`podManagementPolicy: OrderedReady` (the default) means: create pod *N* only when pods *0..N-1* are
**Running and Ready**. It protects clustered applications whose members must start in sequence (a
primary before its replicas). The flip side: **one stuck pod blocks everything after it.**

**Try**

Delete `ordered-0` and watch: it returns first; the others keep running (ordering applies to creation
and scaling, not to a pod restarting on its own).

---

## Step 60: Scaling, and what happens to the volumes

**Do: terminal 1** `kubectl get pods -w`

**Do: terminal 2**

    kubectl scale sts ordered --replicas=5 
    # when ordered-3 is Running (terminal 1):
    kubectl exec ordered-3 -- touch /data/ready
    # when ordered-4 is Running:
    kubectl exec ordered-4 -- touch /data/ready
    kubectl scale sts ordered --replicas=2
    kubectl get pods,pvc
    kubectl scale sts ordered --replicas=3
    kubectl get pods,pvc

**Expect**

- Scaling **up** follows the same rule: `ordered-3` is created, and `ordered-4` only after `ordered-3`
  is Ready.
- Scaling **down** removes pods **from the highest ordinal**: `ordered-4`, then `ordered-3`, then
  `ordered-2`, one at a time, each fully gone before the next.
- After scaling down to 2, **all five claims are still there** (`Retain`).
- Scaling back to 3: `ordered-2` returns with its **old claim** and is `1/1` immediately (its
  `ready` file survived): nobody had to touch it.

**Why**

Reverse-order removal means the "oldest" members (lowest ordinals, usually the primary) are the last
to go. Keeping the claims after scale-down means scaling back up restores each member's data.
Extra claims accumulate if you scale down and never delete them.

**Try**

Change the policy so scale-down also deletes the volumes, then repeat:

    kubectl patch sts ordered -p '{"spec":{"persistentVolumeClaimRetentionPolicy":{"whenScaled":"Delete"}}}'
    kubectl scale sts ordered --replicas=2
    kubectl get pvc                              # data-ordered-2 is gone
    kubectl scale sts ordered --replicas=3       # ordered-2 returns EMPTY: touch /data/ready again

Finish this step with 3 replicas and all three pods `1/1`.

---

## Step 61: Rolling updates: reverse order, and canaries with `partition`

**Do: a normal rolling update**

    kubectl exec ordered-0 -- cat /data/index.html
    kubectl get pods -L controller-revision-hash
    kubectl set env sts/ordered APP_VERSION=v2         # terminal 1: kubectl get pods -w
    kubectl rollout status sts/ordered
    kubectl get pods -L controller-revision-hash
    for i in 0 1 2; do kubectl exec ordered-$i -- cat /data/index.html; done
    kubectl rollout history sts/ordered

**Do: a canary (partition)**

    kubectl patch sts ordered -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
    kubectl set env sts/ordered APP_VERSION=v3
    kubectl rollout status sts/ordered
    kubectl get pods -L controller-revision-hash
    for i in 0 1 2; do kubectl exec ordered-$i -- cat /data/index.html; done
    kubectl patch sts ordered -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
    kubectl rollout status sts/ordered
    for i in 0 1 2; do kubectl exec ordered-$i -- cat /data/index.html; done
    kubectl rollout undo sts/ordered
    kubectl rollout status sts/ordered
    for i in 0 1 2; do kubectl exec ordered-$i -- cat /data/index.html; done

**Expect**

- Before: `ordered-0 v1`. The update to `v2` replaces pods in **reverse order**, `ordered-2` then
  `ordered-1` then `ordered-0`, **one at a time**, waiting for each to be Ready. They are Ready at once
  because their volumes (and `ready` file) are reused. Afterwards all three say `v2`.
- The `controller-revision-hash` label identifies the revision each pod runs.
- With `partition: 2` and the change to `v3`, **only `ordered-2`** is updated (`v3`); `ordered-0` and
  `ordered-1` stay on `v2`. `rollout status` reports a *partitioned roll out complete*.
- Lowering the partition to `0` rolls the rest, **0 and 1**, to `v3`.
- `rollout undo` goes back to the previous revision (`v2`) with the same rolling procedure.

**Why**

Updating in reverse order protects the "first" members, and the Ready-wait protects the quorum.
`partition` turns a rollout into a **staged** one: update the highest ordinal(s), verify, then lower
the partition to continue. It is the StatefulSet's canary release.

---

## Step 62: Taking over: `OnDelete`, and the `Parallel` policy

**Do: OnDelete**

    kubectl patch sts ordered --type=merge -p '{"spec":{"updateStrategy":{"type":"OnDelete","rollingUpdate":null}}}'
    kubectl set env sts/ordered APP_VERSION=v4
    kubectl get pods -L controller-revision-hash          # nothing changed
    kubectl rollout status sts/ordered
    kubectl delete pod ordered-1
    kubectl get pods -w                                   # Ctrl+C when ordered-1 is back
    for i in 0 1 2; do kubectl exec ordered-$i -- cat /data/index.html; done

**Do: Parallel**

    kubectl apply -f sts-parallel.yml
    kubectl get pods -w                                   # terminal 1
    kubectl scale sts par --replicas=5
    kubectl scale sts par --replicas=1
    kubectl patch sts ordered -p '{"spec":{"podManagementPolicy":"Parallel"}}'

**Expect**

- With `OnDelete` the new template changes **nothing** by itself; `rollout status` refuses to track it
  (it only supports `RollingUpdate`). Pods update only when **you delete them**: `ordered-1` returns
  at `v4`, the other two stay on their old version until you delete them too.
- `par-0`, `par-1` and `par-2` are all created **at the same time**, although none is ready. Scaling up or
  down also happens all at once.
- The last `patch` is **rejected**: only a few fields (such as `replicas`, `template`, `updateStrategy`,
  `persistentVolumeClaimRetentionPolicy`, `minReadySeconds`) can change on an existing StatefulSet.
  To switch the policy you must delete and re-create the set.

**Why**

| `updateStrategy` | Updates happen |
|---|---|
| `RollingUpdate` (default) | automatically, in reverse order, one at a time (`partition` to stage) |
| `OnDelete` | only when you delete a pod: for applications that need manual, careful upgrade steps |

| `podManagementPolicy` | Creation / scaling |
|---|---|
| `OrderedReady` (default) | sequential, each waits for the previous to be Ready |
| `Parallel` | all at once, no waiting (keeps names and volumes, drops the ordering) |

---

## Cleanup for this session

    kubectl delete sts ordered par
    kubectl delete -f services.yml
    kubectl delete pvc --all
    kubectl get sts,pods,svc,pvc,pv                  # nothing of yours left

---

## End of Session 2: checkpoint

    kubectl get sts,pods,pvc,pv           # No resources found

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Pod `ordered-N` not created | The previous pod isn't Ready: `touch /data/ready` in it (or fix its probe) |
| `exec` fails: "container not found" / pod not running | Wait until the pod is `Running` (it may still be `0/1`) |
| A pod is `1/1` but I never touched it | Its volume already had `/data/ready` (volumes are reused) |
| `rollout status` waits for ever | A new pod isn't Ready (touch `/data/ready`), or `partition` holds back some pods (that's expected: lower it), or the strategy is `OnDelete` |
| `patch ... updateStrategy` error about `rollingUpdate` | Use the `--type=merge` form shown in Step 62, which clears `rollingUpdate` |
| Extra claims left over | Scaling down keeps claims by default: `kubectl delete pvc --all` |
| Pods take long to delete | This lab sets `terminationGracePeriodSeconds: 3`; `kubectl delete pod <name> --now` shortens it |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] explain `OrderedReady` creation and why one stuck pod blocks the rest
- [ ] say in which order pods are scaled down and updated
- [ ] explain what happens to the claims on scale-down, and the policy that changes it
- [ ] stage an update with `partition`, and roll it back with `rollout undo`
- [ ] explain `OnDelete` and `Parallel`, and which fields can't be changed after creation

## Stretch ideas

- Add `minReadySeconds: 10` to the template's StatefulSet spec and watch how it slows a rollout.
- Add a `livenessProbe` and see how a failing liveness probe restarts the container but keeps the pod.
- Delete the whole set while it runs with 3 pods and watch the order; then scale to 0 first and compare (scale-down is ordered, deletion is not guaranteed to be).
- Try a `PodDisruptionBudget` for the set and read what it protects against.
