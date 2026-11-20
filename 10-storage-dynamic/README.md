# Dynamic provisioning - StorageClass, PVC and PV 

**Work from this directory:** `cd 10-storage-dynamic`

**Goal:** meet the three storage objects (StorageClass, PersistentVolume,
PersistentVolumeClaim), watch a claim turn into a volume automatically, prove that
data outlives a pod, and see what happens when the claim is deleted.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, no volumes of your own.
**Ending state:** everything from this session deleted (claim, volume, data).

**The three objects, in one line each**

- **StorageClass**: the *recipe*: which provisioner creates volumes, and with which policies.
- **PersistentVolume (PV)**: a piece of storage that exists. Cluster-wide, no namespace.
- **PersistentVolumeClaim (PVC)**: a *request* for storage. Lives in a namespace; pods mount claims.

Read the top of `pvc.yml` and `pod.yml` before you start.

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

Docker must be running. No cloud and no extra tools are needed: kind's built-in
storage (local directories on the node containers) is enough for everything here.

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get storageclass              # standard (default) only
    kubectl get pv,pvc -A                 # No resources found

All match: go straight to Step 32.

### Bootstrap (if anything above is missing)

No `lab-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name lab-a --config cluster.yml
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Cluster exists but the namespace default is wrong or missing:

    kubectl config use-context kind-lab-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Leftovers in `lab` from other sessions or experiments:

    kubectl delete ns lab
    kubectl apply -f namespace.yml

PersistentVolumes and StorageClasses are **cluster-wide**:  
deleting the namespace does not remove them. 
List what is left and delete extras by name (only `standard` should remain):

    kubectl get pv,storageclass
    kubectl delete pv <name>
    kubectl delete storageclass <name>

Leftover directories on the nodes, from the static-volume sessions:

    docker exec lab-a-worker sh -c 'rm -rf /mnt/disks'
    docker exec lab-a-worker2 sh -c 'rm -rf /mnt/disks'

Completely broken cluster: `kind delete cluster --name lab-a`, then the first block above.

---

## Step 32: Meet the storage objects in your cluster

**Do**

    kubectl get storageclass
    kubectl describe storageclass standard
    kubectl -n local-path-storage get pods
    kubectl -n local-path-storage get cm local-path-config -o jsonpath='{.data.config\.json}{"\n"}'
    kubectl get pv
    kubectl get pvc -A
    docker exec lab-a-worker ls -la /var/local-path-provisioner

**Expect**

- One class: `standard (default)`, PROVISIONER `rancher.io/local-path`, RECLAIMPOLICY
  `Delete`, VOLUMEBINDINGMODE `WaitForFirstConsumer`, ALLOWVOLUMEEXPANSION empty/false.
- `describe` shows `IsDefaultClass: Yes`.
- A pod `local-path-provisioner-...` Running in namespace `local-path-storage`: this
  is the *provisioner*, the controller that creates volumes for the class.
- Its ConfigMap points at a base directory, `/var/local-path-provisioner`, on each node.
- No PVs and no PVCs yet. The directory on the node may not exist yet (it is
  created on first use).

**Why**

A StorageClass is the cluster's menu of storage types; a provisioner is the
controller behind one entry. kind ships a tiny provisioner that turns a claim into
a plain directory on a node. On a cloud, the provisioner (a CSI driver) would create
an EBS, Persistent Disk or Azure Disk instead; your manifests would look the same.

**Try**

    kubectl get sc standard -o yaml            # see the default-class annotation
    kubectl get csidrivers                     # kind has no CSI driver installed

---

## Step 33: A claim, a pod, and a volume that appears

**Do**

    kubectl apply -f pvc.yml
    kubectl get pvc data-claim
    kubectl describe pvc data-claim
    kubectl get pv                                 # nothing yet
    kubectl apply -f pod.yml
    kubectl wait --for=condition=Ready pod/writer --timeout=120s
    kubectl get pvc,pv
    kubectl get pod writer -o wide
    PV=$(kubectl get pvc data-claim -o jsonpath='{.spec.volumeName}'); echo $PV
    kubectl describe pv $PV
    kubectl -n local-path-storage logs deploy/local-path-provisioner --tail=15

**Expect**

- Right after the first apply, the claim is `Pending` with `STORAGECLASS standard`
  (the default was filled in for you) and an event saying it is **waiting for first
  consumer to be created before binding**. No PV exists.
- After the pod starts: the claim is `Bound` to a PV named `pvc-<uuid>`, size `1Gi`,
  access `RWO`, reclaim policy `Delete`, `CLAIM lab/data-claim`, class `standard`.
- `describe pv` shows a node-local source (a path like
  `/var/local-path-provisioner/pvc-..._lab_data-claim`) and **Node Affinity**: the volume
  belongs to one node, and the pod runs on that node.
- The provisioner's log mentions the claim, the chosen node, and the directory it created.

**Why**

This is *dynamic provisioning*. The claim waits (WaitForFirstConsumer) so that the
scheduler can pick a node for the pod first; the provisioner then creates the
storage **on that node** and the PV, and the claim binds. Nobody created a PV by
hand. If the class used `Immediate` binding, the volume would be created as soon
as the claim existed, before the cluster knew where the pod would run.

**Try**

    kubectl describe pod writer | sed -n '/Volumes:/,/QoS/p'     # data (claim) and scratch (emptyDir)
    kubectl get pvc data-claim -o yaml | grep -E 'storageClassName|volumeName'

---

## Step 34: Persistent versus ephemeral

**Do**

    kubectl exec writer -- sh -c 'echo "from the claim" > /data/note.txt; echo "from emptyDir" > /scratch/note.txt'
    kubectl exec writer -- sh -c 'ls -l /data /scratch; cat /data/note.txt /scratch/note.txt'
    NODE=$(kubectl get pod writer -o jsonpath='{.spec.nodeName}'); echo $NODE
    docker exec $NODE sh -c 'cat /var/local-path-provisioner/*/note.txt'
    kubectl delete pod writer
    kubectl apply -f pod.yml
    kubectl wait --for=condition=Ready pod/writer --timeout=120s
    kubectl exec writer -- sh -c 'ls -l /data /scratch'
    kubectl get pod writer -o wide

**Expect**

- Both files exist while the pod runs.
- The file written to `/data` is visible **on the node's disk** (the directory
  that backs the volume).
- After deleting and recreating the pod: `/data/note.txt` is still there,
  `/scratch` is **empty**. The new pod runs on the **same node**, because the
  volume is tied to it.

**Why**

A pod is disposable; a claim is not. The data lives in the PV, so any pod that
mounts the claim sees it again. An `emptyDir` is scratch space created with the
pod and removed with it (it does survive a *container* restart inside the pod,
but not the pod's deletion).

---

## Step 35: Protection and reclaim: what deletion does

**Do**

    kubectl delete pvc data-claim --wait=false
    kubectl get pvc data-claim
    kubectl describe pvc data-claim | grep -E 'Status|Used By|Finalizers'
    kubectl delete pod writer
    kubectl get pvc,pv
    docker exec $NODE ls /var/local-path-provisioner

**Expect**

- While the pod still uses it, the claim shows `Terminating` and does **not** go
  away. `describe` shows `Used By: writer` and the finalizer
  `kubernetes.io/pvc-protection`.
- After the pod is deleted, the claim disappears at once, then (within seconds)
  the **PV disappears too**, and the directory on the node is removed.

**Why**

Two safety mechanisms. *PVC protection* refuses to delete a claim that a running pod
is using. The *reclaim policy* (`Delete`, copied from the class) decides what
happens to the volume when its claim goes away: with `Delete`, the PV and the
data are removed. Handy for scratch environments, dangerous for anything precious;
Session 3 shows `Retain`.

**Try**

    kubectl get pv -w                  # in a second terminal, then repeat Steps 33-35

---

## End of Session 1: checkpoint

    kubectl get pvc,pv,pods               # No resources found
    docker exec lab-a-worker ls /var/local-path-provisioner      # empty (or missing)
    docker exec lab-a-worker2 ls /var/local-path-provisioner     # empty (or missing)

Session 1 is done. Session 2 (`../02-storage-static/`) can continue from here, or
start from scratch: it has its own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Claim stays `Pending` | Expected until a pod uses it (WaitForFirstConsumer). If the pod exists: `kubectl describe pvc` and the provisioner pod (`kubectl -n local-path-storage get pods`) |
| Claim `Pending` and no pod can use it | `kubectl get storageclass`: is there a default class? Does the claim's `storageClassName` exist? |
| Pod `Pending` | `kubectl describe pod writer`, read Events (unbound claim? image pull?) |
| Claim stuck `Terminating` | A pod still uses it (`Used By:` in `describe pvc`). Delete the pod |
| PV not deleted after the claim | Give the provisioner a few seconds; `kubectl -n local-path-storage logs deploy/local-path-provisioner` |
| `docker exec ... ls`: no such file | The base directory is created on first use; fine |
| `ImagePullBackOff` | No internet, or mistyped image tag. `kubectl describe pod`, read Events |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] say what a StorageClass, a PV and a PVC are, and which are namespaced
- [ ] explain why a claim is `Pending` until a pod exists (WaitForFirstConsumer)
- [ ] follow a claim to its PV, its node and its directory
- [ ] explain why data survives pod deletion but `emptyDir` does not
- [ ] explain PVC protection and what reclaim policy `Delete` does

## Stretch ideas

- Run a Deployment with 3 replicas that all mount one claim: where do the pods land, and why?
- Look at `StatefulSet` and `volumeClaimTemplates`: one claim per replica.
- `kubectl explain pvc.spec` and `kubectl explain pv.spec`: read every field.
- Run two claims at once and compare their directories on the node.
