# Session 19: Authentication - who are you? 

**Work from this directory:** `cd 19-authentication` 

**Goal:** find out how Kubernetes knows who you are. Look at your own admin identity, create a
brand-new user from a certificate, tell an *authentication* failure (401) from an
*authorization* failure (403), and meet the identity of programs: ServiceAccounts and their tokens.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing of your own in it.
**Ending state:** everything from this session deleted, including the keys and kubeconfig files.

**Where this fits.** Every API request goes through a series of checks:

    request -> 1 AUTHENTICATION -> 2 AUTHORIZATION (RBAC) -> 3 mutating admission
            -> 4 schema validation -> 5 validating admission -> 6 stored in etcd

This session is step 1: *who is making the request?* The answer is a **name and a list of groups**.
Nothing here decides what that identity may *do*: that is Session 20.

Read the top of `csr-template.yml` and `sa.yml` as you use them.

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
    openssl version
    curl --version
    python3 --version                     # only used to decode a token (optional)

Docker must be running. No cloud is needed.

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get pods                      # No resources found

All match: go straight to Step 66.

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

## Step 66: Who am I? Your own identity

**Do**

    kubectl config current-context
    kubectl auth whoami
    kubectl config view --minify
    kubectl config view --raw --minify -o jsonpath='{.users[0].user.client-certificate-data}' | base64 --decode | openssl x509 -noout -subject -issuer -dates
    kubectl get clusterrolebindings | grep cluster-admin
    kubectl describe clusterrolebinding kubeadm:cluster-admins
    kubectl auth can-i '*' '*'
    kubectl get users

**Expect**

- `auth whoami` shows **Username `kubernetes-admin`** and groups including
  **`kubeadm:cluster-admins`** and `system:authenticated`.
- `config view --minify` shows three things your kubeconfig ties together: a **cluster** (server
  address and CA), a **user** (here a client certificate and key, shown as `DATA+OMITTED`) and a
  **context** (the pair, plus a default namespace).
- The certificate **subject** is `O=kubeadm:cluster-admins, CN=kubernetes-admin`, issued by the
  cluster's own CA (`kubernetes`). So: **CN is the user name, O is the group.**
- Two ClusterRoleBindings give `cluster-admin` away: `cluster-admin` (to the group `system:masters`)
  and `kubeadm:cluster-admins` (to the group in your certificate).
- `can-i '*' '*'` answers `yes`: you can do anything.
- `kubectl get users` fails: **the server doesn't have a resource type "users".**

**Why**

Kubernetes has **no user database**. A "user" is whatever name appears in a credential the API server
trusts, here a certificate signed by the cluster CA. Your admin power does not come from the
certificate itself but from an ordinary **RBAC binding** of its group to the `cluster-admin` role.

That is a recent improvement: older clusters (and the separate "super-admin" credential) use the group
**`system:masters`**, which is hard-wired to bypass authorization entirely and cannot be revoked by
deleting a binding. With a `kubeadm:cluster-admins` group, an admin's access can be removed like any other.
If your certificate shows `system:masters`, your cluster is older or uses a different tool.

**Try**

    kubectl get clusterrole cluster-admin -o yaml | head -20
    kubectl get clusterrolebinding cluster-admin -o yaml | grep -A3 subjects

---

## Step 67: Create a user: key, request, approval, kubeconfig

**Do: key and request**

    openssl genrsa -out alice.key 2048
    openssl req -new -key alice.key -out alice.csr -subj "/CN=alice/O=dev-team"
    sed "s|__CSR__|$(base64 < alice.csr | tr -d '\n')|" csr-template.yml > csr.generated.yml
    kubectl apply -f csr.generated.yml
    kubectl get csr
    kubectl describe csr alice | head -20

**Do: approve and collect the certificate**

    kubectl certificate approve alice
    kubectl get csr
    kubectl get csr alice -o jsonpath='{.status.certificate}' | base64 --decode > alice.crt
    openssl x509 -in alice.crt -noout -subject -issuer -dates

**Do: build alice's kubeconfig**

    kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 --decode > ca.crt
    SERVER=$(kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.server}'); echo $SERVER
    kubectl config set-cluster lab-a --server=$SERVER --certificate-authority=ca.crt --embed-certs=true --kubeconfig=alice.kubeconfig
    kubectl config set-credentials alice --client-certificate=alice.crt --client-key=alice.key --embed-certs=true --kubeconfig=alice.kubeconfig
    kubectl config set-context alice@lab-a --cluster=lab-a --user=alice --namespace=lab --kubeconfig=alice.kubeconfig
    kubectl config use-context alice@lab-a --kubeconfig=alice.kubeconfig

**Do: use it**

    kubectl --kubeconfig alice.kubeconfig auth whoami
    kubectl --kubeconfig alice.kubeconfig get pods
    kubectl --kubeconfig alice.kubeconfig auth can-i list pods

**Expect**

- The request is `Pending`; after approval, the condition is `Approved,Issued` and the certificate's
  subject is `CN=alice, O=dev-team` (formatting varies), **issued by the cluster CA**, valid for one hour.
- With alice's kubeconfig, `auth whoami` shows **Username `alice`**, groups **`dev-team`** and
  `system:authenticated`.
- `get pods` is **Forbidden**: *User "alice" cannot list resource "pods"...*. The API server knows
  exactly who she is and she is **authenticated**, but she has **no permissions** yet. (Session 20.)

**Why**

You generated the private key on *your* side; it never left your machine. The cluster only signed the
public half. The certificate carries the identity, and anyone holding both certificate and key *is* alice.
A certificate **cannot be revoked** (there is no revocation list): it is valid until it expires, which is why
the request asked for just an hour. In real life you would rather connect an external identity provider
(OIDC), keeping users in one place and issuing short-lived tokens.

**Try**

    kubectl --kubeconfig alice.kubeconfig config view --minify     # alice's three-part kubeconfig
    kubectl create rolebinding tmp --clusterrole=view --user=alice -n lab
    kubectl --kubeconfig alice.kubeconfig get pods                 # now allowed
    kubectl delete rolebinding tmp -n lab

---

## Step 68: 401 versus 403

**Do**

    curl -s --cacert ca.crt $SERVER/version
    curl -s --cacert ca.crt $SERVER/api/v1/namespaces/lab/pods
    curl -s --cacert ca.crt -H "Authorization: Bearer garbage" $SERVER/api/v1/namespaces/lab/pods
    kubectl --kubeconfig alice.kubeconfig get pods -v=6 2>&1 | tail -3
    openssl req -x509 -newkey rsa:2048 -nodes -keyout mallory.key -out mallory.crt -subj "/CN=mallory/O=dev-team" -days 1
    kubectl config set-cluster lab-a --server=$SERVER --certificate-authority=ca.crt --embed-certs=true --kubeconfig=mallory.kubeconfig
    kubectl config set-credentials mallory --client-certificate=mallory.crt --client-key=mallory.key --embed-certs=true --kubeconfig=mallory.kubeconfig
    kubectl config set-context mallory@lab-a --cluster=lab-a --user=mallory --kubeconfig=mallory.kubeconfig
    kubectl config use-context mallory@lab-a --kubeconfig=mallory.kubeconfig
    kubectl --kubeconfig mallory.kubeconfig get pods

**Expect**

- `/version` returns JSON to **anyone**: a few "public info" paths are open to unauthenticated clients.
- The anonymous request for pods returns a Status with **`"code": 403`** and a message like *User
  "system:anonymous" cannot list resource "pods"*. No credentials at all is not an error: it is a valid
  identity, `system:anonymous` (anonymous requests are enabled by default), which simply has no rights.
- The request with a **wrong token** returns **`"code": 401`, `"reason": "Unauthorized"`**.
- alice's verbose output ends with `403 Forbidden`.
- mallory has a certificate, but one **signed by herself**, not by the cluster CA: kubectl fails with
  **`error: You must be logged in to the server (Unauthorized)`**.

**Why**

| Code | Question that failed | Meaning |
|---|---|---|
| **401 Unauthorized** | *authentication*: who are you? | a credential was presented but isn't valid (bad token, certificate from an unknown CA, expired) |
| **403 Forbidden** | *authorization*: may you do this? | the identity is known (even `system:anonymous`) but no rule allows the action |

When debugging, this tells you which half to look at: 401 -> your credentials; 403 -> RBAC (Session 20).

---

## Step 69: Identities for programs: ServiceAccounts

**Do: the account and a token**

    kubectl apply -f sa.yml
    kubectl wait --for=condition=Ready pod/api-client pod/no-token --timeout=90s
    kubectl get serviceaccount
    kubectl get secrets
    jwtdecode() { cut -d. -f2 | python3 -c "import sys,base64,json;s=sys.stdin.read().strip();print(json.dumps(json.loads(base64.urlsafe_b64decode(s+'='*(-len(s)%4))),indent=2))"; }
    TOKEN=$(kubectl create token app-sa --duration=10m)
    echo $TOKEN | jwtdecode
    kubectl config set-cluster lab-a --server=$SERVER --certificate-authority=ca.crt --embed-certs=true --kubeconfig=sa.kubeconfig
    kubectl config set-credentials app-sa --token=$TOKEN --kubeconfig=sa.kubeconfig
    kubectl config set-context app-sa@lab-a --cluster=lab-a --user=app-sa --namespace=lab --kubeconfig=sa.kubeconfig
    kubectl config use-context app-sa@lab-a --kubeconfig=sa.kubeconfig
    kubectl --kubeconfig sa.kubeconfig auth whoami
    kubectl --kubeconfig sa.kubeconfig get pods

**Do: the token inside a pod**

    kubectl exec api-client -- ls /var/run/secrets/kubernetes.io/serviceaccount
    kubectl exec api-client -- cat /var/run/secrets/kubernetes.io/serviceaccount/token | jwtdecode
    kubectl exec api-client -- sh -c 'curl -s --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/api/v1/namespaces/lab/pods'
    kubectl exec no-token -- ls /var/run/secrets/kubernetes.io/serviceaccount

**Expect**

- `get serviceaccount` lists `app-sa` and `default` (every namespace has a `default` one, used by pods that name none).
  `get secrets` is **empty**: modern clusters don't create long-lived token Secrets.
- The decoded token is a **JWT**: `sub` is **`system:serviceaccount:lab:app-sa`**, there is an expiry
  (`exp`, 10 minutes after `iat`), an audience, and a `kubernetes.io` section naming the ServiceAccount.
- With that token, `auth whoami` shows **Username `system:serviceaccount:lab:app-sa`** and groups
  `system:serviceaccounts`, `system:serviceaccounts:lab`, `system:authenticated`. `get pods` is **Forbidden**:
  authenticated, but no permissions.
- Inside `api-client` there are three files: **`ca.crt`, `namespace` and `token`**. The token is also a JWT, **bound to
  that pod** (its claims include the pod name) and valid for about an hour, renewed automatically by the kubelet.
- The in-pod API call answers with a JSON **403** naming `system:serviceaccount:lab:app-sa`:
  **a pod talks to the API as its ServiceAccount, and by default can do nothing.**
- `no-token` has no such directory: it opted out of the token.

**Why**

Every pod runs as *some* ServiceAccount, and a token for it is mounted automatically. Two practical rules:
**give each workload its own ServiceAccount with only the permissions it needs**, and for pods that never call
the Kubernetes API, set `automountServiceAccountToken: false`, so a compromised container can't use a token.
Tokens are **short-lived and bound** (to a pod, with an audience and expiry), unlike the old permanent Secrets.

**Try**

    kubectl auth whoami -o json --as system:serviceaccount:lab:app-sa
    kubectl describe serviceaccount default
    kubectl create token app-sa --duration=10m --bound-object-kind=Pod --bound-object-name=api-client | jwtdecode | head -20

---

## Cleanup for this session

    kubectl delete -f sa.yml
    kubectl delete csr alice
    rm -f alice.key alice.csr alice.crt csr.generated.yml mallory.key mallory.crt *.kubeconfig ca.crt
    unset TOKEN SERVER
    kubectl get pods,sa,csr

`alice.key`, `mallory.key` and the kubeconfig files are credentials: delete them, and never commit such files.

---

## End of Session 19: checkpoint

    kubectl get pods,csr                  # No resources found
    ls *.key *.crt *.kubeconfig 2>/dev/null   # nothing

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `kubectl get csr` shows nothing / apply fails | You are on the wrong cluster, or `csr.generated.yml` still contains `__CSR__` (re-run the `sed`) |
| CSR `Pending` for ever | It needs approving: `kubectl certificate approve alice` |
| `get csr alice -o jsonpath=...` is empty | Not approved yet, or already expired/garbage-collected |
| `certificate-authority-data` is empty | Your current context isn't a kind cluster (some tools use a file path): switch to `kind-lab-a` |
| alice gets `Unauthorized` (401) instead of `Forbidden` | The certificate expired, or the request used a different CA's certificate. Re-create it |
| `python3: command not found` | Skip the decode steps, or decode the middle part of the token with another tool |
| `kubectl create token` unknown | Needs kubectl 1.24 or later |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)
    kubectl auth can-i <verb> <resource> --as <user> -n <ns>     what a subject may do
    kubectl auth whoami              who the API server thinks you are

## Done when you can, without looking

- [ ] explain that Kubernetes has no user objects, and where a user's name and groups come from in a certificate
- [ ] say why your admin access is just an RBAC binding, and what `system:masters` is
- [ ] go from a private key to a working kubeconfig for a new user (CSR, approve, certificate)
- [ ] tell 401 from 403 and say which part of the chain failed
- [ ] explain what a ServiceAccount token is and how a pod uses it
- [ ] say when a pod should have no token at all

## Stretch ideas

- Read how to plug in an OIDC identity provider (`--oidc-issuer-url`), and why it beats certificates for people.
- Shorten `expirationSeconds` to 600 and watch the certificate stop working after it expires.
- Try a `TokenRequest` with a custom `--audience` and read how audiences limit where a token is accepted.
- Look at `kubectl get --raw /apis/authentication.k8s.io/v1` to see the authentication API itself.
