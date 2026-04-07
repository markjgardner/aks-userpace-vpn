# AKS Userspace VPN

Feasibility test for running a WireGuard VPN endpoint inside a multi-tenant AKS cluster, enabling authenticated users to access internal microservices via a VPN tunnel while enforcing namespace-level isolation through Cilium network policies.

## Architecture

```
                    ┌─────────────────────────────────────────────────────┐
                    │                   AKS Cluster                       │
                    │                                                     │
  Internet          │   Namespace: app1                                   │
  ────────┐         │   ┌───────────────────────────────────────────────┐ │
          │         │   │                                               │ │
          ├─── HTTPS ──►│  Ingress ──► web-frontend (nginx)            │ │
          │         │   │                    │                           │ │
          │         │   │                    ▼                           │ │
          │         │   │              api-service (nginx)              │ │
          │         │   │                    │                           │ │
          │         │   │                    ▼                           │ │
          │         │   │            backend-service (nginx)            │ │
          │         │   │                                               │ │
  UserA   │         │   │                                               │ │
  ────────┘         │   │                                               │ │
          │         │   │                                               │ │
          └── UDP 51820►│  wireguard ──► all internal services         │ │
                    │   │   (VPN tunnel provides access)                │ │
                    │   └───────────────────────────────────────────────┘ │
                    │                                                     │
                    │   Namespace: other-tenant                           │
                    │   ┌───────────────────────────────────────────────┐ │
                    │   │  ✗ No access from app1 (network policy)      │ │
                    │   └───────────────────────────────────────────────┘ │
                    └─────────────────────────────────────────────────────┘
```

### Key Design Decisions

- **Tenant isolation**: Each tenant runs in a dedicated namespace. Network policies enforce strict boundaries.
- **WireGuard as a pod**: WireGuard runs as a container inside the tenant namespace using the `linuxserver/wireguard` image. A `LoadBalancer` service exposes UDP port 51820 to the internet.
- **VPN client routing**: Once connected via WireGuard, UserA's traffic enters the cluster through the WireGuard pod. Since the pod is inside the `app1` namespace, intra-namespace network policies allow it to reach all internal services.
- **Cross-namespace isolation**: Default-deny network policies block all traffic leaving the namespace (except DNS). VPN clients cannot reach services in other tenant namespaces.
- **Public web access**: An Ingress resource exposes only the `web-frontend` service to the public internet over HTTPS.

### Network Policy Summary

| Policy | Purpose |
|--------|---------|
| `default-deny` | Deny all ingress and egress in the namespace |
| `allow-intra-namespace` | Allow pod-to-pod traffic within `app1` |
| `allow-dns` | Allow DNS resolution via `kube-system/kube-dns` |
| `allow-ingress-web` | Allow external traffic to `web-frontend` on port 80 |
| `allow-wireguard-external` | Allow external UDP traffic to WireGuard on port 51820 |

## Prerequisites

- AKS cluster with **overlay CNI** and **Cilium** (dataplane and policy engine)
- `kubectl` configured to access the cluster
- An ingress controller installed (e.g., NGINX Ingress Controller)

## Deployment

Apply the manifests in order:

```bash
# 1. Create the namespace
kubectl apply -f k8s/namespace.yaml

# 2. Deploy microservices
kubectl apply -f k8s/microservices/

# 3. Deploy the ingress
kubectl apply -f k8s/ingress.yaml

# 4. Deploy WireGuard
kubectl apply -f k8s/wireguard/

# 5. Apply network policies
kubectl apply -f k8s/network-policies/
```

Or apply everything at once:

```bash
kubectl apply -R -f k8s/
```

## Connecting via WireGuard

1. Get the WireGuard service external IP:
   ```bash
   kubectl get svc wireguard -n app1
   ```

2. Retrieve the generated peer configuration:
   ```bash
   kubectl exec -n app1 deploy/wireguard -- cat /config/peer_user1/peer_user1.conf
   ```

3. Import the configuration into your WireGuard client and connect.

4. Once connected, you can access internal services by their cluster DNS names:
   ```bash
   curl http://web-frontend.app1.svc.cluster.local
   curl http://api-service.app1.svc.cluster.local
   curl http://backend-service.app1.svc.cluster.local
   ```

## File Structure

```
k8s/
├── namespace.yaml                          # app1 namespace
├── ingress.yaml                            # Ingress for web-frontend
├── microservices/
│   ├── web-frontend.yaml                   # Web frontend (nginx) + ClusterIP service
│   ├── api-service.yaml                    # API service (nginx) + ClusterIP service
│   └── backend-service.yaml                # Backend service (nginx) + ClusterIP service
├── wireguard/
│   └── deployment.yaml                     # WireGuard deployment + LoadBalancer service
└── network-policies/
    ├── default-deny.yaml                   # Default deny all
    ├── allow-intra-namespace.yaml          # Allow pod-to-pod within app1
    ├── allow-dns.yaml                      # Allow DNS egress to kube-system
    ├── allow-ingress-web.yaml              # Allow ingress to web-frontend
    └── allow-wireguard-external.yaml       # Allow external WireGuard access
```
