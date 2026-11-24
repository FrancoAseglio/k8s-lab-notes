# StatefulSet basics - names, DNS and volumes that stick

**Work from this directory:** `cd 16-statefulset-basics`

**Goal:** see the three things a StatefulSet gives you that a Deployment doesn't: **stable
names**, **stable network identity (DNS)** and **stable storage (one volume per pod)**, and what
happens to that storage when pods, or the StatefulSet itself, are deleted.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing of your own in it.
**Ending state:** everything from this session deleted, volumes included.

**Deployment versus StatefulSet**

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | random (`web-7d9f-x8k2p`) | ordinals (`web-0`, `web-1`...) |
| A replaced pod | a new identity | the **same** name |
| Storage | one shared claim, or none | **one claim per pod**, always re-attached to the same pod |
| Network | one Service address for all | a DNS name **per pod** (headless Service) |
| Start and stop | all at once | one at a time, in order (by default) |
| Typical use | stateless apps | databases, queues, clustered apps |

Read the top of `service.yml` and `statefulset.yml` before you start.

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

All match: go straight to Step 55.

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

## Step 55: Create a StatefulSet and its headless Service

**Do**

    kubectl apply -f service.yml
    kubectl apply -f statefulset.yml
    kubectl get pods -w                       # Ctrl+C once web-2 is 1/1
    kubectl get sts web
    kubectl get pods -o wide --show-labels
    kubectl get pvc
    kubectl get pv
    kubectl describe sts web | tail -12
    kubectl exec web-1 -- hostname

**Expect**

- Pods appear **one at a time**: `web-0` first; `web-1` only once `web-0` is Running and Ready;
  then `web-2`. A Deployment would create all three at once.
- The names are `web-0`, `web-1`, `web-2`. `kubectl get sts` shows `READY 3/3`.
- Labels include `apps.kubernetes.io/pod-index=0|1|2`, `statefulset.kubernetes.io/pod-name=web-0`
  and a `controller-revision-hash`.
- **Three claims** were created for you: `www-web-0`, `www-web-1`, `www-web-2`, all `Bound` to PVs of
  class `standard`.
- `describe` shows events such as *create Claim www-web-0 Pod web-0 in StatefulSet web success*.
- Inside the pod the hostname is the pod name (`web-1`).

**Why**

The `serviceName` links the StatefulSet to its headless Service. The `volumeClaimTemplates` are
templates: the controller stamps out one PVC per ordinal, named `<template>-<set>-<ordinal>`. That
naming rule is how a pod finds *its* volume again later.

**Try**

    kubectl get sts web -o yaml | head -40          # compare with statefulset.yml
    kubectl get pod web-0 -o yaml | grep -A8 volumes:

---

## Step 56: Stable network identity (DNS)

**Do**

    kubectl apply -f tools.yml
    kubectl wait --for=condition=Ready pod/client pod/dnsutils --timeout=120s
    kubectl exec dnsutils -- nslookup web
    kubectl exec dnsutils -- nslookup web-0.web
    kubectl exec client -- curl -s web-0.web
    kubectl exec client -- curl -s web-1.web
    kubectl exec client -- curl -s web-2.web
    kubectl get pod web-1 -o wide                    # note the IP
    kubectl delete pod web-1
    kubectl get pod web-1 -o wide                    # repeat until READY 1/1 and AGE is a few seconds
    kubectl exec client -- curl -s web-1.web

**Expect**

- `nslookup web` (the Service) returns **three addresses**, one per ready pod (no virtual IP).
- `nslookup web-0.web` returns **one address**: the pod's own IP, under the name
  `web-0.web.lab.svc.cluster.local`.
- Each `curl` answers with the pod's own name (`web-0`, `web-1`, `web-2`), so you really reach
  that pod, not "any pod".
- After deleting `web-1`, the new pod has the **same name** and (usually) a **different IP**, and
  `curl web-1.web` still answers `web-1`.

**Why**

Pod IPs change every time a pod is re-created; the **name** doesn't. A headless Service publishes one
DNS record per pod, so members of a clustered application (a database primary and its replicas, a
quorum) can be addressed by name in each other's configuration. A Deployment only offers one Service
address for "some pod".

**Try**

Delete the headless Service and look again; then bring it back:

    kubectl delete svc web
    kubectl exec dnsutils -- nslookup web-0.web      # fails: no Service, no per-pod names
    kubectl apply -f service.yml
    kubectl exec dnsutils -- nslookup web-0.web      # works again

(The pods never stopped; only the *names* depended on the Service.)

---

## Step 57: Stable storage: each pod keeps its own volume

**Do**

    kubectl exec web-0 -- cat /data/starts.log
    kubectl exec web-2 -- sh -c 'echo "precious data of web-2" > /data/note.txt'
    kubectl get pods -o wide                                   # note the NODE column
    kubectl delete pod web-0 web-2
    kubectl get pods -w                                        # Ctrl+C when both are 1/1 again
    kubectl exec web-0 -- cat /data/starts.log
    kubectl exec web-2 -- cat /data/note.txt
    kubectl exec web-1 -- cat /data/note.txt
    kubectl get pods -o wide
    kubectl get pvc

**Expect**

- `web-0`'s log now has **two lines** (first start, then the restart a moment ago).
- `web-2` still has `note.txt`: the volume came back with the pod.
- `web-1` says *No such file*: **each pod has its own volume.**
- Re-created pods land on the **same nodes** as before, and the claims are unchanged.

**Why**

A re-created pod gets the claim with its own name (`www-web-2`), so it reattaches to the same data.
On kind that volume is a directory on one node (see the storage lab), so the PV pins the pod to that
node. On a cloud with network disks the pod could move to another node and the disk would follow.

**Try**

    kubectl get pv -o custom-columns=NAME:.metadata.name,CLAIM:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]

---

## Step 58: Deleting a StatefulSet, and what happens to the data

**Do**

    kubectl delete sts web
    kubectl get pods
    kubectl get pvc
    kubectl apply -f statefulset.yml
    kubectl get pods -w                                        # Ctrl+C when web-2 is 1/1
    kubectl exec web-2 -- cat /data/note.txt
    kubectl exec web-0 -- cat /data/starts.log
    kubectl patch sts web -p '{"spec":{"persistentVolumeClaimRetentionPolicy":{"whenDeleted":"Delete","whenScaled":"Retain"}}}'
    kubectl get sts web -o jsonpath='{.spec.persistentVolumeClaimRetentionPolicy}{"\n"}'
    kubectl delete sts web
    kubectl get pvc
    kubectl get pv

**Expect**

- After the first delete the pods are gone but **all three claims remain** (`Bound`).
- Re-creating the StatefulSet brings the pods back in order, **with their data**: `note.txt`
  is there and `starts.log` has gained another line.
- After the policy is changed to `whenDeleted: Delete`, deleting the StatefulSet **also deletes the
  claims**, and then the volumes (the class's reclaim policy is `Delete`): `No resources found`.
  That data is gone.

**Why**

By default a StatefulSet **never deletes its volumes** (`Retain`): losing a database because
someone deleted the wrong object would be worse than leaving some storage behind.
`persistentVolumeClaimRetentionPolicy` lets you opt in to automatic cleanup, separately for *deletion* of
the set and for *scaling down* (`whenScaled`): handy for test environments, risky for anything
precious.

**Try**

Repeat the cycle with `kubectl delete sts web --cascade=orphan`: the pods keep running, unmanaged;
`kubectl apply -f statefulset.yml` makes the new StatefulSet adopt them. (Re-apply the file first, since
the previous step deleted the set.)

---

## Cleanup for this session

    kubectl delete sts web --ignore-not-found
    kubectl delete -f service.yml -f tools.yml --ignore-not-found
    kubectl delete pvc --all
    kubectl get sts,pods,svc,pvc,pv                  # nothing of yours left

---

## End of Session 1: checkpoint

    kubectl get sts,pods,pvc,pv           # No resources found

Session 1 is done. Session 2 (`../17-statefulset-ordering-updates/`) can continue from here, or
start from scratch: it has its own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `web-1` never appears | `web-0` isn't Ready yet: `kubectl describe pod web-0`. Nothing after a stuck pod is created (OrderedReady) |
| A claim stays `Pending` | The class waits for the pod to be scheduled; `kubectl describe pvc www-web-0` and `kubectl describe pod web-0` |
| `nslookup web-0.web` fails | Headless Service missing, or its name differs from `serviceName`, or the pod isn't Ready (DNS only lists ready pods) |
| Deleted the StatefulSet but the claims are still there | Expected (`Retain`); delete them with `kubectl delete pvc` |
| Pod `Pending` after re-creating the set | Its volume is pinned to a node that is unavailable: `kubectl describe pod` |
| Pods take long to delete | This lab sets `terminationGracePeriodSeconds: 3`; without it a pod whose main process ignores SIGTERM waits 30 s (`kubectl delete pod <name> --now` shortens it) |
| `ImagePullBackOff` | No internet, or a mistyped image tag. `kubectl describe pod`, read Events |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] list what a StatefulSet guarantees that a Deployment doesn't
- [ ] explain why a StatefulSet needs a headless Service, and what its DNS names look like
- [ ] explain how a re-created pod finds its own volume (the claim name)
- [ ] say what happens to the claims when a pod, then the StatefulSet, is deleted
- [ ] explain `persistentVolumeClaimRetentionPolicy` and when you would change it

## Stretch ideas

- Set `spec.ordinals.start: 1` (recent versions) and see the pod names start at `web-1`.
- Add a `topologySpreadConstraints` or pod anti-affinity rule so the three pods land on different nodes.
- Compare `kubectl get sts web -o yaml` before and after a rollout: find `currentRevision` and `updateRevision`.
- Read `kubectl explain statefulset.spec` from top to bottom.
