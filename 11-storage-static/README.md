# Static provisioning - PersistentVolumes by hand 

**Work from this directory:** `cd 11-storage-static`

**Goal:** play the administrator: create storage by hand, expose it as PersistentVolumes,
and watch claims bind to them. Learn the matching rules, the PV life cycle (Available, Bound,
Released), what `Retain` really means, and how access modes behave.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, no volumes of your own.
**Ending state:** everything from this session deleted: PVs, class, claims and node directories.

**Static versus dynamic.** In Session 1 the provisioner created the PV for you. Here
*you* create the PVs first (the *supply*) and claims (the *demand*) bind to them. This is
how storage worked before provisioners, and it is still how pre-existing disks, NFS shares
and local disks are used.

Read the top of `storageclass-manual.yml`, `pv.yml` and `claims.yml` before you start.

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

All match: go straight to Step 36.

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

PersistentVolumes and StorageClasses are **cluster-wide**: deleting the namespace
does not remove them. List what is left and delete extras by name (only
`standard` should remain among the classes):

    kubectl get pv,storageclass
    kubectl delete pv <name>
    kubectl delete storageclass <name>

Leftover directories on the nodes, from the static-volume sessions:

    docker exec lab-a-worker sh -c 'rm -rf /mnt/disks'
    docker exec lab-a-worker2 sh -c 'rm -rf /mnt/disks'

Completely broken cluster: `kind delete cluster --name lab-a`, then the first block above.

---

## Step 36: Create "disks" and register them as PersistentVolumes

**Do**

    kubectl get nodes
    docker exec lab-a-worker mkdir -p /mnt/disks/vol1
    docker exec lab-a-worker2 mkdir -p /mnt/disks/vol2
    kubectl apply -f storageclass-manual.yml
    kubectl get storageclass
    kubectl apply -f pv.yml
    kubectl get pv
    kubectl describe pv pv-vol1
    kubectl get pvc -A

**Expect**

- The new class `manual-local` has PROVISIONER `kubernetes.io/no-provisioner` (it never
  creates anything) and `WaitForFirstConsumer`.
- Two PVs: `pv-vol1` (1Gi) and `pv-vol2` (2Gi), `RWO`, reclaim `Retain`, STATUS `Available`,
  CLAIM empty.
- `describe pv pv-vol1`: source type local volume, path `/mnt/disks/vol1`, and **Node Affinity**
  `kubernetes.io/hostname in [lab-a-worker]`.
- No claims exist.

**Why**

A kind node is a Docker container, so creating a directory inside it is "attaching a
disk". The PV is the cluster's record of that storage: its size, access modes, class,
reclaim policy, and where it lives. PVs exist whether or not anyone uses them, and are
cluster-wide. A `local` volume lives on exactly one node, so it **must** declare a node
affinity: that is how the scheduler knows where a pod using it can run.

**Try**

Remove the whole `nodeAffinity` block from a copy of `pv.yml` and apply it: the API
server rejects a `local` volume without one.

---

## Step 37: Claims, and the rules of binding

**Do**

    kubectl apply -f claims.yml
    kubectl get pvc
    kubectl describe pvc claim-a | tail -5
    kubectl get pv                                 # still Available: no pod yet
    kubectl apply -f pods.yml
    kubectl get pvc,pv
    kubectl get pods -o wide
    kubectl describe pod pod-c | tail -8
    kubectl describe pvc claim-c | tail -5

**Expect**

- After the claims are applied, all three are `Pending` and both PVs stay `Available`
  (nothing binds until a pod exists: WaitForFirstConsumer).
- After the pods: `claim-a` is `Bound` to **`pv-vol1`** and `claim-b` to **`pv-vol2`**.
  `pod-a` runs on `lab-a-worker` and `pod-b` on `lab-a-worker2`: each pod went where its
  volume is.
- `claim-a` asked for 500Mi but shows **CAPACITY 1Gi**: a claim gets the whole PV.
- `pod-c` stays `Pending`. Its events say no node *didn't find available persistent
  volumes to bind*; `claim-c` (5Gi) can't be satisfied by any PV.

**Why**

A claim binds to a PV only if all of these hold: same `storageClassName`; the PV's
access modes include the claim's; the PV's capacity is **at least** the requested
size; and the PV carries the labels the claim's `selector` asks for. Binding is one to
one, and the claim receives the whole volume, not a slice.

**Try**

Make `claim-c` succeed: copy the `pv-vol2` block of `pv.yml` into a new file as `pv-vol4`
(6Gi, path `/mnt/disks/vol4`, node `lab-a-worker`), create the directory on that node,
apply it, and watch `pod-c` start.

---

## Step 38: The life cycle and what Retain really means

**Do**

    kubectl exec pod-a -- sh -c 'echo "claim-a was here" > /data/who.txt'
    docker exec lab-a-worker cat /mnt/disks/vol1/who.txt
    kubectl delete pod pod-a
    kubectl delete pvc claim-a
    kubectl get pv
    kubectl describe pv pv-vol1 | grep -E 'Status|Claim|Reclaim'
    docker exec lab-a-worker cat /mnt/disks/vol1/who.txt
    kubectl apply -f claims.yml -f pods.yml
    kubectl get pvc claim-a
    kubectl describe pod pod-a | tail -6
    kubectl patch pv pv-vol1 -p '{"spec":{"claimRef":null}}'
    kubectl get pv
    kubectl wait --for=condition=Ready pod/pod-a --timeout=90s
    kubectl exec pod-a -- cat /data/who.txt
    docker exec lab-a-worker sh -c 'rm -rf /mnt/disks/vol1/*'

**Expect**

- After the claim is deleted, `pv-vol1` is **`Released`** (not `Available`): it still
  records the old claim (`Claim: lab/claim-a`). The file is still on the node.
- The re-created `claim-a` and `pod-a` stay **Pending**: a Released volume is not offered
  to new claims, so the pod's events say no PV was found.
- Clearing `claimRef` makes the PV `Available` again, the new claim binds and the pod
  starts. And `/data/who.txt` **still contains "claim-a was here"**: with `Retain`,
  nobody cleaned the volume, so the next claim inherits the old data.

**Why**

The PV life cycle: `Available` (free) -> `Bound` (claimed) -> `Released` (claim gone,
data still there) -> back to `Available` once an admin has dealt with it. `Retain` means
"never delete or wipe this for me": the safe choice for important data, but reuse is a
manual, deliberate step. Note the security angle: a retained volume reused by a different
team would expose the previous team's data unless you wipe it first (the last command).

**Try**

Pre-bind a claim to a specific volume: add `volumeName: pv-vol2` to a claim's `spec`
(see `kubectl explain pvc.spec.volumeName`).

---

## Step 39: Access modes in practice

**Do: A. ReadWriteOnce is "one node", not "one pod"**

    kubectl exec pod-a -- sh -c 'echo shared > /data/shared.txt'
    kubectl apply -f pod-shared.yml
    kubectl wait --for=condition=Ready pod/pod-a2 --timeout=90s
    kubectl get pods -o wide
    kubectl exec pod-a2 -- cat /data/shared.txt

**Do: B. ReadWriteOncePod is "one pod"**

    docker exec lab-a-worker2 mkdir -p /mnt/disks/vol3
    kubectl apply -f rwop.yml
    kubectl get pods -o wide | grep rwop
    kubectl describe pod <the-Pending-rwop-pod> | tail -8
    kubectl delete pod <the-Running-rwop-pod>
    kubectl get pods | grep rwop

**Expect**

- A: `pod-a2` runs on the **same node** as `pod-a`, using the same claim, and reads
  `shared`. Both pods share the one volume.
- B: one `rwop` pod is `Running`; the other is `Pending`, and its events mention the
  **ReadWriteOncePod** access mode (the claim is already in use by another pod).
  After you delete the running one, the other starts.

**Why**

| Mode | Short | Meaning |
|---|---|---|
| ReadWriteOnce | RWO | read-write by **one node** (many pods on it) |
| ReadOnlyMany | ROX | read-only by many nodes |
| ReadWriteMany | RWX | read-write by many nodes (needs a shared filesystem like NFS) |
| ReadWriteOncePod | RWOP | read-write by **one pod** in the whole cluster |

What is actually available depends on the storage behind it. Local disks only offer
RWO/RWOP; RWX needs network storage.

---

## Cleanup for this session

    kubectl delete pod --all
    kubectl delete pvc --all
    kubectl get pv                                 # Released: Retain keeps them
    kubectl delete pv pv-vol1 pv-vol2 pv-vol3 --ignore-not-found
    kubectl delete pv pv-vol4 --ignore-not-found
    kubectl delete storageclass manual-local
    docker exec lab-a-worker sh -c 'rm -rf /mnt/disks'
    docker exec lab-a-worker2 sh -c 'rm -rf /mnt/disks'
    kubectl get pv,pvc,storageclass

Deleting a PV object does **not** touch the data behind it: you removed the directories
yourself. (With a cloud disk the disk would also remain, and keep costing money, until you
delete it there.)

---

## End of Session 2: checkpoint

    kubectl get pv,pvc                    # No resources found
    kubectl get storageclass              # standard (default) only

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Pod `Pending`: "didn't find available persistent volumes to bind" | No `Available` PV matches the claim: class, size, access mode, selector. `kubectl get pv`, `kubectl describe pvc` |
| PV stays `Released` and won't bind again | It still has a `claimRef`: `kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'` |
| Pod `ContainerCreating` for long with a mount error | The directory doesn't exist on that node: `docker exec <node> ls /mnt/disks` |
| PV rejected: "Local volume requires node affinity" | Add the `nodeAffinity` block |
| `docker exec lab-a-worker2`: no such container | Node names are `lab-a-control-plane`, `lab-a-worker`, `lab-a-worker2`: `docker ps` |
| `rwop-2` never starts | Expected while `rwop-1` runs (ReadWriteOncePod) |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] write a PersistentVolume with capacity, access mode, class and reclaim policy
- [ ] explain why a `local` PV needs `nodeAffinity`
- [ ] list the four rules that decide whether a claim binds to a PV
- [ ] explain why a claim of 500Mi can show 1Gi
- [ ] describe Available, Bound and Released, and how to free a Released PV
- [ ] explain what `Retain` does, and its risk when the volume is reused
- [ ] tell RWO, RWX and RWOP apart

## Stretch ideas

- Create a PV with a `selector`-friendly label scheme (`env=dev`, `env=prod`) and write claims that target each.
- Mount a claim `readOnly: true` in a pod (`volumeMounts[].readOnly`).
- Try `hostPath` instead of `local` for a PV and compare what the API requires.
- Read `kubectl explain pv.spec.volumeMode` and the difference between `Filesystem` and `Block`.
