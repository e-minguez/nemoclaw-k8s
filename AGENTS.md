# Copilot Agent Notes

This file documents the reasoning behind the changes made to `nemoclaw-k8s-orig.yaml` when producing `nemoclaw-k8s.yaml`.

## Why a Deployment instead of a Pod

The original manifest used `kind: Pod`. This was changed to `kind: Deployment` for the following reasons:

- **Self-healing**: A bare `Pod` is not restarted by Kubernetes if it crashes or is evicted. A `Deployment` ensures the pod is always reconciled back to the desired state.
- **Rolling updates**: `Deployment` supports rolling restarts (`kubectl rollout restart`) and version upgrades without manual pod deletion.
- **Standard practice**: `Deployment` is the recommended workload primitive for long-running processes in Kubernetes. Bare pods are generally only used for one-shot jobs or debugging.
- **ConfigMap decoupling**: Moving env vars to a `ConfigMap` pairs naturally with `Deployment` — the ConfigMap can be updated and the deployment restarted independently, without touching the pod spec.

The `restartPolicy: Never` from the original pod was removed; it is not valid on `Deployment` pods, and the `Always` default is appropriate here.

## Why a ConfigMap

All environment variables were extracted from the inline `env:` blocks into a `ConfigMap` (`nemoclaw-config`):

- **Separation of concerns**: Configuration is decoupled from the workload spec. Cluster operators can update endpoint URLs or model names without editing the Deployment manifest.
- **Reusability**: Both `dind` and `workspace` containers share the same ConfigMap via `envFrom`, eliminating duplication.
- **Auditability**: ConfigMap changes are visible in `kubectl describe` and version-controlled separately.

## DinD DNS proxy (k3s/rke2 compatibility)

The original manifest assumed a Docker-enabled host. On k3s/rke2 clusters (which use `containerd`, not Docker), several networking issues arose:

### Problem 1: CoreDNS unreachable from DinD bridge (`operation not permitted`)

Containers running inside DinD get IPs on Docker's internal bridge network (`172.17.0.x`). By default, Docker configures these containers to use the K8s CoreDNS server (inherited from the pod's `/etc/resolv.conf`, e.g. `10.43.0.10`). However, the DinD bridge network cannot route UDP packets to the K8s cluster network — resulting in `write: operation not permitted`.

### Problem 2: External DNS unreachable (`i/o timeout` / `network is unreachable`)

Switching to public DNS (`8.8.8.8`) didn't help either. On k3s/rke2 nodes, Docker's `iptables MASQUERADE` rules for the bridge network conflict with the node's `nftables`-based CNI setup. Packets from `172.17.0.x` reach the socket but responses never return, or the network is entirely unreachable.

### Solution: socat UDP DNS proxy on the bridge gateway

The `dind` container now runs a socat UDP proxy that:

1. Listens on `0.0.0.0:53` (i.e. the docker0 bridge gateway `172.17.0.1`, which is always directly reachable by DinD containers — no NAT needed)
2. Forwards queries to the K8s CoreDNS server read from `/etc/resolv.conf`

`daemon.json` is configured with `dns: ["172.17.0.1"]` so all DinD containers use this proxy. CoreDNS resolves both external registries (`ghcr.io`, `docker.io`) and cluster FQDNs.

```
DinD container (172.17.0.x)
  → UDP:53 to 172.17.0.1 (bridge gateway, no NAT required)
    → socat in dind container
      → UDP:53 to K8s CoreDNS (pod network, always reachable)
        → resolves ghcr.io, docker.io, cluster FQDNs ✅
```

### Why not `hostNetwork: true`?

`hostNetwork: true` was tried but caused a different issue: on pod restart, a stale `docker0` bridge interface from the previous run persists in the host network namespace. Docker detects the existing interface, ignores the `bip` setting in `daemon.json`, and reuses the old bridge IP — breaking the DNS proxy binding. Keeping a clean pod network namespace (no `hostNetwork`) avoids this entirely.

## Dynamo TCP proxy

The `workspace` container runs a socat TCP proxy to bridge the NemoClaw installer to the Dynamo vLLM frontend:

```
NemoClaw installer → http://host.openshell.internal:8000/v1
  → /etc/hosts: host.openshell.internal = 127.0.0.1
    → socat TCP on 127.0.0.1:8000 (bound to loopback only, not exposed on node)
      → TCP to vllm-disagg-frontend.dynamo.svc.cluster.local:8000
```

The `bind=127.0.0.1` constraint on socat ensures port 8000 is not exposed beyond the pod's loopback, even if `hostNetwork` were re-enabled in the future.
