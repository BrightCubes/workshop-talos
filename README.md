# BrightCubes Talos Workshop

Build a two-node Talos Linux cluster on your laptop and manage it with `talosctl` only.

Both nodes run as Docker containers (`compose.yaml`): a control plane (`talos-cp`, 172.20.0.100) and a worker (`talos-worker`, 172.20.0.150). They start in maintenance mode, just like a bare-metal machine booted from the Talos ISO, so you go through the real install flow: generate a config, apply it, bootstrap Kubernetes.

> The previous version of this workshop (devcontainer + taskfile, Talos 1.9) lives on the `v1-workshop-talos` branch.

## Prerequisites

- **Docker Desktop, up to date.** Talos 1.14 needs a host kernel of 6.15 or newer when it runs in a container. Check it with:
  ```
  docker info --format '{{.KernelVersion}}'
  ```
  With an older kernel (e.g. Docker Desktop 29.7, kernel 6.12) the node never starts and its log shows `failed to set up /etc overlay ... key="lowerdir+"` ([siderolabs/talos#14476](https://github.com/siderolabs/talos/issues/14476)).
- **talosctl 1.14 or newer**: `brew install siderolabs/tap/talosctl`, or see the [Talos docs](https://docs.siderolabs.com/talos/latest/getting-started/talosctl)
- **kubectl**

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
`bootstrap` starts etcd on the control plane, once per cluster.

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
Apply it to the control plane, then restart the workload:
```
talosctl patch machineconfig -p @patch.yaml
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
