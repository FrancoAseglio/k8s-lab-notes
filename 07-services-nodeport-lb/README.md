# Services II - NodePort and LoadBalancer 

**Work from this directory:** `cd 07-services-nodeport-lb`

**Goal:** understand how a Service is exposed *outside* the cluster: NodePort,
what `externalTrafficPolicy` changes, and why LoadBalancer Services stay
`<pending>` on kind.

**Starting state:** kind cluster `lab-a` (3 nodes, port 30001 mapped), kubectl
context `kind-lab-a` with default namespace `lab`, and the `whoami` Deployment
(3 ready pods) running. Session 1 leaves you here; the bootstrap gets you here
from nothing.
**Ending state:** the extra Services deleted; `whoami` still running.

Read the top of `svc-nodeport.yml` and `svc-lb.yml` before you start.

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

Docker must be running and port **30001** must be free on your machine.

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get deploy whoami             # 3/3
    kubectl get svc                       # no Service using node port 30001

All match: go straight to Step 15.

### Bootstrap (if anything above is missing)

No `lab-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name lab-a --config cluster.yml
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Cluster exists but the namespace default is wrong or missing:

    kubectl config use-context kind-lab-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Leftovers in `lab` from other sessions or experiments (for example `web-svc`
from the first lab, which also uses port 30001):

    kubectl delete ns lab
    kubectl apply -f namespace.yml

Completely broken cluster: `kind delete cluster --name lab-a`, then the first block above.

After the bootstrap, deploy the app:

    kubectl apply -f deploy.yml
    kubectl rollout status deploy/whoami

---

## Step 15: NodePort

**Do**

    kubectl apply -f svc-nodeport.yml
    kubectl get svc whoami-nodeport
    kubectl describe svc whoami-nodeport
    for i in 1 2 3 4 5 6; do curl -s localhost:30001 | grep Hostname; done
    kubectl get nodes -o wide                                  # note each node's INTERNAL-IP
    docker exec lab-a-worker curl -s http://<INTERNAL-IP-of-lab-a-worker2>:30001 | grep Hostname

**Expect**

- `TYPE NodePort`, `PORT(S) 80:30001/TCP`, plus a `CLUSTER-IP`: a NodePort Service
  **is** a ClusterIP Service with an extra port opened on every node.
- `curl localhost:30001` answers from different pods across the loop. The
  request enters through the control-plane container (the port mapping in
  `cluster.yml`) and kube-proxy forwards it to pods on the **worker** nodes.
- The `docker exec` call, run *inside* one node and aimed at another node's IP,
  works too: every node listens on 30001, whether or not it runs a pod.

**Why**

NodePort is the simplest way in from outside: pick a node, use its IP and the
port. Downsides: you must know node IPs, the port range is 30000-32767, and
nothing spreads clients across nodes for you. That is the job of a load balancer.

**Try**

Let Kubernetes choose the port:

    kubectl expose deploy whoami --name=whoami-auto --type=NodePort --port=80
    kubectl get svc whoami-auto                                # e.g. 80:31234/TCP
    docker exec lab-a-worker curl -s localhost:<that-port> | grep Hostname
    curl --max-time 3 localhost:<that-port>                    # from your machine: fails. Why?

(The port exists on the nodes, but only 30001 is mapped to your machine.)

---

## Step 16: externalTrafficPolicy

**Do**

    kubectl get pods -o wide                                   # which nodes run pods? (not the control-plane)
    curl -s localhost:30001 | grep -E 'Hostname|RemoteAddr'
    kubectl patch svc whoami-nodeport -p '{"spec":{"externalTrafficPolicy":"Local"}}'
    kubectl get svc whoami-nodeport -o jsonpath='{.spec.externalTrafficPolicy}{"\n"}'
    curl -s --max-time 3 localhost:30001; echo "curl exit: $?"
    docker exec lab-a-worker curl -s localhost:30001 | grep Hostname
    docker exec lab-a-worker2 curl -s localhost:30001 | grep Hostname
    kubectl patch svc whoami-nodeport -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'    # restore
    curl -s localhost:30001 | grep Hostname

**Expect**

- Pods run on the two workers only; the control-plane node (the one your
  `localhost:30001` enters through) has none.
- With the default policy `Cluster`, any node forwards to any pod. The pod sees
  a **node address** as the client (`RemoteAddr`), because the traffic was
  source-NATed on the way.
- With `Local`, a node only serves pods **running on that node**. Through the
  control-plane the request fails (a timeout, a reset or an empty reply: packets
  are dropped when there is no local pod).
- Run on a worker, `Local` works, and only pods *on that worker* answer.
  Compare the hostnames with `kubectl get pods -o wide`.
- After restoring `Cluster`, everything works again.

**Why**

`Cluster` (default): always works, but an extra hop and the client's IP is hidden.
`Local`: no extra hop and the real client IP is kept, but you must only send
traffic to nodes that have pods (cloud load balancers do this with health checks).

Note the restore used `patch`, not `apply`: `kubectl apply` ignores fields that
are not in your file, so re-applying `svc-nodeport.yml` would not undo the change.

---

## Step 17: LoadBalancer

**Do**

    kubectl apply -f svc-lb.yml
    kubectl get svc whoami-lb -w                               # watch ~10 s, then Ctrl+C
    kubectl describe svc whoami-lb
    kubectl get svc whoami-lb -o jsonpath='{.spec.ports[0].nodePort}{"\n"}'
    docker exec lab-a-worker curl -s localhost:<that-node-port> | grep Hostname

**Expect**

- `EXTERNAL-IP` stays `<pending>`, however long you wait. The Service is also
  given a `CLUSTER-IP` and a node port (`8080:3xxxx/TCP`).
- Through the node port, it works exactly like a NodePort Service.

**Why**

`LoadBalancer` = NodePort + "cloud, please give me an external load balancer".
On AWS/GCP/Azure a cloud controller does that and fills in the address. kind
has no cloud controller, so nothing ever answers the request.

**Try A (Docker Desktop's own Kubernetes)**

Docker Desktop's built-in cluster *does* provide LoadBalancers: EXTERNAL-IP is
`localhost`. Skip this if Kubernetes isn't enabled in Docker Desktop. Note the
explicit `--context`, so you never switch away from `kind-lab-a`:

    kubectl --context docker-desktop get nodes
    kubectl --context docker-desktop apply -f namespace.yml -f deploy.yml -f svc-lb.yml
    kubectl --context docker-desktop -n lab get svc whoami-lb
    curl -s localhost:8080 | grep Hostname
    kubectl --context docker-desktop delete ns lab              # clean up afterwards

(This is a different cluster, so check the context before every command.)

**Try B (a load balancer for kind)**

The `cloud-provider-kind` tool plays the cloud controller for kind clusters.
Install it from its project page, run it in a **second terminal** and keep it
running (on macOS it must run with `sudo`), then watch `kubectl get svc whoami-lb -w`:
the EXTERNAL-IP should change from `<pending>` to an address. How you reach that
address from your machine differs by OS; follow the project's README. Stop it with Ctrl+C.

---

## Cleanup for this session

    kubectl delete -f svc-nodeport.yml -f svc-lb.yml
    kubectl delete svc whoami-auto --ignore-not-found
    kubectl get svc                                    # nothing left

`whoami` keeps running. If you are finished with chapter 7 and want the machine
back: `kind delete cluster --name lab-a`.

---

## End of Session 2: checkpoint

    kubectl get deploy whoami             # 3/3
    kubectl get svc                       # No resources found in lab namespace.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `curl localhost:30001` refused or hangs | Are you on lab-a? Does `kubectl get svc` show `80:30001`? Was the cluster created with `cluster.yml` (mapping 30001)? Did you leave `externalTrafficPolicy: Local` on? Restore with the patch in Step 16 |
| `nodePort: provided port is already allocated` | Another Service uses 30001: `kubectl get svc -A`. A leftover `web-svc` from the first lab? Delete it |
| `docker exec ... curl: not found` | Run the command from your machine through `curl localhost:30001`, or use another node |
| EXTERNAL-IP `<pending>` | Expected on kind (see Step 17) |
| Docker Desktop Try A: `context "docker-desktop" not found` | Enable Kubernetes in Docker Desktop settings, or skip it |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] explain how NodePort relates to ClusterIP, and why any node works
- [ ] trace a request from `localhost:30001` to a pod
- [ ] explain `externalTrafficPolicy` Cluster vs Local and the trade-off
- [ ] explain why a LoadBalancer is `<pending>` on kind, and what fills it on a cloud
- [ ] tell `patch` from `apply` when reverting a live change

## Stretch ideas

- Create a Service of `type: ExternalName` pointing at `kubernetes.io` and look at `nslookup` from a pod (lab 4 has the tools).
- Try `nodePort: 80` and read the error: why is it rejected?
- Run `kubectl get svc whoami-nodeport -o yaml` and find where `externalTrafficPolicy` and `status` live.
