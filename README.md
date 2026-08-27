# Pixie Self-Hosted Setup Guide (Production Cluster)

This guide deploys a fully self-hosted Pixie Control Plane (Pixie Cloud) and Data Plane (Pixie Vizier) on a Kubernetes cluster.

**Important:** There is no official Helm chart for Pixie Cloud (this has been an open GitHub feature request since 2022). The only supported self-hosted method is `git clone` + `kustomize`. This guide uses that method exclusively.

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

```sh
echo "🚀 Installing Pixie CLI..."
printf "y\n/usr/local/bin\n" | sudo bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"
px version

echo "🚀 Installing mkcert (required for local TLS trust)..."
sudo apt-get update -y && sudo apt-get install -y libnss3-tools
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
sudo mv mkcert-v*-linux-amd64 /usr/local/bin/mkcert
mkcert -install

echo "🚀 Installing kustomize..."
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
kustomize version
```

---

## Step 2: Clone Pixie and Check Out a Cloud Release

```sh
git clone https://github.com/pixie-io/pixie.git
cd pixie

export LATEST_CLOUD_RELEASE=$(git tag | perl -ne 'print $1 if /release\/cloud\/v([^\-]*)$/' | sort -t '.' -k1,1nr -k2,2nr -k3,3nr | head -n 1)
echo "Latest cloud release: v${LATEST_CLOUD_RELEASE}"
git checkout "release/cloud/v${LATEST_CLOUD_RELEASE}"
```

---

## Custom Domain (For Production)

If you're deploying to a real/production cluster (not just local minikube), you almost certainly want a real domain instead of the default `dev.withpixie.dev`. Replace all occurrences of `dev.withpixie.dev` in these three files with your domain **before** proceeding:

```sh
export CUSTOM_DOMAIN="pixie.yourcompany.com"

sed -i "s/dev.withpixie.dev/${CUSTOM_DOMAIN}/g" \
  k8s/cloud/public/base/proxy_envoy.yaml \
  k8s/cloud/public/base/domain_config.yaml \
  scripts/create_cloud_secrets.sh
```

If you skip this, Pixie Cloud will only be reachable at `dev.withpixie.dev` (which you'd need to fake via `/etc/hosts` or a local DNS override — fine for testing, not for a real production cluster with real users).

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

---

## Step 6: Set Up DNS Access

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

---

## Step 7: First Login

Open your domain (custom domain or `dev.withpixie.dev`) in **Chrome** (Safari/Firefox have known login issues on self-managed Pixie Cloud).

Default admin credentials:
- Email: `admin@default.com`
- Password: `admin`

**For production, change these immediately** — modify the `ADMIN_IDENTITY` values in `k8s/cloud/base/ory_auth/kratos/kratos_deployment.yaml` **before** deploying Cloud (Step 5), since the admin account is auto-provisioned on first deploy. If you already deployed with defaults, you'll need to redeploy from scratch to change them.

---

## Step 8: Install CLI and Authenticate

```sh
export PX_CLOUD_ADDR=<your-domain>   # e.g. pixie.yourcompany.com or dev.withpixie.dev

px auth login
```

This opens a browser flow against your own Cloud instance (not Pixie's public SaaS).

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

---

## Production TLS Note

The `cloud-proxy-tls-certs` secret generated by `create_cloud_secrets.sh` uses `mkcert` — a **local, self-signed CA** meant for development/testing. It is trusted only on machines where you've run `mkcert -install`.

For a genuine production deployment reachable by real users/browsers without manual CA installation, replace this secret with a certificate from a real CA (e.g., Let's Encrypt via cert-manager, or your org's internal CA) **before** exposing the cluster externally. This is tracked as a known gap in the self-hosted docs (see GitHub issue [#431](https://github.com/pixie-io/pixie/issues/431) for related community discussion on production-friendly cert management).

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