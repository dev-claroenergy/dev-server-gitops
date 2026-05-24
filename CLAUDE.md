# dev-server-gitops

GitOps repo for a single-node k3s cluster (`claro-ai-crm-dev`) running on GCP, used as a **stable, production-shaped dev environment** — no "it's only dev" shortcuts. ArgoCD pulls from this repo and reconciles cluster state.

## How to work in this repo (read before editing)

- **Production-shaped, not production-scale.** Apply *stability* conventions immediately (proper layout, secrets tooling, sync waves, `ServerSideApply`, resource limits). Defer *scale* conventions until needed (HA replicas, ApplicationSets, multi-cluster generators) — on a single node they're YAGNI.
- **Don't propose "we'll fix it later when it matters."** Doing it right the first time is the point.
- **One concern per commit.** Don't bundle a config fix with a refactor. Atomic commits keep `git log` readable and ArgoCD diffs clean.
- **After any ConfigMap or Secret change, restart the affected Deployment.** Kubernetes does not restart pods automatically when ConfigMaps change. Always run `kubectl rollout restart deployment/<name> -n <ns>` after pushing.
- **kubeseal binary lives in the repo root** (`./kubeseal`). It is not installed system-wide. Use `./kubeseal` or the full path when sealing secrets.

## Architecture

App-of-apps pattern, split into two layers governed by two `AppProject` boundaries:

- `bootstrap/` holds the manifests you `kubectl apply` after ArgoCD itself is installed.
  - `bootstrap/projects/infrastructure.yaml` — `AppProject` "infrastructure". Allows cluster-scoped resources, broader source repos.
  - `bootstrap/projects/apps.yaml` — `AppProject` "apps". **No** cluster-scoped resources allowed; this repo only as source.
  - `bootstrap/infrastructure.yaml` — root Application, `project: infrastructure`, watches `infrastructure/` recursively.
  - `bootstrap/apps.yaml` — root Application, `project: apps`, watches `apps/` recursively.
- `infrastructure/` — platform layer. Operators, controllers, CNI, storage, ingress, cert-manager, secrets controller. Provides *capabilities and CRDs*.
- `apps/` — workload layer. User-facing services and resources that *consume* infrastructure's capabilities.

**Bootstrap order:** AppProjects MUST be applied before any Application that references them. Apply `bootstrap/projects/` first, then the two root manifests.

Each subdirectory under `infrastructure/<name>/` or `apps/<name>/` holds one `application.yaml` (an ArgoCD `Application` CR). Manifest-based apps use `base/` + `overlays/<cluster>/` kustomize layout.

Current state:
- `infrastructure/cloudnative-pg/` — CNPG operator, Helm chart from `cloudnative-pg.github.io/charts`, namespace `cnpg-system`.
- `infrastructure/sealed-secrets/` — Bitnami sealed-secrets controller in `kube-system` (renamed to `sealed-secrets-controller` so `kubeseal` defaults work).
- `infrastructure/registry/` — private Docker registry (`registry:2`) at `https://registry.claro-ai-crm.test`. htpasswd auth, TLS via self-signed cert sealed in the overlay.
- `apps/postgres/` — a `postgresql.cnpg.io/v1` `Cluster` in namespace `postgres`, backing Authentik and claro-ai-crm. Kustomize layout: `base/` + `overlays/home-server/`. Includes `Database` CRs for `authentik` and `claro-ai-crm`.
- `apps/authentik/` — SSO/identity provider, Helm chart from `charts.goauthentik.io`, namespace `authentik`. Multi-source Application (chart + values ref + supplementary kustomize overlay containing the SealedSecret, a bundled redis Deployment/Service, and an explicit Namespace). Exposed via Traefik at `http://authentik.claro-ai-crm.test` (HTTP only).
- `apps/claro-ai-crm/` — main application. Two containers: `api` (Hono/Node.js, port 3000) and `web` (nginx serving a Vue SPA, port 80). HTTPS via self-signed cert. OIDC login via Authentik. Database via CNPG postgres cluster.

## Hostname & Ingress

k3s ships Traefik in `kube-system` as the default ingress controller (LoadBalancer service `kube-system/traefik`).

**Cluster ingress IP:** the external IP of the Traefik LoadBalancer service. Verify with:
```bash
kubectl get svc -n kube-system traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

**Traefik ClusterIP** (used for CoreDNS in-cluster resolution): `10.43.167.97`. Verify: `kubectl get svc traefik -n kube-system -o jsonpath='{.spec.clusterIP}'`

**Hostname convention:** `<app>.claro-ai-crm.test`. DNS is provided via `/etc/hosts` entries on each workstation pointing to the node's external IP. DNS provisioning is out of scope for this repo.

**TLS:** The registry and claro-ai-crm use HTTPS with self-signed certs (SealedSecrets in the respective overlays). Authentik is HTTP-only. No cert-manager.

**To expose a new HTTP app:**
1. In the app's chart values (or `Ingress` manifest), set `ingressClassName: traefik` and `hosts: [<app>.claro-ai-crm.test]`.
2. Add the hostname to `/etc/hosts` on each workstation.

No need to touch the AppProject — `Ingress` is namespaced, allowed by `apps`' `namespaceResourceWhitelist: '*'`.

**To expose a TCP service (e.g. Postgres):** Use a `Service` of `type: LoadBalancer`. Example: `apps/postgres/overlays/home-server/postgres-rw-external.yaml` exposes the CNPG primary on `5432`. The external IP is the node IP — no separate DNS entry needed if connecting by IP.

Don't modify operator-managed services. Always create a *new* Service that selects the same pods. CNPG owns `postgres-rw`/`postgres-ro`/`postgres-r` (ClusterIP); we add `postgres-rw-external` (LoadBalancer) alongside.

**Traefik path priority gotcha:** Traefik v3 computes route priority by rule string length. `PathPrefix(/)` (longer string) beats `Path(/me)` (shorter string) — meaning a catch-all `/` Prefix route wins over a more specific Exact route. **Do not try to route specific API paths via Ingress path rules.** Instead, use nginx to own all routing decisions within the web container (see claro-ai-crm nginx-config ConfigMap pattern).

**In-cluster DNS for `*.claro-ai-crm.test`:** pods inside the cluster cannot resolve these hostnames by default (they only exist in workstation `/etc/hosts`). A CoreDNS custom ConfigMap resolves them to the Traefik ClusterIP. See `docs/runbooks/claro-ai-crm-bootstrap.md` Step 1. **Always include both A and AAAA templates** — without the AAAA template, Node.js resolvers get SERVFAIL and abort even though the A record is correct.

## Sync ordering

ArgoCD provides three ordering mechanisms; pick the right one for the right problem:

1. **Sync phases** (`PreSync` / `Sync` / `PostSync`) — for hooks like one-shot migration Jobs. Rarely used here.
2. **Sync waves** (`argocd.argoproj.io/sync-wave: "N"` annotation) — orders resources within a single parent's sync. Lower = earlier. **Only works inside one parent's tree.** Wave annotations on resources in `apps/` cannot order them against resources in `infrastructure/` — those are two independent parents.
3. **Sync options** — per-resource behavior toggles. Most important for cross-tree CRD races:
   - `SkipDryRunOnMissingResource=true` — workloads tolerate "CRD not found" during dry-run and retry the apply after the CRD arrives. Required on any Application that consumes a CRD installed by a different root (e.g. `apps/postgres` consuming the CNPG operator's CRDs from `infrastructure/cloudnative-pg`).

**Within the `apps/` tree:** sync-wave orders Applications relative to each other (`postgres` is wave 1; an Authentik app that depends on postgres would be wave 2; etc.).
**Across `apps/` and `infrastructure/`:** waves don't help. Use `SkipDryRunOnMissingResource=true` and let ArgoCD retries handle eventual consistency.

## Conventions

- One Application per directory under `apps/` or `infrastructure/`. Name the file `application.yaml`.
- Decide infra vs. app by *lifecycle*, not by domain: the CNPG **operator** is infra, but the **`Cluster` CR** that uses CNPG is an app. CRDs and controllers live in `infrastructure/`; consumers of those CRDs live in `apps/`.
- Every Application's `spec.project` matches the directory it lives in: `infrastructure` for `bootstrap/infrastructure.yaml` and everything under `infrastructure/`; `apps` for `bootstrap/apps.yaml` and everything under `apps/`. **Never use `project: default`** — it's a wide-open free-for-all.
- The `apps` AppProject allows exactly one cluster-scoped resource: `Namespace` (no group). Apps that ship namespaced RBAC (Role/RoleBinding) must declare their own `Namespace` with `argocd.argoproj.io/sync-wave: "-1"` to dodge ArgoCD's `rbacReconcile`-before-`CreateNamespace` race. **ClusterRole/ClusterRoleBinding/CRDs/etc. remain forbidden** in this project; those belong in `infrastructure/`.
- When adding a new app that pulls a remote Helm chart, add the chart repo URL to the appropriate AppProject's `sourceRepos` list, or the sync will fail with a sourceRepos validation error.
- **Manifest-based apps** use kustomize layering (`base/` + `overlays/<cluster>/`):
  ```
  apps/<name>/
    application.yaml              # spec.source.path: apps/<name>/overlays/<cluster>
    base/
      kustomization.yaml          # resources: [the universal manifests]
      <resource>.yaml
    overlays/
      <cluster>/                  # named after the target cluster, not "dev"/"prod"
        kustomization.yaml        # resources: [../../base, plus cluster-specific]
        <cluster-specific>.yaml   # SealedSecrets, patches, etc.
  ```
  The Application's `spec.source.path` points at the overlay, never at the base. ArgoCD auto-detects `kustomization.yaml` and runs kustomize.

- **Chart-based apps** use a multi-source Application — no `base/` because the Helm chart IS the universal layer:
  ```
  apps/<name>/
    application.yaml              # spec.sources: chart + repo-ref + repo-path
    overlays/
      <cluster>/
        kustomization.yaml        # resources: [supplementary resources only]
        values.yaml               # Helm values, referenced via $values ref;
                                  # NOT a k8s resource, NOT listed under resources:
        *.sealedsecret.yaml       # supplementary SealedSecrets, etc.
  ```
  The Application has three sources: the chart, a `ref: values` source pointing at this repo (so `$values` resolves), and a path source pointing at the overlay. ArgoCD renders chart + kustomize together as one Application.
- SealedSecrets live in the overlay, not the base. They're encrypted to a specific cluster's master key, so they're inherently cluster-specific.
- Before committing kustomize changes, sanity-check the build: `kubectl kustomize apps/<name>/overlays/<cluster>/`. Catches malformed references before ArgoCD does.
- All Applications use `automated.prune: true` + `automated.selfHeal: true`. Use `ServerSideApply=true` for anything with CRDs or large objects.
- Use `sync-wave` annotations when one Application depends on CRDs or services from another (e.g. `postgres` waits for the CNPG operator).
- Set `revisionHistoryLimit: 3` on all Deployments. Kubernetes default is 10, which causes dead ReplicaSets to accumulate during active development. 3 is enough for rollback.
- **Secrets use Sealed-Secrets.** Never commit a raw `Secret` manifest. Always go through `kubeseal` to produce a `SealedSecret` CR — those ARE committable. Naming convention: `<secret-name>.sealedsecret.yaml`, colocated in the consuming app's kustomize **overlay** (e.g. `apps/postgres/overlays/home-server/postgres-authentik.sealedsecret.yaml`) — not the base, since SealedSecrets are cluster-specific.
- Sealed-Secrets are **strict-scoped by default** — encrypted for the exact `namespace/name` pair. Renaming or moving the `SealedSecret` to a different namespace breaks decryption. This is the security property we want; don't override it without a reason.
- The sealed-secrets controller's master key lives in `kube-system` (selector: `sealedsecrets.bitnami.com/sealed-secrets-key`). It's the only thing that can decrypt SealedSecrets in this repo — backed up to the owner's password manager. Restoration procedure in `README.md`.
- The sealed-secrets Helm chart sets `fullnameOverride: sealed-secrets-controller` so the deployment/service names match `kubeseal`'s compiled-in defaults (`sealed-secrets-controller` in `kube-system`). Without this, every `kubeseal` invocation needs `--controller-name=sealed-secrets --controller-namespace=kube-system` flags. Don't remove the override.
- Storage class is `local-path` (k3s built-in). Single-node — no replication, no HA yet.
- The Postgres cluster has `enableSuperuserAccess: true` so the `postgres` superuser can connect remotely via the LoadBalancer for admin tasks (CREATE DATABASE, CREATE EXTENSION). CNPG defaults this to `false` for safety; we override because the cluster is LAN-only and the password is strong. Application workloads must still use per-app accounts (`authentik`, etc.), not the superuser.

## Private Docker registry

`infrastructure/registry/` runs `registry:2` exposed at `https://registry.claro-ai-crm.test` via Traefik. htpasswd auth (username: `homelab`), sealed creds in `infrastructure/registry/overlays/home-server/registry-auth.sealedsecret.yaml`. TLS terminated by a self-signed cert (10-year validity, CN=`registry.claro-ai-crm.test`) stored in `registry-tls.sealedsecret.yaml`; private key never leaves the cluster.

**Two pieces of host-level state live outside this repo** (one-time setup, documented in `README.md`):
- Workstation: the registry's CA cert at `/etc/docker/certs.d/registry.claro-ai-crm.test/ca.crt`. Extract from cluster: `kubectl get secret registry-tls -n registry -o jsonpath='{.data.tls\.crt}' | base64 -d`. Docker auto-discovers it; no daemon restart needed.
- k3s host: `/etc/rancher/k3s/registries.yaml` with `auth.username` / `auth.password` and `tls.ca_file` pointing to the same CA cert (also copied to the host). Also requires `127.0.0.1 registry.claro-ai-crm.test` in the node's `/etc/hosts` so containerd can resolve the hostname. Containerd pulls authenticated automatically — **no per-namespace imagePullSecrets needed.**

**Build → push → deploy workflow:** `docker build -t registry.claro-ai-crm.test/<app>:<tag>` → `docker push registry.claro-ai-crm.test/<app>:<tag>` → reference `image: registry.claro-ai-crm.test/<app>:<tag>` in any Deployment manifest under `apps/`. The cluster pulls via the containerd config.

**Storage**: 20 GiB PVC on `local-path`. Inspect usage with `kubectl exec -n registry deploy/registry -- du -sh /var/lib/registry`. When it fills up, bump the PVC size and let CSI resize.

**Image deletion**: `REGISTRY_STORAGE_DELETE_ENABLED=true` is set, so `curl -X DELETE` against the v2 API works. The actual disk space is reclaimed only after running `registry garbage-collect` — schedule a CronJob if this becomes a problem.

## Adding a database for a new app

Use CNPG's declarative primitives — never manual `psql` against the cluster for things git should own.

1. Generate a password (`openssl rand -base64 24 | tr -d '/+=' | head -c 32`) and save to your password manager.
2. Seal it into the **postgres namespace** as `postgres-<appname>.sealedsecret.yaml` under `apps/postgres/overlays/home-server/`. Type `kubernetes.io/basic-auth`; keys `username` + `password`.
3. Add a `managed.roles[]` entry to `apps/postgres/base/cluster.yaml` referencing the SealedSecret name. Set `login: true` and `ensure: present`. Don't grant `superuser`/`createdb`/`createrole` unless the app actually needs them.
4. Add a `Database` CR under `apps/postgres/base/databases/<appname>.yaml`, `owner: <appname>`, `cluster.name: postgres`, `ensure: present`. Add it to `apps/postgres/base/kustomization.yaml` resources list.
5. Seal a *second* SealedSecret in the consuming app's namespace with the same password formatted for that app's env (`DATABASE_URL`, `<APP>_DB_PASSWORD`, etc.) Two SealedSecrets, one password — the price of namespace isolation. If this duplication starts to hurt, install Reflector to mirror Secrets across namespaces; until then, manual is fine.
6. Commit. CNPG reconciles: creates/alters the role with the right password, then creates the database. The new app picks up its credential from its own namespace Secret.

**Never** manually `CREATE USER` or `CREATE DATABASE` against the cluster from psql — git becomes the lying source of truth, and the next cluster rebuild loses everything.

## claro-ai-crm application specifics

The `apps/claro-ai-crm/` app has patterns not used elsewhere in this repo. Read this before touching it.

**Two-container layout:**
- `api` — Hono/Node.js backend, port 3000. Handles all business logic, OIDC callback, session management, database queries via Prisma.
- `web` — nginx serving a pre-built Vue SPA, port 80. Proxies API calls to the `api` service internally.

**nginx routing override:** the Vue frontend was built without `VITE_API_BASE_URL` set, so it calls `/me`, `/auth/*`, `/health` directly (no `/api/` prefix). The nginx container's default template only proxied `/api/*`. To fix this without rebuilding the image, a `nginx-config` ConfigMap overrides `/etc/nginx/templates/default.conf.template` — nginx's entrypoint runs `envsubst` on this file at startup. The ConfigMap is in `overlays/home-server/nginx-config.configmap.yaml`. **Do not attempt to fix the routing via Traefik Ingress path rules** — Traefik v3 path priority makes this unreliable (see Traefik gotcha in the Hostname section).

**HTTPS:** the app uses a self-signed cert for `claro-ai-crm.claro-ai-crm.test` stored in `claro-ai-crm-tls.sealedsecret.yaml`. The CA cert must be installed on workstations (`/usr/local/share/ca-certificates/claro-ai-crm.crt`).

**OIDC / Authentik:** the app integrates with Authentik via OIDC. The provider and application in Authentik are **not managed in git** — they live in Authentik's database and must be recreated manually on a fresh cluster. See `docs/runbooks/claro-ai-crm-bootstrap.md` Step 2. Key env vars:
- `OIDC_ISSUER` — must be HTTP (Authentik has no TLS). `NODE_ENV=development` bypasses the app's HTTPS enforcement on the issuer.
- `OIDC_REDIRECT_URI` — must be HTTPS (`https://claro-ai-crm.claro-ai-crm.test/api/auth/callback`) to match the Authentik provider config.

**Database migrations:** Prisma migrations are NOT applied automatically by ArgoCD. On a fresh cluster, run `prisma migrate deploy` via port-forward. See `docs/runbooks/claro-ai-crm-bootstrap.md` Step 3. Use `sslmode=disable` and forward to the **pod** (not the service) to avoid SSL negotiation errors.

**Session cookie:** the app sets a `claro_session` HttpOnly cookie. `secure` is set to `false` when `NODE_ENV=development`. The frontend checks auth by calling `GET /me` — if this returns HTML instead of JSON, the nginx routing is broken.

**Manual steps required on every fresh cluster** (not handled by ArgoCD):
1. CoreDNS custom ConfigMap for `*.claro-ai-crm.test` in-cluster resolution
2. Authentik OIDC provider + application creation
3. `prisma migrate deploy` via port-forward
4. First user role promotion: `UPDATE "User" SET role = 'ADMIN' WHERE email = '...'`

## Repo URL

`https://github.com/dev-claroenergy/dev-server-gitops.git` — referenced from Application `repoURL` fields.

**When forking this repo**, update the URL in **four** places — all must match or ArgoCD rejects the Applications with `InvalidSpecError: repo not permitted`:
1. `bootstrap/projects/infrastructure.yaml` — `sourceRepos`
2. `bootstrap/projects/apps.yaml` — `sourceRepos`
3. `bootstrap/infrastructure.yaml` — `spec.source.repoURL`
4. `bootstrap/apps.yaml` — `spec.source.repoURL`

Files 1–2 and 3–4 are independently applied with `kubectl apply`. Re-apply all four after updating, then force a hard refresh on the root Applications.

## Validation

There is no CI yet. Before committing:

- `kubectl apply --dry-run=client -f <file>` for raw manifests.
- For Application files, use `kubectl explain application.spec.source.directory` (and similar) to confirm field names. **ArgoCD silently ignores unknown keys** — there is no schema validation error for typos. Known traps we've hit in this repo:
  - `heml:` instead of `helm:` (the values block is skipped, chart defaults apply)
  - `recursive: true` instead of `recurse: true` (recursion never happens; root scans only the top level)
- For Helm chart values, **the chart's deprecation messages tell you a field is gone but not what to use instead.** Always sanity-check values structure against the *current* chart schema before pushing:
  ```bash
  helm repo add <name> <url>
  helm repo update
  helm show values <name>/<chart> | less
  ```
  Known traps we've hit:
  - Authentik chart 2026.x: top-level `envFrom:`, `env:`, `envValueFrom:`, `ingress:` are all deprecated. Use `authentik.existingSecret.secretName` for credentials (single Secret with ALL `AUTHENTIK_*` env vars including non-secret ones like DB host); use `server.ingress.*` for ingress. The chart also dropped its bundled redis subchart, so the workload must provide its own — we ship a minimal `redis.yaml` in the overlay.
  - When a chart provisions cluster-scoped RBAC (ClusterRole/ClusterRoleBinding) that the `apps` AppProject refuses, look for a chart toggle to disable it. The Authentik chart's outpost RBAC is toggled off via `serviceAccount.clusterRole.enabled: false`. Don't loosen the AppProject guardrail to accommodate a chart — find the toggle.
- After applying any Application change to the cluster, diff the live spec against git: `kubectl get application <name> -n argocd -o yaml | yq '.spec.source' | diff - <(yq '.spec.source' <file>)`. The cluster and the file should be byte-identical; if not, something patched the live resource without a commit.

## Root Application discovery rule

Root Applications (those in `bootstrap/`) MUST set `directory.include` to filter for only Application files. Otherwise `recurse: true` pulls in every YAML under the tree — including the workload manifests that child Applications are *supposed* to deploy — and the root tries to apply them directly. Use:

```yaml
directory:
  recurse: true
  include: '{application.yaml,**/application.yaml}'
```

Child Applications then deploy their own manifest trees via their own `spec.source.path`, which is not scanned by the root.

## Boundaries

- Don't commit raw `Secret` kinds with populated `data`/`stringData`. Use `SealedSecret` instead — see the secrets convention above.
- Don't commit kubeconfigs, kubeconfig fragments, or the sealed-secrets master key file. The master key file is never long-lived on disk; it goes straight to the password manager after `kubectl get`.
- Don't change `spec.destination.server` from `https://kubernetes.default.svc` unless you're intentionally targeting a remote cluster.
- Don't disable `prune` or `selfHeal` without a reason noted in the commit message — the whole point of this repo is that git is the source of truth.
