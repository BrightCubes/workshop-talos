# BrightCubes Talos Workshop

Build a two-node Talos Linux cluster on your laptop and manage it with `talosctl` only.

Both nodes run as Docker containers (`compose.yaml`): a control plane (`talos-cp`, 172.20.0.100) and a worker (`talos-worker`, 172.20.0.150). They start in maintenance mode, just like a bare-metal machine booted from the Talos ISO, so you go through the real install flow: generate a config, apply it, bootstrap Kubernetes.

> The previous version of this workshop (devcontainer + taskfile, Talos 1.9) lives on the `v1-workshop-talos` branch.

## Prerequisites

You need:
- **Docker Desktop, up to date.** Talos 1.14 needs a host kernel of 6.15 or newer when it runs in a container.
- **talosctl 1.14 or newer**
- **kubectl**
- **k9s** (optional): a terminal dashboard for Kubernetes, handy to watch pods and nodes during the upgrades

### macOS

Install [Docker Desktop](https://www.docker.com/products/docker-desktop/), or update it if you already have it. Then install the tools with [Homebrew](https://brew.sh):
```
brew install siderolabs/tap/talosctl kubectl k9s
```
Already installed? Update them with `brew upgrade siderolabs/tap/talosctl kubectl k9s`.

### Windows (WSL)

The workshop runs inside WSL (Ubuntu), not in PowerShell.

1. In PowerShell, install or update WSL:
   ```
   wsl --install
   wsl --update
   ```
2. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/), or update it. In Docker Desktop, go to **Settings → Resources → WSL integration** and enable it for Ubuntu.
3. Open Ubuntu and install [Homebrew](https://brew.sh):
   ```
   sudo apt-get update && sudo apt-get install -y build-essential curl git
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.bashrc
   eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
   ```
4. Install the tools, the same way as on macOS:
   ```
   brew install siderolabs/tap/talosctl kubectl k9s
   ```
   Already installed? Update them with `brew upgrade siderolabs/tap/talosctl kubectl k9s`.

### Get the repository

Clone this repository and go into it:
```
git clone https://github.com/BrightCubes/workshop-talos.git
cd workshop-talos
```
On Windows, run this inside Ubuntu, in your home directory (`cd ~`), not on the Windows drive (`/mnt/c`).

### Check your setup

Run this before the workshop:
```
docker info --format '{{.KernelVersion}}'
talosctl version --client --short
kubectl version --client
```
The kernel must be 6.15 or newer and talosctl 1.14 or newer. With an older kernel the Talos nodes never start, and their log shows `failed to set up /etc overlay ... key="lowerdir+"` ([siderolabs/talos#14476](https://github.com/siderolabs/talos/issues/14476)). Update Docker Desktop (and on Windows also run `wsl --update`).

To save time and wifi on the day, download the images in advance:
```
docker compose pull
TALOS_VERSION=v1.14.4 docker compose pull
docker pull amigoscode/2048@sha256:45a893ee62267d2825ffefb3b98e860de9716a5bb80091a84967d2bd427990db
```

All commands below are run from the root of this repository.

## 1. Point talosctl and kubectl to this workshop

Use local config files, so your own `~/.talos/config` and `~/.kube/config` are left alone:
```
export TALOSCONFIG=$PWD/talosconfig
export KUBECONFIG=$PWD/kubeconfig
```
Do this again in every new terminal.

## 2. Start the nodes

The Talos version comes from `.env` (`v1.13.12`, we upgrade later).
```
docker compose up -d
```
The nodes are now in maintenance mode, waiting for a config:
```
talosctl version --insecure -n 127.0.0.1
```

## 3. Generate the Talos configuration

```
talosctl gen config workshop https://172.20.0.100:6443 \
  --talos-version v1.13 \
  --kubernetes-version 1.36.3 \
  --additional-sans 127.0.0.1
```
This creates `controlplane.yaml`, `worker.yaml` and `talosconfig`. Each file describes a complete node.

- `--talos-version v1.13` generates a config for the Talos version the nodes run now. Without it, talosctl 1.14 adds settings that 1.13 doesn't know.
- `--additional-sans 127.0.0.1` adds localhost to the certificates, because you reach the nodes through ports published on your laptop.

## 4. Point talosctl at the cluster

```
talosctl config endpoint 127.0.0.1
talosctl config node 172.20.0.100
```
The endpoint is where talosctl connects (the control plane, through the published port). The node is which machine the command is about. The control plane forwards commands to other nodes, so `-n 172.20.0.150` reaches the worker.

## 5. Apply the configuration

```
talosctl apply-config --insecure -n 127.0.0.1 -f controlplane.yaml
talosctl apply-config --insecure -n 127.0.0.1:50001 -f worker.yaml
```
`--insecure` is only needed now: in maintenance mode a node has no certificates yet. Port 50001 is the worker's Talos API.

Optional, watch the nodes come up:
```
talosctl dashboard
```

## 6. Bootstrap Kubernetes

```
talosctl bootstrap
talosctl health
```
`bootstrap` starts etcd on the control plane, once per cluster. `health` keeps retrying until every check is `OK`, so errors like `connection refused` along the way are normal while the cluster starts.

## 7. Get the kubeconfig

```
talosctl kubeconfig $KUBECONFIG
kubectl config set-cluster workshop --server=https://127.0.0.1:6443
kubectl get nodes -o wide
```
The cluster endpoint is `172.20.0.100`, which your laptop can't reach, so we point kubectl at the published port instead.

## 8. Upgrade Talos

Change the version in `.env`:
```
TALOS_VERSION=v1.14.4
```
Upgrade one node at a time, starting with the control plane. Wait until it reports the new version before you continue:
```
docker compose up -d talos-cp
kubectl get nodes -o wide -w
```
```
docker compose up -d talos-worker
kubectl get nodes -o wide -w
```
Tip: open a second terminal, run the two `export` lines from step 1 and start `k9s`. You can then watch nodes and pods change during this upgrade and the next one (type `:nodes` or `:pods` to switch views).

In Docker an upgrade means starting the node from a new image. On real hardware you'd use `talosctl upgrade --image ...`.

## 9. Upgrade Kubernetes

```
talosctl upgrade-k8s --to=1.37.1 --endpoint https://127.0.0.1:6443
```
Kubernetes is upgraded separately from the OS, component by component. `--endpoint` is needed for the same reason as in step 7: by default talosctl talks to the Kubernetes API at `172.20.0.100`, which your laptop can't reach.

## 10. Run a workload

Deploy the 2048 game:
```
kubectl apply -f ./workload
kubectl get deployment -n game-2048
```
No pods start. Find out why:
```
kubectl get events -n game-2048
```
Talos enforces the PodSecurity `baseline` policy by default, and this workload asks for a privileged container.

## 11. Patch the machine config

Everything on a Talos node is changed through its machine config, never on the node itself. Exempt the `game-2048` namespace from PodSecurity with a `patch.yaml`:
```yaml
cluster:
  apiServer:
    admissionControl:
      - name: PodSecurity
        configuration:
          exemptions:
            namespaces:
              - game-2048
```
Apply it to the control plane:
```
talosctl patch machineconfig -p @patch.yaml
```
The kube-apiserver restarts to load the new setting, so kubectl answers `Unable to connect to the server` for about 20 seconds. Wait until this works again:
```
kubectl get nodes
```
Then restart the workload:
```
kubectl rollout restart deployment/game-2048 -n game-2048
kubectl get pods -n game-2048 -o wide
```
Play the game at http://localhost:30080

## Clean up

Remove the nodes, their volumes and the generated files:
```
docker compose down -v
rm -f talosconfig kubeconfig controlplane.yaml worker.yaml patch.yaml
```
Reset `.env` to `TALOS_VERSION=v1.13.12` to start over.

## Notes for trainers

- **sandboxd / workload isolation (new in Talos 1.14)** can't be done in this workshop. Talos never starts `sandboxd` when it runs in a container ([`sandboxd/enabled.go`](https://github.com/siderolabs/talos/blob/v1.14.4/internal/app/machined/pkg/sandboxd/enabled.go)), even with `workloadIsolation: true`. Show it on a real cluster instead.
- A cluster generated for 1.13 and upgraded to 1.14 has no `SecurityProfileConfig` document. A cluster generated for 1.14 gets `workloadIsolation: true` by default.
