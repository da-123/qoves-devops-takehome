# QOVES Senior DevOps take-home — writeup

A trivial HTTP API and a Postgres, run the way I would want to inherit them:
delivered by GitOps, default-deny at the network layer, credentials that never
enter git in any form, and a written answer for what to do at 3am.

Assumptions I made where the brief was deliberately open:

- **One environment.** No `overlays/dev|prod` split. Two nodes, one namespace for
  the app. A second environment would be a new directory of child Applications,
  not a rewrite; I would rather ship one honest environment than a folder
  structure implying environments I never ran.
- **HTTP, not HTTPS.** A self-signed cert on `qoves.local` proves nothing about
  how I would do TLS, so it is a documented gap instead.
- **Prometheus without Alertmanager.** The brief asks for one meaningful alert
  rule, not a paging pipeline. The rule is loaded and evaluated; where it would
  route is in the gaps.
- **Vault in dev mode.** A real store I run, deliberately ephemeral. Reasoning
  and consequences in ADR-3.

---

## 1. Run it

### From nothing to a working cluster

Everything below is one-time bootstrap. After step 5, the cluster is reconciled
from git and nothing is applied by hand again.

**Step 0 — check your context.** These commands install a controller that then
takes over the cluster. A laptop usually has a context pointing at something
real, so verify before and after:

```bash
kubectl config current-context
```

**Step 1 — the cluster.** Two nodes and a CNI that actually enforces policy:

```bash
minikube start -p qoves --nodes 2 --cni calico \
  --kubernetes-version v1.30.3 --cpus 2 --memory 3072
kubectl config use-context qoves
```

**Step 2 — the two things minikube gives me that bare metal would not:**

```bash
minikube -p qoves addons enable ingress          # no cloud load balancer here
minikube -p qoves addons enable metrics-server   # the HPA has nothing to read without it
kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller --timeout=180s
```

**Step 3 — build the image and load it onto both nodes.** `image load` rather
than a registry so the whole exercise runs offline; the pin lives in
`gitops/manifests/api/kustomization.yaml`:

```bash
docker build -t qoves-api:0.1.0 ./app
minikube -p qoves image load qoves-api:0.1.0
```

**Step 4 — install the GitOps controller by hand** (explicitly allowed, and the
only imperative install):

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.12.3/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-repo-server --timeout=300s
kubectl -n argocd rollout status statefulset/argocd-application-controller --timeout=300s
```

**Step 5 — hand the cluster to git.** Two files, and then no more `kubectl apply`:

```bash
kubectl apply -f gitops/bootstrap/project.yaml
kubectl apply -f gitops/bootstrap/root-app.yaml
kubectl -n argocd get applications -w
```

Expect Postgres and the API to sit in `CreateContainerConfigError` for a while:
their Secret does not exist yet, because I have not put the password in Vault.
That is the correct order — the workloads wait on the credential rather than
starting with a default one — and the kubelet retries until it appears.

**Step 6 — put the credential in Vault.** This is the out-of-band step every
secret system has at the bottom. The password is generated locally, written
straight into Vault, and never touches disk or git:

```bash
kubectl -n vault rollout status deploy/vault --timeout=180s
VAULT_POD=$(kubectl -n vault get pod -l app=vault -o jsonpath='{.items[0].metadata.name}')

# Dev mode prints a random root token to the log; it exists only for this bootstrap.
ROOT_TOKEN=$(kubectl -n vault logs "$VAULT_POD" | awk '/Root Token:/ {print $NF}' | tail -1)
PGPASS=$(LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom | head -c 32)

kubectl -n vault exec -i "$VAULT_POD" -- env VAULT_TOKEN="$ROOT_TOKEN" sh -s <<EOF
set -e
vault secrets enable -path=qoves kv-v2
vault kv put qoves/postgres username=qoves password='$PGPASS' database=qoves

vault auth enable kubernetes
vault write auth/kubernetes/config kubernetes_host=https://kubernetes.default.svc:443

vault policy write qoves-app - <<'POLICY'
path "qoves/data/postgres" {
  capabilities = ["read"]
}
POLICY

vault write auth/kubernetes/role/qoves-app \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=qoves-app \
  token_policies=qoves-app \
  ttl=1h
EOF

unset PGPASS ROOT_TOKEN
```

Read that policy carefully, because it is the whole security argument: one
ServiceAccount name, in one namespace, may **read** exactly one path. It cannot
write, cannot list, and cannot see any other secret.

Confirm the chain worked end to end:

```bash
kubectl -n qoves-app get externalsecret postgres-credentials    # STATUS: SecretSynced
kubectl -n qoves-app get secret postgres-credentials -o jsonpath='{.metadata.ownerReferences[0].kind}'
# -> ExternalSecret   (created by the operator, not by any committed manifest)
```

**Step 7 — reach it.** With the docker driver on macOS the node IP lives inside
Docker's VM network and is not routable from the host, so `minikube ip` is the
wrong answer and a tunnel is the right one. On Linux, or with a VM driver, use
`minikube -p qoves ip` directly instead of `127.0.0.1`:

```bash
minikube -p qoves tunnel        # separate terminal, needs sudo, leave it running
echo "127.0.0.1  qoves.local" | sudo tee -a /etc/hosts

curl http://qoves.local/          # hello from the QOVES take-home API
curl -i http://qoves.local/healthz    # 200 ok
curl -o /dev/null -w '%{http_code}\n' http://qoves.local/metrics   # 404 - not published
```

That tunnel is worth naming rather than hiding: it is the local stand-in for the
edge layer that MetalLB or a cloud load balancer provides in production
(section 3).

### Verifying it

```bash
# The GitOps tree, and what ArgoCD thinks of it
kubectl -n argocd get applications

# Everything, as the submission asks for it
kubectl get pods,svc,ingress,netpol -A
kubectl get pvc -n qoves-app
kubectl get hpa -n qoves-app

# No credential anywhere in git, including history
git log -p | grep -c 'postgresql://[^:]*:[^@]*@' || echo "no connection string ever committed"

# Prometheus
kubectl -n monitoring port-forward svc/prometheus 9090:9090
#   sum by (status) (rate(http_requests_total{path="/healthz"}[5m]))
#   curl -s localhost:9090/api/v1/rules | jq -r '.data.groups[].rules[].name'
```

**The NetworkPolicies are not decorative.** Three claims, tested from inside:

```bash
# Allowed: the API resolves DNS and reaches Postgres
kubectl exec -n qoves-app deploy/api -- python -c \
  "import socket; print(socket.gethostbyname('postgres.qoves-app.svc.cluster.local'))"
kubectl exec -n qoves-app deploy/api -- python -c \
  "import socket; socket.create_connection(('postgres',5432),3); print('5432 reachable')"

# Blocked: the API has no route to the internet (by IP, so this tests egress and not DNS)
kubectl exec -n qoves-app deploy/api -- python -c \
  "import socket; socket.create_connection(('1.1.1.1',443),4)" \
  && echo "LEAK" || echo "external egress blocked, as intended"

# Blocked: a pod in the same namespace without the app=api label gets nothing at all
kubectl run netpol-test -n qoves-app --rm -it --restart=Never --image=busybox:1.36.1 \
  --labels app=netpol-test \
  --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":65534,"seccompProfile":{"type":"RuntimeDefault"}},"containers":[{"name":"netpol-test","image":"busybox:1.36.1","stdin":true,"tty":true,"securityContext":{"allowPrivilegeEscalation":false,"capabilities":{"drop":["ALL"]}}}]}}' \
  -- sh -c 'nslookup postgres.qoves-app.svc.cluster.local; nc -z -w3 postgres 5432; echo exit=$?'
# DNS times out and 5432 is unreachable: the allow rules select app=api, and this pod is not it.
```

**Data survives a pod restart:**

```bash
kubectl exec -n qoves-app postgres-0 -- sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "CREATE TABLE IF NOT EXISTS t (id serial, note text); INSERT INTO t (note) VALUES (\$\$survived\$\$); SELECT count(*) FROM t;"'
kubectl delete pod postgres-0 -n qoves-app
kubectl wait --for=condition=Ready pod/postgres-0 -n qoves-app --timeout=180s
kubectl exec -n qoves-app postgres-0 -- sh -c \
  'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT count(*) FROM t;"'
```

### Repo layout

```
app/                                the provided API (source + Dockerfile), unmodified
gitops/
  bootstrap/                        applied by hand, once
    project.yaml                    AppProject: which repos, which namespaces, which cluster-scoped kinds
    root-app.yaml                   the app-of-apps root -> gitops/apps/
  apps/                             one file per component; numeric prefix == sync wave
    00-namespaces.yaml              namespaces, quota, limits
    10-network-policies.yaml        default-deny lands BEFORE any workload
    12-vault.yaml                   the secret store
    15-external-secrets.yaml        the operator that reads it
    20-secrets.yaml                 SecretStore + ExternalSecret (pointers, no values)
    30-postgres.yaml                StatefulSet + PVC
    40-api.yaml                     Deployment, Service, Ingress, HPA
    50-monitoring.yaml              Prometheus + alert
  manifests/<component>/            plain YAML + a kustomization.yaml each
docs/WRITEUP.md                     this file
```

Two properties I care about there: adding a component is **one new file in
`gitops/apps/`**, and the numeric prefixes make the deploy order visible from
`ls` instead of buried in annotations.

### Making a change (the GitOps flow)

Ship a new API version:

1. `docker build`, then push or `minikube image load`.
2. Edit the tag/digest in `gitops/manifests/api/kustomization.yaml` — the single
   place the running version is pinned.
3. Commit, push, open a PR. The PR diff is the change record.
4. ArgoCD reconciles. To skip the poll interval:
   `kubectl -n argocd patch app root --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'`

Roll back with `git revert` and push. `selfHeal: true` means that if someone
does reach for `kubectl edit`, it is reverted within minutes — drift is
corrected, not merely detected.

Two places where I deliberately let the cluster win over git, both annotated in
the child Application that owns them:

- `spec/replicas` on the API Deployment, because the HPA owns it. Without that,
  ArgoCD and the HPA fight forever.
- `status` on the ExternalSecret, which the operator updates on every refresh.

---

## 2. Decisions (ADR-style)

**ADR-1 — CNI: Calico.** minikube's default CNI silently accepts NetworkPolicy
objects and enforces nothing, which is worse than having no policy: the audit
passes and the boundary does not exist. Calico enforces it, ships as a one-flag
option (`--cni=calico`), and is the reference implementation most people's mental
model is built on. Alternatives: **Cilium**, which I would pick for a real
cluster — eBPF datapath, L7 and DNS-aware policy, Hubble for flow visibility, and
identity-based policy that survives hostNetwork sources. I did not, because it
wants more of a two-node minikube's RAM and because its extra power
(CiliumNetworkPolicy) would tempt me away from portable
`networking.k8s.io/v1` policies a reviewer can read at a glance. **Flannel** was
out immediately: no policy support at all.

**ADR-2 — GitOps controller: ArgoCD.** Chosen for the app-of-apps pattern
specifically: a root Application over a directory of children means adding a
component is one commit, and the app tree shows a reviewer exactly what is
managed. Its sync waves also solve real ordering here — CRDs before custom
resources, the secret producer before its consumers — without me writing init
containers. Alternative: **Flux**, smaller footprint, no server-side UI, and
`dependsOn` is arguably a cleaner ordering primitive than wave integers. Either
is defensible; ArgoCD's visible tree is worth more on a team where not everyone
lives in the CLI.

**ADR-3 — Secrets: External Secrets Operator against a Vault I run, using
Kubernetes auth.** The requirement is that no usable credential is in git and
that the app gets it at runtime. I started with **Sealed Secrets** and moved off
it deliberately. Sealed Secrets satisfies the letter of the rule — the ciphertext
is asymmetric and only the in-cluster controller can decrypt it — but it still
means committing secret material, and it drags in three properties I dislike:
the sealing key becomes a single point of catastrophic loss (lose it and every
ciphertext in git is scrap), rotation is a commit rather than a backend
operation, and the encrypted blobs live in git history forever, so their security
rests entirely on a key never leaking, permanently.

External Secrets inverts that: Vault holds the value, and git holds a
`SecretStore` (an address, a mount, a role name) plus an `ExternalSecret` (which
path maps to which key). Both are boring in a PR and useless to an attacker.
Authentication is the part worth defending — I used Vault's **Kubernetes auth
method**, so there is no static token anywhere. The operator asks the API server
for a short-lived token for the `vault-auth` ServiceAccount, presents it to
Vault, and Vault verifies it with a TokenReview call before issuing a lease
scoped to `read` on exactly one path, bound to one SA name in one namespace. The
alternative — a Vault token in a bootstrap Secret — would just relocate the
problem to a credential I have to inject and rotate by hand.

Two side benefits that came out of the switch and were not the reason for it:
the `ExternalSecret` template **composes** `DATABASE_URL` from the parts stored
in Vault, so the password is stored once instead of twice (the Sealed Secrets
version had to store it standalone *and* inside the connection string, two things
to rotate in lockstep); and rotation no longer requires a commit — change it in
Vault and the operator picks it up on refresh.

The honest cost: this is more moving parts than sealing a file, and dev-mode
Vault is **ephemeral** — in-memory storage and auto-unseal, so a Vault restart
loses the stored password and step 6 has to run again. That is a real weakness of
this build, not of the pattern, and I chose it knowingly: the shape of the
production system (a store outside git, short-lived identity-based auth, a
read-only policy per workload) is identical, and swapping dev Vault for a real
HA Vault or AWS Secrets Manager changes the `SecretStore` and nothing else.
Neither Postgres nor the API knows Vault exists; both just read a `Secret` named
`postgres-credentials`. **SOPS with age** was the third option: better than
Sealed Secrets for multi-cluster (one key, many clusters) and it diffs
structurally, but it is still ciphertext in git and needs an ArgoCD plugin to
decrypt at render time.

**ADR-4 — Postgres: a raw StatefulSet, not an operator.** For one replica whose
job is to answer `SELECT 1`, a StatefulSet plus a `volumeClaimTemplate` is ~90
lines I can fully defend, and writing it myself forced me to state my own answers
about PGDATA subdirectories, fast shutdown, and `pg_isready` instead of
inheriting them. CloudNativePG is the better production answer and I would reach
for it the moment this needed **replication, failover, or backups**: synchronous
replicas, automated failover with fencing, PITR via WAL archiving to object
storage, in-place minor upgrades. The cost is a CRD, an operator to keep current,
and indirection between the manifest and the process. At this scale that buys
nothing; at the first "we cannot lose this data" conversation it buys everything.

**ADR-5 — Scaling signal: CPU, and it is the wrong one.** See the Part G answer.
Short version: CPU is what metrics-server can measure without a custom metrics
adapter, so it is what the HPA uses; the signal this API actually needs is
in-flight requests.

**ADR-6 — Probe split: liveness on `/`, readiness on `/healthz`.** The one design
choice here with real consequences, so it is deliberate. Liveness must never
depend on Postgres: if it did, a database blip would restart every API pod and a
dependency outage would become a crash loop that outlives the dependency's
recovery. Readiness *should* depend on Postgres, because every meaningful request
needs data — a pod that cannot reach the DB should leave the Service endpoints
rather than serve 503s. The trade-off I accept: with Postgres fully down, all API
pods go NotReady and the ingress returns 503 for everything, including `/`, which
would otherwise still work. I take that because this API has no DB-free
functionality worth preserving and because fail-closed is easier to reason about
mid-incident. A useful side effect: the kubelet probing `/healthz` every 10
seconds gives the alert a heartbeat, so the DB alert fires on real conditions with
zero user traffic. It is also why Prometheus discovers pods with `role: pod`
rather than `role: endpoints` — endpoint discovery drops NotReady pods, so it
would go blind exactly when the alert needs data.

**ADR-7 — Kustomize, no Helm chart of my own.** Plain YAML with a thin
`kustomization.yaml` per component: a reviewer reads the manifest, not a template
with a values file three directories away. Kustomize earns its place in two
spots — the `images:` transformer that makes the image pin a one-line diff, and
`configMapGenerator`, whose content hash makes an edit to the scrape config or
alert rule roll the Prometheus pod automatically, so the file in git and the
process in the cluster cannot drift. Third-party software (External Secrets) is
consumed as its upstream chart, unvendored, so there is no fork to maintain.

---

## 3. What minikube did for me

Five things I would own on bare metal, each reduced here to a flag:

**Control-plane bootstrap.** `minikube start` gave me an API server, scheduler,
controller-manager, a working PKI and kubelets already joined. On bare metal that
is `kubeadm init` plus `kubeadm join`, and the real work surrounds it: generating
and rotating a CA and the ~10 certificate pairs (kubelet client certs expire in a
year and will take an unattended cluster down), running an odd number of
control-plane nodes, and putting something in front of them — keepalived +
HAProxy, or kube-vip — because "the API server address" must survive losing the
machine it points at. Then a documented, rehearsed upgrade path: drain,
`kubeadm upgrade`, uncordon, one node at a time.

**CNI install.** `--cni=calico` installed the operator, DaemonSet, CRDs and an
IPAM pool. By hand: choose the pod CIDR *before* the cluster exists because
changing it later means rebuilding; decide encapsulation (VXLAN, or BGP peering
with the physical fabric for native routing and no overhead); verify MTU, since
an unnoticed MTU mismatch under encapsulation is the classic "large responses
hang while ping works" outage; and then actually test that enforcement is live
rather than trusting that the objects exist.

**Ingress load-balancing.** The addon gave me an ingress-nginx controller
reachable on the node IP via hostPort. There is no cloud load balancer here,
which is the point: on bare metal a `Service type=LoadBalancer` stays `<pending>`
forever unless I provide the edge. That means **MetalLB** (L2 mode answering ARP
for a VIP, or BGP advertising it to the routers) or kube-vip, an address pool
carved out of the real network, and a decision about whether the controller runs
as a DaemonSet with hostPorts behind DNS round-robin or behind one advertised
VIP. Plus `externalTrafficPolicy: Local` if I want real client IPs — which then
breaks the tidy NetworkPolicy story, because traffic starts arriving from a node
address instead of a pod IP.

**Storage provisioner.** minikube's `standard` StorageClass silently satisfied my
PVC with a hostPath directory on a node: dynamic provisioning with none of the
properties production needs, including the node affinity a real CSI driver puts
on the PV (see Part F for the multi-node trap that leaves). On bare metal I would
run a real CSI driver — Ceph via Rook for replicated RWO volumes that can move
between nodes, Longhorn for something lighter, or `local-path`/LVM if I accept
node-pinned data and get redundancy from Postgres streaming replication instead.
That choice decides whether "the node died" means "the pod reschedules" or "the
data is gone".

**etcd and its backup.** minikube runs a single-member etcd inside the cluster and
I have never had to think about it. In production etcd *is* the cluster: three or
five members on fast dedicated disks (it is fsync-latency-bound), TLS with peer
certs, alerts on fsync duration and leader elections, defragmentation and
revision compaction so the database does not quietly hit its quota and flip to
read-only, and `etcdctl snapshot save` on a schedule to storage **outside** the
cluster with a restore I have actually rehearsed. An untested etcd restore is not
a backup. Note the boundary: an etcd snapshot holds every object in this repo plus
my Secrets, and not one byte of Postgres data. Those need separate,
application-aware backups.

---

## 4. Production gaps

Roughly the order I would fix them:

1. **Postgres is a single pod on node-local storage.** One replica, hostPath PVC,
   no backups — the loudest gap. Fix: CloudNativePG with a synchronous replica,
   WAL archiving and PITR to object storage, plus a rehearsed restore.
2. **No backups at all**, neither Postgres nor etcd. Both need scheduled,
   off-cluster, *tested* restores. Until a restore has been performed, a backup
   is a hypothesis.
3. **Vault is dev mode.** In-memory storage, auto-unseal, one replica: a restart
   loses the stored credential and step 6 must run again. Production Vault means
   real storage (Raft), auto-unseal via a cloud KMS, more than one node, audit
   devices, and a proper Vault-side lifecycle for policies and roles — ideally
   Terraform, so the parts of the secret system that are *not* secret are
   themselves reviewed and versioned.
4. **No TLS anywhere.** Ingress is plain HTTP, the API→DB hop is
   `sslmode=disable`, and Vault's listener is HTTP inside the cluster. Fix:
   cert-manager at the edge, `sslmode=verify-full` for Postgres, TLS on Vault,
   and eventually a mesh (or Cilium mTLS) if I want in-cluster encryption without
   touching every app.
5. **Secrets are injected as environment variables**, so rotation needs a pod
   restart and the value is visible in `/proc` and in any crash dump. The better
   shape is a projected file with the app re-reading on change (or Vault Agent /
   the CSI secrets driver), which makes rotation a no-restart operation.
6. **Single control-plane node, no HA.** See section 3.
7. **Default-deny stops at one namespace.** `kube-system`, `argocd`,
   `ingress-nginx`, `monitoring`, `vault` and `external-secrets` are all open, so
   a compromised pod in `monitoring` is unconstrained — and worth naming
   specifically, anything that can reach Vault's port is only stopped by Vault's
   own auth. Fix: the same default-deny treatment per namespace (starting with
   Vault, where only the operator needs ingress), plus Kyverno or Gatekeeper so a
   new namespace cannot ship without one.
8. **No supply-chain gate.** Images are pinned, which stops drift but says
   nothing about provenance. Fix: cosign signatures verified at admission, and an
   SBOM in CI. A digest proves the bytes did not change; it does not say who
   built them.
9. **Alerts evaluate but route nowhere.** No Alertmanager, no on-call, and only
   application metrics — nothing on nodes, kubelets, etcd, ArgoCD sync failures,
   or ExternalSecret sync failures (which would be my early warning that the
   credential path is broken). Prometheus itself is one pod on an emptyDir, so
   history dies with it.
10. **No CI on this repo.** A broken manifest is currently discovered by ArgoCD.
    Fix: `kustomize build` plus `kubeconform` on every PR, and `gitleaks` so the
    no-secrets-in-git rule is enforced rather than merely intended.
11. **Upgrades are undocumented.** Kubernetes, Calico, ArgoCD, External Secrets
    and Postgres all need a version policy and a rehearsed order. Postgres
    especially: a minor is a restart, a major is a migration.
12. **One cluster.** No staging to break first and no DR target. The app-of-apps
    structure is ready for it (an ArgoCD `ApplicationSet` over a cluster list),
    but I have not run it.

---

## 5. One runbook — the database pod dies

Trigger: the `QovesApiDatabaseUnreachable` alert fires, or `/healthz` returns 503
through the ingress. User impact: every request that touches data fails, and
because readiness depends on the DB (ADR-6), the ingress returns 503 for all
paths.

**Step 1 — confirm what is actually broken (2 minutes).**

```bash
kubectl get pods -n qoves-app -o wide          # is postgres-0 Running? on which node?
kubectl describe pod postgres-0 -n qoves-app   # events: OOMKilled, FailedMount, Pending?
kubectl logs postgres-0 -n qoves-app --previous
kubectl get pvc -n qoves-app                   # is the claim still Bound?
```

Separate three cases, because the fix differs: the pod is **crash-looping**
(logs), the pod is **Pending** (scheduling or volume), or **the pod is fine and
something else is not** (the Secret, or the network).

**Step 2 — triage by case.**

- *Restarted and recovered by itself.* The StatefulSet recreated `postgres-0` and
  reattached the same PVC; `/healthz` goes green within a minute or two. Nothing
  to fix — go to step 4.
- *Pending with `node(s) had volume node affinity conflict`.* The ReadWriteOnce +
  node-local storage failure mode from Part F: the volume is on a node this pod
  can no longer be scheduled to. If the node is coming back, wait. If it is gone,
  so is the data — step 3.
- *CrashLoopBackOff with corruption or `database files are incompatible`.* Do not
  delete anything yet. Preserve evidence first (a debug pod mounting the PVC
  read-only), then restore — step 3.
- *OOMKilled.* Raise the memory limit **in git**
  (`gitops/manifests/postgres/statefulset.yaml`), commit, push, let ArgoCD roll
  it. Editing the live StatefulSet gets reverted by `selfHeal` within minutes,
  which is the system working as designed.
- *`CreateContainerConfigError`, Secret not found.* Postgres is not the problem —
  the credential path is. This is the failure mode this build is most likely to
  hit, because dev-mode Vault is in-memory: if the Vault pod restarted, the
  secret at `qoves/postgres` is gone and the operator cannot refresh it. Diagnose
  and fix:

  ```bash
  kubectl -n qoves-app get externalsecret postgres-credentials -o wide   # SecretSyncedError?
  kubectl -n qoves-app describe externalsecret postgres-credentials      # the Vault error is here
  kubectl -n external-secrets logs deploy/external-secrets --tail=50
  kubectl -n vault get pods                                             # restarted recently?
  ```

  If Vault restarted, re-run step 6 of section 1 — but reuse the **same** password
  if the database already has data initialised with it, because `POSTGRES_PASSWORD`
  only takes effect at `initdb` time. Writing a *new* random password into Vault
  gives you an API that authenticates with a credential the existing database
  does not know, which presents as `password authentication failed` rather than
  anything about Vault. If the password is genuinely lost, rotate it inside
  Postgres (`ALTER USER qoves PASSWORD ...`) to match Vault, rather than
  reinitialising the database.

- *Postgres healthy, API not.* Check the policies with the NetworkPolicy tests in
  section 1, and confirm the Secret exists and has all three keys.

**Step 3 — restore (the case this cluster is not yet ready for).** Honest answer
for *this* build: there is no backup, so if the volume is unrecoverable the data
is lost, and recovery means an empty database — delete the PVC, let the
StatefulSet reprovision, and the app returns with no rows. That is exactly why
backups are gap #2. With the target setup (CloudNativePG + WAL archiving) this
step is instead: create a `Cluster` with a `bootstrap.recovery` stanza pointing at
the backup and a target time just before the incident, let the operator restore
the base backup and replay WAL, verify row counts, then repoint the Service. That
change also goes through git.

**Step 4 — verify.**

```bash
kubectl get pods,pvc -n qoves-app
curl -i http://qoves.local/healthz
# then the persistence check from section 1
```

Confirm the alert clears, and that the API pods are Ready again — they recover on
their own once readiness passes, so no restart is needed.

**Step 5 — afterwards.** Write down the trigger, and if the fix was manual, ask
why it was not a git change. Anything done with `kubectl` to a workload during an
incident needs a follow-up commit; otherwise `selfHeal` quietly undoes the fix
and the next person will not know why.

---

## Appendix — the specific questions in Parts F, G and H

### F. Storage and data

**Access mode, and what it constrains.** The PVC is `ReadWriteOnce`: mountable
read-write by pods on **one node at a time**. For Postgres that is a safety
property rather than a limitation — two postgres processes on one data directory
would corrupt it. What it constrains is scheduling: the pod must land where the
volume can attach.

On a proper CSI driver the scheduler enforces that for you, because the PV
carries node affinity and a pod that cannot honour it sits `Pending` with
`volume node affinity conflict` rather than starting somewhere wrong. minikube's
default provisioner does **not** do this: it creates the directory on whichever
node it runs on and issues a PV with no node affinity at all, so on a two-node
cluster a rescheduled Postgres can mount an empty directory on the other node and
come up as a brand-new database. Failing loudly would be fine; silently starting
empty is the dangerous version. That is why the StatefulSet carries an explicit
`nodeAffinity` onto the control-plane node — I am hand-writing the constraint the
storage layer should have expressed, and it is the same constraint a bare-metal
`local-path` setup imposes. Also worth knowing: RWO is per *node*, not per pod,
so two pods on the same node can both mount it; `ReadWriteOncePod` (1.27+) is the
stricter mode I would actually use. `ReadWriteMany` would not help Postgres at
all.

**What happens if the pod or the node dies.** Pod dies: nothing is lost. The
StatefulSet recreates `postgres-0` with the same identity, reattaches the same
PVC, Postgres replays its WAL and comes up — the case the persistence check in
section 1 demonstrates. Node dies: on this cluster the data is **gone**, because
the volume is a directory on that node's disk with no replication, and the pod
will not even reschedule (its PV points at a node that no longer exists). That is
the difference a real CSI layer makes: with Ceph/Rook or Longhorn the volume is
replicated and reattaches elsewhere, so node loss becomes a reschedule rather
than a data-loss event. With node-local storage, redundancy has to come from
Postgres replication instead of from the volume.

**How I would back it up and restore it.** Two layers. *Logical*: a CronJob
running `pg_dump -Fc` to object storage — simple, portable across major versions,
good for "someone dropped a table", but the recovery point is the last dump.
*Physical*, which is what I would rely on: `pg_basebackup` plus continuous WAL
archiving to S3/MinIO, giving point-in-time recovery to any moment inside the
retention window. CloudNativePG does this natively with a `ScheduledBackup` and
`barmanObjectStore`, which is the main argument for adopting the operator.
Restore is `bootstrap.recovery` against the backup with a `targetTime`, into a
**new** cluster object so the broken one stays available for forensics. Two rules
regardless of tooling: restores are rehearsed on a schedule, and backups live in a
different failure domain from the cluster — an etcd snapshot contains no Postgres
data, and a volume snapshot on the same dead node is not a backup.

### G. Is a CPU-based HPA the right signal here?

No. This API spends its time waiting on a Postgres round-trip, so its bottleneck
is **concurrency, not CPU**: with two gunicorn workers per pod, a pod is
saturated at two in-flight requests while its CPU sits near idle. Load heavy
enough to ruin latency may never push CPU past a 70% target, so the HPA scales
late or not at all. The converse is worse: a slow database *lowers* CPU per
request, so the exact incident where I want more capacity looks quiet to a
CPU-based autoscaler.

What I would scale on instead, in order: **in-flight requests per pod** (or RPS
per pod), because it maps directly to the worker-slot limit — via
`prometheus-adapter` exposing something like
`sum(rate(http_requests_total[1m])) by (pod)`, with the target taken from a load
test that finds the RPS where p95 latency degrades; failing that, **p95 latency**
as an external metric, accepting that it lags. For a worker-per-request app the
structurally better fix is often not autoscaling at all but raising per-pod
concurrency (more workers, or an async worker class) and sizing statically, since
pod count is only a proxy for concurrent database connections — and Postgres has a
hard `max_connections`, so an aggressive HPA can convert a traffic spike into a
"too many clients already" outage. Any real autoscaling story here needs
PgBouncer in front of the database first.

CPU is in the manifest because metrics-server is the only metrics source in this
cluster and the brief asks for a working HPA. It is the right *mechanism* wired to
the wrong *signal*, and I would rather say so than pretend otherwise.

### H. The alert, and why it is actionable

```promql
sum(rate(http_requests_total{path="/healthz",status="503"}[5m])) > 0   for 3m
```

`QovesApiDatabaseUnreachable` fires only when the API has been unable to open a
connection to Postgres for three continuous minutes — a condition that breaks
every request touching data and that **no automation will fix**: not a restart,
not the HPA, not a rollout. That is my bar for paging a human: user-visible
impact plus a required human decision. `for: 3m` rides out a pod restart or a
brief blip so it stays a signal rather than noise, and because the readiness probe
hits `/healthz` every 10 seconds on every pod, the metric moves whether or not
real users are calling — the alert cannot go quiet just because traffic is low.

Its blind spot, stated plainly: if the API pods are gone entirely, nothing
increments the counter and this rule stays silent. In a fuller setup I would pair
it with an `absent()` or `up == 0` rule on the scrape target. Also honest: the app
uses `prometheus_client` under two gunicorn workers with no multiprocess
registry, so `/metrics` returns whichever worker answered the scrape and counters
look jumpy. `rate() > 0` tolerates that; the real fix is
`PROMETHEUS_MULTIPROC_DIR` or one worker per pod.

Metrics are visible via `kubectl -n monitoring port-forward svc/prometheus 9090`
and, for example,
`sum by (status) (rate(http_requests_total{path="/healthz"}[5m]))`.

---

## What I would do next, in priority order

1. CI on this repo: `kustomize build` + `kubeconform` + `gitleaks` on every PR.
   A broken manifest being caught by ArgoCD is too late.
2. Postgres via CloudNativePG with WAL archiving to MinIO, and a restore I have
   actually performed.
3. Real Vault (Raft storage, auto-unseal, TLS) with its policies and roles
   managed as code, so the non-secret parts of the secret system are reviewed
   too.
4. Default-deny in the remaining namespaces — Vault first — plus Kyverno so a new
   namespace cannot ship without policies, a non-root securityContext and a
   pinned image.
5. cosign verification at admission, and cert-manager for real TLS.
6. `prometheus-adapter` so the HPA scales on in-flight requests instead of CPU,
   with PgBouncer in front of Postgres to make scaling safe.
