# NemoClaw on Kubernetes

> **⚠️ Work in progress — not fully working yet.**
> The DinD networking and Ollama integration are functional, but the openshell gateway
> still fails its internal DNS health check (`Cluster DNS resolution failed`) on k3s/rke2.
> Investigation is ongoing — see [AGENTS.md](AGENTS.md) for the full debugging history.

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
│    172.17.0.0/16           (Ollama FQDN)         │
└─────────────────────────────────────────────────┘
```

**nftables MASQUERADE** (`dind` container): Docker's iptables-based MASQUERADE is broken on k3s/rke2 (nftables-based nodes). An nft rule rewrites the source IP of all Docker bridge traffic (`172.16.0.0/12 → pod IP`) so inner containers can reach K8s CoreDNS and external registries. The CoreDNS IP is configured directly in `daemon.json`.

**Ollama TCP proxy** (`workspace` container): socat listens on `127.0.0.1:11434` and forwards TCP to the Ollama K8s service FQDN. The hostname `host.openshell.internal` is added to `/etc/hosts` so the NemoClaw installer can reach Ollama via the stable hostname it expects.

## Notes

- Requires `privileged: true` on the `dind` container for Docker-in-Docker. Ensure the `nemoclaw` namespace allows privileged pods:
  ```bash
  kubectl label ns nemoclaw pod-security.kubernetes.io/enforce=privileged
  ```
- `replicas: 1` only — multiple replicas on the same node would conflict on the Docker bridge subnet.
- No external internet access is required. All image pulls and DNS resolution flow through the cluster network.
