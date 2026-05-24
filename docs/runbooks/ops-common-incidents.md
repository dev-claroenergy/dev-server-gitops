# Operational runbook — common incidents

Fixes for things that break during normal operation. Each section follows the same pattern: **symptom → root cause → fix → verify**.

---

## Node: DiskPressure taint / pods stuck Pending

**Symptom:** pods show `Pending` with event `0/1 nodes are available: 1 node(s) had untolerated taint(s)`. `kubectl describe node` shows `DiskPressure: True`.

**Root cause:** kubelet detected disk usage above its eviction threshold on the 10 GiB (or whatever the current size is) node filesystem. Common triggers: large image pulls all at once, log accumulation, container layer extraction.

**Check actual disk state (from workstation via kubelet stats API):**
```bash
kubectl get --raw /api/v1/nodes/<node-name>/proxy/stats/summary | python3 -c "
import json,sys
fs = json.load(sys.stdin)['node']['fs']
cap = fs['capacityBytes']; avail = fs['availableBytes']
print(f'{avail/1024**3:.2f} GiB free / {cap/1024**3:.1f} GiB total ({avail/cap*100:.1f}% free)')
"
```

**If disk is genuinely full:** free space before proceeding (remove unused images, clean logs, resize disk — see below). Do not remove the taint while disk is still full.

**If disk has recovered but taint persists:** the kubelet has a 5-minute `eviction-pressure-transition-period` before it removes a pressure condition. Wait it out, or remove manually once you've confirmed available space is healthy:
```bash
kubectl taint node <node-name> node.kubernetes.io/disk-pressure:NoSchedule-
```

**Verify:**
```bash
kubectl get node <node-name> -o jsonpath='{.spec.taints}'
# Expected: empty or no disk-pressure entry
kubectl get pods -n argocd  # should show Running pods returning
```

---

## Node: resize GCP disk

**When:** node disk is too small (symptom: DiskPressure, or `df -h` on the node shows <20% free).

**Step 1 — resize the disk in GCP** (from your workstation):
```bash
gcloud compute disks resize <vm-name> --size=50GB --zone=<zone> --project=<project>
```

**Step 2 — expand the partition on the node** (SSH to the node):
```bash
lsblk  # confirm new disk size is visible at the block level (e.g. 50G on sda)
sudo parted /dev/sda resizepart 1 100%
# If growpart is not available, parted handles it directly
```

**Step 3 — resize2fs is usually already done by GCP guest agent.** Verify:
```bash
df -h /
# If capacity still shows old size:
sudo resize2fs /dev/sda1
```

**Step 4 — remove the stale taint** (from workstation):
```bash
kubectl taint node <node-name> node.kubernetes.io/disk-pressure:NoSchedule-
```

---

## ArgoCD: applications not showing / lost after restart

**Symptom:** ArgoCD UI is empty or shows no applications after a fresh install or cluster restart.

**Root cause:** the two root `Application` CRs in `bootstrap/` were never applied (or were lost). AppProjects exist but Applications do not.

**Check:**
```bash
kubectl get applications -n argocd
# If empty, the root Applications are missing
kubectl get appprojects -n argocd
# If infrastructure + apps exist, projects are fine — only applications are missing
```

**Fix — always apply in this order:**
```bash
# Projects first (must exist before any Application that references them)
kubectl apply -f bootstrap/projects/

# Then root Applications
kubectl apply -f bootstrap/infrastructure.yaml -f bootstrap/apps.yaml
```

Wait ~60 seconds for ArgoCD to reconcile, then verify all child Applications appear.

---

## ArgoCD: apps synced to wrong repo (after fork)

**Symptom:** commits to the new repo don't reflect in the cluster. ArgoCD is synced to an old commit SHA that doesn't exist in the current repo.

**Root cause:** when the repo is forked, three separate places hold the old URL. All three must be updated:
1. `application.yaml` files — updated via git commit, ArgoCD picks up automatically
2. `bootstrap/projects/` AppProjects — manually applied, NOT reconciled by ArgoCD
3. `bootstrap/infrastructure.yaml` + `bootstrap/apps.yaml` root Applications — manually applied

**Fix:**
```bash
# After updating all files in git and pushing:

# Re-apply AppProjects (sourceRepos whitelist)
kubectl apply -f bootstrap/projects/

# Re-apply root Applications (repoURL)
kubectl apply -f bootstrap/infrastructure.yaml -f bootstrap/apps.yaml

# Force hard refresh to clear cached InvalidSpecError
kubectl annotate application apps infrastructure -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

---

## Sealed-secrets: `no key could decrypt secret`

**Symptom:** all SealedSecrets show `Status: ErrUnsealFailed` and `no key could decrypt secret`. Actual Secrets are not created. Apps fail to start.

**Root cause:** the sealed-secrets controller generated a new master key (fresh install or pod restart with ephemeral storage). The SealedSecrets in the repo were encrypted with the old key.

**Fix — restore the master key from your password manager:**
```bash
# Paste the full YAML from your password manager entry ("claro-ai-crm sealed-secrets master key")
cat > /tmp/sealed-secrets-master.key << 'EOF'
<paste YAML here>
EOF

kubectl apply -f /tmp/sealed-secrets-master.key
kubectl -n kube-system rollout restart deployment/sealed-secrets-controller

# Shred immediately — this file contains the private key
shred -u /tmp/sealed-secrets-master.key
```

**Verify:**
```bash
kubectl get sealedsecrets -A
# STATUS column should show True (synced) for all, not ErrUnsealFailed

kubectl get secret claro-ai-crm-db -n claro-ai-crm
# Should exist with DATA=2
```

After all secrets are decrypted, CNPG will reconcile postgres, pods will come online, and apps should recover within 2–3 minutes.

---

## Registry: `x509: certificate signed by unknown authority` on image pull

**Symptom:** pods in `ImagePullBackOff` with error `failed to do request: Head "https://registry.claro-ai-crm.test/...": x509: certificate signed by unknown authority`.

**Root cause:** the k3s node's containerd does not trust the registry's self-signed CA cert. This happens on a fresh node or after the cert is rotated.

**Fix — on the k3s node:**
```bash
# Copy the current CA cert from the cluster to the node
kubectl get secret registry-tls -n registry -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  ssh <user>@<node-ip> "sudo tee /etc/rancher/k3s/registry-ca.crt >/dev/null"

# Ensure registries.yaml exists and references the cert
# (if it already exists and is correct, just restart k3s)
sudo systemctl restart k3s
```

---

## Registry: `401 Unauthorized` on docker login

**Symptom:** `docker login registry.claro-ai-crm.test` fails with 401 even with the correct username.

**Diagnosis:** verify the password matches the htpasswd hash:
```bash
python3 -c "
import bcrypt
stored = b'<hash from kubectl get secret registry-auth -n registry -o jsonpath={.data.htpasswd} | base64 -d>'
password = b'<your password>'
print('match' if bcrypt.checkpw(password, stored) else 'no match')
"
```

**Common mistake:** trailing period (`.`) accidentally included when copy-pasting the password from a sentence ending with it.

---

## claro-ai-crm: Traefik routes Ingress paths to wrong service

**Symptom:** `/me` returns HTML (nginx SPA) instead of JSON (API 401).

**Root cause:** Traefik v3 computes route priority by rule string length. `PathPrefix(/)` (longer string) beats `Path(/me)` (shorter string), so the wildcard wins regardless of specificity.

**Fix:** do NOT add more Ingress paths — this will not work. Instead, ensure the nginx ConfigMap override is in place:
```bash
kubectl get configmap nginx-config -n claro-ai-crm
# Should exist

kubectl exec -n claro-ai-crm deploy/web -- cat /etc/nginx/conf.d/default.conf | grep "location ="
# Should show: location = /me, location = /health
```

If the ConfigMap is missing or the nginx config doesn't have the locations, check ArgoCD sync status for the `claro-ai-crm` application and ensure the `nginx-config.configmap.yaml` in the overlay is committed and pushed.

After any ConfigMap fix:
```bash
kubectl rollout restart deployment/web -n claro-ai-crm
```
