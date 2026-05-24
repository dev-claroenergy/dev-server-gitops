# claro-ai-crm — Bootstrap Runbook

Everything a human or AI agent needs to bring the `claro-ai-crm` application fully online on a fresh cluster. Read it top to bottom — the steps are ordered and some depend on earlier ones.

---

## What ArgoCD handles automatically

Once the bootstrap (Tier 0–9 in the root README) is complete and the sealed-secrets master key is restored, ArgoCD automatically deploys and keeps in sync:

| What | Where in repo |
|---|---|
| Namespace `claro-ai-crm` | `apps/claro-ai-crm/overlays/home-server/namespace.yaml` |
| Web Deployment + Service (nginx, port 80) | `apps/claro-ai-crm/base/web.yaml` |
| API Deployment + Service (Node.js/Hono, port 3000) | `apps/claro-ai-crm/base/api.yaml` |
| Ingress with HTTPS (Traefik, self-signed cert) | `apps/claro-ai-crm/base/ingress.yaml` |
| TLS SealedSecret `claro-ai-crm-tls` | `apps/claro-ai-crm/overlays/home-server/claro-ai-crm-tls.sealedsecret.yaml` |
| DB credentials SealedSecret `claro-ai-crm-db` | `apps/claro-ai-crm/overlays/home-server/claro-ai-crm-db.sealedsecret.yaml` |
| App secrets SealedSecret `claro-ai-crm-env` | `apps/claro-ai-crm/overlays/home-server/claro-ai-crm-env.sealedsecret.yaml` |
| ConfigMap `claro-ai-crm-config` (non-secret env vars) | `apps/claro-ai-crm/overlays/home-server/claro-ai-crm-config.configmap.yaml` |
| ConfigMap `nginx-config` (routing template override) | `apps/claro-ai-crm/overlays/home-server/nginx-config.configmap.yaml` |

**Images are NOT managed by ArgoCD.** You must build and push them separately before the pods can start:
```
registry.claro-ai-crm.test/claro-ai-crm/api:dev-v0.0.2
registry.claro-ai-crm.test/claro-ai-crm/web:dev-v0.0.1
```

---

## Manual steps required after ArgoCD sync

These steps cannot be automated via GitOps and must be done by hand on every fresh cluster.

### Step 1 — CoreDNS: in-cluster DNS for `*.claro-ai-crm.test`

Pods inside the cluster cannot resolve `*.claro-ai-crm.test` by default because the domain only exists in `/etc/hosts` on workstations. The API pod must reach Authentik by hostname for OIDC discovery.

Create the CoreDNS custom ConfigMap (k3s auto-mounts it at `/etc/coredns/custom/`):

```bash
kubectl create configmap coredns-custom \
  --namespace=kube-system \
  --from-literal='claro-ai-crm.server=claro-ai-crm.test:53 {
    errors
    template IN A claro-ai-crm.test {
        answer "{{ .Name }} 60 IN A 10.43.167.97"
    }
    template IN AAAA claro-ai-crm.test {
        rcode NOERROR
    }
}' \
  --dry-run=client -o yaml | kubectl apply -f -
```

**Why the AAAA block:** Node.js makes both A and AAAA queries. Without the AAAA block, `template IN A` returns SERVFAIL for IPv6 queries, which causes the resolver to abort entirely even though the A record is correct.

**Get the Traefik ClusterIP** (replace `10.43.167.97` if it differs):
```bash
kubectl get svc -n kube-system traefik -o jsonpath='{.spec.clusterIP}'
```

CoreDNS reloads within 15 seconds (the `reload` directive). Verify:
```bash
kubectl run dns-test -n claro-ai-crm --restart=Never --image=curlimages/curl:8.6.0 \
  --command -- curl -s http://authentik.claro-ai-crm.test/application/o/claro-crm/.well-known/openid-configuration
sleep 8 && kubectl logs dns-test -n claro-ai-crm | head -5
kubectl delete pod dns-test -n claro-ai-crm --force
```
Expected: JSON with `"issuer": "http://authentik.claro-ai-crm.test/..."`.

### Step 2 — Authentik: create OIDC provider and application

The Authentik OIDC configuration is not managed in git (it lives in Authentik's own database). Recreate it via the API using the bootstrap token from the `authentik-env` sealed secret.

```bash
TOKEN=$(kubectl get secret authentik-env -n authentik \
  -o jsonpath='{.data.AUTHENTIK_BOOTSTRAP_TOKEN}' | base64 -d)
```

**Get the authorization and invalidation flow UUIDs:**
```bash
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://authentik.claro-ai-crm.test/api/v3/flows/instances/" | \
  python3 -c "
import json,sys
for f in json.load(sys.stdin)['results']:
    if f['slug'] in ['default-provider-authorization-implicit-consent','default-provider-invalidation-flow']:
        print(f['slug'], f['pk'])
"
```

**Create the OAuth2 provider:**
```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "http://authentik.claro-ai-crm.test/api/v3/providers/oauth2/" \
  -d '{
    "name": "claro-crm",
    "client_id": "claro-crm",
    "client_secret": "<OIDC_CLIENT_SECRET from password manager>",
    "authorization_flow": "<implicit-consent UUID>",
    "invalidation_flow": "<invalidation UUID>",
    "redirect_uris": [
      {"matching_mode": "strict", "url": "https://claro-ai-crm.claro-ai-crm.test/api/auth/callback", "redirect_uri_type": "authorization"},
      {"matching_mode": "strict", "url": "https://claro-ai-crm.claro-ai-crm.test/", "redirect_uri_type": "logout"}
    ],
    "sub_mode": "user_username",
    "include_claims_in_id_token": true
  }'
# Note the pk from the response — needed for the next step
```

**Create the Application:**
```bash
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "http://authentik.claro-ai-crm.test/api/v3/core/applications/" \
  -d '{
    "name": "Claro AI CRM",
    "slug": "claro-crm",
    "provider": <pk from above>,
    "meta_launch_url": "https://claro-ai-crm.claro-ai-crm.test/"
  }'
```

**Verify the OIDC discovery endpoint:**
```bash
curl -s "http://authentik.claro-ai-crm.test/application/o/claro-crm/.well-known/openid-configuration" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); print('issuer:', d['issuer'])"
# Expected: issuer: http://authentik.claro-ai-crm.test/application/o/claro-crm/
```

### Step 3 — Database: run Prisma migrations

The database schema is NOT applied automatically. Prisma migrations must be run from the `@claro/db` package in the application repo against the cluster database.

**Prerequisites:** the `postgres-rw` service must be port-forwarded:
```bash
# Terminal 1 — keep open
kubectl port-forward -n postgres pod/postgres-1 5433:5432
```

**Run migrations from the app repo** (`packages/db` or wherever `@claro/db` lives):
```bash
# Terminal 2
DATABASE_URL="postgresql://claro-ai-crm:<password>@localhost:5433/claro-ai-crm?schema=public&sslmode=disable" \
  pnpm exec prisma migrate deploy
```

Get the password: `kubectl get secret claro-ai-crm-db -n claro-ai-crm -o jsonpath='{.data.password}' | base64 -d`

**Why `sslmode=disable`:** CNPG has SSL enabled (`ssl = on`) but `pg_hba.conf` uses `host` (not `hostssl`), so non-SSL connections are allowed. Without `sslmode=disable`, the Prisma client's SSL negotiation causes a "connection reset by peer" error through the port-forward.

**Why port-forward to the pod (not the service):** `kubectl port-forward svc/postgres-rw` can fail with connection reset errors. Port-forwarding directly to `pod/postgres-1` is more reliable.

**Verify:**
```bash
kubectl exec -n postgres postgres-1 -- psql -U postgres -d claro-ai-crm -c "\dt public.*"
# Expected: User, Session, Lead, ImportBatch, SourceMapping, _prisma_migrations
```

### Step 4 — Set admin role for the first user

The app provisions OIDC users with `BACK_OFFICE` role by default. The first admin user must be promoted manually.

```bash
# Port-forward (same as Step 3, use 5433 to avoid local postgres conflicts)
kubectl port-forward -n postgres pod/postgres-1 5433:5432

# Update role
psql "postgresql://claro-ai-crm:<password>@localhost:5433/claro-ai-crm?sslmode=disable" \
  -c "UPDATE public.\"User\" SET role = 'ADMIN' WHERE email = '<your email>' RETURNING id, email, role;"
```

After updating, log out and back in so the new session picks up the new role.

---

## Workstation setup (one-time per machine)

### Registry CA cert (Docker)

```bash
sudo mkdir -p /etc/docker/certs.d/registry.claro-ai-crm.test
kubectl get secret registry-tls -n registry -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  sudo tee /etc/docker/certs.d/registry.claro-ai-crm.test/ca.crt >/dev/null
```

### App CA cert (browser HTTPS)

```bash
# Extract the cert
kubectl get secret claro-ai-crm-tls -n claro-ai-crm -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  sudo tee /usr/local/share/ca-certificates/claro-ai-crm.crt >/dev/null
sudo update-ca-certificates
```

### DNS (/etc/hosts)

```
<GCP_EXTERNAL_IP>  argocd.claro-ai-crm.test
<GCP_EXTERNAL_IP>  authentik.claro-ai-crm.test
<GCP_EXTERNAL_IP>  registry.claro-ai-crm.test
<GCP_EXTERNAL_IP>  claro-ai-crm.claro-ai-crm.test
```

Get the external IP: `kubectl get svc -n kube-system traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`

---

## k3s node setup (one-time per cluster host)

SSH to the node and configure containerd to pull from the private registry:

```bash
# Copy the registry CA cert to the node
kubectl get secret registry-tls -n registry -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  ssh <user>@<node-ip> "sudo tee /etc/rancher/k3s/registry-ca.crt >/dev/null"

# Add hostname to /etc/hosts on the node so containerd can resolve it
ssh <user>@<node-ip> "echo '127.0.0.1 registry.claro-ai-crm.test' | sudo tee -a /etc/hosts"

# Write the containerd registry config
ssh <user>@<node-ip> "sudo tee /etc/rancher/k3s/registries.yaml >/dev/null <<'EOF'
configs:
  \"registry.claro-ai-crm.test\":
    auth:
      username: homelab
      password: <registry password from password manager>
    tls:
      ca_file: /etc/rancher/k3s/registry-ca.crt
EOF"

# Restart k3s to apply
ssh <user>@<node-ip> "sudo systemctl restart k3s"
```

---

## Known gotchas

### Frontend calls `/me` without `/api/` prefix

The web frontend (`claro-ai-crm/web`) was built with `VITE_API_BASE_URL` unset, so it calls `/me`, `/auth/*` directly — not `/api/me`. The nginx config is overridden via `nginx-config` ConfigMap (mounted at `/etc/nginx/templates/default.conf.template`) to proxy these paths to the API. If you see the login loop again, check that the ConfigMap is mounted correctly in the web pod.

### Traefik `pathType: Exact` loses to `pathType: Prefix /`

Traefik v3 auto-computes route priority by rule string length. `PathPrefix(/)` is longer than `Path(/me)`, so the wildcard route wins. Do not attempt to fix this in the Ingress — use nginx to own the routing instead (see above).

### `NODE_ENV=development` required for HTTP OIDC issuer

The API enforces HTTPS for `OIDC_ISSUER` in production. Since Authentik runs HTTP-only on this cluster, `NODE_ENV=development` bypasses the check. It is set in `claro-ai-crm-config` ConfigMap. Remove it when Authentik gets a TLS ingress.

### Pods don't pick up ConfigMap changes automatically

Kubernetes does not restart pods when a ConfigMap changes. After any ConfigMap update in this namespace, do a full rollout restart:
```bash
kubectl rollout restart deployment/api deployment/web -n claro-ai-crm
```

### Session cookie loop (login keeps redirecting)

Symptom: login succeeds, sessions appear in the `Session` table, but browser loops back to login.

Diagnosis checklist:
1. `kubectl exec -n claro-ai-crm deploy/api -- env | grep NODE_ENV` → must be `development`
2. `curl -s https://claro-ai-crm.claro-ai-crm.test/me` → must return `401 application/json`, NOT `200 text/html`
3. `curl -s http://authentik.claro-ai-crm.test/application/o/claro-crm/.well-known/openid-configuration` from inside cluster → must return JSON

If `/me` returns HTML, the nginx routing is broken — check that `nginx-config` ConfigMap is mounted.
If the discovery endpoint fails, CoreDNS custom config is missing or broken.

---

## Connecting to the database from your workstation

Use `kubectl port-forward` to access postgres directly for admin tasks (migrations, role updates, schema inspection).

**Important:** always forward to the **pod** (not the service) and use a non-standard local port (5433) to avoid conflicts with a locally running postgres. Also require `sslmode=disable` — CNPG has SSL on but `pg_hba.conf` uses `host` (not `hostssl`), so non-SSL connections are valid; the SSL negotiation through the port-forward causes "connection reset by peer" otherwise.

**Get the pod name and credentials:**
```bash
kubectl get pod -n postgres -l cnpg.io/instanceRole=primary
# e.g. postgres-1

PG_PASSWORD=$(kubectl get secret claro-ai-crm-db -n claro-ai-crm \
  -o jsonpath='{.data.password}' | base64 -d)
```

**Open the port-forward in a dedicated terminal and keep it running:**
```bash
# Terminal 1 — leave open for the duration of your session
kubectl port-forward -n postgres pod/postgres-1 5433:5432
# Output: Forwarding from 127.0.0.1:5433 -> 5432
```

**Connect with psql (Terminal 2):**
```bash
psql "postgresql://claro-ai-crm:${PG_PASSWORD}@localhost:5433/claro-ai-crm?sslmode=disable"
```

**Connect with any GUI tool (TablePlus, DBeaver, DataGrip, etc.):**
```
Host:     localhost
Port:     5433
Database: claro-ai-crm
Username: claro-ai-crm
Password: <from above>
SSL:      disable / prefer (not require)
```

**Connect as superuser** (for admin tasks like `CREATE EXTENSION`, inspecting all schemas):
```bash
PG_SUPER=$(kubectl get secret postgres-superuser -n postgres \
  -o jsonpath='{.data.password}' | base64 -d)
psql "postgresql://postgres:${PG_SUPER}@localhost:5433/claro-ai-crm?sslmode=disable"
```

**Run Prisma migrations** (from the `@claro/db` package directory in the app repo):
```bash
DATABASE_URL="postgresql://claro-ai-crm:${PG_PASSWORD}@localhost:5433/claro-ai-crm?schema=public&sslmode=disable" \
  pnpm exec prisma migrate deploy
```

**Check migration status:**
```bash
DATABASE_URL="postgresql://claro-ai-crm:${PG_PASSWORD}@localhost:5433/claro-ai-crm?schema=public&sslmode=disable" \
  pnpm exec prisma migrate status
```

---

## Images: build and push

```bash
# Tag and push (example — actual tags depend on your build pipeline)
docker build -t registry.claro-ai-crm.test/claro-ai-crm/api:dev-v0.0.2 ./apps/api
docker push registry.claro-ai-crm.test/claro-ai-crm/api:dev-v0.0.2

docker build -t registry.claro-ai-crm.test/claro-ai-crm/web:dev-v0.0.1 ./apps/web
docker push registry.claro-ai-crm.test/claro-ai-crm/web:dev-v0.0.1
```

After pushing a new image tag, update the tag in the relevant `base/*.yaml` and push to git. ArgoCD deploys automatically. Then restart the deployment to prevent stuck pods:
```bash
kubectl rollout restart deployment/api -n claro-ai-crm  # or /web
```
