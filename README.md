# NemoClaw on Kubernetes

Kubernetes manifests for running [NemoClaw](https://nvidia.com/nemoclaw) on a k3s/rke2 cluster using Docker-in-Docker (DinD) for sandbox isolation.

## Files

| File | Description |
|---|---|
| `nemoclaw-k8s.yaml` | Production manifest: `ConfigMap` + `Deployment` |
| `nemoclaw-k8s-orig.yaml` | Original reference manifest (`Pod`-based) |

## Prerequisites

```bash
kubectl create namespace nemoclaw
```

## Usage

1. Edit the `nemoclaw-config` ConfigMap in `nemoclaw-k8s.yaml` to match your cluster:

   | Key | Description |
   |---|---|
   | `DYNAMO_HOST` | `host:port` of your vLLM/Dynamo frontend service |
   | `NEMOCLAW_MODEL` | Model name to use |
   | `NEMOCLAW_SANDBOX_NAME` | Name for the NemoClaw sandbox |
   | `COMPATIBLE_API_KEY` | API key for the endpoint |

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
│  │  socat DNS  │        │  socat TCP proxy   │  │
│  │  proxy      │        │  NemoClaw installer│  │
│  └─────────────┘        └────────────────────┘  │
│         │                        │              │
│    DinD bridge             K8s pod network       │
│    172.17.0.0/16           (Dynamo FQDN)         │
└─────────────────────────────────────────────────┘
```

**DinD DNS proxy** (`dind` container): socat listens on `0.0.0.0:53` (the docker0 bridge gateway) and forwards UDP DNS queries to the K8s CoreDNS server. This allows containers running inside DinD to resolve image registries and cluster FQDNs without needing external internet access or NAT routing.

**Dynamo TCP proxy** (`workspace` container): socat listens on `127.0.0.1:8000` and forwards TCP to the Dynamo vLLM frontend service FQDN. The hostname `host.openshell.internal` is added to `/etc/hosts` pointing to `127.0.0.1` so the NemoClaw installer can reach the endpoint via a stable hostname.

## Notes

- Requires `privileged: true` on the `dind` container for Docker-in-Docker. Ensure the `nemoclaw` namespace allows privileged pods:
  ```bash
  kubectl label ns nemoclaw pod-security.kubernetes.io/enforce=privileged
  ```
- `replicas: 1` only — multiple replicas on the same node would conflict on the Docker bridge subnet.
- No external internet access is required. All image pulls and DNS resolution flow through the cluster network.
