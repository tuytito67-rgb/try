
# Pixie Observability Setup on Minikube

This guide walks you through setting up a self-hosted Pixie Control Plane (Pixie Cloud) and Data Plane (Pixie Vizier) on a Minikube cluster.

---

### Step 1: Install Pixie CLI

Download and install the official Pixie CLI binary:

```sh {"terminalRows":"15"}
echo "🚀 Installing Pixie CLI..."
printf "y\n/usr/local/bin\n" | sudo bash -c "$(curl -fsSL https://withpixie.ai/install.sh)"
px version
```

---

### Step 2: Verify Minikube Prerequisites

Ensure Minikube is allocated sufficient resources (at least 4 vCPUs and 8GB RAM):

```sh {"terminalRows":"10"}
echo "🚀 Verifying Minikube Environment..."
if ! minikube status | grep -q "Running"; then
  minikube start --cpus=4 --memory=8192 --driver=docker
else
  echo "✅ Minikube cluster is active."
fi
kubectl cluster-info
```

---

### Step 3: Deploy Pixie Cloud (Control Plane)

We MUST deploy the Pixie Cloud first. This creates the `plc` namespace and the `cloud-proxy-service`.

```sh {"terminalRows":"15"}
echo "🚀 Deploying Pixie Cloud..."
helm repo add pixie https://pixie-io.github.io/pixie
helm repo update

helm upgrade --install pixie-cloud pixie/pixie-cloud \
  --namespace plc \
  --create-namespace \
  --set devMode=true

echo "⏳ Waiting for Pixie Cloud to be ready (this may take a few minutes)..."
kubectl wait --for=condition=ready pod -l app=cloud-proxy -n plc --timeout=300s
```

---

### Step 4: Authenticate CLI Against Self-Hosted Cloud

Now that Pixie Cloud is running, we open a port and authenticate the CLI:

```sh {"terminalRows":"15"}
echo "🔑 Preparing Authentication..."

# Stop any old port-forwarding
pkill kubectl || true

# Forward the port so we can reach the cloud
kubectl port-forward --address 0.0.0.0 -n plc svc/cloud-proxy-service 8443:443 > /dev/null 2>&1 &
sleep 5

export PL_CLOUD_ADDR=127.0.0.1:8443
export PL_TESTING_ENV=dev

echo "⚠️ IMPORTANT FOR CODESPACES:"
echo "1. Go to the 'Ports' tab in VS Code."
echo "2. Make port 8443 'Public'."
echo "3. Click the globe icon to open the URL."
echo "4. Add '/login?local_mode=true' to the end of the URL and hit Enter."
echo "5. Log in with any email, copy the token, and paste it below:"

px auth login --manual
```

---

### Step 5: Deploy Pixie Vizier (eBPF Agents)

Once authenticated, deploy the eBPF agents and connect them to the local cloud:

```sh {"terminalRows":"15"}
echo "🚀 Deploying Pixie Operator and Vizier..."

# Ensure environment variables are still set
export PL_CLOUD_ADDR=127.0.0.1:8443
export PL_TESTING_ENV=dev

# Install Operator
helm repo add pixie-operator https://pixie-operator-charts.storage.googleapis.com
helm upgrade --install pixie-operator pixie-operator/pixie-operator-chart \
  --namespace pixie \
  --create-namespace

# Deploy Vizier
px deploy -y

# Fix the internal cloud address so agents can communicate inside the cluster
kubectl patch secret pl-cluster-secrets -n pl -p '{"stringData":{"PL_CLOUD_ADDR": "cloud-proxy-service.plc.svc.cluster.local:443"}}'

# Restart agents to pick up the new address
kubectl delete pods -n pl -l app=vizier

echo "✅ Deployment Complete! Checking Pods..."
kubectl get pods -n pl
```
```
