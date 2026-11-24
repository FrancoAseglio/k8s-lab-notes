# Secrets - create, consume, protect 

**Work from this directory:** `cd 14-secrets`  

**Goal:** create Secrets of several types, prove that base64 is not protection, consume them as
environment variables and as files and compare the exposure, then control who can read them
with RBAC, and see how updates and immutability behave.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing of your own in it.  
**Ending state:** everything from this session deleted, including the generated key files.

**Secrets versus ConfigMaps.** Same shape (named keys, namespaced, injected as env or files), but
for sensitive data. What makes them different is the *handling*: values are not shown by `describe`,
volumes are held in memory, RBAC has separate rules for them, and they can be encrypted at rest
(Session 3). What they do **not** do by default: encrypt anything. Keep that in mind throughout.

Read the top of `secret.yml` and `rbac.yml` as you use them.

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

Docker must be running. Nothing else is needed: no cloud, no extra tools.

Step 48 also uses `openssl` (present on macOS and most Linux systems) and `base64`.

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get secrets,pods              # No resources found

All match: go straight to Step 48.

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

## Step 48: Create Secrets, and decode them

**Do**

    kubectl create secret generic db-creds --from-literal=username=appuser --from-literal=password='S3cr3t!pass'
    kubectl get secret db-creds
    kubectl describe secret db-creds
    kubectl get secret db-creds -o yaml
    kubectl get secret db-creds -o jsonpath='{.data.password}{"\n"}'
    kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 --decode; echo
    echo -n 'S3cr3t!pass' | base64
    echo 'S3cr3t!pass' | base64
    kubectl apply -f secret.yml
    kubectl get secret api-key -o yaml

**Do: other types**

    openssl req -x509 -newkey rsa:2048 -nodes -keyout tls.key -out tls.crt -subj "/CN=demo.lab.test" -days 30
    kubectl create secret tls demo-tls --cert=tls.crt --key=tls.key
    kubectl create secret docker-registry regcred --docker-server=registry.example.com --docker-username=me --docker-password=pw --docker-email=me@example.com
    kubectl get secrets
    kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 --decode; echo
    kubectl create secret generic bad-tls --type=kubernetes.io/tls --from-literal=foo=bar

**Expect**

- `describe` shows only the **sizes** of the values (`password: 11 bytes`), never the values.
- `get -o yaml` shows the values **base64-encoded**, and the one-line `base64 --decode` turns
  `password` back into `S3cr3t!pass`. Nothing protected it.
- The two `base64` commands give **different results**: `echo` without `-n` adds a newline that
  gets encoded too, a classic source of "wrong password" bugs.
- `api-key`: `stringData.token` was encoded for you and the object shows only `data` (with both
  `token` and `legacy`): `stringData` is write-only.
- `get secrets` shows the **TYPE** column: `Opaque`, `kubernetes.io/tls`,
  `kubernetes.io/dockerconfigjson`. The decoded `regcred` is a JSON registry auth document.
- The last command is **rejected**: a `kubernetes.io/tls` Secret must have `tls.crt` and
  `tls.key`. Types come with validation.

**Why**

**Base64 is an encoding for transporting bytes, not a security measure.** Any user who may read
the Secret reads the password. What protects a Secret is *who may read it* (Step 50) and *where it
is stored* (Session 3).

**Try**

    kubectl get secret demo-tls -o jsonpath='{.data.tls\.crt}' | base64 --decode | openssl x509 -noout -subject -dates

---

## Step 49: Consume a Secret: environment variable or file?

**Do**

    kubectl apply -f pod-secret-env.yml -f pod-secret-vol.yml
    kubectl wait --for=condition=Ready pod/secret-env pod/secret-vol --timeout=90s
    kubectl exec secret-env -- env | grep DB_
    kubectl get pod secret-env -o yaml | grep -B1 -A4 secretKeyRef
    kubectl describe pod secret-env | grep -A3 'Environment'
    kubectl exec secret-env -- sh -c 'cat /proc/1/environ | tr "\0" "\n" | grep DB_'
    kubectl exec secret-vol -- ls -lL /etc/secret
    kubectl exec secret-vol -- cat /etc/secret/password; echo
    kubectl exec secret-vol -- sh -c 'mount | grep /etc/secret'
    kubectl exec secret-vol -- env | grep DB_ || echo "no DB_ variables"

**Expect**

- The environment-variable pod has `DB_USER` and `DB_PASS` in clear text, readable by any process
  (even straight from `/proc/1/environ`).
- The pod's YAML and `describe` show only a **reference** (`<set to the key 'password' in secret
  'db-creds'>`), never the value.
- The volume pod has two files, `username` and `password`, with mode **`-r--------`**
  (0400), mounted on a **`tmpfs`** (in memory, never written to the node's disk), and **no**
  `DB_` variables.

**Why**

| | Environment variable | Mounted file |
|---|---|---|
| Visible to every process and child | yes | only if it can read the file |
| Shows up in crash dumps, debug output, `exec env` | yes | no |
| Permissions | none | `defaultMode` per file |
| Stored on the node's disk | no | no (tmpfs) |
| Updated when the Secret changes | **no** (restart) | **yes** (except `subPath`) |

Environment variables are convenient (many apps expect them), files are safer. Whatever you
choose, never print the value in logs.

---

## Step 50: Who may read a Secret? RBAC

**Do**

    kubectl apply -f rbac.yml
    kubectl auth can-i get secret/db-creds --as=system:serviceaccount:lab:app-sa
    kubectl auth can-i get secret/api-key --as=system:serviceaccount:lab:app-sa
    kubectl auth can-i list secrets --as=system:serviceaccount:lab:app-sa
    kubectl auth can-i get secret/db-creds --as=system:serviceaccount:lab:other-sa
    kubectl get secret db-creds --as=system:serviceaccount:lab:app-sa -o jsonpath='{.data.username}{"\n"}'
    kubectl get secret api-key --as=system:serviceaccount:lab:app-sa
    kubectl get secrets --as=system:serviceaccount:lab:app-sa
    kubectl get secret db-creds --as=system:serviceaccount:lab:other-sa
    kubectl get role read-db-creds -o yaml

**Expect**

- `can-i`: `yes` for `app-sa` on `db-creds`; `no` for `api-key`, `no` for `list`, `no` for `other-sa`.
- `app-sa` reads `db-creds` (it prints the base64 of the user name) but gets **Forbidden** for
  `api-key` and for listing; `other-sa` gets **Forbidden** too, with a message like *User
  "system:serviceaccount:lab:other-sa" cannot get resource "secrets" in API group "" in the namespace "lab"*.

**Why**

Access control is the real protection. Remember:

- `get`, `list` **and** `watch` on Secrets all return the values. Granting `list secrets` is
  granting *read everything*.
- A wildcard rule (`resources: ["*"]`) includes Secrets.
- Anyone allowed to **create pods** in a namespace can mount any Secret of that namespace, so "can
  create pods" effectively means "can read Secrets".
- Give each workload its own ServiceAccount with the narrowest rule (`resourceNames`).

**Try**

    kubectl auth can-i --list --as=system:serviceaccount:lab:other-sa -n lab
    kubectl get rolebinding app-sa-reads-db-creds -o yaml

---

## Step 51: Updates and immutability

**Do**

    kubectl exec secret-vol -- cat /etc/secret/password; echo
    kubectl patch secret db-creds -p '{"stringData":{"password":"N3w-p4ssw0rd"}}'
    # wait up to ~2 minutes, then:
    kubectl exec secret-vol -- cat /etc/secret/password; echo
    kubectl exec secret-env -- env | grep DB_PASS
    kubectl apply -f secret-immutable.yml
    kubectl patch secret locked -p '{"stringData":{"k":"v2"}}'
    kubectl get secret locked -o jsonpath='{.immutable}{"\n"}'

**Expect**

- The **mounted file** shows the new password after a delay; the **environment variable** in the
  other pod still shows the old one until that pod restarts.
- The patch of the immutable Secret is **rejected** (*field is immutable when `immutable` is set*).

**Why**

The same rules as ConfigMaps: volumes follow, environment variables don't. To rotate a credential
reliably, create a **new Secret** (`db-creds-v2`) and roll the workload onto it; an immutable Secret
turns that into the only possible path.

---

## Cleanup for this session

    kubectl delete pod --all
    kubectl delete secret db-creds api-key demo-tls regcred locked --ignore-not-found
    kubectl delete -f rbac.yml
    rm -f tls.key tls.crt
    kubectl get secrets,pods,sa                  # only the default ServiceAccount left

`tls.key` is a private key: delete it, and never commit such files.

---

## End of Session 2: checkpoint

    kubectl get secrets,pods              # No resources found
    ls tls.*                              # no such file

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Pod `CreateContainerConfigError` | The Secret or key doesn't exist: `kubectl describe pod`. Did Step 48 create `db-creds`? |
| Pod `ContainerCreating` for long | `kubectl describe pod`: a mounted Secret is missing |
| Wrong password inside the app | Probably a trailing newline in the encoded value: use `echo -n`, or `stringData` |
| `base64: invalid option -- d` | Use `base64 --decode` (works on macOS and Linux) |
| `error: unknown flag --as` / Forbidden on your own commands | `--as` needs admin rights, which your kind user has; check `kubectl config current-context` |
| `kubectl auth can-i` answers `no` for what you granted | Namespace (`-n lab`), exact `resourceNames`, and the subject name in the RoleBinding |
| `openssl: command not found` | Install openssl, or skip the TLS part |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] create a Secret from literals, files and YAML (`data` vs `stringData`), and name three types
- [ ] decode a Secret in one command, and explain why that isn't a vulnerability of Kubernetes but of its use
- [ ] compare environment variables and mounted files for Secrets
- [ ] write a Role that allows `get` on one named Secret, and test it with `auth can-i --as`
- [ ] explain why `list secrets` and "can create pods" are both dangerous permissions
- [ ] rotate a Secret safely (new versioned Secret, rollout)

## Stretch ideas

- Use `imagePullSecrets` with `regcred` in a pod spec, and read what happens with a wrong one (needs a private image to test for real).
- Set `automountServiceAccountToken: false` on a ServiceAccount and read why.
- Try `kubectl create secret generic --from-file=ssh-privatekey=... --type=kubernetes.io/ssh-auth`.
- Read about external secret managers (External Secrets Operator, HashiCorp Vault) and why teams use them.
