# Session 20: Authorization - RBAC 

**Work from this directory:** `cd 20-authorization-rbac`  

**Goal:** learn the second check in the chain: *may this identity do this?* Write Roles and
bindings, discover sub-resources and API groups the hard way (by being refused), use the built-in
`view`/`edit`/`admin` roles, give a ServiceAccount exactly the access it needs, and see why nobody can
simply promote themselves.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing of your own in it. You do **not** need Session 19: this session
uses **impersonation** (`--as`), so no certificates are required.
**Ending state:** everything from this session deleted, including the ClusterRoles it created.

**Impersonation.** As the cluster admin you can run any command *as another identity*:

    kubectl get pods --as alice --as-group dev-team -n lab

The API server then authorizes the request as `alice` in group `dev-team`. That's identical to alice using
her own certificate, which makes it the standard way to test RBAC. (Impersonating is itself a permission
that only admins hold.)

**The model in four lines**

    Role / ClusterRole              = a list of ALLOWED actions (apiGroups + resources + verbs)
    RoleBinding / ClusterRoleBinding = who gets that list (users, groups, ServiceAccounts)
    namespaced Role + RoleBinding    -> only inside one namespace
    ClusterRole + ClusterRoleBinding -> across the whole cluster

There is **no deny** and no ordering: access is the **union** of everything granted, and if nothing allows
a request, it is refused.

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

Docker must be running. No cloud and no extra tools are needed.

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get pods                      # No resources found

All match: go straight to Step 70.

### Bootstrap (if anything above is missing)

No `lab-a` cluster yet (check with `kind get clusters`):

    kind create cluster --name lab-a --config cluster.yml
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Cluster exists but the namespace default is wrong or missing:

    kubectl config use-context kind-lab-a
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Leftovers in `lab` from other sessions or experiments:

    kubectl delete ns lab
    kubectl apply -f namespace.yml

Completely broken cluster: `kind delete cluster --name lab-a`, then the first block above.

---

## Step 70: A Role and a RoleBinding

**Do**

    kubectl apply -f pods.yml
    kubectl wait --for=condition=Ready pod/web pod/db --timeout=90s
    kubectl auth can-i list pods --as alice -n lab
    kubectl get pods --as alice -n lab
    kubectl apply -f rbac-1-pod-reader.yml
    kubectl auth can-i list pods --as alice -n lab
    kubectl get pods --as alice -n lab
    kubectl delete pod web --as alice -n lab
    kubectl get pods --as alice -n kube-system
    kubectl get nodes --as alice
    kubectl get secrets --as alice -n lab
    kubectl auth can-i --list --as alice -n lab
    kubectl describe role pod-reader -n lab
    kubectl describe rolebinding alice-reads-pods -n lab

**Expect**

- Before the binding: `can-i` answers **no** and `get pods` is **Forbidden**: *User "alice" cannot list resource
  "pods" in API group "" in the namespace "lab"*. (Note how the message names the **user, verb, resource, API group
  and namespace**: everything you need to write the missing rule.)
- After applying the Role and binding: `yes`, and `get pods` lists `web` and `db`.
- **delete** is still Forbidden (the Role only has get, list, watch). The same Role does **nothing outside `lab`**
  (`kube-system` is Forbidden), nothing for cluster-scoped things (`nodes` is Forbidden *at the cluster scope*), and
  nothing for other resources (`secrets`).
- `can-i --list` shows the pods rules plus a few basics every authenticated user has.

**Why**

A **Role** is a list of allowed actions; a **RoleBinding** says who gets it. alice appears only as a *name* in the
binding: nothing "creates" her. Permissions are *additive*: start from nothing and grant exactly what is needed.

**Try**

    kubectl get role pod-reader -n lab -o yaml
    kubectl auth can-i get pods --as alice -n lab --subresource=log

---

## Step 71: Verbs, sub-resources and API groups

**Do**

    kubectl logs web --as alice -n lab
    kubectl exec web --as alice -n lab -- echo "hi"
    kubectl get deployments --as alice -n lab
    kubectl api-resources -o wide | grep -E '^(pods|deployments|secrets) '
    kubectl apply -f rbac-2-logs-exec.yml
    kubectl logs web --as alice -n lab
    kubectl exec web --as alice -n lab -- echo "hi from alice"
    kubectl get deployments --as alice -n lab
    kubectl apply -f rbac-3-deployments.yml
    kubectl get deployments --as alice -n lab
    kubectl auth can-i --list --as alice -n lab

**Expect**

- Reading logs is refused: *cannot get resource **"pods/log"***; exec is refused: *cannot create resource **"pods/exec"***; listing
  deployments is refused: *cannot list resource "deployments" in API group **"apps"***. Having `get pods` covered none
  of them.
- `api-resources -o wide` shows each resource's **API group** (in the APIVERSION column: `v1` is the core group,
  `apps/v1` is `apps`) and its **verbs**.
- After `rbac-2`, logs work and `exec` runs: alice can now do what a pod can do. The change took effect **at once**:
  nobody restarted anything. Deployments are still refused.
- After `rbac-3` (a rule for group `apps`), `get deployments` answers *No resources found*: **allowed**, just empty.

**Why**

Three things to remember when writing rules:

| | |
|---|---|
| **Sub-resources** | `pods/log` (get), `pods/exec` (create), `pods/portforward`, `pods/status`... each its own permission |
| **API groups** | `""` core, `apps`, `batch`, `rbac.authorization.k8s.io`... a rule for `pods` says nothing about `deployments` |
| **Verbs** | `get` (one by name), `list`, `watch`, `create`, `update`, `patch`, `delete`... Note `list` and `watch` return **full objects**, as `get` does |

When something is Forbidden, the message tells you the exact resource and group to add.

**Try**

Restrict a rule to one pod by name:

    kubectl patch role pod-reader -n lab --type=json -p '[{"op":"add","path":"/rules/0/resourceNames","value":["web"]}]'
    kubectl get pod web --as alice -n lab -o name
    kubectl get pod db --as alice -n lab -o name
    kubectl get pods --as alice -n lab          # list is refused: resourceNames can't filter a plain list
    kubectl apply -f rbac-3-deployments.yml     # restore

---

## Step 72: ClusterRoles, groups, aggregation and the built-in roles

**Do: cluster-wide access for a group**

    kubectl apply -f cluster-roles.yml
    kubectl get nodes --as carol --as-group ops-team
    kubectl get nodes --as carol
    kubectl get nodes --as alice --as-group ops-team
    kubectl get clusterrole all-readers -o yaml | sed -n '/rules:/,$p'

**Do: the built-in roles, bound inside one namespace**

    kubectl create secret generic demo --from-literal=k=v -n lab
    kubectl create rolebinding dev-view --clusterrole=view --group=dev-team -n lab
    kubectl create rolebinding ops-edit --clusterrole=edit --user=erin -n lab
    kubectl create rolebinding lead-admin --clusterrole=admin --user=frank -n lab
    kubectl auth can-i list pods --as dave --as-group dev-team -n lab
    kubectl auth can-i list secrets --as dave --as-group dev-team -n lab
    kubectl auth can-i create deployments --as dave --as-group dev-team -n lab
    kubectl auth can-i list secrets --as erin -n lab
    kubectl auth can-i create rolebindings --as erin -n lab
    kubectl auth can-i create rolebindings --as frank -n lab
    kubectl auth can-i list pods --as frank -n kube-system
    kubectl get clusterroles | grep -v '^system:'
    kubectl get clusterrole view -o yaml | head -12

**Expect**

- `carol` in group `ops-team` can list **nodes** (cluster-scoped, via a ClusterRoleBinding); carol alone cannot;
  `alice` given the group `ops-team` can. **Permissions follow groups**, not individuals.
- The aggregated `all-readers` ClusterRole now contains rules for **pods and configmaps**, which you never wrote into it.
- The matrix:

  | Subject | list pods | list secrets | create deployments | create rolebindings |
  |---|---|---|---|---|
  | dave (group dev-team, role **view**) | yes | **no** | no | no |
  | erin (role **edit**) | - | **yes** | - | **no** |
  | frank (role **admin**) | - | - | - | **yes** |

  (`-` = not tested in the commands above.)

  `view` is read-only and **excludes Secrets**; `edit` reads and writes almost everything including Secrets but
  **cannot manage RBAC**; `admin` can also manage Roles and RoleBindings **in the namespace**.
- `frank` cannot list pods in `kube-system`: a **ClusterRole bound with a RoleBinding is limited to that namespace**.
- `view` shows an `aggregationRule`: the built-in roles are aggregated, so new resource types can add themselves.

**Why**

Reuse beats repetition: the four user-facing ClusterRoles (**`cluster-admin`, `admin`, `edit`, `view`**) cover most needs.
*Binding scope* decides reach: a ClusterRole with a **RoleBinding** applies to one namespace (a handy pattern: define
roles once, bind them per namespace), with a **ClusterRoleBinding** everywhere. Bind to **groups**, not people, and let
your identity provider manage membership.

---

## Step 73: ServiceAccounts, privilege escalation and reading a Forbidden

**Do: a pod calling the API**

    kubectl apply -f sa-api.yml
    kubectl wait --for=condition=Ready pod/api-client --timeout=90s
    apicall() { kubectl exec api-client -- sh -c "curl -s -o /dev/null -w '%{http_code}\n' --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H \"Authorization: Bearer \$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)\" https://kubernetes.default.svc$1"; }
    apicall /api/v1/namespaces/lab/pods
    kubectl apply -f sa-binding.yml
    apicall /api/v1/namespaces/lab/pods
    apicall /api/v1/namespaces/kube-system/pods
    kubectl auth can-i list pods --as system:serviceaccount:lab:app-sa -n lab

**Do: privilege escalation prevention**

    kubectl apply -f rbac-editor.yml
    kubectl auth can-i create roles --as bob -n lab
    kubectl apply -f role-secrets.yml --as bob
    kubectl apply -f rbac-editor-escalate.yml
    kubectl apply -f role-secrets.yml --as bob
    kubectl get role secret-reader -n lab

**Expect**

- The in-pod request returns **403** before the binding, **200** after it for `lab`, and **403** still for `kube-system`.
  The pod's identity is `system:serviceaccount:lab:app-sa`: least privilege, namespace-scoped.
- bob may create Roles (`yes`), but applying `role-secrets.yml` as bob is **Forbidden**, with a message like:
  *user "bob" ... **is attempting to grant RBAC permissions not currently held**: {APIGroups:[""], Resources:["secrets"], Verbs:["get"]}*.
- After `rbac-editor-escalate.yml` adds the verb **`escalate`**, the same command **succeeds**.

**Why**

Without escalation prevention, anyone allowed to edit Roles could grant themselves everything. The API server therefore refuses
to let you create or bind a Role containing permissions **you don't hold**, unless you have **`escalate`** (create) or **`bind`**
(bind). Together with **`impersonate`**, these verbs are the keys to the kingdom.

**Reading a Forbidden, and the debugging routine**

> `User "alice" cannot delete resource "pods" in API group "" in the namespace "lab"`

| Part | Tells you |
|---|---|
| User / groups | the identity the server decided you are (check with `kubectl auth whoami`) |
| `delete` | the verb |
| `pods`, API group `""` | resource and group (add a `/sub` for sub-resources) |
| namespace / cluster scope | where a rule is needed |

1. Read the message: it is the missing rule.
2. `kubectl auth can-i <verb> <resource> --as <who> -n <ns>` to confirm.
3. `kubectl auth whoami` (or `--as`) to be sure about the **identity and groups**.
4. Find the bindings: `kubectl get rolebindings,clusterrolebindings -A -o wide | grep <name-or-group>`.
5. `kubectl describe role <name>`: is the verb / group / sub-resource there?

**Permissions that deserve extra care**

| Permission | Why it is powerful |
|---|---|
| `get`/`list`/`watch` on `secrets` | reads every password and token in the namespace |
| `create` on `pods` | can mount any Secret and run any image: effectively "read Secrets" |
| `create` on `pods/exec` | shell in any pod it can reach |
| `escalate`, `bind` on roles | can grant themselves anything |
| `impersonate` | can act as anyone, including admins |
| `*` on `*` | everything |
| membership of `system:masters` | bypasses RBAC entirely |

---

## Cleanup for this session

    kubectl delete ns lab
    kubectl apply -f namespace.yml
    kubectl delete clusterrole node-reader reader-pods reader-configmaps all-readers
    kubectl delete clusterrolebinding ops-read-nodes
    kubectl get clusterroles | grep -E 'node-reader|reader-|all-readers' || echo "cluster roles removed"

Namespaced Roles, RoleBindings, pods and ServiceAccounts disappeared with the namespace. **ClusterRoles and
ClusterRoleBindings are cluster-wide**, so they needed deleting separately.

---

## End of Session 20: checkpoint

    kubectl get pods,roles,rolebindings   # No resources found
    kubectl get clusterrole node-reader   # NotFound

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Still Forbidden after applying a Role | Is there a **RoleBinding**? Right namespace, right subject name/group, right `apiGroups`/`resources`/`verbs`? `kubectl auth can-i ... --as ...` |
| `--as` has no effect, everything is allowed | You forgot `--as` on that command (you ran it as the admin) |
| `Error from server (Forbidden): ... cannot impersonate` | You are not the admin (wrong kubeconfig) |
| `exec` Forbidden although `get pods` is allowed | Needs `create` on `pods/exec` as well |
| Deployments Forbidden although `pods` is allowed | Different API group: `apps` |
| `apicall` prints nothing | The function is only defined in the terminal where you typed it; paste it again |
| `roles ... is forbidden: ... attempting to grant RBAC permissions not currently held` | That's the escalation check (Step 73) |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)
    kubectl auth can-i <verb> <resource> --as <user> -n <ns>     what a subject may do
    kubectl auth whoami              who the API server thinks you are

## Done when you can, without looking

- [ ] write a Role and RoleBinding, and say what `apiGroups`, `resources` and `verbs` mean
- [ ] explain why RBAC has no deny and no ordering
- [ ] explain sub-resources (`pods/log`, `pods/exec`) and API groups, and the usual mistakes with them
- [ ] choose between Role + RoleBinding, ClusterRole + RoleBinding and ClusterRole + ClusterRoleBinding
- [ ] describe `view`, `edit` and `admin`, and what `view` leaves out
- [ ] test permissions with `auth can-i` and `--as`, and read a Forbidden message
- [ ] explain privilege escalation prevention and the verbs `escalate`, `bind`, `impersonate`

## Stretch ideas

- Write a Role that lets a user `get` and `patch` exactly one Deployment (`resourceNames`) and nothing else.
- Use `kubectl get clusterrolebindings -o wide` to audit who holds `cluster-admin` on your cluster.
- Install the `kubectl-who-can` plugin (krew) and ask "who can get secrets?".
- Read `kubectl auth reconcile --help`: apply RBAC manifests without clobbering existing bindings.
