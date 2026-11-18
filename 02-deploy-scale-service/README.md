# Session 2: Deployments, scaling and Services 

**Work from this directory:** `cd 02-deploy-scale-service` 

**Goal:** see self-healing in action, scale a workload, and reach it through
a Service.

**Starting state:** kind cluster `lab-a` (3 nodes, port 30001 mapped), kubectl
context `kind-lab-a` with default namespace `lab`, and nothing deployed.
Session 1 leaves you exactly here; the bootstrap below gets you here from 0.
**Ending state:** `web-deploy` (3 pods) and `web-svc` are running in ns `lab`,  
reachable at `http://localhost:30001`.

Read the top of `deploy.yml` and `svc.yml` before you start.

---

## Golden rule

Before any command that changes something, 
know **which cluster and which namespace** you are on:

    kubectl config current-context
    kubectl config get-contexts           # NAMESPACE column; empty = default

Applying to the wrong cluster, or looking in the wrong namespace, are the two
most common real-world mistakes.

## Prerequisites

    docker version
    kind version
    kubectl version --client

Docker must be running and port **30001** must be free on your machine.

---

## Starting checks

    kubectl config current-context    # kind-lab-a
    kubectl get nodes                 # 3 nodes, all Ready
    kubectl config get-contexts       # NAMESPACE lab on kind-lab-a
    kubectl get pods                  # No resources found in lab namespace.

All four match: go straight to Step 4.

### Bootstrap (if anything above is missing)

No `lab-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name lab-a --config cluster.yml
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Cluster exists but the namespace default is wrong or missing:

    kubectl config use-context kind-lab-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Leftovers from earlier experiments in `lab`:

    kubectl delete ns lab
    kubectl apply -f namespace.yml

Completely broken cluster: `kind delete cluster --name lab-a`,  
then the first block above.

---

## Step 4: Deployment

**Do**

    kubectl apply -f deploy.yml
    kubectl get deploy web-deploy -w        # Ctrl+C when 3/3
    kubectl get deploy,rs,pods -o wide
    kubectl get pods --show-labels
    kubectl describe deploy web-deploy

Then delete one pod:

    kubectl delete pod <one-pod-name>
    kubectl get pods

**Expect**

- `READY 3/3`, one ReplicaSet named `web-deploy-<hash>` with 3 pods.
- Pod names are `web-deploy-<hash>-<random>`; the hash equals the ReplicaSet's.
- Pods are spread across `lab-a-worker` and `lab-a-worker2`.
- After deleting a pod, a new one with a different name appears within seconds.
- `describe` shows `Namespace: lab`, `Scaled up replica set ... from 0 to 3`, 
  and `RollingUpdateStrategy: 1 max unavailable, 1 max surge`.

**Why**

The ReplicaSet continuously compares *desired* (3) with *actual* pods and
fixes the difference. That's "self-healing" at the heart of Kubernetes.

**Try**

    kubectl get deploy -A       # from any namespace's point of view

---

## Step 5: Scaling

**Do**

    kubectl scale deploy web-deploy --replicas=6
    kubectl get pods -o wide
    kubectl scale deploy web-deploy --replicas=3
    kubectl get pods

**Expect**

- Going to 6: three new pods start immediately; no new ReplicaSet is created.
- Going back to 3: three pods go `Terminating` and disappear.

**Why**

Scaling only changes `spec.replicas`. It does not touch the pod template, so
it is not a rollout and creates no new revision.

**Try**

- Scale to 6, then re-apply the file: `kubectl apply -f deploy.yml`.
  It goes back to 3, because the file says 3. 
  Live changes are overwritten by the file (this is "drift").
- Scale to 0 and back: `kubectl scale deploy web-deploy --replicas=0`.
  The Deployment still exists; it just has no pods.

**Make sure you finish at 3 replicas.**

---

## Step 6: Service

**Do**

    kubectl apply -f svc.yml
    kubectl get svc web-svc
    kubectl get endpointslices
    curl localhost:30001

**Expect**

- `TYPE NodePort`, `PORT(S) 80:30001/TCP`, `EXTERNAL-IP <none>` (normal for NodePort).
- The EndpointSlice lists one `ip:80` per ready pod (3 addresses).
- `curl` prints the nginx welcome HTML. 
  You can open `http://localhost:30001` in a browser too.

**Why**

The Service finds pods by label (`app: web`) within its own namespace, so
there is no link between `svc.yml` and `deploy.yml` other than that label and
the shared namespace. EndpointSlices are the live list of pods currently
receiving traffic. Not-ready pods are removed.

**Try**

- In one terminal run `kubectl get endpointslices -w`, 
  then in another scale the Deployment up and down. 
  Watch the address list change. (Finish back at 3.)
- Reach the Service by DNS name **from another namespace**:

      kubectl run tmp -n default --rm -it --image=busybox --restart=Never -- wget -qO- web-svc.lab

  It returns the nginx page: namespaces do not isolate the network. 
  Now try `wget -qO- web-svc` from the same pod: it fails, because 
  the short name only resolves inside the Service's own namespace (`lab`).

**If curl fails:** see Troubleshooting at the end of this file.

---

## End of Session 2: checkpoint

    kubectl get deploy,rs,pods,svc        # 3/3 deployment, 1 ReplicaSet, 3 pods, web-svc
    curl -s -o /dev/null -w "%{http_code}\n" localhost:30001     # 200

Session 2 is done. Session 3 (`../03-update-rollback-cleanup/`) can continue from
this state, or start from scratch: it has its own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `curl localhost:30001` refused or hangs | Are you on lab-a (`current-context`)? Does `kubectl get svc` show `80:30001`? Was the cluster created with `cluster.yml`? |
| Service has no endpoints | `kubectl get endpointslices`; compare the `svc.yml` selector with `kubectl get pods --show-labels` |
| `No resources found` after an apply | Wrong namespace: `kubectl get pods -n lab` or `-A` |
| `namespaces "lab" not found` | `kubectl apply -f namespace.yml` first |
| Replica count "reverted" to 3 | You re-applied `deploy.yml`; the file says 3 |
| Pod `Pending` / `ImagePullBackOff` | `kubectl describe pod <name>`, read Events |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] explain Deployment > ReplicaSet > Pod, and what self-healing means
- [ ] scale with the command and with the file, and explain the difference
- [ ] explain how a Service finds pods (labels, same namespace)
- [ ] reach the app from your machine, and from a pod in another namespace
- [ ] read EndpointSlices to see which pods receive traffic

## Stretch ideas

- Add `resources.requests/limits` to `deploy.yml` and see what `describe pod` shows.
- Add a `readinessProbe` (HTTP GET `/` on port 80) and watch the Service endpoints.
- Create a ClusterIP Service and reach it with `kubectl port-forward svc/<name> 8081:80`.
- Deploy the same app in a second namespace `dev` with another NodePort.
