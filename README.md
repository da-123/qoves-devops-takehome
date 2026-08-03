# QOVES take-home — a small API run properly on self-managed Kubernetes

A trivial Flask API and a Postgres, delivered by GitOps onto a two-node minikube
cluster with Calico: default-deny networking, credentials that live in Vault
rather than in git, an HPA, and a Prometheus alert.

**The reasoning is in [`docs/WRITEUP.md`](docs/WRITEUP.md)** — decisions and the
alternatives I rejected, what minikube did for me, production gaps, and a
runbook. Read that first; this file is only how to run it.

## Prerequisites

`docker` (daemon running, VM given ~6 CPUs / 8 GB), `minikube`, `kubectl`,
`helm`, `git`, `jq`, `curl`. No `kubeseal` and no `vault` CLI — Vault is
configured by exec'ing into its own pod.

> **Check your kubectl context before anything else.** These commands install
> ArgoCD and then hand it the whole cluster. Pointing them at a real cluster by
> accident is not a recoverable mistake:
>
> ```bash
> kubectl config current-context   # must be `qoves` once the cluster is up
> ```

## Stand it up

Full step-by-step (including the Vault bootstrap) is in
[section 1 of the writeup](docs/WRITEUP.md#1-run-it). The short version:

```bash
minikube start -p qoves --nodes 2 --cni calico \
  --kubernetes-version v1.30.3 --cpus 2 --memory 3072
kubectl config use-context qoves
minikube -p qoves addons enable ingress
minikube -p qoves addons enable metrics-server

docker build -t qoves-api:0.1.0 ./app
minikube -p qoves image load qoves-api:0.1.0

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.3/manifests/install.yaml
kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=300s

kubectl apply -f gitops/bootstrap/project.yaml
kubectl apply -f gitops/bootstrap/root-app.yaml     # the last imperative command
```

Then write the database password into Vault (writeup §1, step 6), start
`minikube -p qoves tunnel` in its own terminal, point `qoves.local` at
`127.0.0.1` in `/etc/hosts`, and:

```bash
curl http://qoves.local/healthz     # -> ok
```

## How it fits together

| Layer      | Choice                                        | Why (short)                                                       |
| ---------- | --------------------------------------------- | ----------------------------------------------------------------- |
| Cluster    | minikube, 2 nodes, pinned Kubernetes          | Two nodes make scheduling and spread real                         |
| CNI        | Calico                                        | The default CNI accepts NetworkPolicy and enforces nothing        |
| Delivery   | ArgoCD app-of-apps                            | One root Application over a directory of children; waves order it |
| App        | Deployment + HPA behind ingress-nginx         | No cloud LB; the controller runs on the node                      |
| Database   | Postgres StatefulSet + PVC (ReadWriteOnce)    | Small enough to fully defend; CloudNativePG when it needs failover |
| Networking | Default-deny both ways, 4 explicit allows     | DNS, ingress→API, API→DB, Prometheus→API. Nothing else            |
| Secrets    | External Secrets Operator + Vault (k8s auth)  | Git holds a pointer, never a value — not even ciphertext          |
| Metrics    | Prometheus + one alert                        | Alerts on the one thing no automation can fix: the DB is gone     |

## Repo layout

```
app/                              the provided API (source + Dockerfile), unmodified
gitops/bootstrap/                 the only two files applied by hand
gitops/apps/                      child Applications; numeric prefix = ArgoCD sync wave
gitops/manifests/namespaces/      namespaces, ResourceQuota, LimitRange
gitops/manifests/vault/           the secret store I run (dev mode)
gitops/manifests/secrets/         SecretStore + ExternalSecret — pointers only
gitops/manifests/postgres/        StatefulSet + PVC template, headless Service
gitops/manifests/api/             Deployment, Service, Ingress, HPA, PDB, image pin
gitops/manifests/network-policies/ default-deny + 4 explicit allows
gitops/manifests/monitoring/      Prometheus, RBAC, scrape config, one alert
docs/WRITEUP.md                   the reasoning
```

## Deploying a change

Edit the image pin in `gitops/manifests/api/kustomization.yaml`, commit, push.
ArgoCD reconciles; `git revert` rolls back. No `kubectl apply` or `kubectl edit`
on workloads — `selfHeal: true` reverts anyone who tries.

## Self-check against the brief

- [x] The whole stack is reconciled from git; only ArgoCD itself was installed by hand
- [x] The NetworkPolicies block real traffic, and the writeup shows the test
- [x] **No secret value in the repo at all** — git holds a Vault path and a role name
- [x] Images pinned to explicit tags, never `:latest`
- [x] `/healthz` returns 200 through the ingress; data survives a pod restart
- [x] Writeup covers all five required sections plus the Part F/G/H questions
