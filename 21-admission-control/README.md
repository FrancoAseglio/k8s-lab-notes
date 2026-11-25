# Session 21: Admission control - the last gate 

**Work from this directory:** `cd 21-admission-control` 

**Goal:** understand what happens *after* a request is authenticated and authorized: **admission
controllers** can change it (mutating) and refuse it (validating). You will meet the built-in ones
(LimitRange, ResourceQuota), **Pod Security Admission**, and the two policy types you can write yourself in
CEL, then follow one request through every check from this lab.

**Starting state:** kind cluster `lab-a` (3 nodes, Kubernetes 1.37), kubectl context `kind-lab-a`
with default namespace `lab`. You do **not** need the other sessions.
**Ending state:** all namespaces and cluster-wide policies of this session deleted.

**Where this fits**

    request -> authentication -> authorization (RBAC) -> MUTATING admission -> schema validation
            -> VALIDATING admission -> stored in etcd

- **Mutating** admission runs first and may *change* the object (add defaults, labels, sidecars).
- **Validating** admission runs last and may only *accept or refuse* it, and it sees the final, mutated object.
- Admission applies to **writes** (create, update, delete), never to reads, and it applies to **everyone**,
  cluster-admin included: RBAC saying "yes" doesn't override a policy.
- Admission only judges **new requests**: objects that already exist are not touched.

Read the top of each YAML file as you use them.

---

## Golden rule

Before any command that changes something, know **which cluster and which
namespace** you are on:

    kubectl config current-context
    kubectl config get-contexts           # NAMESPACE column; empty = default

Applying to the wrong cluster, or looking in the wrong namespace, are the two
most common real-world mistakes.

> **Run this lab on the kind cluster only.** You will create users, roles, policies and
> namespaces. Make sure `kubectl config current-context` prints **`kind-lab-a`**, and *not*
> `docker-desktop` or any real cluster. (Your admin credential is a superuser there too.)

## Prerequisites

    docker version
    kind version
    kubectl version --client
    kubectl version                       # the SERVER version must be 1.36 or later for step 77

Docker must be running. No cloud and no extra tools are needed. Step 77 (the mutating policy) needs a
Kubernetes server of version 1.36 or later; your kind cluster (1.37) has it.

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get ns adm-demo psa-baseline psa-restricted psa-warn vap-demo       # NotFound for all: fine
    kubectl get validatingadmissionpolicy,mutatingadmissionpolicy               # No resources found

All match: go straight to Step 74.

### Bootstrap (if anything above is missing)

No `lab-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name lab-a --config cluster.yml
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Cluster exists but the namespace default is wrong or missing:

    kubectl config use-context kind-lab-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Leftovers from an earlier attempt at this session (namespaces and cluster-wide policies):

    kubectl delete ns adm-demo psa-baseline psa-restricted psa-warn vap-demo --ignore-not-found
    kubectl delete validatingadmissionpolicybinding no-latest-tag-binding max-replicas-binding --ignore-not-found
    kubectl delete validatingadmissionpolicy no-latest-tag max-replicas --ignore-not-found
    kubectl delete mutatingadmissionpolicybinding label-admitted-pods-binding --ignore-not-found
    kubectl delete mutatingadmissionpolicy label-admitted-pods --ignore-not-found

Completely broken cluster: `kind delete cluster --name lab-a`, then the first block above.

---

## Step 74: The pipeline and the built-in controllers

**Do**

    kubectl -n kube-system get pods -l component=kube-apiserver -o yaml | grep -E 'admission'
    kubectl apply -f namespaces.yml
    kubectl apply -f limitrange.yml -f quota.yml
    kubectl describe limitrange,resourcequota -n adm-demo
    kubectl apply -f pod-plain.yml -n adm-demo --dry-run=server -o yaml | grep -E 'serviceAccountName|resources:|cpu|memory'
    kubectl apply -f pod-plain.yml -n adm-demo
    kubectl get pod plain -n adm-demo -o jsonpath='{.spec.serviceAccountName}{"\n"}{.spec.containers[0].resources}{"\n"}'
    kubectl apply -f pod-big.yml -n adm-demo
    kubectl run second --image=busybox:1.37 -n adm-demo -- sleep 3600
    kubectl run third --image=busybox:1.37 -n adm-demo -- sleep 3600
    kubectl describe resourcequota pods-quota -n adm-demo

**Expect**

- The API server's arguments list only `--enable-admission-plugins=NodeRestriction`: that's the
  *extra* plugin kubeadm turns on. A set of **default** plugins (NamespaceLifecycle, LimitRanger, ServiceAccount,
  DefaultStorageClass, ResourceQuota, PodSecurity, the policy and webhook plugins...) is on without being listed.
- The `--dry-run=server` output of a pod you wrote without resources **already contains** `serviceAccountName: default`
  and CPU/memory requests and limits: **admission added them** (`50m`/`32Mi` requests, `100m`/`64Mi` limits).
  The stored pod has the same.
- `pod-big.yml` is **Forbidden**: *maximum memory usage per Container is 128Mi, but limit is 256Mi*.
- `second` is created (2 pods in the namespace); `third` is refused:
  **`exceeded quota: pods-quota, requested: pods=1, used: pods=2, limited: pods=2`**.

**Why**

Admission controllers are plugins compiled into the API server, running in a fixed order after RBAC. You have met
them before without knowing: the default StorageClass that appeared on your claims (lab 5) and the ServiceAccount token
mounted into pods (Session 19) were both added by admission. `LimitRange` shows both roles at once: it **mutates**
(fills in defaults) and **validates** (enforces the maximum).

Beyond the built-ins there are two extension mechanisms, which you will use in this session: **policies written in CEL**
(inside the API server, no extra process) and **admission webhooks** (calls to your own HTTPS service: more powerful, but a
service to run and secure; `failurePolicy` decides whether requests fail or pass when it is down).

**Try**

    kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations     # none on a bare cluster

---

## Step 75: Pod Security Admission

**Do**

    kubectl get ns psa-baseline psa-restricted psa-warn --show-labels
    kubectl apply -f pod-privileged.yml -n psa-baseline
    kubectl apply -f pod-plain.yml -n psa-baseline
    kubectl apply -f pod-plain.yml -n psa-restricted
    kubectl apply -f pod-compliant.yml -n psa-restricted
    kubectl apply -f pod-plain.yml -n psa-warn
    kubectl apply -f deploy-root.yml -n psa-restricted
    kubectl get deploy,rs -n psa-restricted
    kubectl describe rs -n psa-restricted | tail -6
    kubectl label --dry-run=server --overwrite ns adm-demo pod-security.kubernetes.io/enforce=restricted

**Expect**

- Namespaces carry the labels `pod-security.kubernetes.io/<mode>=<level>`.
- **`baseline` refuses the privileged pod**: *violates PodSecurity "baseline:latest": privileged (container "busybox" must not
  set securityContext.privileged=true)*. It **accepts** the plain root pod.
- **`restricted` refuses the plain pod** and lists *everything* it lacks: `allowPrivilegeEscalation != false`, `unrestricted
  capabilities`, `runAsNonRoot != true`, `seccompProfile`. It **accepts** `pod-compliant.yml`.
- In `psa-warn` (only warn/audit) the plain pod is **created**, with a warning *would violate PodSecurity "restricted:latest"*.
- The **Deployment** is created, with a warning, but `get deploy` shows **0/1** and the ReplicaSet has `CURRENT 0`: its events show
  `FailedCreate ... violates PodSecurity "restricted:latest"`. Enforcement judges **pods**, so the failure surfaces late.
- The `--dry-run=server` label change prints a warning that **existing pods in `adm-demo` would violate** `restricted`, and
  names them, without changing anything.

**Why**

Pod Security Admission replaces the removed PodSecurityPolicy. It is **built in and configured only with namespace
labels**: three **levels** (*privileged* = no limits, *baseline* = blocks known escalations like privileged containers and hostPath,
*restricted* = current hardening best practice) and three **modes** (*enforce* rejects, *warn* tells the user, *audit* logs).
The safe way to adopt it: start with `warn` and `audit`, use the server-side dry run to see what would break, then `enforce`.
Since it only judges Pods, set `warn` too so people creating Deployments hear about problems immediately.

**Try**

    kubectl apply -f pod-privileged.yml -n psa-warn           # allowed: nothing enforces, only a warning
    kubectl get pods -n psa-warn

---

## Step 76: ValidatingAdmissionPolicy (CEL)

**Do: a policy without parameters**

    kubectl apply -f vap-no-latest.yml
    kubectl get validatingadmissionpolicy,validatingadmissionpolicybinding
    kubectl run ok --image=busybox:1.37 -n vap-demo -- sleep 3600
    kubectl run bad1 --image=busybox:latest -n vap-demo -- sleep 3600
    kubectl run bad2 --image=busybox -n vap-demo -- sleep 3600
    kubectl run elsewhere --image=busybox:latest -n psa-baseline -- sleep 3600

**Do: a policy with a parameter**

    kubectl apply -f vap-replicas.yml
    kubectl create deployment web --image=nginx:1.27 --replicas=5 -n vap-demo
    kubectl create deployment web --image=nginx:1.27 --replicas=2 -n vap-demo
    kubectl patch configmap replica-limits -n vap-demo -p '{"data":{"maxReplicas":"1"}}'
    kubectl scale deploy web --replicas=2 -n vap-demo
    kubectl patch configmap replica-limits -n vap-demo -p '{"data":{"maxReplicas":"3"}}'
    kubectl scale deploy web --replicas=3 -n vap-demo

**Do: the late failure, and changing the consequence**

    kubectl create deployment latest-app --image=nginx:latest -n vap-demo
    kubectl get deploy,rs -n vap-demo
    kubectl describe rs -n vap-demo -l app=latest-app | tail -5
    kubectl patch validatingadmissionpolicybinding no-latest-tag-binding --type=merge -p '{"spec":{"validationActions":["Warn"]}}'
    kubectl run warned --image=busybox:latest -n vap-demo -- sleep 3600
    kubectl patch validatingadmissionpolicybinding no-latest-tag-binding --type=merge -p '{"spec":{"validationActions":["Deny"]}}'

**Expect**

- `ok` (tagged `1.37`) is created. `bad1` (`:latest`) and `bad2` (no tag) are **refused** with a message like
  *ValidatingAdmissionPolicy 'no-latest-tag' with binding 'no-latest-tag-binding' denied request: every container image must have an
  explicit tag other than :latest*. In `psa-baseline`, which isn't labelled `policy=enforced`, the same image is allowed: the
  **binding's namespace selector** decides where a policy applies.
- Creating the Deployment with 5 replicas is refused: *replicas must be at most 3 (requested 5)*; with 2 it works.
- After lowering the **ConfigMap** to `1`, `kubectl scale ... --replicas=2` is refused with *at most 1*, without touching the policy:
  **parameters** let you tune a policy as data. Restoring it to `3` makes the scale to 3 work. (`kubectl scale` is caught because the
  policy lists the `deployments/scale` sub-resource too.)
- `latest-app`: the **Deployment is created** (the policy matches Pods, not Deployments) but `get deploy` shows 0/1 and the ReplicaSet's
  events show the pod creation being **denied by the policy**: the same "late failure" as with Pod Security.
- With `validationActions: [Warn]` the pod is **created** and the client gets a **warning** with the same message; set back to `Deny`.

**Why**

A **ValidatingAdmissionPolicy** is declarative and runs inside the API server: no webhook service, no TLS, no outage if a service is down.
It has two halves: the **policy** (WHAT: resources + CEL expressions + messages) and the **binding** (WHERE and HOW: namespace selector and
actions `Deny`, `Warn`, `Audit`). A good way to roll out a new policy is `Audit` first (see what *would* be denied), then `Warn`, then `Deny`.
Always remember: policies see **new requests only**; they don't remove existing violations.

**Try**

    kubectl get validatingadmissionpolicy no-latest-tag -o yaml | sed -n '/validations/,$p'
    kubectl run existing-ok --image=busybox:1.37 -n vap-demo -- sleep 3600     # still fine

---

## Step 77: MutatingAdmissionPolicy (CEL)

**Do**

    kubectl apply -f map-label.yml
    kubectl get mutatingadmissionpolicy,mutatingadmissionpolicybinding
    kubectl run labelled --image=busybox:1.37 -n vap-demo -- sleep 3600
    kubectl get pod labelled -n vap-demo --show-labels
    kubectl run notlabelled --image=busybox:1.37 -n psa-baseline -- sleep 3600
    kubectl get pod notlabelled -n psa-baseline --show-labels

**Expect**

- The pod in `vap-demo` carries an **extra label**, `admission.lab/policy=label-admitted-pods`, next to the `run=labelled` that kubectl
  added. **You never wrote it**: the API server did, before storing the pod.
- The pod in `psa-baseline` (not selected by the binding) has only `run=notlabelled`.
- If `kubectl apply` says *no matches for kind "MutatingAdmissionPolicy"*, the server is older than 1.36.

**Why**

Mutating policies **change** requests: add default labels or annotations, inject a sidecar, set a security context, fill in missing
fields. Previously that meant a mutating *webhook*; now the same can be a CEL expression inside the API server. Because mutation runs
**before** validation, your validating policies and Pod Security see the *result*: a mutating policy can make a non-compliant pod compliant
(for example by adding a `securityContext`), which is a powerful, and potentially surprising, thing.

**Try**

Look at how the stored pod differs from what you typed: `kubectl get pod labelled -n vap-demo -o yaml | head -30`.

---

## Step 78: One request through every check

This step runs a request through everything in this lab and shows which gate stops it.

**Do**

    kubectl create rolebinding dev1-edit --clusterrole=edit --user=dev1 -n vap-demo
    kubectl create rolebinding dev1-edit --clusterrole=edit --user=dev1 -n psa-restricted
    SERVER=$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.server}')
    kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 --decode > ca.crt
    curl -s --cacert ca.crt -H "Authorization: Bearer garbage" $SERVER/api/v1/namespaces/vap-demo/pods | grep -E '"code"|"reason"'
    kubectl run t1 --image=busybox:1.37 -n vap-demo --as nobody -- sleep 3600
    kubectl run t2 --image=busybox:latest -n vap-demo --as dev1 -- sleep 3600
    kubectl run t3 --image=busybox:1.37 -n psa-restricted --as dev1 -- sleep 3600
    kubectl run t4 --image=busybox:1.37 -n adm-demo -- sleep 3600
    kubectl run t5 --image=busybox:1.37 -n vap-demo --as dev1 -- sleep 3600
    kubectl get pod t5 -n vap-demo --show-labels

**Expect, and which gate said it**

| Request | Result | Gate |
|---|---|---|
| bad token | `"code": 401`, `Unauthorized` | **1 Authentication** |
| `t1` as `nobody` | *User "nobody" cannot create resource "pods"...* (Forbidden) | **2 Authorization (RBAC)** |
| `t2` as `dev1` (RBAC allows it) | *ValidatingAdmissionPolicy 'no-latest-tag' ... denied request* | **5 Validating admission (policy)** |
| `t3` as `dev1` (RBAC allows it) | *violates PodSecurity "restricted:latest": ...* | **5 Validating admission (Pod Security)** |
| `t4` (admin, namespace full) | *exceeded quota: pods-quota...* | **5 Validating admission (ResourceQuota)** |
| `t5` as `dev1` | created, with the **extra label** | passed every gate; **3 Mutating admission** added the label |

**Why: a decoder for the messages you will meet**

| The message says | It was refused at | Look at |
|---|---|---|
| `Unauthorized` / `You must be logged in` | authentication | credentials, certificate, token, expiry |
| `User "x" cannot <verb> resource "y" in API group ...` | authorization | RBAC: roles and bindings |
| `violates PodSecurity "<level>"` | Pod Security Admission | the pod's `securityContext`, the namespace labels |
| `ValidatingAdmissionPolicy '<name>' ... denied request` | a validating policy | the named policy and binding |
| `admission webhook "<name>" denied the request` | a validating webhook | that webhook's service |
| `exceeded quota` / `maximum ... per Container` | ResourceQuota / LimitRange | the quotas and limits of the namespace |
| `... is invalid` | schema validation | the object itself |

Notice what RBAC could **not** do: `dev1` was *allowed* to create pods, yet `t2` and `t3` were refused. Authorization asks "may you
create pods?"; admission asks "is **this pod** acceptable?". You need both.

---

## Cleanup for this session

    kubectl delete validatingadmissionpolicybinding no-latest-tag-binding max-replicas-binding
    kubectl delete validatingadmissionpolicy no-latest-tag max-replicas
    kubectl delete mutatingadmissionpolicybinding label-admitted-pods-binding
    kubectl delete mutatingadmissionpolicy label-admitted-pods
    kubectl delete ns adm-demo psa-baseline psa-restricted psa-warn vap-demo
    rm -f ca.crt
    unset SERVER
    kubectl get validatingadmissionpolicy,mutatingadmissionpolicy,ns

Policies and their bindings are **cluster-wide**: deleting the namespaces does not remove them, so they are deleted first.
They only applied to namespaces labelled `policy=enforced`, but leaving them behind would surprise you in a later lab.

---

## End of Session 21: checkpoint

    kubectl get validatingadmissionpolicy,mutatingadmissionpolicy       # No resources found
    kubectl get ns | grep -E 'adm-demo|psa-|vap-demo' || echo "namespaces removed"

---

## Troubleshooting

| Symptom | Check |
|---|---|
| A policy denies nothing | Is there a **binding**? Does its namespace selector match the namespace's labels (`policy=enforced`)? Is `validationActions` `Deny`? |
| New policy not effective at once | Give the API server a few seconds to compile and activate it, then retry |
| `no matches for kind "MutatingAdmissionPolicy"` | Server older than 1.36: `kubectl version` |
| CEL error in `describe validatingadmissionpolicy` | Look at `status.typeChecking` and the policy's conditions for the expression problem |
| `kubectl scale` bypasses a policy | The policy must also list the `<resource>/scale` sub-resource (as `vap-replicas.yml` does) |
| Deployment `0/1` and no error | Admission refused the **pods**: `kubectl describe rs` shows the reason |
| Pod refused in the `psa-*` namespaces unexpectedly | `kubectl get ns <name> --show-labels`, and compare your pod with `pod-compliant.yml` |
| `exceeded quota` in `adm-demo` | Expected (2 pods): delete a pod or raise the quota |
| `curl: (60) SSL certificate problem` | `ca.crt` missing: regenerate it as in Step 78 |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)
    kubectl apply --dry-run=server -f <file> -o yaml    what the API server WOULD store
    kubectl auth can-i <verb> <resource> --as <user> -n <ns>

## Done when you can, without looking

- [ ] place admission in the request flow, and say what mutating and validating admission each do
- [ ] explain why RBAC "yes" does not override admission, and that admission applies to writes only
- [ ] name two built-in admission controllers and what each does
- [ ] label a namespace for Pod Security Admission, and choose level and mode (and say why to start with `warn`)
- [ ] explain why a Deployment can be created while its pods are refused
- [ ] write the two halves of a ValidatingAdmissionPolicy and say what a binding adds
- [ ] use parameters, and the `Deny`/`Warn`/`Audit` actions, to roll a policy out safely
- [ ] decode an error message and say which gate produced it

## Stretch ideas

- Make the `no-latest-tag` policy also check `initContainers` (and `ephemeralContainers`).
- Add `audit-annotations` to a policy and read where they appear.
- Write a policy that requires an `owner` label on every Namespace.
- Use the mutating policy to add a default `securityContext` and watch Pod Security accept a pod it would have refused.
- Read about admission **webhooks**, `failurePolicy`, and why a webhook outage can take down a cluster; look at Kyverno and OPA Gatekeeper as policy engines.
