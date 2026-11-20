# StorageClasses in depth - defaults, binding, expansion, reclaim 

**Work from this directory:** `cd 12-storage-classes` 

**Goal:** explore what a StorageClass controls: which class a claim gets (and the three
meanings of `storageClassName`), when binding happens, whether a volume can grow, and what
happens to the data when the claim is deleted.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, no volumes of your own, only the `standard` class.
**Ending state:** everything from this session deleted: classes, volumes, claims, directories.

**StorageClass fields at a glance**

| Field | Controls |
|---|---|
| `provisioner` | which controller creates volumes (`no-provisioner` = nobody) |
| `reclaimPolicy` | `Delete` or `Retain`: what happens to the volume when its claim is deleted |
| `volumeBindingMode` | `Immediate` or `WaitForFirstConsumer`: when binding/provisioning happens |
| `allowVolumeExpansion` | whether a bound claim may be resized |
| `parameters` / `mountOptions` | provisioner-specific settings / mount flags |

Read the top of the YAML files as you use them.

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

All match: go straight to Step 40.

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

## Step 40: Classes, the default, and three meanings of `storageClassName`

**Do**

    kubectl apply -f storageclass-retain.yml
    kubectl get storageclass
    kubectl apply -f pvc-variants.yml
    kubectl get pvc
    kubectl get pvc pvc-default -o jsonpath='{.spec.storageClassName}{"\n"}'
    kubectl describe pvc pvc-default | tail -4
    kubectl describe pvc pvc-none | tail -4

**Expect**

- Two classes: `standard (default)` and `lab-retain`, the latter with RECLAIMPOLICY `Retain`.
- Three claims, all `Pending`:
  - `pvc-default` (no class written): its class is now **`standard`**. The API server filled in the default.
    Event: waiting for first consumer.
  - `pvc-named`: class `lab-retain`. Same wait.
  - `pvc-none` (`storageClassName: ""`): **empty** STORAGECLASS. Event: no persistent
    volumes available and no storage class is set. It will never bind (no PV without a class exists).

**Why**

`storageClassName` has three meanings: *omitted* = "the default class", *a name* =
"that class", *empty string* = "**no** class": no provisioner will ever act on it. That
last form is how a claim says "only a hand-made, class-less PV, please". Mixing up
*omitted* and *empty* is a classic mistake.

**Try (and restore!)**

Remove the default flag and see what an unnamed claim becomes:

    kubectl patch storageclass standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
    kubectl get storageclass
    kubectl apply -f pvc-nodefault.yml
    kubectl get pvc pvc-nodefault                       # empty class
    kubectl delete pvc pvc-nodefault
    kubectl patch storageclass standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    kubectl get storageclass                            # standard (default) again

---

## Step 41: Binding modes: Immediate versus WaitForFirstConsumer

**Do**

    docker exec lab-a-worker mkdir -p /mnt/disks/imm /mnt/disks/wffc
    kubectl apply -f binding-modes.yml
    kubectl get storageclass
    kubectl get pv,pvc
    kubectl apply -f pod-wffc.yml
    kubectl wait --for=condition=Ready pod/pod-wffc --timeout=90s
    kubectl get pv,pvc
    kubectl get pod pod-wffc -o wide

**Expect**

- Two new static classes: `lab-immediate` (`Immediate`) and `manual-local` (`WaitForFirstConsumer`).
- Right after the apply: **`claim-imm` is already `Bound`** to `pv-imm` although no pod uses it;
  **`claim-wffc` is `Pending`** and `pv-wffc` is still `Available`.
- After `pod-wffc` is created: `claim-wffc` binds to `pv-wffc` and the pod runs on `lab-a-worker`.

**Why**

With `Immediate` the claim binds as soon as it exists, without knowing where the pod will
run. For storage that exists in only one place (a node's disk, a cloud availability zone) the
chosen volume can make the pod impossible to schedule. `WaitForFirstConsumer` lets the
scheduler pick the node *and* the volume together, which is why it is the recommended mode for
topology-bound storage and why kind's default class uses it.

**Try**

Create a pod for `claim-imm` and note where it lands: the PV's node affinity guides the scheduler.

---

## Step 42: Volume expansion: a permission, not a feature

**Do**

    kubectl apply -f pod-default.yml
    kubectl wait --for=condition=Ready pod/default-pod --timeout=120s
    kubectl get storageclass -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner,EXPAND:.allowVolumeExpansion
    kubectl patch pvc pvc-default -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
    kubectl apply -f expand.yml
    kubectl wait --for=condition=Ready pod/expand-pod --timeout=120s
    kubectl patch pvc pvc-expand -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
    kubectl get pvc -o custom-columns=NAME:.metadata.name,CLASS:.spec.storageClassName,REQUESTED:.spec.resources.requests.storage,CAPACITY:.status.capacity.storage
    kubectl describe pvc pvc-expand | tail -8
    kubectl patch pvc pvc-expand -p '{"spec":{"resources":{"requests":{"storage":"500Mi"}}}}'

**Expect**

- `EXPAND` is empty for `standard` and `lab-retain`, `true` for `lab-expand`.
- Resizing `pvc-default` is **rejected**: an error saying only dynamically provisioned claims whose
  class supports resizing can be resized.
- Resizing `pvc-expand` is **accepted**, but the table shows REQUESTED `2Gi` while CAPACITY is still
  `1Gi`, and the events note that nothing in the cluster can do the expansion (an
  `ExternalExpanding`-style message). The flag only grants permission.
- Shrinking is rejected: a claim can never be reduced.

**Why**

`allowVolumeExpansion: true` lets you *ask*. Something must *do* it: a CSI driver with an
expander, or a cloud provider, which resizes the disk and its filesystem (sometimes requiring a pod
restart). kind's local-path provisioner can't, so you can see the request pending forever.

**Try**

Fields of a StorageClass are mostly immutable. Predict, then try:

    kubectl patch storageclass lab-retain -p '{"reclaimPolicy":"Delete"}'

(Only a few fields, such as `allowVolumeExpansion`, may change. To change anything else,
create a new class.)

**Tidy up before Step 43**

    kubectl delete pod default-pod expand-pod
    kubectl delete pvc pvc-default pvc-expand

---

## Step 43: Reclaim policy in practice: Retain, and rescuing a Delete volume

**Do: A. a class that retains**

    kubectl apply -f pod-retain.yml
    kubectl wait --for=condition=Ready pod/retain-pod --timeout=120s
    PV=$(kubectl get pvc pvc-named -o jsonpath='{.spec.volumeName}'); echo $PV
    kubectl get pv $PV
    kubectl exec retain-pod -- sh -c 'echo "keep me" > /data/keep.txt'
    NODE=$(kubectl get pod retain-pod -o jsonpath='{.spec.nodeName}'); echo $NODE
    kubectl delete pod retain-pod
    kubectl delete pvc pvc-named
    kubectl get pv
    docker exec $NODE sh -c 'cat /var/local-path-provisioner/*/keep.txt'
    kubectl delete pv $PV
    docker exec $NODE sh -c 'cat /var/local-path-provisioner/*/keep.txt'
    docker exec $NODE sh -c 'ls /var/local-path-provisioner'
    docker exec $NODE sh -c 'rm -rf /var/local-path-provisioner/*_lab_pvc-named'

**Do: B. protect a volume whose class says Delete**

    kubectl apply -f protect.yml
    kubectl wait --for=condition=Ready pod/protect-pod --timeout=120s
    PV2=$(kubectl get pvc pvc-protect -o jsonpath='{.spec.volumeName}'); echo $PV2
    kubectl get pv $PV2
    kubectl exec protect-pod -- sh -c 'echo "precious" > /data/precious.txt'
    NODE2=$(kubectl get pod protect-pod -o jsonpath='{.spec.nodeName}'); echo $NODE2
    kubectl patch pv $PV2 -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
    kubectl get pv $PV2
    kubectl delete pod protect-pod
    kubectl delete pvc pvc-protect
    kubectl get pv
    docker exec $NODE2 sh -c 'cat /var/local-path-provisioner/*/precious.txt'

**Expect**

- A: the PV for `pvc-named` has RECLAIM POLICY `Retain`. After the claim is deleted it is
  **`Released`**, and `keep.txt` is still on the node. Even after you delete the **PV object**, the
  directory and the file are still on the node: `Retain` never removes the backing storage.
  You clean it up by hand (last command: it deletes only that volume's directory).
- B: the PV starts as `Delete` (from `standard`). After you patch **the PV** to `Retain`, deleting the
  claim leaves it **`Released`** with the data intact (`precious`), although the class says `Delete`.

**Why**

The class's `reclaimPolicy` is only the *initial* value copied into each PV. From then on the **PV's own
`persistentVolumeReclaimPolicy`** is what counts, and it is editable. Before deleting a claim whose
data you might need, check the PV's policy, and patch it to `Retain` if in doubt. Also: PVs are
cluster-wide, so deleting a *namespace* removes its claims but leaves `Retain` volumes behind.

---

## Cleanup for this session

    kubectl delete ns lab
    kubectl delete pv pv-imm pv-wffc $PV2 --ignore-not-found
    kubectl delete storageclass lab-retain lab-immediate manual-local lab-expand --ignore-not-found
    for n in lab-a-worker lab-a-worker2; do docker exec $n sh -c 'rm -rf /mnt/disks /var/local-path-provisioner/*_lab_*'; done
    kubectl get pv,storageclass
    kubectl apply -f namespace.yml

Deleting the namespace removes `pvc-none` and the remaining claims and pods. The last command
re-creates the empty `lab` namespace so the context default still works. (If `$PV2` is empty
because you opened a new terminal, delete the leftover `pvc-...` PV by name: `kubectl get pv`.)

---

## End of Session 3: checkpoint

    kubectl get pv,pvc -A                 # No resources found
    kubectl get storageclass              # standard (default) only

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Claim `Pending` | Is there a pod using it (WaitForFirstConsumer)? `kubectl describe pvc` shows the reason |
| `pvc-none` never binds | Expected: `storageClassName: ""` and no class-less PV |
| `claim-wffc` bound to a leftover PV | The claims select PVs by label `lab=binding-modes`; check `kubectl get pv --show-labels` |
| `storageclass ... forbidden: updates to ... are forbidden` | StorageClass fields are mostly immutable; create a new class |
| PVC resize rejected | The class has no `allowVolumeExpansion: true` (Step 42) |
| PVC resize accepted but CAPACITY unchanged | Nothing implements expansion on kind (expected) |
| `cat .../keep.txt`: no such file | You may be on the wrong node: `kubectl get pod retain-pod -o wide` before deleting it; use `$NODE` |
| Two defaults, or none, after the Step 40 experiment | Re-run the last `kubectl patch` of that step; `kubectl get storageclass` shows `(default)` |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] name the fields of a StorageClass and what each controls
- [ ] explain omitted, named and `""` for `storageClassName`
- [ ] explain Immediate versus WaitForFirstConsumer, and when each is dangerous
- [ ] explain why `allowVolumeExpansion` alone doesn't resize anything
- [ ] explain where the reclaim policy really lives (the PV), and how to protect a volume
- [ ] explain why deleting a namespace can leave volumes behind

## Stretch ideas

- Make `lab-retain` the default class (and `standard` not), observe what an unnamed claim gets, then restore.
- Add `mountOptions` or `parameters` to a StorageClass and read what `kubectl describe sc` reports.
- Look at the local-path provisioner's ConfigMap to see how paths are chosen per node (`nodePathMap`).
- Read about VolumeSnapshots and CSI (`kubectl api-resources | grep -i snapshot`): what is missing on kind?
