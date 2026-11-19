# Ingress II - host-based and path-based routing 

**Work from this directory:** `cd 05-ingress-routing` 

**Goal:** route requests to different apps by host name and by URL path,
see what the controller does and doesn't change, and learn to diagnose the
classic Ingress failures (wrong class, no endpoints, wrong backend).

**Starting state:** kind cluster `ingress-a` (ports 80/443 mapped) with
Traefik running in namespace `traefik`; kubectl context `kind-ingress-a` with
default namespace `lab`; no Ingress objects. Session 1 leaves you almost here
(delete its catch-all Ingress, see below); the bootstrap gets you here from nothing.
**Ending state:** everything deleted, including the cluster.

Read the top of `apps.yml`, `ingress-host.yml` and `ingress-path.yml` before you start.

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

## Starting checks

    kubectl config current-context            # kind-ingress-a
    kubectl config get-contexts               # NAMESPACE lab on kind-ingress-a
    kubectl -n traefik get deploy traefik     # 1/1
    kubectl get ingress                       # No resources found in lab namespace.
    curl -si localhost | head -1              # HTTP/1.1 404 Not Found

If you come from Session 1, `get ingress` still lists its catch-all `whoami`
Ingress: remove it with `kubectl delete ingress whoami`, otherwise it would
answer requests that the new rules don't match.

All match: go straight to Step 21.

### Bootstrap (if anything above is missing)

No `ingress-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name ingress-a --config cluster.yml
    helm repo add traefik https://helm.traefik.io/traefik
    helm repo update
    helm upgrade --install traefik traefik/traefik --namespace traefik --create-namespace -f traefik-values.yml
    kubectl -n traefik rollout status deploy/traefik
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Cluster exists but the context or namespace default is wrong:

    kubectl config use-context kind-ingress-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Traefik missing in an existing cluster: run the `helm` lines above.
Leftover Ingresses (for example the catch-all one from Session 1):

    kubectl delete ingress --all -n lab

Completely broken cluster: `kind delete cluster --name ingress-a`, then the first block above.

---

## Step 21: Host-based routing

**Do**

    kubectl apply -f apps.yml
    kubectl wait --for=condition=Available deploy/red deploy/blue --timeout=120s
    kubectl apply -f ingress-host.yml
    kubectl get ingress apps-host
    kubectl describe ingress apps-host
    curl -s -H "Host: red.lab.test" localhost | grep -E '^Name|^Hostname|^Host:'
    curl -s -H "Host: blue.lab.test" localhost | grep -E '^Name|^Hostname|^Host:'
    curl -si localhost | head -1
    curl -si -H "Host: green.lab.test" localhost | head -1

**Expect**

- `get ingress`: `HOSTS red.lab.test,blue.lab.test`, `CLASS traefik`.
- `describe` lists two rules, each with its backend Service and the pod IPs behind it.
- The red request answers `Name: red` and a `Hostname:` starting with `red-`;
  the blue one `Name: blue`. The `Host:` line the app received is the one you sent.
- No `Host` header match (plain `localhost`, or `green.lab.test`): **404**.

**Why**

The controller reads the `Host` header of each request and picks the rule.
One IP and one pair of ports serve any number of sites. The names don't need to
exist in DNS for this lab: you tell curl what to put in the header.

**Try**

Make curl believe the names are real, so the URL carries the host name:

    curl --resolve red.lab.test:80:127.0.0.1 http://red.lab.test/ | grep '^Name'

Or add `127.0.0.1 red.lab.test blue.lab.test apps.lab.test` to `/etc/hosts` and
open `http://red.lab.test` in a browser (remove the line afterwards).

---

## Step 22: Path-based routing

**Do**

    kubectl apply -f ingress-path.yml
    kubectl describe ingress apps-path
    curl -s -H "Host: apps.lab.test" localhost/red | grep -E '^Name|^GET'
    curl -s -H "Host: apps.lab.test" localhost/blue/some/deeper/path | grep -E '^Name|^GET'
    curl -si -H "Host: apps.lab.test" localhost/green | head -1
    curl -si -H "Host: apps.lab.test" localhost/ | head -1

**Expect**

- `/red` -> `Name: red`, and the app logged the request line `GET /red HTTP/1.1`.
- `/blue/some/deeper/path` -> `Name: blue`, with the **full** path in the request
  line: with `pathType: Prefix` everything below `/blue` matches.
- `/green` and `/` -> **404**: no rule matches.

**Why**

The backend receives the original path, **prefix included**. If red-svc were an
app that serves its pages at `/`, `/red` would 404 inside it. A plain Ingress
can't strip the prefix: that takes controller-specific extras (Traefik
middlewares, nginx annotations). Host-based routing avoids the problem, which is
why it is the more common choice for separate apps.

**Try**

- Which Ingress answers `curl -H "Host: red.lab.test" localhost/blue`? (Predict, then test.)
- Edit `ingress-path.yml` to `pathType: Exact` for `/red`, re-apply, and compare
  `/red` with `/red/x`.
- Test `curl -H "Host: apps.lab.test" localhost/redfish`: does a Prefix of `/red`
  match it? The spec says prefixes match whole path elements; check what your
  controller actually does.

---

## Step 23: Break it on purpose, then diagnose

**Do: 1. an Ingress for a class nobody runs**

    kubectl patch ingress apps-host --type=merge -p '{"spec":{"ingressClassName":"nginx"}}'
    kubectl get ingress apps-host
    kubectl get ingressclass
    curl -si -H "Host: red.lab.test" localhost | head -1
    kubectl patch ingress apps-host --type=merge -p '{"spec":{"ingressClassName":"traefik"}}'

**Do: 2. a backend with no ready pods**

    kubectl scale deploy red --replicas=0
    curl -si -H "Host: red.lab.test" localhost | head -1
    curl -si -H "Host: blue.lab.test" localhost | head -1
    kubectl scale deploy red --replicas=2
    kubectl rollout status deploy/red

**Do: 3. a backend Service that doesn't exist**

    kubectl patch ingress apps-host --type=json -p '[{"op":"replace","path":"/spec/rules/0/http/paths/0/backend/service/name","value":"red-svcc"}]'
    kubectl describe ingress apps-host
    curl -si -H "Host: red.lab.test" localhost | head -1
    kubectl -n traefik logs deploy/traefik --tail=10
    kubectl apply -f ingress-host.yml                  # restore

**Expect**

1. The Ingress still exists (`CLASS nginx`) but no controller claims it, so
   the request returns **404**. This is exactly what happens when manifests written
   for ingress-nginx (`ingressClassName: nginx`) are applied to a cluster that now
   runs another controller.
2. The route exists but has no backend: Traefik typically answers **503 Service Unavailable**
   (the exact code is controller-specific). Blue keeps working: the failure is
   isolated to that route. Scaling back restores it.
3. `describe ingress` shows the backend with an error such as
   `red-svcc:80 (<error: services "red-svcc" not found>)`. The request fails
   (404 or 503 depending on the controller) and the controller's logs mention the
   missing service. Re-applying the file fixes it.

**Why (the ladder)**

When a request through an Ingress fails, go down this list:

1. Does a controller own this Ingress? (`ingressClassName`, `get ingressclass`)
2. Does the rule match what you sent? (`Host` header, path, `pathType`)
3. Does the backend Service exist, with the right port? (`describe ingress` shows errors)
4. Does the Service have ready endpoints? (`get endpointslices`, as in lab 2)
5. What does the controller say? (logs, dashboard)

**Try**

Give an Ingress a `defaultBackend` (see `kubectl explain ingress.spec.defaultBackend`)
so unmatched requests go to the blue app instead of returning 404.

---

## Cleanup

    kubectl delete -f ingress-host.yml -f ingress-path.yml
    kubectl delete ns lab
    kind delete cluster --name ingress-a
    kind get clusters
    kubectl config get-contexts
    docker ps

After the cluster is gone, no context may be current. Pick one:
`kubectl config use-context kind-lab-a` (if that cluster exists) or `docker-desktop`.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Everything returns 404 | `kubectl get ingress`: right `CLASS`? Is the `Host` header what the rule says? Are the apps in the same namespace as the Ingress? |
| A rule works for `curl -H Host:` but not in a browser | The browser needs the name to resolve: `/etc/hosts` or `curl --resolve` |
| 503 | The Service has no ready endpoints: `kubectl get endpointslices`, `kubectl get pods` |
| Changes to the Ingress seem ignored | Give the controller a few seconds; `kubectl describe ingress` for errors; controller logs |
| `curl localhost` refused | Are you on `kind-ingress-a`? Traefik running? `docker port ingress-a-control-plane` |
| `Error from server (NotFound): namespaces "lab" not found` | Apply `namespace.yml` |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] write an Ingress with host rules and one with path rules
- [ ] explain that path prefixes are passed to the backend unchanged
- [ ] explain why an Ingress can exist and still be ignored (IngressClass)
- [ ] use the 5-step ladder to diagnose a 404 or 503
- [ ] explain what is controller-specific and what is standard Ingress

## Stretch ideas

- Add TLS: create a self-signed certificate, a `kubernetes.io/tls` Secret, and a `tls:` section; test with `curl -k https://...` on port 443.
- Use a Traefik `Middleware` (a custom resource) to strip the `/red` prefix.
- Compare with Gateway API: install its CRDs and write a `Gateway` and `HTTPRoute` for the same routes.
- Add `annotations` to an Ingress and see which are controller-specific.
