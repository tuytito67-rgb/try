---
shell: bash
skipPrompts: true
---

# Pixie Self-Hosted Setup Guide (Production Cluster)

> This guide deploys a fully self-hosted Pixie Control Plane (Pixie Cloud) and Data Plane (Pixie Vizier) on a Kubernetes cluster, using the official `git clone` + `kustomize` method (there is no supported Helm chart for Pixie Cloud itself).

## What changed in this revision (adapted to run via `runme`)

Running this guide's blocks through `runme` surfaced a real bug: `export`/`cd` from one cell does __not__ reliably carry over into the next cell (each cell can execute as its own shell), so a plain `cd pixie` in Step 2 was gone by the time the Custom Domain cell ran its `sed`, and relative paths like `k8s/cloud/public/base/proxy_envoy.yaml` failed with "No such file or directory" — even though those files genuinely exist in the repo. Fixes applied throughout:

- Every cell that touches files inside the cloned repo sets **`cwd=pixie`** at the cell level (resolved relative to the folder that contains this `README.md`), instead of depending on a `cd` from an earlier cell.
- __`PX_CLOUD_ADDR`__ is exported fresh inside _every_ cell that calls the `px` CLI — the CLI reads it from the environment on each invocation, it does not persist across shells.
- Step 2 now resolves the release tag, checks it out, pins the image tag in `kustomization.yaml`, and saves the tag to a plain file (`pixie/.pixie_cloud_release`) — all in __one__ cell, so nothing downstream depends on an exported variable surviving.
- `minikube tunnel` and `dev_dns_updater` are marked __`background=true`__ so they don't block the rest of the runbook.
- `minikube delete --all --purge` runs before `minikube start`, for a clean slate.
- **Step 1 (install the `px` CLI) now runs before the Codespace section** — the original ordering had the Codespace's eBPF sanity check calling `px deploy`/`px run` before the CLI was ever installed.
- Optional / conditional / alternative cells (custom domain, secret cleanup, alternate `px deploy` flags, troubleshooting diagnostics, the eBPF sanity check) are marked **`excludeFromRunAll=true`** so a "Run All" doesn't fire them unintentionally.
- Cells with placeholder `export` values use **`promptEnv=never`** — since this is meant to run headlessly, edit the placeholder value directly in the cell before running it, rather than relying on an interactive prompt that may not appear in your execution context.

---

## Prerequisites

- A Kubernetes cluster with:
   - At least **4 vCPUs / 8GB RAM** free (Elasticsearch alone needs significant memory — for a production cluster, consider 16GB+ RAM headroom)
   - `PersistentVolume` support enabled
   - Privileged pod access allowed (required for `vizier-pem-*` eBPF agents)

- Tools installed locally: `kubectl` (configured against your cluster), `git`, `openssl`, `go` (for the DNS updater binary)
- A domain name if you're not using the default `dev.withpixie.dev` (recommended for a real/production cluster — see [Custom Domain](#custom-domain-for-production) section below)

---

## Step 1: Install Prerequisite Tools

```sh { name=install-prereq-tools }
echo "🚀 Installing Pixie CLI..."
printf "y\n/usr/local/bin\n" | sudo bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"
px version

echo "🚀 Installing mkcert (required for local TLS trust)..."
if command -v apt-get >/dev/null; then
  sudo apt-get update -y && sudo apt-get install -y libnss3-tools
elif command -v apk >/dev/null; then
  sudo apk add --no-cache nss-tools
elif command -v dnf >/dev/null; then
  sudo dnf install -y nss-tools
elif command -v yum >/dev/null; then
  sudo yum install -y nss-tools
else
  echo "No apt-get/apk/dnf/yum found — skipping certutil install."
  echo "mkcert's CA will still install into the generic system trust store,"
  echo "but Chrome-on-Linux (NSS-based cert store) may not auto-trust it without certutil."
fi
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
sudo mv mkcert-v*-linux-amd64 /usr/local/bin/mkcert
mkcert -install

echo "🚀 Installing kustomize..."
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
kustomize version
```

> If `apt-get` is missing (some Codespace/devcontainer base images aren't Debian-based), mkcert still installs its CA into the generic system trust store, but Chrome-on-Linux may not auto-trust it without `certutil`. If you hit a cert warning at Step 7, either import `$(mkcert -CAROOT)/rootCA.pem` into Chrome manually (`chrome://settings/certificates` → Authorities) or click through the warning — it's a throwaway dev cert either way.

---

## Running This Inside a GitHub Codespace

> ⚠️ **Read this before you invest time in the rest of the guide.** Pixie's whole value comes from `vizier-pem-*` — eBPF agents that need real access to the node's kernel (kernel headers/BTF, `/sys`, `/proc`) to compile and attach BPF programs. A Codespace already runs inside a container-on-VM setup; running a K8s cluster *inside that* via Docker-in-Docker nests things one layer deeper again. That exact pattern (dockerd-in-a-VM — Rancher Desktop, Colima, similar setups) shows up repeatedly in Pixie's own GitHub issues with PEM/eBPF either failing outright or only partially working. Do the fast sanity check below before you spend the ~30-60 minutes standing up the full Cloud stack.

Skip this whole section if you're deploying to a real cluster, not a Codespace.

### Suggested cluster setup

Codespaces has no cloud LoadBalancer provider, and this guide's `cloud-proxy-service` / `vzconn-service` are both `LoadBalancer` type — plain `kind` will leave them stuck `Pending`. Use **`minikube` + the `docker` driver + `minikube tunnel`** instead of `kind`/`k3d` — it's the one local setup Pixie's own docs already anticipate ("If you are running Pixie Cloud on minikube, you likely need to run `minikube tunnel`").

**`.devcontainer/devcontainer.json`** (add this if the repo doesn't already have one):

```json { name=devcontainer-config ignore=true }
{
  "name": "pixie-self-hosted",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": { "moby": true },
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {},
    "ghcr.io/devcontainers/features/go:1": {}
  },
  "hostRequirements": { "cpus": 8, "memory": "16gb" }
}
```

Pick at least an **8-core / 16GB** Codespace machine type when you create it — this guide already asks for 4 vCPU/8GB free *for Pixie itself*, and minikube + Docker-in-Docker overhead eats into that before Pixie even starts.

Once the Codespace is up, start with a clean slate and bring up the cluster:

```sh { name=minikube-reset-and-start }
minikube delete --all --purge
minikube start --driver=docker --cpus=6 --memory=12000mb
```

Leave the tunnel running in the background — this replaces Step 6 below entirely:

```sh { name=minikube-tunnel background=true }
minikube tunnel
```

### Fast eBPF sanity check — do this before anything else

Five minutes instead of an hour. Requires the `px` CLI from Step 1 above.

```sh { name=ebpf-sanity-deploy excludeFromRunAll=true }
# Points at Pixie's hosted cloud by default — no self-hosted Cloud needed
# for this check. Do NOT export PX_CLOUD_ADDR before running this.
px deploy
```

```sh { name=ebpf-sanity-status excludeFromRunAll=true }
px run px/agent_status
px run px/cluster
```

If PEM pods crash-loop, or `px/cluster` comes back empty/erroring while `agent_status` shows unhealthy PEMs, eBPF isn't working on this Codespace — better to know that now than after building the whole self-hosted stack.

Tear this down once you've confirmed it either way:

```sh { name=ebpf-sanity-teardown excludeFromRunAll=true }
px deploy --clear
```

### Domain / TLS: pick one

The rest of this guide's `mkcert` + `dev.withpixie.dev` + `/etc/hosts` flow assumes your browser and your cluster are the same machine. In a Codespace they aren't — pick one of these instead:

- __Option A — ride Codespaces' own HTTPS forwarding (less setup, some risk).__ Use your Codespace's forwarded hostname as `CUSTOM_DOMAIN` in the [Custom Domain](#custom-domain-for-production) step below, e.g. `verbose-space-fishstick-abc123-443.app.github.dev`. Forward the `443` port and whatever port `vzconn-service` uses, mark them Public in the Ports panel, and skip `mkcert`/`dev_dns_updater` entirely — GitHub's edge presents a real trusted cert for `*.app.github.dev`, no local CA install needed. Risk: Pixie's Envoy routes by the `Host` header matching `domain_config.yaml`, so if login redirects loop or you get a routing error, mismatched Host headers through GitHub's proxy is the likely cause.

- **Option B — keep `dev.withpixie.dev` exactly as the guide does it (more setup, closer to upstream).** These commands run on your **local machine**, not inside the Codespace, so they are not runnable from here:

```sh { ignore=true }
# Run this on your LOCAL machine, not inside the Codespace:
gh codespace ports forward 443:443 <vzconn-port>:<vzconn-port>
```

Or forward the ports privately from VS Code Desktop connected remotely (those show up as plain `127.0.0.1` tunnels, not wrapped in GitHub's proxy), so the raw TLS reaches your browser untouched. Then, on your **local** machine: add `127.0.0.1 dev.withpixie.dev` to your hosts file, and install the Codespace's `mkcert` root CA locally too (`cat "$(mkcert -CAROOT)/rootCA.pem"` inside the Codespace, copy it out, add to your local trust store) — `mkcert -install` only trusts a CA on the machine it ran on, and your browser is a different machine from the Codespace.

---

## Step 2: Clone Pixie and Check Out a Cloud Release

```sh { name=clone-and-checkout-release }
[ -d pixie ] || git clone https://github.com/pixie-io/pixie.git
cd pixie
git fetch --tags

LATEST_CLOUD_RELEASE=$(git tag | perl -ne 'print $1 if /release\/cloud\/v([^\-]*)$/' | sort -t '.' -k1,1nr -k2,2nr -k3,3nr | head -n 1)
if [ -z "$LATEST_CLOUD_RELEASE" ]; then
  echo "❌ Could not detect a release/cloud/vX.Y.Z tag. Run 'git tag | grep release/cloud' manually and check." >&2
  exit 1
fi
echo "Latest cloud release: v${LATEST_CLOUD_RELEASE}"

git checkout "release/cloud/v${LATEST_CLOUD_RELEASE}"

# Pin the image tag in the kustomization file — easy to miss, but without it
# `kustomize build` can silently pull `latest` instead of this pinned release.
perl -pi -e "s|newTag: latest|newTag: \"${LATEST_CLOUD_RELEASE}\"|g" k8s/cloud/public/kustomization.yaml

# Persist the resolved version to a plain file: exports do NOT survive between
# separately-executed runme cells, so later cells read this file instead.
echo "${LATEST_CLOUD_RELEASE}" > .pixie_cloud_release

echo "✅ Checked out release/cloud/v${LATEST_CLOUD_RELEASE}, pinned image tag, saved to pixie/.pixie_cloud_release"
```

---

## Custom Domain (For Production)

> **In a Codespace:** see [Domain / TLS: pick one](#domain--tls-pick-one) above — Option A uses this section with your Codespace's forwarded hostname instead of a domain you own; Option B skips this section and keeps `dev.withpixie.dev`.

If you're deploying to a real/production cluster (not just local minikube), you almost certainly want a real domain instead of the default `dev.withpixie.dev`. This replaces all occurrences of `dev.withpixie.dev` in three files with your domain. __Edit `CUSTOM_DOMAIN` below before running.__

```sh { name=set-custom-domain cwd=pixie excludeFromRunAll=true promptEnv=never }
CUSTOM_DOMAIN="pixie.yourcompany.com"   # <-- edit this

test -f k8s/cloud/public/base/proxy_envoy.yaml || {
  echo "❌ Not inside the pixie repo checkout — did the clone-and-checkout-release cell run first?" >&2
  exit 1
}

sed -i "s/dev.withpixie.dev/${CUSTOM_DOMAIN}/g" \
  k8s/cloud/public/base/proxy_envoy.yaml \
  k8s/cloud/public/base/domain_config.yaml \
  scripts/create_cloud_secrets.sh

echo "✅ Replaced dev.withpixie.dev with ${CUSTOM_DOMAIN} in 3 files"
```

If you skip this, Pixie Cloud will only be reachable at `dev.withpixie.dev` (which you'd need to fake via `/etc/hosts` or a local DNS override — fine for testing, not for a real production cluster with real users).

---

## Step 3: Create Namespace and Cloud Secrets

```sh { name=create-namespace-and-secrets cwd=pixie }
kubectl create namespace plc
./scripts/create_cloud_secrets.sh
kubectl get secrets -n plc
```

This creates: `cloud-auth-secrets`, `pl-hydra-secrets`, `pl-db-secrets`, `cloud-session-secrets`, `service-tls-certs`, and `cloud-proxy-tls-certs` (the last one is a self-signed cert via `mkcert` — see [Production TLS](#production-tls-note) note below for real deployments).

⚠️ __If you've run this before and it's failing on "already exists"__: `create_cloud_secrets.sh` is __not idempotent__ — clean up first, then re-run the cell above.

```sh { name=reset-cloud-secrets cwd=pixie excludeFromRunAll=true }
kubectl delete secrets --all -n plc
```

---

## Step 4: Deploy Cloud Dependencies (Elastic, NATS, Postgres)

```sh { name=deploy-cloud-deps cwd=pixie }
kustomize build k8s/cloud_deps/base/elastic/operator | kubectl apply -f -
kustomize build k8s/cloud_deps/public | kubectl apply -f -
```

This takes several minutes — Elasticsearch pods go through `Init:0/2` before reaching `Running`. Re-run the cell below every so often to check progress; don't proceed to Step 5 until everything is `Running`/`Ready`:

```sh { name=check-cloud-deps-pods excludeFromRunAll=true }
kubectl get pods -n plc
```

If a pod stays stuck in `Init` or `Pending` for more than ~10 minutes:

```sh { name=describe-stuck-pod excludeFromRunAll=true promptEnv=never }
POD_NAME="paste-the-pod-name-here"   # <-- edit this
kubectl describe pod "$POD_NAME" -n plc
```

Look at the `Events:` section at the bottom — usually insufficient CPU/memory or a PersistentVolume provisioning issue on a larger cluster.

---

## Step 5: Deploy Pixie Cloud

```sh { name=deploy-pixie-cloud cwd=pixie }
kustomize build k8s/cloud/public/ | kubectl apply -f -
kubectl get pods -n plc
```

Note: one or more `create-hydra-client-job` pods may show `Error` — this is expected as long as **another instance** of that job pod completes successfully (Kubernetes retries jobs automatically).

---

## Step 6: Set Up DNS Access

> **In a Codespace with minikube:** skip this whole step. `minikube tunnel` (already running in the background from the setup section above) handles exposing both services — just confirm they show an `EXTERNAL-IP` of `127.0.0.1` with the commands below, then go straight to whichever Domain/TLS option you picked.

On a real cluster, `cloud-proxy-service` and `vzconn-service` are typically `LoadBalancer` type services and should get real external IPs automatically:

```sh { name=check-loadbalancer-ips }
kubectl get service cloud-proxy-service -n plc
kubectl get service vzconn-service -n plc
```

__If you set a custom domain in the earlier step:__ point your real DNS (A records) at the `EXTERNAL-IP` shown above for `cloud-proxy-service` and `vzconn-service`. This is the proper production approach — skip the `dev_dns_updater` cells below entirely.

**If you're still using the default `dev.withpixie.dev` domain (testing only):**

```sh { name=build-dev-dns-updater cwd=pixie excludeFromRunAll=true }
go build src/utils/dev_dns_updater/dev_dns_updater.go
```

```sh { name=run-dev-dns-updater background=true cwd=pixie excludeFromRunAll=true }
./dev_dns_updater --domain-name="dev.withpixie.dev" --kubeconfig=$HOME/.kube/config --n=plc
```

---

## Step 7: First Login

Open your domain (custom domain or `dev.withpixie.dev`) in **Chrome** (Safari/Firefox have known login issues on self-managed Pixie Cloud).

Default admin credentials:

- Email: `admin@default.com`
- Password: `admin`

__For production, change these immediately__ — modify the `ADMIN_IDENTITY` values in `k8s/cloud/base/ory_auth/kratos/kratos_deployment.yaml` __before__ deploying Cloud (Step 5), since the admin account is auto-provisioned on first deploy. If you already deployed with defaults, you'll need to redeploy from scratch to change them.

---

## Step 8: Authenticate the CLI

The `px` CLI is already installed from Step 1. This just points it at your own Cloud instance instead of Pixie's public SaaS.

```sh { name=pixie-auth-login interactive=true promptEnv=never }
export PX_CLOUD_ADDR=dev.withpixie.dev   # <-- edit if you set a custom domain or a Codespace hostname

# In a headless environment (Codespace, CI, SSH) there's no local browser to
# open, so --manual prints a URL for you to open yourself instead.
px auth login --manual
```

---

## Step 9: Deploy Pixie Vizier

```sh { name=deploy-pixie-vizier promptEnv=never }
export PX_CLOUD_ADDR=dev.withpixie.dev   # <-- match Step 8; px reads this fresh on every invocation

px deploy --dev_cloud_namespace plc
```

For a memory-constrained cluster, lower the PEM (per-node agent) memory limit instead:

```sh { name=deploy-pixie-vizier-low-memory excludeFromRunAll=true promptEnv=never }
export PX_CLOUD_ADDR=dev.withpixie.dev   # <-- match Step 8

px deploy --dev_cloud_namespace plc --pem_memory_limit=1Gi
```

If your cluster already has Operator Lifecycle Manager (OLM) installed, use this instead:

```sh { name=deploy-pixie-vizier-olm-present excludeFromRunAll=true promptEnv=never }
export PX_CLOUD_ADDR=dev.withpixie.dev   # <-- match Step 8

px deploy --dev_cloud_namespace plc --deploy_olm=false
```

Verify:

```sh { name=verify-vizier-pods }
kubectl get pods -n pl
```

---

## Production TLS Note

The `cloud-proxy-tls-certs` secret generated by `create_cloud_secrets.sh` uses `mkcert` — a __local, self-signed CA__ meant for development/testing. It is trusted only on machines where you've run `mkcert -install`.

For a genuine production deployment reachable by real users/browsers without manual CA installation, replace this secret with a certificate from a real CA (e.g., Let's Encrypt via cert-manager, or your org's internal CA) **before** exposing the cluster externally.

---

## Using Pixie

```sh { name=pixie-live-view promptEnv=never }
export PX_CLOUD_ADDR=dev.withpixie.dev   # <-- match Step 8

px live px/cluster
```

Or open the web UI at your domain.

---

## Troubleshooting

- **Elastic pods stuck in `Init`**:

```sh { name=troubleshoot-node-resources excludeFromRunAll=true }
kubectl describe nodes | grep -A 5 "Allocated resources"
```

- __`create_cloud_secrets.sh` fails with "already exists"__: run the `reset-cloud-secrets` cell above first (script is not idempotent).

- **Browser can't reach Cloud UI / cert errors**: confirm `mkcert -install` ran on the machine you're browsing from, or that you've replaced the self-signed cert per the [Production TLS](#production-tls-note) note.

- __`px deploy` fails with "must be logged in"__: re-run the `pixie-auth-login` cell (Step 8) with `PX_CLOUD_ADDR` set to your Cloud's domain first — remember it must be exported in the _same_ cell/shell as the `px` command that needs it.

## Reference

- Official self-hosted guide: https://docs.px.dev/installing-pixie/install-guides/self-hosted-pixie/
- Pixie repo: https://github.com/pixie-io/pixie
- Runme cell-level configuration reference: https://docs.runme.dev/configuration/cell-level