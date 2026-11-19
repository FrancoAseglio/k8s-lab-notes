# Services I - selectors, endpoints and ClusterIP 

**Work from this directory:** `cd 06-services-clusterip`

**Goal:** understand what a Service really is: a selector, a stable virtual IP
and a live list of *ready* pods. See how traffic is spread, how labels and
readiness decide who receives it, and how to read the evidence.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing deployed.
**Ending state:** `whoami` Deployment (3 ready pods) and the `whoami-svc`
ClusterIP Service running in `lab`.

**The app:** `whoami` is a tiny web server that answers every request with a
page about itself (its pod name, IPs, who called, and the request headers).
It lets you *see* which pod answered. Read the top of `deploy.yml` and
`svc-clusterip.yml` before you start.

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
    kubectl get all                       # nothing in namespace lab

All four match: go straight to Step 11.

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

---

## Step 11: Create the app and a ClusterIP Service

**Do**

    kubectl apply -f deploy.yml
    kubectl rollout status deploy/whoami
    kubectl apply -f svc-clusterip.yml
    kubectl get svc whoami-svc
    kubectl describe svc whoami-svc
    kubectl get pods -o wide
    kubectl get endpointslices -l kubernetes.io/service-name=whoami-svc
    kubectl get endpoints whoami-svc          # the legacy object: prints a deprecation warning

**Expect**

- `get svc`: `TYPE ClusterIP`, a `10.96.x.x` address, `EXTERNAL-IP <none>`, `PORT(S) 80/TCP`.
- `describe svc`: `Selector: app=whoami`, `Port: 80/TCP`, `TargetPort: 80/TCP`
  and an `Endpoints:` line with 3 addresses `10.244.x.x:80`.
- Those 3 addresses are exactly the pod IPs shown by `get pods -o wide`.
- The EndpointSlice lists the same 3 addresses. `kubectl get endpoints` shows them
  too but warns that the v1 Endpoints API is deprecated: EndpointSlices replaced it.

**Why**

A Service is a *selector* plus a *stable address*. It does not contain pods; it
points at them by label. A controller watches pods and keeps an EndpointSlice up
to date: the live list of **ready** pod IPs matching the selector. kube-proxy,
on every node, uses that list to forward traffic sent to the ClusterIP.
The ClusterIP itself is virtual: no network interface owns it, so you cannot ping it.

**Try**

    kubectl get svc whoami-svc -o yaml                              # no pod list in there
    kubectl get endpointslices -l kubernetes.io/service-name=whoami-svc -o yaml

In the EndpointSlice look at `endpoints[].conditions.ready`, `nodeName` and `targetRef`.

---

## Step 12: Reach it from inside, and from your machine

**Do: from inside the cluster**

    kubectl apply -f client-pod.yml
    kubectl wait --for=condition=Ready pod/client --timeout=90s
    kubectl exec client -- curl -s whoami-svc
    kubectl exec client -- sh -c 'for i in 1 2 3 4 5 6 7 8 9 10; do curl -s whoami-svc | grep Hostname; done'
    kubectl exec client -- sh -c 'curl -s whoami-svc whoami-svc whoami-svc | grep Hostname'
    kubectl get pod client -o wide            # compare its IP with RemoteAddr in the page

**Do: from your machine**

    kubectl get svc whoami-svc -o jsonpath='{.spec.clusterIP}{"\n"}'
    curl --max-time 3 http://<the-cluster-ip>           # from your own terminal doesn't reach svc
    kubectl port-forward svc/whoami-svc 8080:80         # leave running, use a second terminal:
    for i in 1 2 3 4 5; do curl -s localhost:8080 | grep Hostname; done
    # Ctrl+C the port-forward when done

**Expect**

- The page lists `Hostname: whoami-<hash>-<id>` (the answering pod) and
  `RemoteAddr:` = the **client pod's IP**: inside the cluster the source address
  is preserved.
- In the 10-request loop all 3 pods answer, in no fixed order. A new connection
  is balanced independently each time (it is *not* strict round robin).
- The third command sends 3 requests over **one** connection (curl reuses it):
  the same pod answers all 3. Balancing happens per connection, not per request.
- `curl` to the ClusterIP from your machine times out: that address exists only
  inside the cluster.
- Through `port-forward` it works, but **always the same pod** answers: a
  port-forward to a Service picks one pod at startup and sticks to it.

**Why**

ClusterIP Services are for pod-to-pod traffic. `port-forward` is a debugging
tunnel that bypasses the Service's load balancing: don't use it to judge how a
Service balances traffic.

**Try**

    kubectl exec client -- cat /etc/resolv.conf      # preview of lab 4: how the name resolves
    kubectl exec client -- curl -s whoami-svc.lab.svc.cluster.local | head -3

---

## Step 13: Selectors and labels decide everything

**Part A: quarantine one pod**

    kubectl get pods --show-labels
    kubectl get pods -o name | head -1               # pick a pod name
    kubectl label pod <pod-name> app=quarantined --overwrite
    kubectl get pods --show-labels
    kubectl get endpointslices -l kubernetes.io/service-name=whoami-svc
    kubectl logs <pod-name>                          # still running and still readable
    kubectl delete pod <pod-name>                    # tidy up

**Expect**

- You briefly see **4** pods: the relabelled one no longer matches the Deployment's
  selector, so the ReplicaSet lets it go and creates a replacement.
- The Service's endpoints contain 3 addresses and **not** the quarantined pod.
- The quarantined pod keeps running, receives no Service traffic, and you can
  still inspect it. A handy trick for debugging a misbehaving pod without
  killing the evidence.

**Part B: break the Service selector**

    kubectl patch svc whoami-svc -p '{"spec":{"selector":{"app":"whoami-typo"}}}'
    kubectl describe svc whoami-svc | grep -i endpoints
    kubectl get endpointslices -l kubernetes.io/service-name=whoami-svc
    kubectl exec client -- curl -s --max-time 3 -o /dev/null -w "%{http_code}\n" whoami-svc; echo "curl exit: $?"
    kubectl apply -f svc-clusterip.yml                # restore
    kubectl describe svc whoami-svc | grep -i endpoints

**Expect**

- `Endpoints: <none>`; the EndpointSlice has no addresses.
- The request **fails**, typically at once with "connection refused" (code `000`,
  curl exit 7): the Service exists and its name resolves, but there is nothing to
  send traffic to. After the restore the 3 endpoints are back.

**Why**

"Service exists, DNS works, requests fail" very often means **no endpoints**:
either the selector doesn't match the pod labels, or the pods aren't ready.
`kubectl get endpointslices` is the first thing to check.

---

## Step 14: Readiness decides who receives traffic

**Do**

    POD=$(kubectl get pods -l app=whoami -o jsonpath='{.items[0].metadata.name}')
    IP=$(kubectl get pod $POD -o jsonpath='{.status.podIP}')
    echo $POD $IP
    kubectl exec client -- curl -s -X POST -d 500 http://$IP/health     # make this pod's /health fail
    kubectl get pods -w                                                  # Ctrl+C once it shows 0/1
    kubectl describe endpointslices -l kubernetes.io/service-name=whoami-svc
    kubectl exec client -- sh -c 'for i in $(seq 1 12); do curl -s whoami-svc | grep Hostname; done'
    kubectl exec client -- curl -s -X POST -d 200 http://$IP/health     # heal it
    kubectl get pods
    kubectl exec client -- sh -c 'for i in $(seq 1 12); do curl -s whoami-svc | grep Hostname; done'

**Expect**

- Within a few seconds the pod shows `READY 0/1` but stays `Running` with
  `RESTARTS 0`: a failing readiness probe never restarts a container.
- In the EndpointSlice that pod is listed with `Ready: false`, and
  `kubectl describe svc` shows only 2 addresses.
- During the first loop, `$POD` **never** answers; only the other two do.
- After you heal it, it goes back to `1/1` and appears in the second loop.

**Why**

Only *ready* pods are used by a Service. This is what makes rolling updates
(lab 1, Session 3) safe: new pods get traffic only once their probe passes, and
a pod that stops being healthy is taken out of rotation automatically.

**Try**

- `kubectl scale deploy whoami --replicas=0`, then check the endpoints and `curl`.
  Scale back to 3 before moving on.
- Read about `publishNotReadyAddresses` in `kubectl explain service.spec`.

---

## End of Session 1: checkpoint

    kubectl get deploy,pods,svc           # whoami 3/3, 3 pods 1/1, whoami-svc
    kubectl get endpointslices -l kubernetes.io/service-name=whoami-svc    # 3 addresses
    kubectl delete pod client             # tidy up

Session 1 is done. Session 2 (`../07-services-nodeport-lb/`) can continue from
here, or start from scratch: it has its own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Service exists but requests fail | `kubectl get endpointslices -l kubernetes.io/service-name=<svc>`: empty means selector mismatch or pods not ready |
| Pods `Running` but `0/1` | Readiness probe failing: `kubectl describe pod <name>`, read Events. Did you leave a pod broken in Step 14? POST `-d 200` to it |
| `curl` in the client pod: "Could not resolve host" | Wrong name or namespace (see lab 4). From the `lab` namespace use plain `whoami-svc` |
| `kubectl exec` says the pod isn't found | `kubectl get pods`: is the `client` pod running? Is your namespace `lab`? |
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

- [ ] say what a Service is made of (selector, stable IP/name, port mapping)
- [ ] read a Service's endpoints and match them to pod IPs
- [ ] explain why a ClusterIP can't be reached from your laptop
- [ ] explain that balancing is per connection, and why `port-forward` hits one pod
- [ ] make a pod stop receiving traffic (relabel it, or fail its readiness)
- [ ] explain "no endpoints" and the two usual causes

## Stretch ideas

- Add a second port to `svc-clusterip.yml` (`name:` is then required on each port).
- Set `sessionAffinity: ClientIP` on the Service and repeat the loop in Step 12.
- Use `kubectl get pods -L app` to display the `app` label as a column.
- Create a Service with no selector and write its EndpointSlice by hand (advanced): `kubectl explain endpointslice`.
