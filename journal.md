# Journal: barman-plugin-scheduledbackups cleanup reproduction on gcp-internal

## Safety scope
- **Target context:** `gcp-internal` (GKE `fathom-internal`). Every command uses `--context gcp-internal`.
- **DO NOT TOUCH:**
  - `postgres` namespace / `internal-postgres-cluster` (existing CNPG cluster)
  - `cnpg-system` deployment `internal-cloudnative-pg` (existing operator, v1.27.1) — we only ADD the barman plugin controller alongside it
  - `cert-manager`, `argocd`, any `prd`/`stg` contexts
- **WILL CREATE (additive, isolated):**
  - `minio` namespace: standalone MinIO pod + Service + bucket (NO operator)
  - barman plugin (`plugin-barman-cloud`) deploy in `cnpg-system` (adds `barmancloud.cnpg.io` CRDs + controller)
- **WILL VERIFY** after each step that existing workloads are unaffected.

## Reproduction note
Test endpoint is hardcoded `https://minio.minio.svc.cluster.local` (+ kube-root-ca.crt CA).
Standalone MinIO will run **HTTP on :9000**, so the reproduction test run will override
`endpointURL` to `http://minio.minio.svc.cluster.local:9000` and drop `endpointCA`.
TLS is irrelevant to the cleanup-timeout reproduction.

## Command log

### Phase 0 — pre-flight safety check (read-only)
- `kubectl config current-context` → `gcp-internal` ✓
- `kubectl --context gcp-internal get clusters.postgresql.cnpg.io -A` → only `postgres/internal-postgres-cluster` (healthy, must not touch)
- `kubectl --context gcp-internal get ns minio` → NotFound (safe to create)
- `kubectl --context gcp-internal get crd objectstores.barmancloud.cnpg.io` → NotFound (plugin will add)

### Phase 1 — standalone MinIO (no operator), HTTP :9000
- `kubectl --context gcp-internal apply -f repro/minio-standalone.yaml`
  → created ns/minio, secret/minio-creds, deploy/minio, svc/minio
- `kubectl --context gcp-internal -n minio wait --for=condition=ready pod -l app=minio --timeout=120s` → condition met
- `kubectl --context gcp-internal apply -f repro/minio-make-bucket.yaml` (image: `minio/mc:latest`; first tried `RELEASE.2024-08-17T01-24-54Z` → not found for mc)
- `kubectl --context gcp-internal -n minio logs job/minio-make-bucket` → `Bucket created successfully local/mybucket` ✓

### Phase 2 — barman plugin into cnpg-system (additive)
- `helm --kube-context gcp-internal upgrade --install plugin-barman-cloud -n cnpg-system --create-namespace --wait ./charts/plugin-barman-cloud` → deployed, rev 1
- Post-checks:
  - `objectstores.barmancloud.cnpg.io` CRD present ✓
  - `plugin-barman-cloud` deploy 1/1 ✓
  - `internal-cloudnative-pg` (existing operator) still 1/1 ✓
  - `internal-postgres-cluster` still `Cluster in healthy state` ✓ (untouched)

### Phase 3 — manual reproduction in `repro-sb` namespace
Goal: inspect teardown in real time (chainsaw would auto-clean-up opaquely).
Repro values: `repro/sb-values.yaml` (HTTP endpoint override; tracked test files untouched).

- `kubectl --context gcp-internal create ns repro-sb`
- `helm --kube-context gcp-internal install scheduledbackups -n repro-sb --values repro/sb-values.yaml ./charts/cluster`
- `kubectl -n repro-sb wait cluster/scheduledbackups-cluster --for=jsonpath='{.status.phase}'=Cluster\ in\ healthy\ state --timeout=300s` → condition met (62s), pod 2/2 (postgres + plugin-barman-cloud)

**Diagnostics captured before cleanup:**
- pod `terminationGracePeriodSeconds` = **1800** (30 min) — CNPG default
- Cluster finalizers = (none)
- ObjectStore finalizers = (none)
- PVC finalizers = `kubernetes.io/pvc-protection` (standard)

**Cleanup (mirror of the chainsaw test) + timing:**
- 21:44:06 start → `delete scheduledbackups`, `delete backups`, `helm uninstall` → 21:44:17 helm returned
- Pod went `Terminating` → gone by ~21:46:30 (**~2m13s** graceful shutdown)
- 21:48:13 `delete ns repro-sb` → gone 21:48:52 (**39s**)

**Surprise (non-blocking):** `helm uninstall` does NOT delete the ObjectStore — it's a helm hook
(`helm.sh/hook: pre-install,pre-upgrade,pre-rollback`) with no `hook-delete-policy`. It lingers until
namespace GC removes it. It carries **no finalizer**, so it does NOT block namespace deletion (39s GC).

### Conclusion (definitive)
- On a well-resourced cluster (GKE 16-vCPU/64GB nodes), full cleanup = **~3m**, within budget.
- No stuck finalizer (Cluster/ObjectStore have none). No deterministic bug.
- The pod's graceful Postgres shutdown (~2–3m, bounded by `smartShutdownTimeout`) sits right at the 5m
  cleanup boundary **on GitHub runners** (kind-in-Docker on a constrained VM) → flaky >5m timeouts.
- `terminationGracePeriodSeconds=1800` is just the cap, not the actual shutdown time.
- **Fix:** raise `cleanup: 5m → 10m` in the scheduledbackups test (accommodate slow-runner shutdown).
  The graceful shutdown is correct; it just needs headroom on GitHub runners.

### Phase 4 — GKE cleanup (restore baseline)
- `helm --kube-context gcp-internal uninstall plugin-barman-cloud -n cnpg-system`
  → uninstalled (chart retained the `objectstores.barmancloud.cnpg.io` CRD via resource-policy: keep)
- `kubectl --context gcp-internal delete crd objectstores.barmancloud.cnpg.io` (no instances remained)
- `kubectl --context gcp-internal delete ns minio` → deleted
- `repro-sb` namespace already deleted during the reproduction run

**Post-cleanup verification (cluster == original baseline):**
- `cnpg-system` deploys → only `internal-cloudnative-pg` (1/1) ✓
- `barmancloud` CRD → NotFound ✓
- `minio` ns → NotFound ✓
- no `repro`/`chainsaw` namespaces ✓
- `postgres/internal-postgres-cluster` → still `Cluster in healthy state` (untouched) ✓
- `cert-manager` → 2/2 (untouched) ✓

**Net: gcp-internal left exactly as found. Nothing outside `minio`/`repro-sb`/the plugin deploy was created or modified.**



