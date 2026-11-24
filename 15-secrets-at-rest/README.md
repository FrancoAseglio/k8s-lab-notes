# Secrets at rest - what etcd really stores 

**Work from this directory:** `cd 15-secrets-at-rest`  

**Goal:** look at a Secret *inside etcd*, the cluster's database, and see that by default it is
stored as plain bytes. Then enable **encryption at rest** on the API server, watch the stored form
change, and learn the migration procedure for a cluster that already holds Secrets.

**Starting state:** no cluster named `secrets-a`. This session builds its **own single-node
cluster** (so it doesn't touch `lab-a`) and needs nothing from the other sessions.
**Ending state:** the cluster and the key material deleted.

**What encryption at rest is, and isn't**

- It protects the data *on disk*: an etcd snapshot, a stolen backup, a disk image.
- It does **not** protect against anyone who can read the Secret through the API (RBAC does that),
  nor against someone controlling the control-plane node (the key sits on it, unless you use a KMS).

Read the top of every YAML file in this directory before you start.

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

Docker must be running. The commands also use `sed`, `head`, `base64` and `strings` (all present on
macOS and Linux). Run everything from this directory, and **avoid spaces in its path**.

---

## Starting checks

    kind get clusters                     # no `secrets-a`
    ls enc 2>/dev/null                    # no `enc` directory

### Reset (if a previous attempt is in the way)

    kind delete cluster --name secrets-a
    rm -rf enc kind-encrypted.generated.yml

---

## Step 52: A cluster with the plumbing in place, encryption still off

**Do**

    mkdir enc
    KEY=$(head -c 32 /dev/urandom | base64)
    sed "s|__KEY__|$KEY|" encryption-identity-first.yml > enc/encryption-config.yaml
    sed "s|__ENC_DIR__|$PWD/enc|" kind-encrypted.yml > kind-encrypted.generated.yml
    kind create cluster --name secrets-a --config kind-encrypted.generated.yml
    kubectl config current-context
    kubectl get nodes
    kubectl -n kube-system get pod kube-apiserver-secrets-a-control-plane -o yaml | grep -i encryption
    kubectl apply -f namespace.yml
    kubectl config set-context --current --namespace=lab

Define a helper for talking to etcd (it lives in this terminal only; paste it again in a new one):

    etcdq() { kubectl -n kube-system exec etcd-secrets-a-control-plane -- etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key "$@"; }

Create a Secret and read it **straight from etcd**:

    kubectl create secret generic db-creds --from-literal=password='S3cr3t-plain'
    etcdq get /registry/secrets/lab/db-creds --print-value-only | strings

**Expect**

- One node `secrets-a-control-plane`, `Ready`. The API server pod's arguments contain
  `--encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml`: the machinery is wired.
- The `strings` output shows readable fragments such as `Secret`, `db-creds`, `password` and,
  in the clear, **`S3cr3t-plain`**.

**Why**

Everything the API server stores lives in etcd under keys like `/registry/<resource>/<namespace>/<name>`.
Here the config lists `identity` (= no encryption) **first**, so Secrets are written as they are.
Notice also that the value is **not even base64** in etcd: base64 only exists in the JSON/YAML the
API shows you. In etcd it is the raw bytes. Anyone with access to etcd (or a copy of its disk)
reads every Secret in the cluster.

**Try**

    etcdq get /registry/secrets/lab/db-creds | head -3       # the key line, then the value
    etcdq get /registry/secrets --prefix --keys-only         # every Secret in the cluster

---

## Step 53: Turn encryption on

**Do**

    sed "s|__KEY__|$KEY|" encryption-aescbc-first.yml > enc/encryption-config.yaml
    docker exec secrets-a-control-plane sh -c 'mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/ && sleep 25 && mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/'
    kubectl get nodes                                        # retry until it answers (up to about a minute)
    kubectl create secret generic db-creds-new --from-literal=password='S3cr3t-new'
    etcdq get /registry/secrets/lab/db-creds-new --print-value-only | head -c 60 | cat -v; echo
    etcdq get /registry/secrets/lab/db-creds --print-value-only | strings | grep S3cr3t
    kubectl get secret db-creds-new -o jsonpath='{.data.password}' | base64 --decode; echo
    kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 --decode; echo

**Expect**

- The first command only swaps which provider is first (**aescbc** now, `identity` last). The API
  server reads this file **at startup**, so the second command makes the kubelet stop and
  recreate its pod: for about a minute `kubectl` can't connect.
- The new Secret in etcd starts with **`k8s:enc:aescbc:v1:key1:`** followed by unreadable bytes
  (that prefix names the provider and the key).
- The *old* Secret is **still plain**: `S3cr3t-plain` shows up. Encryption applies to *new writes*.
- Through the API both Secrets decode normally: encryption is **transparent** to users and pods.

**Why**

The API server tries providers in order: the first one **encrypts**; reading tries each until one
can decode. `identity` stays last so existing plain data remains readable. If you removed it now,
the old Secret would become unreadable.

**If the API server doesn't come back**

    docker exec secrets-a-control-plane ls /etc/kubernetes/manifests /tmp
    docker exec secrets-a-control-plane sh -c 'crictl ps -a | grep apiserver'

If `kube-apiserver.yaml` is still in `/tmp`, move it back into `/etc/kubernetes/manifests/`. If the
container keeps exiting, `docker exec secrets-a-control-plane crictl logs <container-id>` shows why
(typically a typo in `enc/encryption-config.yaml`).

---

## Step 54: Rewrite the existing Secrets

**Do**

    kubectl get secrets -A -o json | kubectl replace -f -
    etcdq get /registry/secrets/lab/db-creds --print-value-only | head -c 60 | cat -v; echo
    etcdq get /registry/secrets/lab/db-creds --print-value-only | strings | grep -c S3cr3t
    kubectl get secret db-creds -o jsonpath='{.data.password}' | base64 --decode; echo
    etcdq get /registry/secrets --prefix --keys-only

**Expect**

- `replace` rewrites every Secret in every namespace (the API server re-saves each one, now
  encrypted). It prints `replaced` for each.
- The old `db-creds` now begins with `k8s:enc:aescbc:v1:key1:` too; the count of lines
  containing `S3cr3t` is `0`.
- `kubectl get secret` still returns the same decoded value: **users see no difference.**

**Why: the adoption procedure**

1. Configure the encrypting provider **first** and `identity` **last**; restart the API server.
2. **Rewrite** every Secret (`kubectl get secrets -A -o json | kubectl replace -f -`).
3. Verify in etcd; once nothing is plain any more, optionally remove `identity`.

Key rotation follows the same pattern: add the new key first, rewrite everything, then retire the old
key.

**What this did and did not do**

| Threat | Protected by encryption at rest? |
|---|---|
| Stolen etcd snapshot or disk | **yes** (without the key) |
| A user with `get secrets` RBAC | no, the API decrypts for them |
| Someone with access to the control-plane node | no, the key file is on it |
| A pod that mounts the Secret | no, it receives the decrypted value |

Real clusters keep the key outside the node with a **KMS provider** (a cloud key service or Vault) and
combine it with tight RBAC and audit logging.

**Try**

    etcdq get /registry/configmaps/lab/kube-root-ca.crt --print-value-only | strings | head -3     # ConfigMaps are NOT covered by this config

Adding `configmaps` to the `resources:` list in the EncryptionConfiguration would cover them too.

---

## Cleanup for this session

    kind delete cluster --name secrets-a
    rm -rf enc kind-encrypted.generated.yml
    unset KEY
    kind get clusters
    kubectl config get-contexts

`enc/encryption-config.yaml` held the encryption key: it is deleted. If no context is current
afterwards, pick one: `kubectl config use-context kind-lab-a` (if it exists) or `docker-desktop`.

---

## End of Session 3: checkpoint

    kind get clusters                     # no `secrets-a`
    ls enc 2>/dev/null                    # nothing

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `kind create cluster` fails | Absolute path in `kind-encrypted.generated.yml` (`__ENC_DIR__` replaced?), no spaces in the path, `enc/encryption-config.yaml` exists, Docker has access to that folder |
| API server never appears after Step 52 | Typo in `enc/encryption-config.yaml`? `docker exec secrets-a-control-plane crictl ps -a` and `crictl logs` |
| `grep encryption` prints nothing | The kubeadm patch wasn't applied: the cluster was created from the wrong file, or the Kubernetes version's kubeadm format differs from both patches |
| `etcdctl: executable file not found` | The etcd image may not ship `etcdctl`; check `kubectl -n kube-system get pod etcd-secrets-a-control-plane -o yaml` for the image and read the Kubernetes docs "Encrypting Confidential Data at Rest" for alternatives |
| `etcdq` says command not found | The function exists only in the terminal where you defined it: paste it again |
| `strings: command not found` | Use `... | cat -v`, or `| hexdump -C | head` |
| Old Secret unreadable after Step 53 | `identity` provider missing from the config, or a different key than before. Restore the previous config and key |
| `kubectl` hangs for a minute after the swap | Expected while the API server restarts |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] read a Secret from etcd, and say what the stored form looks like by default
- [ ] explain why base64 appears in the API but not in etcd
- [ ] explain provider order (first encrypts, all can decrypt) and why `identity` goes last
- [ ] describe the three steps to adopt encryption on a live cluster
- [ ] list what encryption at rest protects against, and what it doesn't
- [ ] explain why a KMS provider is preferable to a key file

## Stretch ideas

- Switch to `secretbox` or `aesgcm` providers (note the rotation caveats for `aesgcm` in the Kubernetes docs) and compare the etcd prefix.
- Add a second key (`key2`) first in the list, restart, rewrite all Secrets, then retire `key1`: a full key rotation.
- Add `configmaps` to the encrypted resources and verify with `etcdq`.
- Read about KMS v2 providers, and about audit logging of Secret reads.
