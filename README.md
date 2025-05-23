# BrightCubes Talos Workshop

# Export TALOSCONFIG
export TALOSCONFIG=./talosconfig

# Starting the Talos docker containers
docker compose up -d
## task 01

# Generate the Talos configuration 
talosctl gen config workshop https://172.20.0.100:6443 --kubernetes-version=1.31.6
## task 02

# Update Talos Config
talosctl config endpoint 172.20.0.100
talosctl config node 172.20.0.100
## task 03

# Apply Talos Config
talosctl apply-config -f controlplane.yaml -n 172.20.0.100 --insecure
talosctl apply-config -f worker.yaml -n 172.20.0.150 --insecure
## task 04

# Dashboard (optional)
talosctl dashboard

# Bootstrap
talosctl bootstrap
talosctl health
## task 05

# Download kubeconfig
talosctl kubeconfig
## task 06

# Upgrade Talos CP & Worker
Change the version from 1.9.5 to 1.10.1 for both images (compose.yaml)
watch show-talos-version
docker compose up -d talos-cp
docker compose up -d talos-worker
## task 07

# Upgrade Kubernetes
talosctl upgrade-k8s --to=1.32.4
## task 08

# Run Workload
kubectl apply -f ./workload
## task 09

# Create a PodSecurity patch
patch.yaml

cluster:
  apiServer:
    admissionControl:
      - name: PodSecurity
        configuration:
          exemptions:
            namespaces:
              - game-2048
## task 10

# Apply the patch.yaml
talosctl patch machineconfig -p @patch.yaml
## task 11

# Restart Deployment
kubectl rollout -n game-2048 restart deployments/game-2048
## task 12