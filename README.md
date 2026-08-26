Here is the complete and corrected `README.md` file, fully compatible with **Runme Notebooks**:

```markdown
# Pixie Observability Setup on Kubernetes

This guide walks you through setting up a self-hosted Pixie environment inside your Kubernetes cluster.

---

### Step 1: Install Pixie CLI

Download and install the official Pixie CLI binary into `/usr/local/bin`:

> **Note:** We use `sudo` and `printf` to automate the interactive prompts for non-interactive environments like Runme.

```sh {"terminalRows":"15"}
echo "=================================================="
echo "🚀 Installing Pixie CLI with sudo..."
echo "=================================================="

printf "y\n/usr/local/bin\n" | sudo bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"

echo ""
echo "=================================================="
echo "✅ Verifying Installation..."
echo "=================================================="
px version
```

---

### Step 2: Deploy Pixie Cloud (Control Plane)

Add the official Pixie Helm repository and deploy the Pixie Cloud control plane into your local Kubernetes cluster:

```sh {"terminalRows":"15"}
echo "=================================================="
echo "🚀 Adding Pixie Helm Repository..."
echo "=================================================="
helm repo add pixie https://pixie-io.github.io/pixie
helm repo update

echo ""
echo "=================================================="
echo "📦 Installing Pixie Cloud in namespace 'plc'..."
echo "=================================================="
helm install pixie-cloud pixie/pixie-cloud \
  --namespace plc \
  --create-namespace \
  --set devMode=true

echo ""
echo "=================================================="
echo "⏳ Waiting for Pixie Cloud Pods to be Ready..."
echo "=================================================="
kubectl get pods -n plc
```

---

### Step 3: Deploy Pixie Vizier (Data Plane / eBPF Agents)

Deploy Pixie Vizier agents into your cluster and connect them to your local Pixie Cloud instance:

```sh {"terminalRows":"15"}
echo "=================================================="
echo "🚀 Deploying Pixie Vizier..."
echo "=================================================="
px deploy --cloud_addr=pixie-cloud.plc.svc.cluster.local:443 --use_direct_connection

echo ""
echo "=================================================="
echo "✅ Checking Pixie Vizier Pods Status..."
echo "=================================================="
kubectl get pods -n pl
```

---

### Step 4: Verify & Access Pixie Observability

You can inspect cluster performance directly from your terminal or open the Web UI dashboard:

#### Option A: Terminal Live Telemetry (CLI)
```sh
# Stream live cluster metrics using Pixie TUI
px live px/cluster
```

#### Option B: Access Pixie Web UI (Dashboard)
```sh
echo "=================================================="
echo "🌐 Forwarding Pixie Web UI to 0.0.0.0:8080..."
echo "=================================================="
kubectl port-forward --address 0.0.0.0 svc/cloud-proxy-service -n plc 8080:443
```
```