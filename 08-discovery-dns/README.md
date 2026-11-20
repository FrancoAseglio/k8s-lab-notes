# Service discovery I - registration, DNS and kube-proxy 

**Work from this directory:** `cd 08-discovery-dns` 

**Goal:** follow a Service from creation to a working request: how it gets
"registered" in DNS, what a pod's DNS setup looks like, why the search path
exists, how the old environment-variable mechanism works, and how kube-proxy
turns a ClusterIP into a pod IP.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing deployed.
**Ending state:** the `whoami` app, two Services and two helper pods in `lab`.
CoreDNS logging switched back off.

Read the top of `svc-headless.yml` and `dnsutils-pod.yml` before you start.

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
    kubectl -n kube-system get deploy coredns      # READY 2/2 (note the number)

All match: go straight to Step 24.

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

## Step 24: Registration - what happens when a Service is created

**Do: look at the DNS service first**

    kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
    kubectl -n kube-system get svc kube-dns
    kubectl -n kube-system get endpointslices -l kubernetes.io/service-name=kube-dns
    kubectl -n kube-system get cm coredns -o yaml

**Do: create the app and a client, then watch a name appear**

    kubectl apply -f deploy.yml -f dnsutils-pod.yml
    kubectl rollout status deploy/whoami
    kubectl wait --for=condition=Ready pod/dnsutils --timeout=120s
    kubectl exec dnsutils -- nslookup whoami-svc          # the Service doesn't exist yet
    kubectl apply -f svc-clusterip.yml
    kubectl exec dnsutils -- nslookup whoami-svc
    kubectl get svc whoami-svc

**Expect**

- Two CoreDNS pods (label `k8s-app=kube-dns`; the label is `kube-dns` for
  historic reasons). A Service named `kube-dns` in `kube-system` with a fixed
  ClusterIP (`10.96.0.10` on a default kind cluster) and ports 53/UDP, 53/TCP, 9153/TCP.
- The `coredns` ConfigMap holds the `Corefile`: a `kubernetes cluster.local ...`
  block (the plugin that answers Service names from the API), `forward . /etc/resolv.conf`
  (everything else goes upstream) and `cache 30`.
- Before the Service exists, `nslookup whoami-svc` **fails** ("can't resolve" / NXDOMAIN).
  Right after `apply`, it answers with `whoami-svc.lab.svc.cluster.local` and the
  **same ClusterIP** that `kubectl get svc` shows.

**Why**

Nobody "registers" anything. CoreDNS's `kubernetes` plugin **watches the API server**
for Services and EndpointSlices and builds its answers from them. Creating a
Service is the registration; deleting it is the deregistration. (The DNS name
format is `<service>.<namespace>.svc.cluster.local`.)

**Try**

    kubectl exec dnsutils -- nslookup kubernetes.default      # the API server's own Service
    kubectl get svc kubernetes -n default

---

## Step 25: Headless Service - DNS returns the pods

**Do**

    kubectl apply -f svc-headless.yml
    kubectl get svc whoami-svc whoami-headless
    kubectl exec dnsutils -- nslookup whoami-svc
    kubectl exec dnsutils -- nslookup whoami-headless
    kubectl get pods -o wide

**Expect**

- `whoami-headless` has `CLUSTER-IP None`.
- `nslookup whoami-svc` returns **one** address: the ClusterIP.
  `nslookup whoami-headless` returns **three**: exactly the pod IPs of
  `get pods -o wide`.

**Why**

A normal Service hides the pods behind one virtual IP and kube-proxy picks a
pod. A headless Service has no virtual IP: DNS returns the pods themselves and
the *client* chooses. That is what StatefulSets and databases use to reach
specific pods, and how client-side load balancing works.

**Try**

    kubectl scale deploy whoami --replicas=5
    kubectl exec dnsutils -- nslookup whoami-headless          # 5 addresses (after a few seconds)
    kubectl scale deploy whoami --replicas=3

---

## Step 26: How a pod resolves a name (search path and ndots)

**Do**

    kubectl exec dnsutils -- cat /etc/resolv.conf
    kubectl exec dnsutils -- nslookup whoami-svc
    kubectl exec dnsutils -- nslookup whoami-svc.lab
    kubectl exec dnsutils -- nslookup whoami-svc.lab.svc.cluster.local

**Expect**

- `resolv.conf` has `nameserver 10.96.0.10` (the `kube-dns` ClusterIP, which
  kubelet injects into every pod), a `search` line starting with
  `lab.svc.cluster.local svc.cluster.local cluster.local`, and `options ndots:5`.
- All three names return the same ClusterIP.

**Do: watch the queries (CoreDNS logging)**

Turn query logging on: add a line containing only `log` below the `errors` line
of the Corefile, save, and restart CoreDNS.

    kubectl -n kube-system edit cm coredns
    kubectl -n kube-system rollout restart deploy/coredns
    kubectl -n kube-system rollout status deploy/coredns

Second terminal (leave it running):

    kubectl -n kube-system logs -l k8s-app=kube-dns --tail=0 -f

First terminal:

    kubectl exec dnsutils -- nslookup whoami-svc.lab

**Expect (in the log)**

For one lookup, several queries. First the name with the search suffixes appended
(`whoami-svc.lab.lab.svc.cluster.local.`, answered `NXDOMAIN`), then the one that
matches (`whoami-svc.lab.svc.cluster.local.`, answered `NOERROR`). The exact
lines and the A/AAAA pairs depend on the tool.

**Turn logging off again** (edit the ConfigMap, delete the `log` line, then):

    kubectl -n kube-system rollout restart deploy/coredns
    kubectl -n kube-system rollout status deploy/coredns

**Why**

A name with fewer dots than `ndots` (5) is tried **with each search suffix first**,
then as written. That is why the short name `whoami-svc` works inside the
namespace, and `whoami-svc.lab` works from anywhere. The price: every lookup of
an external name such as `kubernetes.io` (1 dot) first generates failing queries
against the cluster domains.

**Try**

With CoreDNS logging on (and internet access):

    kubectl exec dnsutils -- nslookup kubernetes.io            # watch the failed search attempts
    kubectl exec dnsutils -- nslookup kubernetes.io.           # the trailing dot skips the search path

---

## Step 27: Two more mechanisms - environment variables and kube-proxy

**Do: environment variables (the original discovery mechanism)**

    kubectl exec dnsutils -- env | grep -i whoami || echo "no whoami variables"
    kubectl apply -f client-pod.yml
    kubectl wait --for=condition=Ready pod/client --timeout=90s
    kubectl exec client -- env | grep WHOAMI_SVC

**Expect**

- The `dnsutils` pod was created **before** the Service, so it has no `WHOAMI_SVC_*` variables.
- The `client` pod was created **after** it and has several, such as
  `WHOAMI_SVC_SERVICE_HOST` (the ClusterIP) and `WHOAMI_SVC_SERVICE_PORT` (80).
  (Service name upper-cased, dashes become underscores.)

**Why**

Kubelet injects variables only for Services that exist **when the pod starts**,
and only from the pod's own namespace. Create the pod first and it never sees the
Service. DNS has neither limitation, which is why DNS is the mechanism to use.

**Do: how a ClusterIP becomes a pod (kube-proxy)**

    kubectl -n kube-system get cm kube-proxy -o yaml | grep -n 'mode:'
    docker exec lab-a-worker iptables-save | grep 'lab/whoami-svc'
    kubectl get endpointslices -l kubernetes.io/service-name=whoami-svc
    kubectl get pods -o wide

If the mode shown is `nftables`, use this instead of `iptables-save`:

    docker exec lab-a-worker nft list ruleset | grep -B2 -A6 'whoami-svc'

**Expect**

- The mode is usually empty or `iptables` on kind.
- The grep shows rules that mention `lab/whoami-svc`: one matching the
  ClusterIP and port 80 in the service chain; then, in a per-service chain, three
  rules that pick an endpoint randomly (`-m statistic --mode random --probability ...`
  with 1/3, then 1/2, then the rest); and per endpoint a `DNAT` to a pod IP and port 80.
- The three DNAT addresses are the pod IPs from `get pods -o wide` and the EndpointSlice.
  Chain names and rule layout vary with the kube-proxy version.

**Why**

This closes the loop of the whole chapter. **DNS** turns the name into a ClusterIP.
**kube-proxy** (on every node) programs rules from the Service and its EndpointSlices so
that a packet sent to the ClusterIP gets a pod IP as its destination, chosen at random
once per connection. Nothing listens on the ClusterIP. When endpoints change, kube-proxy
rewrites the rules.

**Try**

    kubectl scale deploy whoami --replicas=5
    docker exec lab-a-worker iptables-save | grep 'lab/whoami-svc'      # 5 endpoint rules now
    kubectl scale deploy whoami --replicas=3

---

## End of Session 1: checkpoint

    kubectl get deploy,svc,pods                                   # whoami 3/3, whoami-svc, whoami-headless, dnsutils, client
    kubectl -n kube-system get deploy coredns                     # READY 2/2, restarted with logging off
    kubectl -n kube-system get cm coredns -o yaml | grep -w log || echo "logging is off"

Session 1 is done. Session 2 (`../09-discovery-namespaces-debug/`) can continue
from here or start from scratch: it has its own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `nslookup` can't resolve a Service that exists | `kubectl get svc`: right name and namespace? From another namespace use `<svc>.<ns>` |
| `nslookup` hangs or times out on everything | CoreDNS down? `kubectl -n kube-system get pods -l k8s-app=kube-dns`; see Session 2 |
| The `log` line makes CoreDNS crash | Indentation (it must align with `errors`): `kubectl -n kube-system logs -l k8s-app=kube-dns`. Remove the line, restart |
| No `log` output | CoreDNS reloads its config after a delay; `rollout restart` forces it. Query again |
| `iptables-save: command not found` / no output | The node may use nftables: use the `nft` command in Step 27 |
| `docker exec lab-a-worker`: no such container | Node names are `lab-a-control-plane`, `lab-a-worker`, `lab-a-worker2`: check `docker ps` |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] explain how a Service gets a DNS name (CoreDNS watches the API)
- [ ] give the full DNS name of a Service
- [ ] explain a headless Service and when you'd use one
- [ ] read a pod's `resolv.conf` and explain `search` and `ndots:5`
- [ ] explain why environment-variable discovery depends on creation order
- [ ] describe what kube-proxy does with a ClusterIP

## Stretch ideas

- Edit the Corefile to add a custom `hosts` block and resolve a made-up name.
- Compare `kubectl exec client -- nslookup` and `curl` for the same name: which resolver does each use?
- Set `dnsPolicy`/`dnsConfig` (e.g. `ndots: 1`) on a pod and see how its `resolv.conf` and queries change.
- Read `kubectl explain pod.spec.dnsPolicy`.
