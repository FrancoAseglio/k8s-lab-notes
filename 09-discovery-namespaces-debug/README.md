# Service discovery II - namespaces, FQDNs and troubleshooting 

**Work from this directory:** `cd 09-discovery-namespaces-debug`

**Goal:** see how namespaces and DNS interact, learn how discovery fails in each
layer (names, Services, ports, pods), and practice a systematic troubleshooting
routine on a deliberately broken app.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing deployed. This session uses **explicit
`-n` flags** on most commands, so the default namespace matters less.
**Ending state:** everything deleted, including the cluster. CoreDNS restored.

Read the top of `app-lab.yml`, `app-dev.yml` and `tools.yml` before you start. Do
**not** read `broken.yml` before Step 31.

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
    kubectl get ns dev                    # NotFound is fine: Step 28 creates it
    kubectl -n kube-system get deploy coredns      # READY 2/2 (note the number)

All match: go straight to Step 28.

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

## Step 28: The same names in two namespaces

**Do: set up**

    kubectl apply -f namespace.yml -f namespace-dev.yml
    kubectl apply -f app-lab.yml -f app-dev.yml -f tools.yml -f dnsutils-pod.yml
    kubectl -n lab rollout status deploy/whoami
    kubectl -n dev rollout status deploy/whoami
    kubectl -n lab wait --for=condition=Ready pod/client pod/dnsutils --timeout=120s
    kubectl -n dev wait --for=condition=Ready pod/client --timeout=120s
    kubectl get deploy,svc -A | grep -E 'NAMESPACE|whoami|lab-only'

**Do: call the "same" Service from both namespaces**

    kubectl -n lab exec client -- curl -s whoami-svc | grep '^Name'
    kubectl -n dev exec client -- curl -s whoami-svc | grep '^Name'
    kubectl -n lab exec client -- curl -s whoami-svc.dev | grep '^Name'
    kubectl -n dev exec client -- curl -s whoami-svc.lab | grep '^Name'
    kubectl -n lab exec client -- curl -s whoami-svc.dev.svc.cluster.local | grep '^Name'
    kubectl -n lab exec client -- cat /etc/resolv.conf
    kubectl -n dev exec client -- cat /etc/resolv.conf
    kubectl -n lab exec dnsutils -- nslookup whoami-svc.dev.svc.cluster.local

**Expect**

- The same Deployment and Service names exist in both namespaces without conflict.
- The short name `whoami-svc` reaches **your own namespace's** copy: `Name: lab`
  from the lab client, `Name: dev` from the dev client.
- `whoami-svc.dev` from the lab client reaches `dev`; `whoami-svc.lab` from the dev
  client reaches `lab`. The fully qualified name (`...svc.cluster.local`) works from anywhere.
- The two `resolv.conf` files differ only in the first search entry
  (`lab.svc.cluster.local` vs `dev.svc.cluster.local`): that single line is what
  makes the short name namespace-relative.

**Why**

Namespaces scope *names*. DNS names embed the namespace
(`<service>.<namespace>.svc.cluster.local`), and each pod's search path starts with
its own. Namespaces are **not** a network boundary: any pod can call any Service by its
qualified name. (Blocking that takes NetworkPolicies.)

**Try**

    kubectl get endpointslices -A | grep whoami            # one slice per namespace
    kubectl -n dev exec client -- curl -s whoami-svc.lab.svc | grep '^Name'    # partial names: svc is also in the search path

---

## Step 29: When the *name* fails

**Do: a. a Service that lives in another namespace**

    kubectl -n dev exec client -- curl -s --max-time 5 lab-only-svc; echo "curl exit: $?"
    kubectl -n dev exec client -- curl -s lab-only-svc.lab | grep '^Name'

**Do: b. a typo**

    kubectl -n lab exec dnsutils -- nslookup whoami-svcc

**Do: c. DNS itself is down**

    kubectl -n kube-system get deploy coredns                 # note READY, e.g. 2/2
    kubectl -n kube-system scale deploy coredns --replicas=0
    kubectl -n lab exec client -- curl -s --max-time 5 whoami-svc; echo "curl exit: $?"
    CIP=$(kubectl -n lab get svc whoami-svc -o jsonpath='{.spec.clusterIP}')
    kubectl -n lab exec client -- curl -s --max-time 5 http://$CIP | grep '^Name'

**Restore DNS now (use the replica count you noted):**

    kubectl -n kube-system scale deploy coredns --replicas=2
    kubectl -n kube-system rollout status deploy/coredns
    kubectl -n lab exec client -- curl -s whoami-svc | grep '^Name'

**Expect**

- a. The first call fails: `curl: (6) Could not resolve host: lab-only-svc`
  (exit code 6). With `.lab` it works. The Service is fine; the *name* isn't visible
  from `dev`.
- b. `nslookup` reports it can't resolve / NXDOMAIN.
- c. With CoreDNS at 0 replicas, every name lookup fails (exit 6, quickly or
  after a delay). But `curl http://<ClusterIP>` **still works**: Services don't depend on DNS
  to function, only your ability to *find* them does. After restoring, names work again.

**Why**

Exit code **6** from curl means "could not resolve the name": the problem is in
DNS/namespace/typo territory, not in the Service or the app.

---

## Step 30: When the *Service* fails (the name resolves, the request doesn't)

**Do: a. the port the Service forwards to is wrong**

    kubectl -n lab patch svc whoami-svc --type=json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":8080}]'
    kubectl -n lab describe svc whoami-svc | grep -E 'TargetPort|Endpoints'
    kubectl -n lab exec client -- curl -s --max-time 5 -o /dev/null -w "%{http_code}\n" whoami-svc; echo "curl exit: $?"
    kubectl -n lab apply -f app-lab.yml                       # restore

**Do: b. the client uses the wrong Service port**

    kubectl -n lab exec client -- curl -s --max-time 5 -o /dev/null -w "%{http_code}\n" whoami-svc:8080; echo "curl exit: $?"

**Do: c. bypass the Service and ask a pod directly**

    POD_IP=$(kubectl -n lab get pods -l app=whoami -o jsonpath='{.items[0].status.podIP}')
    kubectl -n lab exec client -- curl -s http://$POD_IP | grep '^Name'

**Expect**

- a. `describe` shows endpoints ending in `:8080` (the Service forwards to a port
  where nothing listens). The request fails **immediately**: `connection refused`
  (curl exit 7, code `000`). After the restore, endpoints end in `:80` again.
- b. Nothing listens on Service port 8080 at all: the request **hangs until
  the timeout** (curl exit 28).
- c. The pod answers: the app is healthy, so the fault is in front of it.

**Why**

*Refused* (instant) = something answered "nothing here": you reached a host, but no
process listens on that port. *Timeout* (slow) = nobody answered: wrong address or port
on the Service, or a firewall. Combined with "name resolves" this narrows the layer.

| What you see | Most likely layer |
|---|---|
| `Could not resolve host` (exit 6) | name: typo, wrong namespace, DNS down |
| `connection refused` (exit 7), no endpoints | Service: selector mismatch, pods not ready |
| `connection refused`, endpoints exist | Service `targetPort` is not the container port |
| timeout (exit 28) | wrong Service port, or traffic blocked |
| pod answers directly but Service doesn't | everything between: Service config |

**The routine, in order**

1. Does the name resolve? (`nslookup`, or curl's exit code 6)
2. Does the Service exist, in the right namespace?
3. Does it have **endpoints**? (`kubectl get endpointslices`)
4. Are the pods **ready**? (`kubectl get pods`)
5. Is the port right? (`describe svc`: Port vs TargetPort vs the container port)
6. Does the pod itself answer? (`curl <pod-ip>`)

---

## Step 31: The challenge - fix the broken app

`broken.yml` deploys an app called `shop` that is **broken in several ways**. From
the `dev` namespace, this command should print `Name: shop`:

    kubectl -n dev exec client -- curl -s --max-time 5 shop-svc.lab | grep '^Name'

**Do**

    kubectl apply -f broken.yml

Now use *only* the routine above and `kubectl` to diagnose, one layer at a time.
Don't read `broken.yml`: finding the faults with `kubectl get/describe` is the exercise.
Fix them in the cluster (for example `kubectl edit`, `kubectl patch`, or by editing
the file and re-applying it). Stop when the command above prints `Name: shop`.

<details>
<summary>Hints (open one at a time)</summary>

1. Start from the client in `dev`. What exact error does curl give? Which layer is that?
2. Once you reach the right name, is there anything to send traffic to? Check the endpoints.
3. Compare the Service's `Selector` with `kubectl get pods --show-labels`.
4. When endpoints are still empty, look at the pods' READY column and `describe pod`.
5. When endpoints exist but the request still fails, compare `TargetPort` with the container's port.

</details>

<details>
<summary>Solution (spoilers)</summary>

There are **four** faults, hiding behind each other:

1. **Wrong name from `dev`:** `shop-svc` doesn't resolve from the `dev` namespace
   (curl exit 6). Use `shop-svc.lab`. (Not a manifest bug: the command in the challenge
   already uses the right name.)
2. **Selector mismatch:** `describe svc shop-svc` shows `Selector: app=shop-web`, but
   the pods are labelled `app=shop`, so there are no endpoints (connection refused).
   Fix: `kubectl -n lab patch svc shop-svc -p '{"spec":{"selector":{"app":"shop"}}}'`.
3. **Readiness probe on the wrong port:** the pods then show `READY 0/1`;
   `describe pod` Events show `Readiness probe failed: ... :8081 connection refused`, so
   they still aren't endpoints. Fix the probe port to 80 in the Deployment (for example
   `kubectl -n lab edit deploy shop`); the pods restart and become ready.
4. **Wrong `targetPort`:** now endpoints exist but end in `:8080`, and requests are refused.
   Fix: `kubectl -n lab patch svc shop-svc --type=json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":80}]'`.

Then the challenge command prints `Name: shop`.

</details>

---

## Cleanup

    kubectl -n kube-system get deploy coredns        # READY back to its original count (2/2)
    kubectl delete ns lab dev
    kind delete cluster --name lab-a
    kind get clusters
    kubectl config get-contexts

If no context is current afterwards, pick one:
`kubectl config use-context docker-desktop`.

---

## End of Session 2: checkpoint

Before the cleanup:

    kubectl -n kube-system get deploy coredns                       # 2/2
    kubectl -n dev exec client -- curl -s shop-svc.lab | grep '^Name'   # Name: shop  (after Step 31)

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Names stop resolving everywhere | You may have left CoreDNS at 0 replicas: `kubectl -n kube-system scale deploy coredns --replicas=2` |
| `kubectl exec` needs `-n` | This session targets two namespaces on purpose; always pass `-n lab` or `-n dev` |
| `Error from server (NotFound): namespaces "dev" not found` | Apply `namespace-dev.yml` before `app-dev.yml` and `tools.yml` |
| `client` pod can't start | `kubectl -n lab describe pod client`: image pull problem? (needs internet) |
| `nslookup` not found in `client` | Use the `dnsutils` pod (`-n lab`) for `nslookup`; `client` only has curl |
| Step 31: nothing seems wrong after one fix | Expected: each fault hides the next. Re-run the routine from step 1 |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] write the fully qualified name of a Service
- [ ] explain why a short name only works inside its own namespace
- [ ] say that namespaces scope names, not network traffic
- [ ] explain why a Service still works by IP when DNS is down
- [ ] tell "refused" from "timeout" and name the likely layer
- [ ] follow the 6-step routine to fix a broken app

## Stretch ideas

- Add a `NetworkPolicy` that blocks traffic from `dev` to `lab`, and see what changes (kind's default CNI may not enforce it: read about it first).
- Break the same app again in a new way and have someone else fix it.
- Add an `ExternalName` Service in `dev` pointing at `whoami-svc.lab.svc.cluster.local`.
- Look at `kubectl get events -A --sort-by=.lastTimestamp` while you break and fix things.
