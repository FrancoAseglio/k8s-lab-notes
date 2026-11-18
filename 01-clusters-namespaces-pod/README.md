# Session 1: Clusters, contexts, namespaces and a single Pod 

**Work from this directory:** `cd 01-clusters-namespaces-pod`  

**Goal:** create two clusters, switch between them, organise work into a
namespace, and see why a bare Pod is not enough.

**Starting state:** nothing. No kind clusters from this lab. 
(This session needs nothing from any other session.)
**Ending state:** clusters `lab-a` (3 nodes) and `lab-b` (1 node) exist; 
you are on `kind-lab-a`, whose default namespace is `lab`; 
the `lab` namespace exists and is empty.

Read `cluster.yml`, `namespace.yml` and `pod.yml` before starting.

## Golden rule

Before any command that changes something,  
know **which cluster and which namespace** you are on:

    kubectl config current-context
    kubectl config get-contexts      # NAMESPACE column; empty = default

Applying to the wrong cluster, or looking in the wrong namespace,  
are the two most common real-world mistakes.

## Prerequisites

    docker version
    kind version
    kubectl version --client

Docker must be running and port **30001** must be free on your machine.

---

## Step 1: Cluster creation and context switching

**Do**

    kind create cluster --name lab-a --config cluster.yml
    kind create cluster --name lab-b
    kind get clusters
    kubectl config get-contexts
    kubectl config use-context kind-lab-b
    kubectl get nodes
    kubectl config use-context kind-lab-a
    kubectl get nodes
    docker ps

**Expect**

- `kind get clusters` lists `lab-a` and `lab-b`.
- `get-contexts` shows `kind-lab-a` and `kind-lab-b`; `*` marks current one.
  kind switches to the newest cluster automatically, so after creating lab-b
  you are on `kind-lab-b`. The NAMESPACE column is empty (= `default`).
- On lab-b: 1 node (`lab-b-control-plane`).
  On lab-a: 3 nodes (`lab-a-control-plane`, `lab-a-worker`, `lab-a-worker2`).
- `docker ps` shows 4 containers: each node is a container.

**Why**

kubectl reads `~/.kube/config`. A *context* = cluster + user + namespace.
`use-context` only changes which cluster your commands go to;  
nothing inside either cluster changes. 

**Try**

    kubectl cluster-info --context kind-lab-b   # query without switching
    kubectl get nodes --context kind-lab-b      # one-off, per command

**Finish this step on `kind-lab-a`.**

---

## Step 2: Namespaces

**Do**

    kubectl config current-context      # should say kind-lab-a
    kubectl get ns
    kubectl get pods                    # current namespace (default): empty
    kubectl get pods -n kube-system
    kubectl get pods -A
    kubectl apply -f namespace.yml
    kubectl get ns
    kubectl describe ns lab

**Expect**

- `get ns` lists `default`, `kube-node-lease`, `kube-public`, `kube-system`
  and `local-path-storage` (kind). After the apply, `lab` appears, `Active`.
- `get pods` says `No resources found in default namespace.`
- `-n kube-system` shows Kubernetes' own pods (coredns, etcd, kube-apiserver, 
   kindnet...). These are the cluster's own components, running as pods.
- `-A` shows pods from every namespace, with a NAMESPACE column.
- `describe ns lab` shows no resource quota and no limit ranges. It is empty.

**Why**

A namespace is a named partition inside one cluster.  Without `-n`, kubectl 
uses the *current namespace of your context* (`default` unless changed).
The same name can exist in several namespaces, and deleting a namespace
deletes everything inside it.

**Try**

    kubectl api-resources --namespaced=true     # things that live in a ns
    kubectl api-resources --namespaced=false    # cluster-wide things
    kubectl get ns lab -o yaml                  # see the object as stored

---

## Step 3: A single Pod, and the namespace gotcha

**Do**

    kubectl apply -f pod.yml
    kubectl get pods                          # current ns is still default
    kubectl get pods -n lab
    kubectl get pods -A | grep web-pod

Now make `lab` your context's default namespace:

    kubectl config set-context --current --namespace=lab
    kubectl config get-contexts
    kubectl get pods -o wide
    kubectl describe pod web-pod
    kubectl logs web-pod
    kubectl exec -it web-pod -- sh            # type `exit` to leave
    kubectl delete pod web-pod
    kubectl get pods

**Expect**

- The apply prints `pod/web-pod created`, yet the first `get pods` says
  `No resources found in default namespace.`: 
  you are looking in the wrong place. `-n lab` (or `-A`) finds it.
- After `set-context`, `get-contexts` shows `lab` in the NAMESPACE column of
  `kind-lab-a`, and plain `kubectl get pods` now finds the pod.
- First `get pods`: `ContainerCreating`, then `Running`.
- `-o wide` shows the pod IP and the worker node it landed on.
- `describe` ends with an **Events** section: 
  Scheduled > Pulling > Pulled > Created > Started.  
  Read Events first whenever something is wrong.
- After the delete: `No resources found in lab namespace.`  
  The pod does not come back.

**Why**

- The manifest says where the object goes (`namespace: lab`); 
  `kubectl get` looks where your context says. 
  When they differ you get "No resources found" for something that exists. 
  Always suspect the namespace first.
- A bare Pod has no controller. Deleting it is final. 
  That is why you almost never create pods directly.

**Try**

- The namespace setting belongs to the *context*. 
  Switch to lab-b with `kubectl config use-context kind-lab-b`, 
  check `get-contexts` (its namespace is still empty = default), 
  then switch back to lab-a (still `lab`).
- Override the default for one command: `kubectl get pods -n kube-system`.

---

## End of Session 1: checkpoint

    kubectl config current-context  # kind-lab-a
    kubectl get nodes               # 3 nodes
    kubectl config get-contexts     # kind-lab-a shows NAMESPACE lab
    kubectl get ns lab              # Active
    kubectl get pods                # No resources found in lab namespace.

All five match? Session 1 is done. 
Session 2 (`../02-deploy-scale-service/`) can start from here,  
or from scratch: it has its own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `kind create` fails on port 30001 | Another process uses it: `lsof -i :30001` |
| `No resources found` right after an apply | Wrong namespace. Try `kubectl get pods -n lab` or `-A` |
| `namespaces "lab" not found` on apply | Apply `namespace.yml` before `pod.yml` |
| Pod stuck in `ContainerCreating` for long | First image pull is slow; `kubectl describe pod web-pod`, read Events |
| `ImagePullBackOff` | Typo in image name/tag, or no internet. Events show the exact error |
| `no such file` when applying | You are in the wrong directory: `cd` into this session's directory |
| Commands hit the wrong cluster | `kubectl config current-context`, then `kubectl config use-context kind-lab-a` |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] create and delete a kind cluster
- [ ] list contexts, switch between them, and say which one is current
- [ ] explain what a namespace is, and what is and is not isolated by it
- [ ] set the default namespace of a context, and use `-n` and `-A`
- [ ] explain why a deleted Pod stays gone

## Stretch ideas

- Create a namespace `dev` and apply the pod there. Compare `get pods -A`.
- Add a `ResourceQuota` to `lab` limiting the pods number, try to exceeding.
- Run `kubectl explain pod.spec.containers` and read configurable options.
