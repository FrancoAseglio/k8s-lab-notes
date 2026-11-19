# Ingress I - cluster and controller 

**Work from this directory:** `cd 04-ingress-setup`  

**Goal:** build a cluster that can receive web traffic on ports 80/443, install
an Ingress controller, and route a first request through it. Understand the
parts: Ingress (rules) + IngressClass + controller + the Service that exposes it.

**Starting state:** a kind cluster named `ingress-a` doesn't exist yet. (This
session needs nothing from the other sessions. It uses its own cluster, so
`lab-a` can stay or go.)  
**Ending state:** cluster `ingress-a`, Traefik running in namespace `traefik`,
the `whoami` app in `lab`, and one Ingress sending every request to it.

Read the top of `cluster.yml`, `traefik-values.yml` and `ingress.yml` before you start.

> **Why Traefik?** Most tutorials install *ingress-nginx*. The Kubernetes
> project **retired it in March 2026**: no more releases or security fixes.
> Traefik is maintained and handles standard Ingress objects. (Gateway API is
> the Ingress successor; see the stretch ideas.)

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
    helm version

Docker must be running; ports **80** and **443** must be free on your machine.
Helm is needed to install the Ingress controller (macOS: `brew install helm`).

---

## Step 18: A cluster with ports 80 and 443

**Do**

    kind create cluster --name ingress-a --config cluster.yml
    kubectl config current-context           # kind-ingress-a
    kubectl get nodes
    docker port ingress-a-control-plane

**Expect**

- 3 nodes: `ingress-a-control-plane`, `ingress-a-worker`, `ingress-a-worker2`.
- `docker port` lists mappings such as `30080/tcp -> 0.0.0.0:80` and
  `30443/tcp -> 0.0.0.0:443` (plus the API server port).
- kind switched your context to `kind-ingress-a`.

**Why**

The path of every request in this lab is:
`curl localhost:80` -> node container port 30080 -> the controller's Service
(a NodePort) -> the controller pod -> your app. The mapping makes the first
hop possible; Step 19 makes the rest.

**Try**

If `lab-a` still exists, `kubectl config get-contexts` shows both clusters.
Both can run at once (different ports), but each uses memory: delete `lab-a`
if the machine is struggling.

---

## Step 19: Install the Ingress controller

**Do**

    helm repo add traefik https://helm.traefik.io/traefik
    helm repo update
    helm upgrade --install traefik traefik/traefik --namespace traefik --create-namespace -f traefik-values.yml
    kubectl -n traefik rollout status deploy/traefik
    kubectl -n traefik get all
    kubectl get ingressclass
    kubectl describe ingressclass traefik
    helm list -n traefik
    curl -si localhost | head -3

**Expect**

- One Deployment `traefik` (1/1) and one Service `traefik` of `TYPE NodePort`
  with `80:30080/TCP,443:30443/TCP`: the node ports from `traefik-values.yml`.
- An IngressClass named `traefik` whose controller is `traefik.io/ingress-controller`.
  `describe` shows in its annotations whether it is the default class.
- `curl -si localhost` returns **`HTTP/1.1 404 Not Found`** (body `404 page not found`).
  That is success: the request travelled host:80 -> node -> Traefik, and
  Traefik answered because **no route exists yet**.
- `helm list` shows the chart and Traefik versions that were installed.
  The chart may also have installed Traefik's own CRDs (`kubectl get crd | grep traefik`);
  we won't use them.

**Why**

An Ingress controller is just a reverse proxy running in pods that watches the
API server for Ingress objects. Kubernetes defines the Ingress *API* but ships
no controller: you choose and install one. The IngressClass says which
controller owns which Ingress objects (so several can coexist). The controller
itself is reached through a normal Service: a NodePort here, a LoadBalancer on a cloud.

**Try**

Look at the controller's dashboard, which will show routers once you have Ingresses
(use the command printed at the end of the Helm install if it differs):

    kubectl -n traefik port-forward deploy/traefik 9000:8080
    # open http://localhost:9000/dashboard/   (Ctrl+C to stop)

---

## Step 20: The first Ingress

**Do**

    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab
    kubectl apply -f app.yml
    kubectl rollout status deploy/whoami
    curl -si localhost | head -1             # still 404: the app exists, but no Ingress yet
    kubectl apply -f ingress.yml
    kubectl get ingress
    kubectl describe ingress whoami
    curl -s localhost
    curl -s localhost | grep -E 'Hostname|RemoteAddr|X-Forwarded|X-Real-Ip'
    for i in 1 2 3 4 5 6; do curl -s localhost | grep Hostname; done

**Expect**

- Before the Ingress: `404`. After: the whoami page.
- `get ingress`: `CLASS traefik`, `HOSTS *`, `PORTS 80`. `ADDRESS` is typically
  empty here (no load balancer to report).
- `describe ingress`: a rule for host `*`, path `/`, backend `whoami-svc:80`
  with the 3 pod IPs listed.
- Headers added by the controller: `X-Forwarded-For`, `X-Forwarded-Host`,
  `X-Forwarded-Port`, `X-Forwarded-Proto`, `X-Real-Ip`.
- `RemoteAddr` is now the **Traefik pod's IP**, not yours. The app sees the proxy,
  and learns the real client only from those headers.
- The `Hostname` changes between requests: the controller spreads requests
  across the pods. Controllers typically send traffic straight to the pod
  addresses from the Service's EndpointSlice, bypassing the ClusterIP.

**Why**

The Ingress object is only configuration; the controller turned it into a
route. One entry point (ports 80/443) can now front many Services, and
you can add or remove routes at any time by applying Ingress objects, with no
new load balancer and no new ports per app.

**Try**

    kubectl get ingress whoami -o yaml            # compare with ingress.yml
    kubectl delete ingress whoami                 # then curl: 404 again within seconds
    kubectl apply -f ingress.yml
    kubectl -n traefik logs deploy/traefik --tail=20

---

## End of Session 1: checkpoint

    kubectl config current-context        # kind-ingress-a
    kubectl -n traefik get deploy traefik # 1/1
    kubectl get ingress                   # whoami, CLASS traefik
    curl -s -o /dev/null -w "%{http_code}\n" localhost       # 200

Session 1 is done. Session 2 (`../05-ingress-routing/`) can continue from here
(it first removes this catch-all Ingress), or start from scratch: it has its
own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `kind create` fails: port 80 or 443 in use | Another process (or Docker Desktop's Kubernetes with a LoadBalancer on 80): `lsof -i :80`. Free it, or change `hostPort` in `cluster.yml` and add the port to your curl commands |
| `curl localhost` refused or hangs | Are you on `kind-ingress-a`? `docker port ingress-a-control-plane` shows the mappings? Traefik running (`kubectl -n traefik get pods`)? |
| `404 page not found` after applying the Ingress | `kubectl describe ingress`: right class (`ingressClassName: traefik`)? Backend Service exists with endpoints? Ingress in the same namespace as the Service? |
| `helm` can't reach the repository | No internet, or `helm repo update` not run. `helm repo list` |
| Helm error about unknown values | Chart values change between versions: `helm show values traefik/traefik`, compare with `traefik-values.yml` |
| `Error from server (NotFound): namespaces "lab" not found` | Apply `namespace.yml` first |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] say what an Ingress, an IngressClass and an Ingress controller each are
- [ ] explain why an Ingress does nothing without a controller
- [ ] trace a request from `curl localhost` to a pod, naming every hop
- [ ] explain why the app sees the proxy's IP, and where the client IP is
- [ ] explain why this lab doesn't use ingress-nginx

## Stretch ideas

- Run `helm show values traefik/traefik | less` and find where the NodePort values live.
- Pin the chart version: `helm upgrade --install ... --version <x.y.z>`.
- Look up Gateway API (`gateway.networking.k8s.io`), the successor to Ingress.
- Install a second controller with a different IngressClass and compare.
