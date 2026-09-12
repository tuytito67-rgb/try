# Pixie Self-Hosted Setup Guide (Production Cluster)

> **📋 Review notes (checked against the live docs.px.dev self-hosted guide, September 2026)**
> - Pixie the project still looks actively maintained — no sign of a shutdown/sunset.
> - The default self-hosted domain is still `dev.withpixie.dev`. `getcosmic.ai` you may see elsewhere is a *different*, newer **hosted** offering ("Cosmic Cloud") added in 2024 — unrelated to this self-hosted path, don't mix the two up.
> - Every command in Steps 2–9 below (clone/checkout, custom-domain sed, namespace/secrets, deps, cloud deploy, DNS updater flags, `px deploy` flags) matches the current official guide **word for word**.
> - A few things below are *not* verifiable from the public docs (they live inside the repo's scripts) — flagged inline with 🔎 where they appear. Worth double-checking against the actual release tag you check out, rather than trusting this guide blindly.
> - No `Ingress` Kubernetes resource appears anywhere in this guide — it uses `LoadBalancer`-type Services (`cloud-proxy-service`, `vzconn-service`) instead. If you edited an Ingress somewhere, that change lives in a file that wasn't included here.
> - **Update:** added a [Codespaces section](#running-this-inside-a-github-codespace) since that's your actual environment. The biggest open question there is whether eBPF/PEM attaches at all inside a nested Docker-in-Docker setup — everything else (DNS, TLS, exposing the LoadBalancer services) is solvable plumbing, but that one I can't confirm without you actually testing it.

This guide deploys a fully self-hosted Pixie Control Plane (Pixie Cloud) and Data Plane (Pixie Vizier) on a Kubernetes cluster.

**Important:** There is no official Helm chart for Pixie Cloud (this has been an open GitHub feature request since 2022). The only supported self-hosted method is `git clone` + `kustomize`. This guide uses that method exclusively.

> **✅ Confirmed still true.** The official Helm docs only cover deploying *Vizier* (the data plane) — even the "self-hosting Pixie Cloud" Helm example just points Vizier at your own Cloud namespace. There's still no Helm path for standing up Cloud itself.

---

## Prerequisites

- A Kubernetes cluster with:
  - At least **4 vCPUs / 8GB RAM** free (Elasticsearch alone needs significant memory — for a production cluster, consider 16GB+ RAM headroom)
  - `PersistentVolume` support enabled
  - Privileged pod access allowed (required for `vizier-pem-*` eBPF agents)
- Tools installed locally: `kubectl` (configured against your cluster), `git`, `openssl`, `go` (for the DNS updater binary)
- A domain name if you're not using the default `dev.withpixie.dev` (recommended for a real/production cluster — see [Custom Domain](#custom-domain-for-production) section below)

> 🔎 `openssl` isn't mentioned on the public docs page (it's used inside `create_cloud_secrets.sh`, not documented separately) — plausible, but worth confirming against the actual script in the release tag you check out.

---

## Running This Inside a GitHub Codespace

> ⚠️ **Read this before you invest time in the rest of the guide.** Pixie's whole value comes from `vizier-pem-*` — eBPF agents that need real access to the node's kernel (kernel headers/BTF, `/sys`, `/proc`) to compile and attach BPF programs. A Codespace already runs inside a container-on-VM setup; running a K8s cluster *inside that* via Docker-in-Docker nests things one layer deeper again. That exact pattern (dockerd-in-a-VM — Rancher Desktop, Colima, similar setups) shows up repeatedly in Pixie's own GitHub issues with PEM/eBPF either failing outright or only partially working. I can't tell you which you'll get on GitHub's specific Codespace kernel without actually testing it, and I have no way to run that test myself. **Do the fast sanity check below before you spend the ~30-60 minutes standing up the full Cloud stack.**

### Suggested cluster setup

Codespaces has no cloud LoadBalancer provider, and this guide's `cloud-proxy-service` / `vzconn-service` are both `LoadBalancer` type — plain `kind` will leave them stuck `Pending`. Going with **`minikube` + the `docker` driver + `minikube tunnel`** rather than `kind`/`k3d`, because it's the one local setup Pixie's *own* docs already anticipate ("If you are running Pixie Cloud on minikube, you likely need to run `minikube tunnel`") — you're on a path the maintainers actually thought about, not an unlisted one.

**`.devcontainer/devcontainer.json`** (add this if the repo doesn't already have one):
```json
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

Once the Codespace is up:
```sh
minikube start --driver=docker --cpus=6 --memory=12000mb
minikube tunnel   # leave running in its own terminal/tab — this replaces Step 6 below entirely
```

### Fast eBPF sanity check — do this before anything else

```sh
px deploy   # points at Pixie's hosted cloud by default, no self-hosted Cloud needed for this check
px run px/agent_status
px run px/cluster
```
Five minutes instead of an hour. If PEM pods crash-loop, or `px/cluster` comes back empty/erroring while `agent_status` shows unhealthy PEMs, eBPF isn't working on this Codespace — better to know that now than after building the whole self-hosted stack. Tear this down (`px deploy --clear`, or just recreate the minikube cluster) once you've confirmed it either way.

### Domain / TLS: pick one

The rest of this guide's `mkcert` + `dev.withpixie.dev` + `/etc/hosts` flow assumes your browser and your cluster are the same machine. In a Codespace they aren't — pick one of these instead:

- **Option A — ride Codespaces' own HTTPS forwarding (less setup, some risk).** Use your Codespace's forwarded hostname as `CUSTOM_DOMAIN` in the [Custom Domain](#custom-domain-for-production) step, e.g. `verbose-space-fishstick-abc123-443.app.github.dev`. Forward the `443` port and whatever port `vzconn-service` uses, mark them Public in the Ports panel, and skip `mkcert`/`dev_dns_updater` entirely — GitHub's edge presents a real trusted cert for `*.app.github.dev`, no local CA install needed. Risk I can't rule out from documentation alone: Pixie's Envoy routes by the `Host` header matching `domain_config.yaml`, so if login redirects loop or you get a routing error, mismatched Host headers through GitHub's proxy is the likely cause.
- **Option B — keep `dev.withpixie.dev` exactly as the guide does it (more setup, closer to upstream).** Use `gh codespace ports forward 443:443 <vzconn-port>:<vzconn-port>`, or forward the ports privately from VS Code Desktop connected remotely (those show up as plain `127.0.0.1` tunnels, not wrapped in GitHub's proxy), so the raw TLS reaches your browser untouched. Then, on your **local** machine: add `127.0.0.1 dev.withpixie.dev` to your hosts file, and install the Codespace's `mkcert` root CA locally too (`cat "$(mkcert -CAROOT)/rootCA.pem"` inside the Codespace, copy it out, add to your local trust store) — `mkcert -install` only trusts a CA on the machine it ran on, and your browser is a different machine from the Codespace.

---

## Step 1: Install Prerequisite Tools

```sh
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

> 🔎 The official docs just run `bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"` with no `sudo` and no piped answers. The `printf "y\n/usr/local/bin\n" | sudo bash -c ...` here is your own non-interactive automation — should work as long as the installer still asks exactly those two prompts (confirm + path) in that order, but that's not something the docs guarantee across versions, so it's worth a dry run before you rely on it in CI.
> 🖥️ **Seen on your run:** `sudo: apt-get: command not found` — this Codespace's base image isn't Debian/Ubuntu-based, so no `apt-get`. mkcert still created and installed its CA into the generic system trust store fine, but check `which certutil` once mkcert is installed — if it's missing, Chrome-on-Linux may not auto-trust the cert (it uses the NSS store, which needs `certutil` from `libnss3-tools`/`nss-tools`). If it does throw a warning in the browser at Step 7, either import `$(mkcert -CAROOT)/rootCA.pem` into Chrome manually (`chrome://settings/certificates` → Authorities) or just click through the warning — it's a throwaway dev cert anyway.
> ⚠️ **Run each step's block as one paste into a single open terminal**, not through a separate "Run" click per code block or auto-generated task. Several later steps (`export CUSTOM_DOMAIN=...`, `export PX_CLOUD_ADDR=...`, and the `export LATEST_CLOUD_RELEASE=...` right below) only exist for the rest of that same shell session — if whatever's executing these blocks spins up a fresh shell per block, those exports silently vanish before the next line needs them, which is exactly what happened with `LATEST_CLOUD_RELEASE` below.

---

## Step 2: Clone Pixie and Check Out a Cloud Release

```sh
git clone https://github.com/pixie-io/pixie.git
cd pixie

export LATEST_CLOUD_RELEASE=$(git tag | perl -ne 'print $1 if /release\/cloud\/v([^\-]*)$/' | sort -t '.' -k1,1nr -k2,2nr -k3,3nr | head -n 1)
echo "Latest cloud release: v${LATEST_CLOUD_RELEASE}"
git checkout "release/cloud/v${LATEST_CLOUD_RELEASE}"
```

> ✅ Matches the official guide exactly.
> 🔎 One thing the official guide has that this section doesn't: after checking out the tag, it also runs `perl -pi -e "s|newTag: latest|newTag: \"${LATEST_CLOUD_RELEASE}\"|g" k8s/cloud/public/kustomization.yaml` to pin the image tag in the kustomization file. Worth adding before Step 4/5 if it's not happening elsewhere in your pipeline — otherwise `kustomize build` may pull `latest` instead of the pinned release.

---

## Custom Domain (For Production)

> 🖥️ **In the Codespace:** see [Running This Inside a GitHub Codespace](#running-this-inside-a-github-codespace) above — Option A uses this section with your Codespace's forwarded hostname instead of a domain you own; Option B skips this section and keeps `dev.withpixie.dev`.

If you're deploying to a real/production cluster (not just local minikube), you almost certainly want a real domain instead of the default `dev.withpixie.dev`. Replace all occurrences of `dev.withpixie.dev` in these three files with your domain **before** proceeding:

```sh
export CUSTOM_DOMAIN="pixie.yourcompany.com"

sed -i "s/dev.withpixie.dev/${CUSTOM_DOMAIN}/g" \
  k8s/cloud/public/base/proxy_envoy.yaml \
  k8s/cloud/public/base/domain_config.yaml \
  scripts/create_cloud_secrets.sh
```

If you skip this, Pixie Cloud will only be reachable at `dev.withpixie.dev` (which you'd need to fake via `/etc/hosts` or a local DNS override — fine for testing, not for a real production cluster with real users).

> ✅ Exact match with the official docs — same three files, same instruction.

---

## Step 3: Create Namespace and Cloud Secrets

```sh
kubectl create namespace plc
```

⚠️ **If you've run this before and it's failing on "already exists" for secrets or namespace:** the `create_cloud_secrets.sh` script is **not idempotent** — it will fail on re-run if secrets already exist. Clean up first:

```sh
kubectl delete secrets --all -n plc
```

Then generate the secrets:

```sh
./scripts/create_cloud_secrets.sh
```

This creates: `cloud-auth-secrets`, `pl-hydra-secrets`, `pl-db-secrets`, `cloud-session-secrets`, `service-tls-certs`, and `cloud-proxy-tls-certs` (the last one is a self-signed cert via `mkcert` — see [Production TLS](#production-tls-note) note below for real deployments).

Verify:

```sh
kubectl get secrets -n plc
```

> ✅ `kubectl create namespace plc` and `./scripts/create_cloud_secrets.sh` match the official guide exactly.
> 🔎 The exact list of 6 secret names and the "not idempotent, delete first" behavior aren't documented on the public page — they're implementation details of the script itself. This is a very plausible, commonly-hit gotcha, but I couldn't independently confirm the secret names from the docs; run `kubectl get secrets -n plc` after the script once to sanity-check them for the release tag you're on.

---

## Step 4: Deploy Cloud Dependencies (Elastic, NATS, Postgres)

```sh
kustomize build k8s/cloud_deps/base/elastic/operator | kubectl apply -f -
kustomize build k8s/cloud_deps/public | kubectl apply -f -
```

This takes several minutes — Elasticsearch pods go through `Init:0/2` before reaching `Running`. Watch progress:

```sh
watch kubectl get pods -n plc
```

Do **not** proceed to Step 5 until all pods here are `Running`/`Ready`. If a pod stays stuck in `Init` or `Pending` for more than ~10 minutes, check:

```sh
kubectl describe pod <pod-name> -n plc
```

Look at the `Events:` section at the bottom — usually insufficient CPU/memory or a PersistentVolume provisioning issue on a larger cluster.

> ✅ Both `kustomize build` commands match the official guide exactly.

---

## Step 5: Deploy Pixie Cloud

```sh
kustomize build k8s/cloud/public/ | kubectl apply -f -
```

Wait for pods:

```sh
kubectl get pods -n plc
```

Note: one or more `create-hydra-client-job` pods may show `Error` — this is expected as long as **another instance** of that job pod completes successfully (Kubernetes retries jobs automatically).

> ✅ Matches the official guide almost word-for-word, including the hydra-client-job note.

---

## Step 6: Set Up DNS Access

> 🖥️ **In the Codespace with minikube:** skip this whole step. `minikube tunnel` (already running from the setup section above) handles exposing both services — just confirm they show an `EXTERNAL-IP` of `127.0.0.1` with the `kubectl get service` commands below, then go straight to whichever Domain/TLS option you picked.

On a real cluster, `cloud-proxy-service` and `vzconn-service` are typically `LoadBalancer` type services and should get real external IPs automatically (no `minikube tunnel` needed — that was only for local minikube).

```sh
kubectl get service cloud-proxy-service -n plc
kubectl get service vzconn-service -n plc
```

**If you set a custom domain in the earlier step:** point your real DNS (A records) at the `EXTERNAL-IP` shown for `cloud-proxy-service` and `vzconn-service`. This is the proper production approach — skip the `dev_dns_updater` step below entirely.

**If you're still using the default `dev.withpixie.dev` domain (testing only):**

```sh
go build src/utils/dev_dns_updater/dev_dns_updater.go
./dev_dns_updater --domain-name="dev.withpixie.dev" --kubeconfig=$HOME/.kube/config --n=plc
```

Keep this process running in the background/foreground as needed — it patches local DNS resolution.

> ✅ Service names, flag names (`--domain-name`, `--kubeconfig`, `--n`), and the real-DNS-vs-dev-updater split all match the official guide.

---

## Step 7: First Login

Open your domain (custom domain or `dev.withpixie.dev`) in **Chrome** (Safari/Firefox have known login issues on self-managed Pixie Cloud).

Default admin credentials:
- Email: `admin@default.com`
- Password: `admin`

**For production, change these immediately** — modify the `ADMIN_IDENTITY` values in `k8s/cloud/base/ory_auth/kratos/kratos_deployment.yaml` **before** deploying Cloud (Step 5), since the admin account is auto-provisioned on first deploy. If you already deployed with defaults, you'll need to redeploy from scratch to change them.

> ✅ Chrome-only note, default credentials, file path, and "redeploy from scratch" caveat all match the official guide exactly.

---

## Step 8: Install CLI and Authenticate

```sh
export PX_CLOUD_ADDR=<your-domain>   # e.g. pixie.yourcompany.com or dev.withpixie.dev

px auth login
```

This opens a browser flow against your own Cloud instance (not Pixie's public SaaS).

> 🔎 The official flow installs the CLI *here* (after Cloud is up), not back in Step 1. Not wrong to install it earlier like this guide does, just note the CLI has nothing to authenticate against until Cloud is actually deployed — so don't be surprised if `px auth login` right after Step 1 fails.

---

## Step 9: Deploy Pixie Vizier

```sh
px deploy --dev_cloud_namespace plc
```

For a memory-constrained cluster, you can lower the PEM (per-node agent) memory limit:

```sh
px deploy --dev_cloud_namespace plc --pem_memory_limit=1Gi
```

If your cluster already has Operator Lifecycle Manager (OLM) installed:

```sh
px deploy --dev_cloud_namespace plc --deploy_olm=false
```

Verify:

```sh
kubectl get pods -n pl
```

> ✅ All three flags (`--dev_cloud_namespace`, `--pem_memory_limit=1Gi`, `--deploy_olm=false`) match the current official guide exactly.

---

## Production TLS Note

The `cloud-proxy-tls-certs` secret generated by `create_cloud_secrets.sh` uses `mkcert` — a **local, self-signed CA** meant for development/testing. It is trusted only on machines where you've run `mkcert -install`.

For a genuine production deployment reachable by real users/browsers without manual CA installation, replace this secret with a certificate from a real CA (e.g., Let's Encrypt via cert-manager, or your org's internal CA) **before** exposing the cluster externally. This is tracked as a known gap in the self-hosted docs (see GitHub issue [#431](https://github.com/pixie-io/pixie/issues/431) for related community discussion on production-friendly cert management).

> 🔎 The mkcert/self-signed nature of that cert is confirmed by the official docs. I could not independently verify that issue **#431** specifically is the right link — double-check it resolves to a cert-management discussion before you publish this internally. (There is a real, merged PR — "Create vizier and cloud cert-manager compatible secrets" — showing Pixie has been moving toward cert-manager-compatible secrets, so the underlying gap you're pointing at is real even if the issue number needs a check.)

---

## Using Pixie

```sh
px live px/cluster
```

Or open the web UI at your domain.

## Troubleshooting

- **Elastic pods stuck in `Init`**: check node resources with `kubectl describe nodes | grep -A 5 "Allocated resources"`.
- **`create_cloud_secrets.sh` fails with "already exists"**: run `kubectl delete secrets --all -n plc` first (script is not idempotent).
- **Browser can't reach Cloud UI / cert errors**: confirm `mkcert -install` ran on the machine you're browsing from, or that you've replaced the self-signed cert per the [Production TLS](#production-tls-note) note.
- **`px deploy` fails with "must be logged in"**: run `px auth login` with `PX_CLOUD_ADDR` set to your Cloud's domain first.

## Reference

- Official self-hosted guide: https://docs.px.dev/installing-pixie/install-guides/self-hosted-pixie/
- Pixie repo: https://github.com/pixie-io/pixie