
# Pixie Observability Setup on Minikube

This guide walks you through setting up Pixie Observability in your Minikube cluster using the official Pixie Helm repository and CLI.

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

### Step 3: Add Official Pixie Helm Repository & Deploy Operator

Add the official Pixie Helm Chart repository and install/upgrade the Pixie Operator:

```sh {"terminalRows":"15"}
echo "=================================================="
echo "🚀 Adding Official Pixie Helm Repository..."
echo "=================================================="
helm repo add pixie-operator https://pixie-operator-charts.storage.googleapis.com
helm repo update

echo ""
echo "=================================================="
echo "📦 Deploying Pixie Operator in namespace 'pixie'..."
echo "=================================================="
helm upgrade --install pixie-operator pixie-operator/pixie-operator-chart \
  --namespace pixie \
  --create-namespace

echo ""
echo "=================================================="
echo "⏳ Waiting for Pixie Operator Resources to Initialize..."
echo "=================================================="
sleep 10
kubectl get pods -A | grep -E "px-operator|pixie" || kubectl get pods -A
```

---

### Step 4: Deploy Pixie Vizier (eBPF Agents)

Deploy Pixie Vizier eBPF agents into your Minikube cluster:

```sh {"terminalRows":"15"}
echo "=================================================="
echo "🚀 Deploying Pixie Vizier via CLI..."
echo "=================================================="
px deploy

echo ""
echo "=================================================="
echo "✅ Checking Pixie Vizier Pods Status..."
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

#### Option B: Open Pixie Web Dashboard
```sh {"terminalRows":"10"}
echo "=================================================="
echo "🌐 Opening Pixie Console..."
echo "=================================================="
px auth login
```
