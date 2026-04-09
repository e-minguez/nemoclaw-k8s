# NemoClaw on Kubernetes

> **⚠️ Status: Not working — Investigation in progress.**
> Despite CIDR separation and direct DNS configuration, the openshell gateway
> still fails with `Cluster DNS resolution failed` and `network unreachable` (172.16.1.2:6443)
> on k3s/rke2 clusters. See [AGENTS.md](AGENTS.md) for the latest technical findings.

Kubernetes manifests for running [NemoClaw](https://nvidia.com/nemoclaw) on a k3s/rke2 cluster using Docker-in-Docker (DinD) for sandbox isolation.

## Files

| File | Description |
|---|---|
| `nemoclaw-k8s.yaml` | Production manifest: `ConfigMap` + `Deployment` |
| `nemoclaw-k8s-orig.yaml` | Original reference manifest (`Pod`-based) |

## Usage

1. Edit the `nemoclaw-config` ConfigMap in `nemoclaw-k8s.yaml` to match your cluster:

   | Key | Description |
   |---|---|
   | `OLLAMA_SERVICE` | `host:port` of your Ollama service (default: `ollama.ollama-system.svc.cluster.local:11434`) |
   | `NEMOCLAW_MODEL` | Ollama model tag to pull and use (e.g. `qwen2.5:14b`) |
   | `NEMOCLAW_SANDBOX_NAME` | Name for the NemoClaw sandbox |

2. Apply:
   ```bash
   kubectl apply -f nemoclaw-k8s.yaml
   ```

3. Follow logs:
   ```bash
   # DinD daemon
   kubectl logs -n nemoclaw deploy/nemoclaw -c dind -f

   # NemoClaw installer
   kubectl logs -n nemoclaw deploy/nemoclaw -c workspace -f
   ```

## Architecture

The pod runs two containers sharing volumes for the Docker socket:

```
┌─────────────────────────────────────────────────┐
│  Pod (nemoclaw)                                 │
│                                                 │
│  ┌─────────────┐        ┌────────────────────┐  │
│  │    dind     │        │     workspace      │  │
│  │             │        │                    │  │
│  │  dockerd ───┼──sock──┼─▶ docker CLI       │  │
│  │  nftables   │        │  socat TCP proxy   │  │
│  │  MASQUERADE │        │  NemoClaw installer│  │
│  └─────────────┘        └────────────────────┘  │
│         │                        │              │
│    DinD bridge             K8s pod network       │
│    172.16.0.0/12           (Ollama FQDN)         │
└─────────────────────────────────────────────────┘
```

**CIDR Separation:** The inner k3s cluster is configured with `10.44.0.0/16` (Pod) and `10.45.0.0/16` (Service) to avoid conflicts with the outer cluster's `10.42/10.43` ranges.

**Direct DNS:** The `dind` script detects the outer nameserver and injects it directly into Docker's `daemon.json`, ensuring consistent resolution for inner containers.

**nftables MASQUERADE:** An nft rule rewrites the source IP of all Docker bridge traffic (`172.16.0.0/12 → pod IP`) and explicitly allows `FORWARD` traffic to ensure inner containers can reach external registries and the control plane.

## Notes

- Requires `privileged: true` on the `dind` container for Docker-in-Docker. Ensure the `nemoclaw` namespace allows privileged pods:
  ```bash
  kubectl label ns nemoclaw pod-security.kubernetes.io/enforce=privileged
  ```
- `replicas: 1` only — multiple replicas on the same node would conflict on the Docker bridge subnet.
