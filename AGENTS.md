# Copilot Agent Notes

This file documents the reasoning behind the changes made to `nemoclaw-k8s-orig.yaml` when producing `nemoclaw-k8s.yaml`.

## Why a Deployment instead of a Pod

The original manifest used `kind: Pod`. This was changed to `kind: Deployment` for the following reasons:

- **Self-healing**: A bare `Pod` is not restarted by Kubernetes if it crashes or is evicted. A `Deployment` ensures the pod is always reconciled back to the desired state.
- **Rolling updates**: `Deployment` supports rolling restarts (`kubectl rollout restart`) and version upgrades without manual pod deletion.
- **Standard practice**: `Deployment` is the recommended workload primitive for long-running processes in Kubernetes. Bare pods are generally only used for one-shot jobs or debugging.
- **ConfigMap decoupling**: Moving env vars to a `ConfigMap` pairs naturally with `Deployment` — the ConfigMap can be updated and the deployment restarted independently, without touching the pod spec.

## Network Architecture & CIDR Separation

The original manifest assumed a Docker-enabled host. On k3s/rke2 clusters, several networking issues arose due to overlapping CIDRs.

### The Problem: Overlapping CIDRs

By default, k3s uses `10.42.0.0/16` for Pods and `10.43.0.0/16` for Services. If the inner k3s cluster (running inside DinD) uses the same defaults, routing becomes ambiguous. Packets destined for the inner cluster might be routed to the outer cluster and vice-versa.

### The Solution: Explicit CIDR Separation

We now explicitly configure the inner k3s cluster to use non-conflicting ranges:
- **Cluster CIDR:** `10.44.0.0/16`
- **Service CIDR:** `10.45.0.0/16`
- **Cluster DNS:** `10.45.0.10`

These are injected via `/etc/rancher/k3s/config.yaml` and `flannel/subnet.env` inside the openshell container.

## DNS Strategy

### Direct DNS via daemon.json

Previously, a `socat` UDP/TCP proxy was used to bridge DNS queries. This was found to be fragile. The current approach is more direct:

1.  **Upstream Detection:** The `dind` startup script detects the real upstream nameserver from `/etc/resolv.conf` (ignoring `127.0.0.1`).
2.  **Docker Configuration:** This nameserver is injected directly into Docker's `daemon.json` under the `"dns"` key.
3.  **Result:** Containers started by Docker (including the openshell gateway) inherit this upstream DNS directly, bypassing the need for a local proxy.

## nftables and Forwarding

Docker's `iptables` rules often conflict with k3s's `nftables` setup. We use `nft` directly to:
1.  **MASQUERADE:** Ensure traffic from Docker bridge networks (`172.16.0.0/12`, `192.168.0.0/16`) is masqueraded to the pod's IP.
2.  **FORWARD:** Explicitly set the `FORWARD` chain policy to `accept` to ensure packets can flow between the Docker bridges and the pod's `eth0`.

## Current Status: Still Not Working

Despite these changes, the deployment fails with:
- **`Cluster DNS resolution failed`**: The openshell installer reports this even when DNS seems to be resolving.
- **`network is unreachable`**: Logs show `dial tcp 172.16.1.2:6443: connect: network is unreachable`. This suggests that the inner k3s node cannot reach its own control plane IP or that the tunnel setup is failing.

### Next Investigation Steps:
- Verify if `172.16.x.x` routes are being correctly installed inside the `dind` container.
- Check for `iptables` vs `nftables` conflicts specifically in the `FORWARD` chain.
- Investigate why the health check thinks DNS is failing even when `ghcr.io` resolution is successful.

## k3s image pre-loading

The `k3s-inject` watcher still handles pre-pulling and injecting the `pause` and `coredns` images to speed up k3s startup and help meet the tight health-check window (~14s).
