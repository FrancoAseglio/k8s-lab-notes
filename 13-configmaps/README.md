# ConfigMaps - create, inject, update 

**Work from this directory:** `cd 13-configmaps` 

**Goal:** create ConfigMaps in every way `kubectl` offers, feed them to a pod as
environment variables and as files, find out exactly what updates when you change
one (and what doesn't), and make one immutable.

**Starting state:** kind cluster `lab-a` (3 nodes), kubectl context `kind-lab-a`
with default namespace `lab`, and nothing of your own in it.  
**Ending state:** everything from this session deleted.

**Why ConfigMaps exist.** An image should be the same in every environment. Settings
that change per environment (log level, URLs, feature flags, whole config files) live
in a ConfigMap, stored in the cluster and injected into pods when they start. For
*confidential* values use a Secret (Session 2).

Read the top of `configmap.yml` and the pod files as you use them.

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

---

## Starting checks

    kubectl config current-context        # kind-lab-a
    kubectl get nodes                     # 3 nodes, all Ready
    kubectl config get-contexts           # NAMESPACE lab on kind-lab-a
    kubectl get configmap                 # only kube-root-ca.crt
    kubectl get pods                      # No resources found

All match: go straight to Step 44.

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

## Step 44: Four ways to create a ConfigMap

**Do**

    kubectl create configmap from-literals --from-literal=LOG_LEVEL=info --from-literal=APP_MODE=demo
    kubectl create configmap from-file --from-file=app.properties
    kubectl create configmap from-file-key --from-file=settings=app.properties
    kubectl create configmap from-env --from-env-file=app.env
    kubectl apply -f configmap.yml
    kubectl get configmap
    kubectl describe configmap from-file
    kubectl get configmap from-env -o yaml
    kubectl get configmap from-file-key -o jsonpath='{.data}{"\n"}'
    kubectl create configmap sample --from-literal=a=b --dry-run=client -o yaml

**Expect**

- `get configmap` lists your five ConfigMaps (plus an automatic `kube-root-ca.crt` that
  exists in every namespace) with a DATA column counting keys:
  `from-literals` 2, `from-file` 1, `from-file-key` 1, `from-env` 3, `app-config` 4.
- `from-file`: **one key named `app.properties`** whose value is the entire file.
- `from-file-key`: the same content under the key you chose, **`settings`**.
- `from-env`: **one key per line** of `app.env` (`LOG_LEVEL`, `APP_MODE`, `GREETING`).
- In YAML everything sits under `data:` as strings.
- `--dry-run=client -o yaml` prints the YAML *without creating anything*: the quick way to
  write a manifest.

**Why**

| Flag | One key per... | Key name |
|---|---|---|
| `--from-literal=k=v` | each flag | `k` |
| `--from-file=path` | file | the file name |
| `--from-file=name=path` | file | `name` |
| `--from-env-file=path` | **line** of the file | the left side of each `KEY=value` |

A ConfigMap is namespaced, can hold up to 1 MiB, and stores strings (use `binaryData` for
binary content).

**Try**

    kubectl create configmap from-dir --from-file=.      # every file in this directory becomes a key
    kubectl describe configmap from-dir | head -20
    kubectl delete configmap from-dir

---

## Step 45: ConfigMap values as environment variables

**Do**

    kubectl apply -f pod-env.yml
    kubectl wait --for=condition=Ready pod/env-demo --timeout=90s
    kubectl logs env-demo
    kubectl exec env-demo -- env | sort | grep -E 'LOG_LEVEL|MODE|CFG_'
    kubectl apply -f pod-env-missing.yml
    kubectl get pods
    kubectl describe pod env-missing | tail -6
    kubectl logs env-optional

**Expect**

- The log shows two lines with `level=info mode=demo`: one expanded by the **kubelet**
  (`$(LOG_LEVEL)`), one by the **shell** (`$LOG_LEVEL`), then three imported variables:
  `CFG_APP_MODE`, `CFG_GREETING`, `CFG_LOG_LEVEL`.
- `MODE` is the key `APP_MODE` **renamed**; the `CFG_` ones came from `envFrom` with a prefix.
- `env-missing` is stuck in **`CreateContainerConfigError`**; its events say it couldn't find
  the key `NOPE` in the ConfigMap. `env-optional` runs and prints `NOPE=[]`.

**Why**

`env`+`configMapKeyRef` picks and renames single keys; `envFrom` imports a whole ConfigMap
(add a `prefix` to avoid clashes). Environment variables are read **once, when the
container starts**: they never change while the container runs. Kubernetes only replaces `$(NAME)` when NAME is a defined variable: any other `$(...)`, such as the
shell's `$(date)`, is left alone for the shell. A missing *required*
reference blocks the pod, which is usually what you want: better to fail visibly than to run
with a half-configured app.

**Try**

Copy `pod-env.yml` and also add `envFrom` with `configMapRef: {name: app-config}`: the keys
`greeting.txt` and `app.properties` are **not valid variable names**, so they are skipped. The
pod still starts, and `kubectl describe pod` shows a warning event about invalid names.

---

## Step 46: ConfigMap data as files

**Do**

    kubectl apply -f pod-files.yml
    kubectl wait --for=condition=Ready pod/files-demo --timeout=90s
    kubectl exec files-demo -- ls -la /etc/config
    kubectl exec files-demo -- cat /etc/config/greeting.txt
    kubectl exec files-demo -- cat /etc/config/app.properties
    kubectl exec files-demo -- ls -lLR /etc/subset
    kubectl exec files-demo -- ls -l /etc/single
    kubectl exec files-demo -- touch /etc/config/new.txt

**Expect**

- `/etc/config` has **one file per key** (4). The entries are symbolic links through a hidden
  `..data` link: that indirection is how Kubernetes swaps the whole set atomically on update.
- `/etc/subset/conf/settings.properties` is the only file there (only the selected key, at the
  renamed path), with mode `-r--r--r--` (0444).
- `/etc/single` contains just `greeting.txt` (the `subPath` mount): other files in that
  directory would not be hidden.
- Writing into `/etc/config` fails: *read-only file system*.

**Why**

Mounting is the natural way to deliver **whole config files** (nginx.conf, application.yml):
the app reads them like any other file. `items` picks and renames keys; `defaultMode` sets
permissions; `subPath` places one file exactly, without covering the rest of a directory (at a
price you'll see next).

---

## Step 47: What updates, and what doesn't

**Do: terminal 1**

    kubectl apply -f pod-update.yml
    kubectl wait --for=condition=Ready pod/watcher --timeout=90s
    kubectl logs -f watcher                 # leave running

**Do: terminal 2**

    kubectl patch configmap app-config -p '{"data":{"LOG_LEVEL":"debug"}}'
    # watch terminal 1 for up to ~2 minutes, then Ctrl+C there
    kubectl delete pod watcher
    kubectl apply -f pod-update.yml
    kubectl wait --for=condition=Ready pod/watcher --timeout=90s
    kubectl logs watcher --tail=2

**Do: immutability**

    kubectl apply -f configmap-immutable.yml
    kubectl patch configmap app-config-immutable -p '{"data":{"LOG_LEVEL":"debug"}}'
    kubectl get configmap app-config-immutable -o jsonpath='{.immutable}{"\n"}'

**Expect**

- Right after the patch the log still says `env=info volume=info subpath=info`. After a
  delay (typically up to about a minute, sometimes two) it becomes
  **`env=info volume=debug subpath=info`**: only the **volume** followed.
- After the pod is re-created, all three say `debug`: a restart re-reads everything.
- The patch of the immutable ConfigMap is **rejected** (a message like *field is immutable when
  `immutable` is set*), and `.immutable` is `true`.

**Why**

| Consumed as | After the ConfigMap changes |
|---|---|
| environment variable | **unchanged** until the container restarts |
| file in a mounted volume | **updated** by the kubelet after a delay (the app must notice and reload it) |
| file mounted with `subPath` | **never updated** (restart needed) |

The practical consequence: a ConfigMap edit does **not** roll out by itself. Common patterns are
`kubectl rollout restart deployment/<name>`, or an **immutable, versioned** ConfigMap
(`app-config-v2`) referenced from the pod template, so changing the name changes the template and
triggers an ordinary rolling update (with an obvious rollback).

**Try**

    kubectl create configmap app-config-v2 --from-literal=LOG_LEVEL=debug
    kubectl get configmap

(In a Deployment, you would now edit the pod template to reference `app-config-v2`.)

---

## Cleanup for this session

    kubectl delete pod --all
    kubectl delete configmap from-literals from-file from-file-key from-env app-config app-config-immutable app-config-v2 --ignore-not-found
    kubectl get pods,configmap                    # only kube-root-ca.crt left

---

## End of Session 1: checkpoint

    kubectl get pods                      # No resources found
    kubectl get configmap                 # kube-root-ca.crt only

Session 1 is done. Session 2 (`../14-secrets/`) can continue from here, or start from
scratch: it has its own bootstrap.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Pod `CreateContainerConfigError` | `kubectl describe pod`: the ConfigMap or key doesn't exist (typo? wrong namespace?). Create it or fix the reference |
| `configmap "app-config" not found` | Apply `configmap.yml` (Step 44); `env-demo` also needs `from-env` |
| Variable missing in the container | Key name not a valid variable name (dots, dashes), so `envFrom` skipped it: `kubectl describe pod` events |
| Mounted file didn't change | Wait up to ~2 minutes; remember `subPath` mounts never update; env vars need a restart |
| `error validating ... must be a string` | In YAML values must be strings: quote `"8080"`, `"true"` |
| `kubectl patch` says the ConfigMap is immutable | Create a new one (new name) instead |
| `no such file` when applying | Wrong directory: `cd` into this session's directory |

## Handy flags

    kubectl get <thing> -o wide      more columns (node, IP)
    kubectl get <thing> -w           watch live (Ctrl+C to stop)
    kubectl get <thing> -o yaml      the full object as stored in the cluster
    kubectl get <thing> -n <ns>      look in one namespace
    kubectl get <thing> -A           look in all namespaces
    kubectl describe <thing> <name>  details + Events (debug here first)

## Done when you can, without looking

- [ ] create a ConfigMap from literals, a file, an env-file and YAML, and say what each gives you
- [ ] inject one key, renamed, and a whole ConfigMap (with prefix) as environment variables
- [ ] mount a ConfigMap as files, select and rename keys, and set permissions
- [ ] state what updates live and what doesn't (env, volume, subPath)
- [ ] explain why immutable, versioned ConfigMaps make rollouts and rollbacks cleaner
- [ ] say when a ConfigMap is the wrong tool (anything confidential)

## Stretch ideas

- Run nginx and replace its `default.conf` with a ConfigMap mounted via `subPath`.
- `kubectl kustomize` has a `configMapGenerator` that appends a content hash to the name: try it, and connect it to the immutable pattern.
- Use a ConfigMap as a *projected* volume together with a Secret (`kubectl explain pod.spec.volumes.projected`).
- Find out what happens to a running pod when its ConfigMap is deleted.
