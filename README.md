
# Pixie Observability Setup on Minikube

This guide details the deployment of a self-hosted Pixie Control Plane (Pixie Cloud) and eBPF Data Plane (Pixie Vizier) on a local Minikube cluster based on official documentation.

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

### Step 2: Verify Minikube Prerequisites

Ensure Minikube is allocated sufficient resources for Pixie Cloud and eBPF kernel tracing (at least 4 vCPUs and 8GB RAM):

```sh {"terminalRows":"10"}
echo "=================================================="
echo "🚀 Verifying Minikube Environment..."
echo "=================================================="

if ! minikube status | grep -q "Running"; then
  echo "Minikube is not running. Starting cluster with required allocation..."
  minikube start --cpus=4 --memory=8192 --driver=docker
else
  echo "✅ Minikube cluster is active."
fi

kubectl cluster-info
```

---

### Step 3: Deploy Pixie Cloud (Control Plane)

Deploy the self-hosted Pixie Cloud control plane services into the `plc` namespace using Helm:

```sh {"terminalRows":"15"}
echo "=================================================="
echo "🚀 Adding Pixie Official Helm Repository..."
echo "=================================================="
helm repo add pixie https://pixie-io.github.io/pixie
helm repo update

echo ""
echo "=================================================="
echo "📦 Deploying Pixie Cloud into namespace 'plc'..."
echo "=================================================="
helm install pixie-cloud pixie/pixie-cloud \
  --namespace plc \
  --create-namespace \
  --set devMode=true

echo ""
echo "=================================================="
echo "⏳ Waiting for Pixie Cloud Pods to Initialize..."
echo "=================================================="
kubectl wait --for=condition=ready pod -l app=cloud-proxy -n plc --timeout=300s || kubectl get pods -n plc
```

---

### Step 4: Deploy Pixie Vizier (Data Plane / eBPF Agents)

Deploy Pixie Vizier into the `pl` namespace and connect eBPF collectors to your local Pixie Cloud instance:

```sh {"terminalRows":"15"}
echo "=================================================="
echo "🚀 Deploying Pixie Vizier eBPF Agents..."
echo "=================================================="
px deploy \
  --cloud_addr=cloud-proxy-service.plc.svc.cluster.local:443 \
  --use_direct_connection \
  --use_testing_certs

echo ""
echo "=================================================="
echo "✅ Checking Pixie Vizier Agents Status..."
echo "=================================================="
kubectl get pods -n pl
```

---

### Step 5: Verify & Access Pixie Telemetry

#### Option A: Stream Telemetry via CLI (TUI)
```sh
# Query live cluster telemetry directly in terminal
px live px/cluster
```

#### Option B: Access Pixie Dashboard (Web UI)
Forward the local cloud proxy service to access the Pixie UI at `https://localhost:8080`:

```sh {"terminalRows":"10"}
echo "=================================================="
echo "🌐 Forwarding Pixie Web UI to https://localhost:8080..."
echo "=================================================="
kubectl port-forward -n plc svc/cloud-proxy-service 8080:443 --address 0.0.0.0
```
