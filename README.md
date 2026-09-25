# Trident Port™

> **Sailing to a Safe Harbor** — move Kubernetes persistent volumes from any storage
> provisioner onto NetApp Trident CSI.

---

## Architecture & Capabilities

Trident Port moves the data behind a `PersistentVolumeClaim` onto a
Trident-provisioned volume, then repoints the application at it — stopping what
holds the claim, in the correct order, and restoring it once the cutover completes.

**Source:** any provisioner — NFS subdir, Longhorn, Rook/Ceph (RBD and CephFS),
local-path, OpenEBS, vSphere CSI, Portworx, hand-made PVs, and Trident itself.

**Destination:** any Trident backend — `ontap-nas`, `ontap-nas-economy`,
`ontap-nas-flexgroup`, `ontap-san`, `ontap-san-economy`, `azure-netapp-files`,
FSx for ONTAP.

### Application Availability & Cutover Strategies

Trident Port operates a controlled cutover model: the destination volume is
provisioned and populated ahead of time, and the application is repointed to it in a
scheduled, bounded window rather than migrated live. What differs between methods is
*how much of the work is completed before that window opens*:

| Method | The application, from start to finish |
|---|---|
| **Offline copy** | `████████████████████████` |
| **Snapshot** | `░░░░░░░░░░░░░░░░░░░░░░░█` |
| **Live sync** | `█░░░░░░░░░░░░░░░░░░░░░░█` |
| **SnapMirror** | `░░░░░░░░░░░░░░░░░░░░░░░█` |
| **Block cutover** (`start`/`cutover`) | `█░░░░░░░░░░░░░░░░░░░░░░█` |

`░` the application is up and serving · `█` it is down

**Offline copy performs the entire transfer inside the cutover window** — the row is
black from the first byte to the last. The other four methods move the transfer
ahead of the window, at the cost of a prerequisite. **Live sync carries a second
window at the start**: its `start` phase replaces every pod once to inject the sync
sidecar, so that cost is paid up front and only the final delta remains in the
cutover window. **Block cutover's `start`/`cutover` shares that same shape** — a
brief window to repoint the application onto the mirror, then a second brief window
to switch once the resync is done.

---

## The Methods

| Method | Application stopped for | Touches the array | Requires |
|---|---|---|---|
| **Offline copy** (`copy`) | the full transfer | no | nothing beyond a destination claim |
| **Snapshot** (`snapshot`) | the final delta only | no | a `VolumeSnapshotClass` whose driver matches the source provisioner, and a filesystem volume |
| **Live sync** (`live-sync`) | the sidecar restart at `start`, then the final cut | no, but a continuous read on the source | Kubernetes 1.29+, a filesystem volume, and room to inject a sidecar |
| **SnapMirror** (`snapmirror`) | the break and the mount | yes: the transfer competes with production I/O on the same aggregates | ONTAP at both ends, cluster peering, credentials on both |
| **Block cutover** (`block-cutover`) | two brief windows (repoint, then switch) — or the switch only, without `start`/`cutover` | no, but a device-level resync on the node | a raw `Block` volume already on a node, and a destination claim |

`tp import` adopts an existing NFS export into a Trident claim without copying it —
the source is mounted read-only and never stopped.

The planning engine evaluates every method for the target environment — ranking what
is available and stating, for anything excluded, the precise reason: *"snapshot is
unavailable because the source CSI driver does not register a VolumeSnapshotClass."*

**Design considerations:**

- **SnapMirror addresses the economy storage class** (`ontap-nas-economy`,
  `ontap-san-economy`). Requires Trident **26.06+** (Tech Preview); pre-flight
  validates the version rather than failing at cutover.
- **SnapMirror operates at the FlexVol level, not the individual claim.** On
  `ontap-*-economy`, several namespaces may share a single FlexVol, and the array
  performs its delta, break and resize **once per FlexVol**. `--plan-only` identifies
  which namespaces share a FlexVol.
- **Three cutover shapes govern how shared FlexVols are handled:**
  - **`together`** cuts the entire FlexVol in one operation. Every namespace sharing
    it must be named — `--with-flexvol-neighbours` folds a neighbour in.
  - **`waves`** cuts one namespace per window, at the cost of fragmenting the
    estate — a qtree cannot move between FlexVols, so each wave re-copies the
    FlexVol in full.
  - **`clonedWaves`** avoids fragmentation cost: one baseline transfer, then each
    wave cuts over its own FlexClone — a single transfer instead of N, with reduced
    source-side snapshot retention.
- **Live sync's `start` phase requires a restart** — the sidecar replaces every pod
  once before synchronization begins. Only the final cutover delta is new.
- **A given build may include a subset of these methods.** *"Not included in this
  build"* is distinct from *"the target cluster cannot support this."* Offline copy
  is always available.
- **Block cutover is never selected automatically** — the only method above that the
  planning engine skips; an operator names it directly. `tp mixed-cutover` coordinates
  it with Live Sync when both are needed in the same namespace.

---

## What a Run Does

```mermaid
flowchart TD
    A[discover<br/>claims by StorageClass] --> B[pre-flight<br/>capacity, RBAC, SCC, versions]
    B -->|refuses| X[stop, nothing created]
    B --> C[provision<br/>destination claims]
    C --> D[copy<br/>rsync / tar / dd]
    D --> E[validate<br/>counts, sizes, or every byte]
    E --> F[STOP<br/>workloads, in order]
    F --> G[final delta]
    G --> H[swap<br/>repoint claims]
    H --> I[START<br/>and wait until serving]
    I --> J[report + rollback script]
    style F fill:#c62828,color:#fff
    style G fill:#c62828,color:#fff
    style H fill:#c62828,color:#fff
    style I fill:#c62828,color:#fff
```

**The four highlighted stages are the cutover window.** Everything preceding them
executes with the application running, regardless of method. Every phase records its
own duration to a permanent trace, and reporting reads that record rather than
asserting an estimate.

**Source volumes are never deleted.** They are set to `Retain` before anything else
happens, so they persist beyond their claims. Leftover volumes are surfaced for
reclamation; nothing is removed without an explicit operator action.

---

## Supported and Validated Environments

Trident Port is validated against the following platforms:

- **RKE2**
- **Red Hat OpenShift**
- **K3s**
- **Upstream Kubernetes 1.29+** (1.29+ required for live sync's native sidecar support)

| | |
|---|---|
| Kubernetes | any distribution Trident supports · **1.29+** for live sync |
| NetApp Trident | 24+ · **26.06+** for qtree import in SnapMirror mode |
| Trident backend | at least one `TridentBackendConfig` bound, provisioning into the destination StorageClass |
| Credentials | a Kubernetes Secret holding the ONTAP credentials |
| Registry | private image — a `docker-registry` pull secret per namespace |
| Licence | a token bound to the cluster's `kube-system` namespace UID |

---

## Installation

There are three ways to install Trident Port: **Option A**, an operator that
manages the deployment for you (recommended); **Option B**, Helm directly; or
**Option C**, a standalone container outside Kubernetes. All three end at the
same place — a running pod you reach with `kubectl port-forward`, where you
activate your licence.

Everything you need for all three options is on this page. Nothing else to
read first.

### Before you start

- `kubectl`, configured for the target cluster — `kubectl cluster-info` should
  succeed before you go any further.
- Helm 3.10 or newer for Options A and B — check with `helm version`.
- At least one Trident `TridentBackendConfig` already `Bound` — check with
  `kubectl get tbc -A`.
- A GitHub personal access token (PAT) with **only** the `read:packages`
  scope. NetApp / CalzDevOps issues this to you as part of onboarding — do not
  create your own. Below, `<github-user>` is your GitHub username and `<PAT>`
  is this token.
- **Option A only:** two files NetApp hands you at onboarding time,
  `install.yaml` and `config/sample.yaml`. They are account-specific and are
  not published in this repository. Save them locally before you start, e.g.
  in `~/trident-port-onboarding/`, and run the commands below from that
  folder.

### Option A — Operator (recommended)

Use this if you want upgrades, RBAC and adoption handled for you.

**Step 1 — create the operator's namespace and its pull secret.**

```bash
kubectl create namespace trident-port-system --save-config
kubectl -n trident-port-system create secret docker-registry ghcr \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<PAT>
```

You should see `namespace/trident-port-system created` and `secret/ghcr
created`. If instead you see `AlreadyExists`, that is fine — continue. Any
other error, stop and check `kubectl config current-context` points at the
right cluster.

**Step 2 — install the operator.**

```bash
kubectl apply -f install.yaml
kubectl -n trident-port-system rollout status deploy/trident-port-operator --timeout=300s
```

You should see a list of created objects, ending with `deployment
"trident-port-operator" successfully rolled out`. If it never returns, run
`kubectl -n trident-port-system get pods` — `ImagePullBackOff` means the
secret from Step 1 is missing or wrong.

**Step 3 — create the product's own namespace and pull secret (a separate
one, in a separate namespace), then hand it the sample configuration.**

```bash
kubectl create namespace trident-port
kubectl -n trident-port create secret docker-registry ghcr \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<PAT>
kubectl apply -f sample.yaml
kubectl -n trident-port rollout status deploy/trident-port --timeout=300s
```

You should see `deployment "trident-port" successfully rolled out`. If not,
`kubectl -n trident-port describe pod <pod-name>` — the most common cause is
the same missing pull secret, in this namespace this time.

**Step 4 — get your exact port-forward command and licence-activation
instructions.** The operator writes them into the `TridentPort` object once
the deployment is up:

```bash
kubectl -n trident-port get tridentport trident-port \
  -o jsonpath='{.status.conditions[?(@.type=="Deployed")].message}'
```

Follow what this prints — it is generated for your install. Then skip to
["Opening the UI"](#opening-the-ui-and-activating-your-licence) below.

### Option B — Helm

Use this if you manage Helm releases directly, without the operator.

**Step 1 — log the cluster and Helm itself into the registry.** These are two
separate logins — the cluster needs one to pull the image, Helm needs the
other to pull the chart:

```bash
kubectl create namespace trident-port

kubectl -n trident-port create secret docker-registry ghcr \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<PAT>

echo "<PAT>" | helm registry login ghcr.io -u <github-user> --password-stdin
```

The last command should print `Login Succeeded`. A `401` or `403` means the
PAT lacks `read:packages`, or has expired — ask whoever issued it for a fresh
one.

**Step 2 — get the chart's default values file, and edit the few fields that
matter.**

```bash
helm show values oci://ghcr.io/calzdevops/charts/trident-port \
  --version 0.2.44 > my-values.yaml
```

You should now have a file `my-values.yaml`, a few hundred lines, starting
with `image:`. Open it and set:

- `image.pullSecrets`, to `[ghcr]` — the secret you created in Step 1.
- `config.DEST_SC` — the destination StorageClass migrations write into. Leave
  it blank and set it later from the web UI (Settings → Environment) if you
  don't know it yet.
- `existingSecret` — only if you already have a Kubernetes Secret holding your
  ONTAP password; otherwise leave it blank and enter ONTAP details from the UI
  after install.

Every other field already has a working default, each explained by a comment
directly above it in the file — leave the rest alone for a first install.

**Step 3 — install.**

```bash
helm install trident-port oci://ghcr.io/calzdevops/charts/trident-port \
  --version 0.2.44 \
  --namespace trident-port --create-namespace \
  --values my-values.yaml
```

You should see `STATUS: deployed`, followed by a `NOTES` block with a
port-forward command and how to fetch the API token — read it, it is specific
to your install. An error naming a pull secret means Step 1's secret name
does not match `image.pullSecrets` in your values file; an error naming
`read:packages` or `401` means the `helm registry login` in Step 1 needs to be
run again.

**Step 4 — confirm the pod is actually running** (`deployed` describes the
Helm release, not the pod):

```bash
kubectl rollout status deployment/trident-port -n trident-port --timeout=300s
```

You should see `deployment "trident-port" successfully rolled out`. If not,
`kubectl -n trident-port get pods`: `ImagePullBackOff` again points at the
pull secret; `Pending` means run `kubectl describe pod <name>` for the
scheduling reason (resources, taints, node selectors).

`image.tag` is intentionally left empty in the values file — the chart
installs its own `appVersion`, so the chart version and the image version move
together, one decision instead of two.

### Option C — Standalone Container

Use this if you do not want to install anything into the cluster itself — a
container on a bastion host or your laptop, using whatever your own
kubeconfig can already do.

**Step 1 — log in to the registry and pull the image.**

```bash
echo "<PAT>" | docker login ghcr.io -u <github-user> --password-stdin
docker pull ghcr.io/calzdevops/trident-port:v1.2.0-rc55
```

You should see `Login Succeeded`, then the image layers downloading, ending
with `Status: Downloaded newer image`.

**Step 2 — run it, with your kubeconfig and a folder for session state
mounted in.**

```bash
mkdir -p ./tracking

docker run -d \
  --name trident-port \
  -p 8000:8000 \
  -v ~/.kube:/root/.kube:ro \
  -v "$(pwd)/tracking:/app/tracking" \
  ghcr.io/calzdevops/trident-port:v1.2.0-rc55
```

Mount the kubeconfig at `/root/.kube`, not the container's own home directory
— the container copies it internally with the right ownership on startup;
mounting it directly anywhere else leaves it unreadable by the process that
needs it.

You should see a container ID printed, and `docker ps` showing it `Up`. If
`docker logs trident-port` shows a permissions error on `/app/tracking`, the
host folder needs to be writable by the container (`chmod 777 ./tracking` on
a lab machine, or add `--user $(id -u)` to the command above).

**Step 3 — confirm it is actually serving, not just running.**

```bash
docker exec trident-port curl -sf http://localhost:8000/api/health
```

You should see a JSON body with no error field. If this fails immediately
after Step 2, wait a few seconds and retry (first boot); if it never
succeeds, read `docker logs trident-port` for the actual error.

### Opening the UI and activating your licence

**Options A and B** — forward the port:

```bash
kubectl -n trident-port port-forward svc/trident-port 8000:8000
```

**Option C** — the container already publishes port 8000 on the host; skip
straight to opening the browser.

Open `http://localhost:8000`. The first screen is licence activation — paste
the token NetApp gave you and confirm. Every other page needs this
deployment's own API token, which the browser will ask for the first time it
gets a `401`; fetch it with:

```bash
kubectl -n trident-port get secret trident-port-api-token \
  -o jsonpath='{.data.token}' | base64 -d
```

**This install is reachable only from inside the cluster's network by
default** (`ClusterIP`). The token above guards the API from a stranger who
can reach the address — it is a shared secret, not a per-user login — so keep
the default exposure unless there is an authenticating proxy in front.

### Verifying the install

```bash
kubectl -n trident-port get pods
```

You should see two pods, both `1/1 Running`: `trident-port-xxxx` (the
application) and `trident-port-webhook-xxx` (a small admission helper the
chart installs automatically — its presence is expected, not a sign of a
double install).

### Uninstalling

```bash
# Option A — remove the CR, then the operator itself
kubectl -n trident-port delete tridentport trident-port
kubectl delete -f install.yaml

# Option B — Helm
helm uninstall trident-port -n trident-port

# Option C — container
docker rm -f trident-port
```

Session history and rollback scripts live on a persistent volume
(`trident-port-tracking`) that survives uninstall on purpose, so a reinstall
does not lose them. Delete it explicitly if you want it gone:

```bash
kubectl -n trident-port delete pvc trident-port-tracking
```

### Licensing

A licence token, bound to the target cluster's `kube-system` namespace UID, is
required to migrate data. Contact NetApp or your account team to obtain one.

---

## Enterprise Packaging & Security

Trident Port ships as **a hardened, fully compiled artifact**, ensuring immutable
execution, a minimized attack surface, and no runtime dependency on host-level
scripts. The customer-facing image contains no interpretable source and no shell
tooling beyond what the running product itself requires — what is delivered is what
was built and validated.

**RBAC is namespace-agnostic, and scoped to exactly what each operation needs**
(`helm/trident-port/templates/rbac.yaml`, a single `ClusterRole`): read access on
core objects including `secrets` (get/list/watch, **never write** — this is how the
product reaches ONTAP credentials without requiring a copy per namespace),
create/delete on `batch/jobs` and `pods`, patch on
`deployments`/`statefulsets`/`daemonsets`/`replicasets`, and scoped access on each
supported operator's own CRD group — never a wildcard over custom resources.

**The copy workload is not a privileged container.** It runs as root
(`runAsUser: 0`) with `allowPrivilegeEscalation: false` and no additional
capabilities — sufficient to preserve filesystem ownership and write a raw block
device. Requesting `privileged` access is unnecessary and is rejected under
OpenShift's `restricted-v2` policy by design; the remediation the product surfaces is
`oc adm policy add-scc-to-user anyuid`. Exactly one code path requires an additional
capability: mounting a foreign NFS export, which requires `CAP_SYS_ADMIN`.

**Trident's own controller may be restarted as part of a SnapMirror import** — shared
CSI infrastructure, restarted by Trident itself, not by this product directly, when
an import is refused while a volume name is still held in memory. This occurs **at
most once per run, and only when an import was actually refused**; `--no-trident-restart`
disables this behavior, stopping the cutover at the first refused import with
remediation steps provided.

**Interface access is authenticated and scoped to a single deployment.** The default
Service is `ClusterIP`; the API runs with the deployment's own RBAC, so an
`X-Trident-Port-Token` header (generated and persisted at install time, validated in
constant time) is required on every request except health checks, static assets and
the activation screen.

**Licence entitlement is bound to the target cluster's identity** (its `kube-system`
namespace UID) and does not transfer if the token is copied to another cluster.
Completed migrations are recorded in a signed usage ledger (HMAC), protecting the
integrity of the usage count independent of the cluster operator.

**Every operation is auditable.** `/metrics` exposes Prometheus-format migration
counts, licence status and audit counters; `tracking/trace.jsonl` is an append-only,
SIEM-ready record of the CLI and web interface, including every refusal.

---

## Command Reference

**Licensing model**: read-only planning (`survey`, `plan`) and critical disaster
recovery operations (`rollback`, `stand-down`) are permanently unlocked and do not
require an active licence entitlement. Operations that migrate or mutate data require
a validated licence.

**Reading and planning — no licence required**

| Command | What it does |
|---|---|
| `tp survey` | assesses whether a cluster is eligible for migration; read-only, dry-runs a copy pod's admission |
| `tp plan` | reports what would happen to one namespace's claims, method by method |
| `tp space` | projects whether the source volume fills before a baseline completes; sampled over time |
| `tp validate` | compares source and destination; performs no writes |
| `tp sessions` | lists migrations on this installation, or follows one in progress |
| `tp report` | reports what happened to a namespace's volumes, matching the interface's view |
| `tp array-leftovers` | lists array volumes no longer referenced by the cluster; explains, does not delete |
| `tp pre-engagement` | produces the read-mostly assessment kit for a discovery engagement |
| `tp rollback` | restores reclaim policies, lists or removes leftovers |
| `tp start` | restores application availability after a stopped run |
| `tp support-bundle` | collects a complete diagnostic bundle |
| `tp stand-down` | scales this deployment to zero if a licence lapses |

**Data operations — licensed**

| Command | What it does |
|---|---|
| `tp auto` | migrates a whole namespace in one coordinated stop: discover, pre-flight, copy, validate, cut over, verify |
| `tp migrate` | migrates a single volume |
| `tp batch` | migrates several namespaces in sequence, each to its own destination |
| `tp snapshot` | copies from a clone while the application remains online (`bulk`), then cuts over (`cutover`) |
| `tp live-sync` | `start`, `status`, `stop`, `cutover` |
| `tp snapmirror` | replicates FlexVols between arrays and adopts their contents |
| `tp import` | brings a foreign NFS export into a Trident claim |
| `tp cutover` | repoints claims to the new volumes |
| `tp stop` | stops the workloads holding the claims |
| `tp block-cutover` | mirrors a Block PVC's device to its destination and switches; `start`/`cutover` also repoint the application's own claim onto it |
| `tp mixed-cutover` | `start`/`cutover` — the same repoint as `live-sync` and `block-cutover`, coordinated for both mixed in one namespace |

### A Namespace, End to End

```bash
# Rehearsal: discovery and pre-flight only. Nothing is created, stopped or copied.
tp auto --namespace shop --dest-sc ontap-nas --dry-run

# Execute: one coordinated stop, every claim in the namespace.
tp auto --namespace shop --dest-sc ontap-nas

# Multiple source classes in one namespace: a destination per source class.
tp auto --namespace shop --dest-sc ontap-nas --sc-map "longhorn=ontap-nas,ceph-rbd=ontap-san"

# Copy and validate only, leaving applications in place.
tp auto --namespace shop --dest-sc ontap-nas --exclude cache-0 --no-cutover
```

**Four parameters govern the cutover window**, each defaulting to full concurrency
except adoption:

| Flag | Governs | Default |
|---|---|---|
| `--copy-parallel` | the copy — a Job, a pod and an image pull per volume | all |
| `--stop-parallel` | stopping workloads, within each ordering tier | all |
| `--start-parallel` | restoring workloads — already down, so higher concurrency introduces no additional risk | all |
| `--adopt-parallel` | `tridentctl import` concurrency | **1** |

`--bwlimit` caps the read rate in rsync's own notation (`20M`, `200K`) and is the
only throttling control.

### The Other Methods

```bash
# Snapshot: copy from a clone with the application online, then take the delta.
tp snapshot bulk    --namespace shop --pvc data-0 data-1 --dest-sc ontap-nas
tp snapshot cutover --namespace shop --pvc data-0 data-1

# Live sync: a sidecar keeps the destination current; the cut is the final delta.
tp live-sync start   --namespace shop --pvc data-0 --dest-sc ontap-nas
tp live-sync cutover --namespace shop --pvc data-0 --dest-sc ontap-nas

# SnapMirror, ONTAP to ONTAP. Plan first; the baseline runs with the application online.
tp snapmirror --namespaces shop --dest-svm svm_new --plan-only
tp snapmirror --namespaces shop --dest-svm svm_new --cutover-only

# Block cutover: mirrors a raw Block device, then repoints the claim onto it.
tp block-cutover start   --namespace shop --pvc data-0 --dest-sc ontap-san
tp block-cutover cutover --namespace shop --resume <session-id>

# Adopt an existing NFS export. The source is mounted read-only.
tp import --namespace shop --server 10.0.0.5 --path /exports/archive \
  --claim archive --dest-sc ontap-nas --size 500Gi
```

### Validation and Rollback

`tp validate` compares counts, sizes and effective modification times by default;
`--mode full` (or `strict`) reads every byte on both sides. **Raw block volumes are
always compared across the full device** — a matching prefix is the absence of
counter-evidence, not confirmation of a match. Excluded by design: a formatted block
destination's `lost+found`, and Trident's own `.snapshot`/`trident_pvc_*` paths.

Every run leaves a session under `tracking/sessions/<id>` with its plan, its
per-phase log and a rollback script.

```bash
tp sessions
tp sessions --session sess-8a21 --follow
tp rollback --namespace shop                                    # restores reclaim policies, lists leftovers
tp rollback --namespace shop --to-source --session sess-8a21 --yes
```

**`--to-source` is a reverse migration, not an undo.** It repoints the application to
its original volume; data written to the destination since cutover remains there.
This is stated explicitly before the operation executes.

---

## The Web Interface

|  |  |
|---|---|
| ![Dashboard: cluster overview with migration counts, PVC/pod/StorageClass state and Trident backends](docs/img/web-dashboard.png) | ![Cluster Map: every namespace as a node, coloured by migration state](docs/img/web-cluster-map.png) |
| ![Assessment: which mode each namespace's volumes should take, and why — before anything is installed](docs/img/web-assessment.png) | ![New Migration: the six ways to move data, picked per namespace](docs/img/web-new-migration.png) |
| ![SnapMirror: ONTAP-to-ONTAP replication, connection profiles and cutover grouping](docs/img/web-snapmirror.png) | ![Migrations: every run this cluster has been through, live or finished](docs/img/web-migrations.png) |
| ![A migration in progress: phase tracker, pods and PVCs, live log](docs/img/web-migration-detail.png) | ![Migration history: a completed run's phase-by-phase window, byte counts and validation per volume](docs/img/web-reports.png) |
| ![Cluster Explorer — Operators: every operator controller in the cluster, its CRs and its migration method](docs/img/web-operators.png) | ![Cleanup: released and orphaned volumes left behind by past migrations, reclaim policy called out per row](docs/img/web-cleanup.png) |
| ![Settings: .env status, licence, product version and commit, cluster health](docs/img/web-settings.png) | |

| Page | Path | Purpose |
|---|---|---|
| Dashboard | `/dashboard` | cluster overview, claim counts, configuration state |
| Assessment | `/assessment` | pre-flight and the migration plan, on one screen |
| Cluster Map | `/cluster-map` | every namespace as a node, coloured by state, with operator overlay |
| New Migration | `/migrations/new` | the guided migration wizard |
| Auto Migration | `/auto-migration` | a whole namespace in one launch |
| Batch Migration | `/batch-migration` | multiple namespaces, sequential or parallel against a live capacity ceiling |
| Space Race | `/space` | projects whether the source will fill before the baseline completes |
| Cutover | `/cutover` | the swap, as an isolated operation |
| Snapshot · Live Sync · SnapMirror · Block Cutover | `/snapshot-migration` `/live-sync` `/snapmirror` `/block-cutover` | one page per method |
| Import NFS | `/import-nfs` | adopt an external export |
| Sessions | `/sessions` | every run, live or finished |
| Migrations · Reports | `/history` `/reports` | migration history, one row per volume |
| Rollbacks | `/rollbacks` | the undo path, per session |
| Cluster Explorer | `/cluster` | namespaces, pods, claims, volumes, StorageClasses, backends |
| Cleanup | `/cleanup` | source claims and orphaned Released volumes |
| Settings | `/settings` | configuration, connection testing, permissions, licence |

---

## Applications With an Operator

A StatefulSet owned by an operator must be stopped **through its Custom Resource**,
or the operator restores it automatically — measured at under one second. Detection
is by `ownerReferences` and CR name; the registry covers **40 kinds** (PostgreSQL,
MongoDB, MySQL/MariaDB, Kafka, Elasticsearch, OpenSearch, Redis, Valkey, Cassandra,
ScyllaDB, CockroachDB, TiDB, ClickHouse, Couchbase, MinIO, etcd, RabbitMQ,
Prometheus and others).

```mermaid
flowchart TD
    A{Is the workload owned<br/>by a known operator?} -->|yes| B[stop and restore<br/>through its CR]
    A -->|no| C{Unknown CRD with a<br/>scale subresource?}
    C -->|yes| D[scale generically,<br/>stated explicitly]
    C -->|no| E[STOP AND WARN<br/>never a blind scale]
    B --> F{Does the CRD refuse<br/>replicas: 0?}
    F -->|11 of the 40 do| G[scale the workload directly<br/>— requires --hold-operator]
    F -->|no| H[patch the CR]
    style E fill:#ef6c00,color:#fff
```

**The replica count is read and recorded before any scale-down occurs**, in order: an
annotation from a prior run, then the CR's own spec, then a default of 1. The
annotation takes precedence because an interrupted run can leave the spec itself at
zero, and restoring that value would report success against a namespace that never
returns to service.

An operator serving *other* namespaces is never suspended without explicit
authorization: `--hold-operator` is an operator-level acknowledgment that nothing
reconciles there for the duration of the window.

### The Four Cutover Mechanisms for Operator-Managed Workloads

1. **Native pause, by CR** — a dedicated pause field (`spec.offline`, `spec.stop`).
2. **Operator hold** — the operator's Deployment to 0, StatefulSet scaled directly.
3. **Zero, by CR** — patches the CR's own replica field; the operator scales itself down.
4. **Orphan + PV Retain** — deletes the owner, with the PV already set to `Retain`.
   Available today for KubeVirt.

### Validated Compatibility, by Operator Kind

Every migration path is measured end to end against a real cluster before it is
published as supported. `✅` denotes a validated, production-ready path; `WIP`
denotes active engineering work; `excluded` marks a deliberate scope decision.

Each entry lists the mechanism applied (1–4 above, or `generic` for kinds without a
dedicated CRD path) alongside its validation status.

| Kind | Mechanism · Status | Kind | Mechanism · Status |
|---|---|---|---|
| `Airflow` | generic · WIP | `Alertmanager` | 3 · WIP |
| `ArangoDB` | generic · WIP | `Artifactory` | generic · ✅ |
| `CassandraDatacenter` | 2 · ✅ | `CephCluster` (Rook) | excluded |
| `ClickHouseInstallation` | 1 · WIP | `Cluster` (CloudNativePG) | 1 · WIP |
| `CouchDB` | generic · ✅ | `CouchbaseCluster` | 2 · WIP |
| `CrdbCluster` (CockroachDB) | 2 · ✅ | `Dragonfly` | 3 · ✅ |
| `Elasticsearch` | 3 · WIP | `EtcdCluster` | 3 · WIP |
| `Gitea` | generic · ✅ | `Harbor` | generic · ✅ |
| `Hazelcast` | 3 · WIP | `InfluxDB` | generic · WIP |
| `InnoDBCluster` (Oracle) | 2 · WIP | `Jenkins` | generic · WIP |
| `KafkaNodePool` | 2/3 · WIP | `Keycloak` | 3 · WIP |
| `MariaDB` | 3 · ✅ | `Memcached` | — · WIP (chart has no volume) |
| `Milvus` | 3 · ✅ | `MongoDBCommunity` | 3 · WIP |
| `MySQLCluster` (MOCO) | 1 · ✅ | `MysqlCluster` (Presslabs) | 3 · ✅ |
| `NATS` | generic · ✅ | `Neo4j` | generic · ✅ |
| `OpenSearchCluster` | 3 · WIP | `PerconaServerMongoDB` | 3 · ✅ |
| `PerconaXtraDBCluster` | 3 · ✅ | `Prometheus` | 3 · WIP |
| `Pulsar` | generic · WIP | `RabbitmqCluster` | 3 · ✅ |
| `RedisCluster` | 3 · WIP | `RedisEnterpriseCluster` | 3 · WIP |
| `ScyllaCluster` | 3 · WIP | `Seaweed` | 2 · ✅ |
| `SolrCloud` | 3/2 · WIP | `TemporalCluster` | 2 · ✅ |
| `Tenant` (MinIO) | 3 · WIP | `TidbCluster` | 3 · WIP |
| `Valkey` | 3 · WIP | `Vault` | excluded |
| `VirtualMachine` (KubeVirt) | 4 · ✅ | `VMCluster` | 1 · ✅ |
| `ZookeeperCluster` | 2 · WIP | `postgresql` (Zalando) | 3 · ✅ |

**21 validated, production-ready paths**, each measured clean end to end against a
real cluster. `KafkaNodePool` and `SolrCloud` list two mechanisms because neither
alone stops them fully — both have been evaluated, and both remain active work.
Kinds requiring a third-party licensed operator to validate (MongoDB Enterprise,
CrunchyData PostgreSQL Operator, Sonatype Nexus Repository via Red Hat Connect) are
tracked separately and engaged on a per-opportunity basis. One kind (RedisFailover,
via the Spotahome operator) is omitted here as its upstream project is no longer
maintained.

`VirtualMachine`'s disk migrates through its own dedicated DataVolume swap
(`tp compat kv-dv-swap.sh swap <ns> <vm> <datavolume> <target-pv> <target-sc>`),
mechanism 4's shape rather than a CR patch — validated end to end, including data
integrity inside the guest. `MySQLCluster` (MOCO) is the validated case of mechanism 1
working exactly as designed: a dedicated `spec.offline` field, not a replica count.
`Gitea`, `CouchDB`, `Neo4j`, `ArangoDB`, `InfluxDB`, `Harbor`, `Pulsar`, `Artifactory`,
`Jenkins` and `Airflow` are validated through the generic unknown-operator path
(direct, stated scaling); `CouchDB`, `NATS`, `Gitea` and `Neo4j` are confirmed there
today. `CephCluster` (Rook) is excluded by design: storage infrastructure, like
Trident itself, is not a migration target for this product. `Vault` is excluded by
design as well: migrating its Raft store underneath it is a correctness risk outside
the product's scope — validated against real hardware, the storage transfer itself
completes cleanly, but Vault's own seal state requires its native unseal procedure
regardless, so a clean storage copy alone does not constitute a working handoff.
`Milvus` (mechanism 3 entry) follows the same pattern as `TemporalCluster`: its own CR
patch is not exercised in practice because state resides in dependent services (etcd,
MinIO) rather than the patched component — the namespace migration itself, through
those dependencies' volumes, completes cleanly end to end. `Memcached`'s current chart
ships with no persistent volume — nothing to migrate, pending a chart revision.

---

## Operational Safeguards

- **Infrastructure namespaces** are protected by default; `--i-know-this-is-infrastructure`
  is required to proceed — stopping workloads in a namespace running storage
  infrastructure removes volumes rather than quiescing them.
- **A namespace under active GitOps reconciliation (Argo CD or Flux)** is detected and
  the operation is declined, naming the owning resource and the exact suspend
  command. `--allow-gitops` is the documented override, with both associated risks
  stated explicitly.
- **A second run against a namespace already in flight is refused, not queued.**
  `tp auto`, `tp batch`, `tp snapmirror` and `tp live-sync` all route through a single
  session guard:

  ```
  [ERROR] a migration of 'shop' is already running: shop-20260911-101500
    (started …, phase 'copy'). Two concurrent runs on one namespace would
    read volumes the first run is actively restoring.
  ```

- **A dry run creates, stops or copies nothing** — enforced at the engine level.
- **A long-running copy is never silent.** Progress is reported on a fixed interval,
  with elapsed time and the exact command to follow the live log.

---

## What Survives a Cutover

**Label and annotation retention follows a deny-list, not an allow-list**
(`tp_core/pvcmeta.py`). Every label on the source claim carries to the replacement
unfiltered — there is no maintained list of "supported" annotations to extend as
tooling evolves. `backup.velero.io/backup-volumes`, `argocd.argoproj.io/tracking-id`,
`cdi.kubevirt.io/storage.populatedFor`, cost-centre labels: all are preserved.

**Three annotation families are dropped, each for a stated reason** — properties true
of the binding being replaced, and false of its replacement:

| Dropped | Rationale |
|---|---|
| `pv.kubernetes.io/*`, `volume.kubernetes.io/*`, `volume.beta.kubernetes.io/*` | written by the binding machinery; the API server writes its own on the new claim |
| `trident.netapp.io/*` | Trident's record of a prior SnapMirror import — true of the volume just imported onto, false of the one it is migrating to |
| `kubectl.kubernetes.io/last-applied-configuration` | describes the object as declared before the migration; a later `kubectl apply` would diff against a spec that no longer exists |

Every deliberately dropped annotation is named in the migration report, distinguishing
an intentional design decision from data loss.

---

## Observability

- **`/metrics`** — Prometheus text exposition: migration counts by status, licence
  validity, tier and limit, verified ledger entries, audit counters.
- **`tracking/trace.jsonl`** — append-only JSON lines, a unified record for the CLI
  and the web interface, including every refusal. Suitable for SIEM ingestion.
- **`tracking/sessions/<id>/`** — the plan, per-phase logs and trace for every run;
  the basis of a support bundle.

---

## Troubleshooting and Expected Results

| Situation | Observed Behavior | Resolution |
|---|---|---|
| A second run against a namespace already migrating | Refused: a migration of that namespace is already in progress | By design — concurrent runs on one namespace would overwrite each other's copy in progress. Wait for completion, or follow the active session |
| `tp auto` against an infrastructure namespace | Declined by default, requiring `--i-know-this-is-infrastructure` | Stopping workloads where storage itself runs can remove volumes rather than quiesce them. Requires explicit acknowledgment |
| A namespace under active Argo CD/Flux reconciliation | Declined, naming the owning resource and the exact suspend command | `--allow-gitops` is the documented override, with both associated risks stated |
| `--dry-run` performs no changes | Enforced at the engine level | Any state change during a rehearsal is treated as a defect |
| A long-running copy shows no progress percentage | A status line is emitted on a fixed interval, with elapsed time and the exact command to follow the live log | rsync-based transfers cannot produce a reliable ETA; progress is reported by interval instead |
| OpenShift rejects the copy pod | Pod unschedulable under `restricted-v2` | Do not request `privileged` — the remediation the product prints is `oc adm policy add-scc-to-user anyuid` |
| A generic Kubernetes namespace under `PodSecurity: restricted` rejects the copy pod | Same failure class as the OpenShift case, on a different platform | The product prints the exact remediation: `kubectl label ns <ns> pod-security.kubernetes.io/enforce=baseline --overwrite` |
| `Forbidden` on a newly onboarded operator's CRD | That operator is not yet declared in the product's `ClusterRole` | RBAC is granted per operator, by explicit rule — never a wildcard. A new operator kind requires that rule before this product can manage it |

**A successful migration** is confirmed when `tp validate` agrees with the
byte-level checksum comparison, the migration report shows all cutover phases
with measured durations, and a complete session record exists under
`tracking/sessions/<id>/`, including its plan, per-phase log and rollback script.

---

## Support

- **`tp support-bundle`** produces a complete diagnostic archive (logs, trace,
  session state) for submission to support.
- Before contacting support, capture the active session (`tp sessions --session
  <id> --follow`) and the exact error text — the majority of conditions in the
  table above are self-resolving with that information alone.
- Contact your NetApp account team or supporting partner for engagement-specific
  support channels.

---

## Prerequisites and Best Practices

Beyond the environment requirements above, the following should be confirmed
ahead of a migration window rather than discovered during one:

- **RBAC, and on OpenShift the appropriate SCC exception, applied in advance** —
  on OpenShift, `oc adm policy add-scc-to-user anyuid` (never request
  `privileged`: rejected under `restricted-v2` by design); on a generic cluster
  enforcing `PodSecurity: restricted`, the `baseline` label the product itself
  documents.
- **A `VolumeSnapshotClass` registered with a driver matching the source
  provisioner**, when the snapshot method is intended — the planning engine will
  otherwise exclude it, but resolving this ahead of time avoids a mid-window
  surprise.
- **The destination `TridentBackendConfig` already in `Bound` state** before a
  run begins.
- **ONTAP credentials provisioned as a reachable Kubernetes Secret.**
- **A registry pull secret present in every target namespace.**

---

## Performance and Sizing

**Figures below are measured, not projected, and scoped to the environment in
which they were captured.** From validation against a vSphere CSI → Trident
environment, one representative dataset:

| Mode | Measured Cutover Window | Scales With Dataset Size |
|---|---|---|
| Offline copy | 155 s | Yes — the entire transfer occurs inside the window |
| Snapshot | 65 s | No, predominantly fixed cost — only the final delta is inside the window |
| Swap, per volume | 6 s floor | No — a pointer repoint, not a data transfer |

- **Offline copy places the entire transfer inside the cutover window** — window
  duration scales with dataset size.
- **Snapshot and live-sync remove the bulk transfer from the cutover window** —
  the window is dominated by the final delta, largely independent of total
  volume size.
- **SnapMirror cuts by FlexVol, not by individual claim** — on economy storage
  classes, namespaces sharing a FlexVol are cut together in a single array
  operation (see FlexVol-sharing behavior under [The Four Methods](#the-four-methods)).

For sizing guidance specific to a customer's dataset and topology, engage your
NetApp account team for a scoped assessment.

---

## Documentation

| | |
|---|---|
| [`documentation/`](documentation/README.md) | the manual: overview, installation, methods, operations, rehearsal and engagement procedure |
| [`docs/INSTALL.md`](docs/INSTALL.md) | Helm installation, values, licence activation, standalone container |
| [`operator/README.md`](operator/README.md) | the operator deployment path |
| [`documentation/07-precheck-runbook.md`](documentation/07-precheck-runbook.md) | pre-window checks, including OpenShift SCC and RBAC requirements |
| [`pre-engagement/`](pre-engagement/README.md) | the read-only assessment kit, designed to run before any installation |

---

## Licence

Trident Port is commercial software.
Copyright © 2026 Carlos Alzaga. All rights reserved.

Licensing: [github.com/CalzDevOps](https://github.com/CalzDevOps)

---

*Trident Port — Sailing to a Safe Harbor*
