# Session 3: Rolling updates, rollback, failure and cleanup 

**Work from this directory:** `cd 03-update-rollback-cleanup`

**Goal:** change a live app without downtime, inspect history, roll back,
survive a bad release, and clean everything up.

**Starting state:** kind cluster `lab-a`, kubectl context `kind-lab-a` with
default ns `lab`, `web-deploy` (3 pods, image `nginx:1.25`) and `web-svc`
running. 
Session 2 leaves you here; the bootstrap below gets you here from nothing.
**Ending state:** everything deleted, including the clusters.

Read the strategy section of `deploy.yml` (maxSurge / maxUnavailable) first.

---

## Golden rule

Before any command that changes something, 
know **which cluster and which namespace** you are on:

    kubectl config current-context
    kubectl config get-contexts      # NAMESPACE column; empty = default

Applying to the wrong cluster, or looking in the wrong namespace, are the two
most common real-world mistakes.

## Prerequisites

    docker version
    kind version
    kubectl version --client

Docker must be running and port **30001** must be free on your machine.

---

## Starting checks

    kubectl config current-context                       # kind-lab-a
    kubectl config get-contexts                          # NAMESPACE lab on kind-lab-a
    kubectl get deploy,svc                               # web-deploy 3/3 and web-svc
    kubectl describe deploy web-deploy | grep Image      # nginx:1.25
    curl -s -o /dev/null -w "%{http_code}\n" localhost:30001     # 200

All match: go straight to Step 7.

### Bootstrap (if anything above is missing)

No `lab-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name lab-a --config cluster.yml
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab
    kubectl apply -f deploy.yml -f svc.yml
    kubectl rollout status deploy/web-deploy

Cluster exists but the app is missing, or the image is not `nginx:1.25`
(`apply` resets it to what the file says):

    kubectl config use-context kind-lab-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab
    kubectl apply -f deploy.yml -f svc.yml
    kubectl rollout status deploy/web-deploy

For a pristine history (revision numbers starting at 1), delete the app:
    
    `kubectl delete -f svc.yml -f deploy.yml`, 

then apply again.

Completely broken cluster: 
    
    `kind delete cluster --name lab-a`, then the first block above.

---

## Step 7: Rolling update

Use two terminals (both on `kind-lab-a`; 
the namespace default is stored in your kubeconfig, so it applies to both).

**Do: terminal 1 (watch)**

    kubectl get pods -w

**Do: terminal 2 (change)**

    kubectl annotate deploy web-deploy kubernetes.io/change-cause="update nginx 1.25 to 1.27"
    kubectl set image deploy/web-deploy nginx=nginx:1.27
    kubectl rollout status deploy/web-deploy
    kubectl get rs
    kubectl describe deploy web-deploy | grep Image

**Expect**

- Terminal 1: new pods appear and old ones terminate in small batches.
  At most 4 pods exist at once (3 + maxSurge 1) and at least 2 stay
  available (3 - maxUnavailable 1).
- `rollout status` ends with `successfully rolled out`.
- `get rs` shows **two** ReplicaSets: the old one at 0 pods, the new one at 3.
- Image is now `nginx:1.27`.

**Why**

Changing the pod template creates a new ReplicaSet. The Deployment scales it 
up and the old one down, within the maxSurge/maxUnavailable limits. 
The old ReplicaSet is kept (at 0) as the rollback target.

**Try**

Run a request loop in another terminal *during* the update (before you run
`set image`):

    while true; do curl -s -o /dev/null -w "%{http_code}\n" localhost:30001; sleep 0.5; done

You should only see `200`: no downtime. (Ctrl+C to stop.)

---

## Step 8: Inspect and roll back

**Do**

    kubectl rollout history deploy/web-deploy
    kubectl rollout history deploy/web-deploy --revision=2
    kubectl rollout undo deploy/web-deploy
    kubectl rollout status deploy/web-deploy
    kubectl describe deploy web-deploy | grep Image
    kubectl rollout history deploy/web-deploy

**Expect**

- History lists revisions 1 and 2. CHANGE-CAUSE shows your annotation for
  revision 2 and `<none>` for revision 1.
- `--revision=2` shows that revision's pod template (image `nginx:1.27`).
- After `undo`, the image is back to `nginx:1.25`.
- History now shows revisions 2 and **3**. Revision 1 is gone because rolling
  back to it re-labelled it as revision 3. A rollback is a new rollout that
  reuses an old ReplicaSet, not a time machine.

**Why**

Rollback is just "point the Deployment at an old ReplicaSet's template",
done with the same rolling mechanism, so it is also zero-downtime.

**Try**

Roll back to a specific revision: 

    `kubectl rollout undo deploy/web-deploy --to-revision=2`.

The change-cause annotation is not updated automatically on undo, so
re-annotate before each change if you want readable history.

---

## Step 9: Break it on purpose, then recover

**Do**

    kubectl annotate deploy web-deploy kubernetes.io/change-cause="update break" --overwrite
    kubectl set image deploy/web-deploy nginx=nginx:does-not-exist
    kubectl rollout status deploy/web-deploy      # waits, then errors after ~60s
    kubectl get pods
    kubectl describe pod <a-failing-pod>          # read the Events
    curl localhost:30001                          # still works!
    kubectl rollout undo deploy/web-deploy
    kubectl rollout status deploy/web-deploy

**Expect**

- Some new pods show `ErrImagePull`, then `ImagePullBackOff`.
- The rollout **stalls**: roughly 2 old pods stay Running and 2 new pods are
  stuck. The limits (max 1 unavailable) stop Kubernetes from removing more
  working pods.
- `rollout status` reporting the deployment exceeded its progress deadline.
- `describe pod` Events show `Failed to pull image ... not found`.
- `curl` still returns 200 because the surviving old pods keep serving.
- After `undo`, broken pods are removed and you are back to 3 healthy pods.

**Why**

This is the safety net of rolling updates: a bad version never takes down
the whole app. Kubernetes does not roll back by itself; you detect the
problem and run `undo`.

---

## Step 10: Cleanup

**Do**

    kubectl get all                      # everything the app consists of
    kubectl delete ns lab
    kubectl get ns
    kubectl get all -n lab
    kind delete cluster --name lab-a
    kind delete cluster --name lab-b
    kind get clusters
    kubectl config get-contexts
    docker ps

**Expect**

- `get all` lists the Deployment, ReplicaSets, pods and Service in `lab`.
- `delete ns lab` takes a few seconds (the namespace shows `Terminating`) and
  removes **everything inside it** with a single command: no need to delete
  the Service and Deployment one by one.
- `get all -n lab` then says `No resources found in lab namespace.`
- After the clusters are deleted, the `kind-*` contexts disappear and no
  context may be marked current. Select one explicitly:

      kubectl config use-context docker-desktop

- `docker ps` no longer shows the kind containers.

**Why**

Namespaces make cleanup safe and complete: group an app's objects in one
namespace and you can remove it all at once. (The inverse is also true:
deleting a namespace is destructive, so always check which cluster you are
on first.)

**Try (before deleting the clusters)**

Delete only `kubectl delete -f svc.yml -f deploy.yml` instead, and compare:  
the `lab` namespace stays behind, empty.

---

## Lab complete

Nothing from this lab is left on your machine.  
Below: troubleshooting, a self-check and ideas for going further.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `rollout status` never finishes | Expected in Step 9 (it errors after ~60 s). Otherwise `kubectl get pods` and `describe pod` |
| `CHANGE-CAUSE` shows `<none>` | You did not annotate before the change; it is not set automatically |
| Revision numbers look odd after `undo` | Normal: an undo creates a new revision from an old ReplicaSet |
| Request loop shows non-200 codes | Are you on lab-a? Was the Service applied? `kubectl get endpointslices` |
| `curl localhost:30001` refused | `kubectl get svc`; the cluster must come from `cluster.yml` |
| `delete ns lab` seems stuck | It shows `Terminating` for a few seconds; wait, then `kubectl get ns` |
| `kind delete cluster --name lab-b` prints nothing useful | `lab-b` only exists if you did Session 1; deleting a missing cluster is harmless |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] perform a rolling update and explain maxSurge / maxUnavailable
- [ ] read `rollout history` and `--revision=N`
- [ ] roll back, and explain why revision numbers move forward
- [ ] diagnose a failed rollout with `describe` and recover with `undo`
- [ ] explain why the app kept serving during the failed update
- [ ] clean up a whole app by deleting its namespace

## Stretch ideas

- Change `maxSurge`/`maxUnavailable` (0 and 1, or 100%) and compare rollouts.
- `kubectl rollout pause deploy/web-deploy`, change the image, then `resume`.
- Switch `strategy.type` to `Recreate` and watch the downtime with the loop.
- Add a `readinessProbe` and see how a failing probe blocks a rollout.
- Lower `revisionHistoryLimit` and see which revisions disappear.
