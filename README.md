# GateKey Kustomize

Kustomize manifests for deploying [GateKey](https://github.com/dye-tech/GateKey) - a modern Zero-Trust VPN control plane.

## Overview

This repository provides Kustomize bases and overlays for flexible GateKey deployments. Use different overlays to deploy:

- **Control Plane** - Central management (server, web UI, database)
- **Full Stack** - All-in-one deployment with VPN gateway
- **Remote Gateways** - Standalone VPN gateways at edge locations
- **Mesh Networking** - Hub and spoke topology for site-to-site VPN

## Prerequisites

- Kubernetes 1.19+
- kubectl with Kustomize support (v1.14+)
- For gateways/hubs: LoadBalancer or NodePort capability

## Network Requirements

Components requiring **inbound connections** (need LoadBalancer/NodePort):

| Component | Port | Protocol |
|-----------|------|----------|
| Web UI | 8080 | TCP |
| Server API | 8080 | TCP |
| OpenVPN Gateway/Hub | 1194 | UDP |
| WireGuard Gateway/Hub | 51820 | UDP |

Components requiring **outbound only** (no service exposure needed):

| Component | Connects To |
|-----------|-------------|
| OpenVPN Spoke | Hub (UDP 1194) |
| WireGuard Spoke | Hub (UDP 51820) |

## Directory Structure

```
.
├── base/                    # Base manifests for each component
│   ├── server/              # GateKey API server
│   ├── web/                 # Web UI
│   ├── postgresql/          # Database
│   ├── openvpn-gateway/     # OpenVPN standalone gateway
│   ├── openvpn-hub/         # OpenVPN mesh hub
│   ├── openvpn-spoke/       # OpenVPN mesh spoke
│   ├── wireguard-gateway/   # WireGuard standalone gateway
│   ├── wireguard-hub/       # WireGuard mesh hub
│   └── wireguard-spoke/     # WireGuard mesh spoke
└── overlays/                # Pre-configured deployment scenarios
    ├── control-plane/       # Server + Web + DB
    ├── full-stack/          # Control plane + OpenVPN gateway
    ├── openvpn-gateway/     # Remote OpenVPN gateway
    ├── openvpn-hub/         # OpenVPN mesh hub
    ├── openvpn-spoke/       # OpenVPN mesh spoke
    ├── wireguard-gateway/   # Remote WireGuard gateway
    ├── wireguard-hub/       # WireGuard mesh hub
    └── wireguard-spoke/     # WireGuard mesh spoke
```

## Quick Start

### 1. Control Plane Deployment

Deploy the central management components:

```bash
# Create namespace
kubectl create namespace gatekey

# Edit secrets first
cd overlays/control-plane
# Edit kustomization.yaml to set your passwords

# Deploy
kubectl apply -k overlays/control-plane
```

### 2. Get Admin Password

```bash
kubectl get secret gatekey-secrets -n gatekey -o jsonpath='{.data.admin-password}' | base64 -d
```

### 3. Access Web UI

```bash
kubectl port-forward svc/gatekey-web 8080:8080 -n gatekey
# Open http://localhost:8080
```

## Deployment Scenarios

### Scenario 1: Control Plane Only

Deploy server, web UI, and database. Use when VPN gateways will be deployed separately.

```bash
# 1. Copy and customize the overlay
cp -r overlays/control-plane overlays/my-control-plane
cd overlays/my-control-plane

# 2. Edit kustomization.yaml - update secretGenerator with real values:
#    - database-password
#    - jwt-secret (generate with: openssl rand -base64 32)
#    - admin-password

# 3. Deploy
kubectl apply -k .
```

### Scenario 2: Full Stack (All-in-One)

Deploy everything in a single cluster including an OpenVPN gateway.

```bash
# 1. Copy and customize
cp -r overlays/full-stack overlays/my-deployment
cd overlays/my-deployment

# 2. Edit kustomization.yaml:
#    - Set database credentials
#    - Set jwt-secret and admin-password
#    - Set openvpn-gateway-token (get from admin panel after first deploy)

# 3. Deploy without gateway token first
kubectl apply -k .

# 4. Create gateway in admin panel, get token

# 5. Update kustomization.yaml with token, re-apply
kubectl apply -k .
```

### Scenario 3: Remote OpenVPN Gateway

Deploy a gateway at a remote location that connects to your central control plane.

**Prerequisites:**
- Running control plane (Scenario 1 or 2)
- Gateway token from admin panel

```bash
# 1. Copy and customize
cp -r overlays/openvpn-gateway overlays/site-a-gateway
cd overlays/site-a-gateway

# 2. Edit kustomization.yaml:
#    - Set server-url to your control plane URL
#    - Set openvpn-gateway-token

# 3. Deploy
kubectl apply -k .
```

### Scenario 4: Remote WireGuard Gateway

Same as Scenario 3, but using WireGuard for better performance.

```bash
cp -r overlays/wireguard-gateway overlays/site-a-wg-gateway
cd overlays/site-a-wg-gateway

# Edit kustomization.yaml with server-url and token
kubectl apply -k .
```

### Scenario 5: OpenVPN Mesh Hub

Deploy a hub for site-to-site mesh networking. Spokes at branch offices connect to this hub.

```bash
cp -r overlays/openvpn-hub overlays/mesh-hub
cd overlays/mesh-hub

# Edit kustomization.yaml:
#    - Set server-url
#    - Set openvpn-hub-token

kubectl apply -k .

# Note the hub's external IP for spoke configuration
kubectl get svc gatekey-openvpn-hub -n gatekey
```

### Scenario 6: OpenVPN Mesh Spoke

Deploy at branch offices to connect to the mesh hub.

```bash
cp -r overlays/openvpn-spoke overlays/branch-office-1
cd overlays/branch-office-1

# Edit kustomization.yaml:
#    - Set server-url (control plane)
#    - Set hub-address (hub's external IP/hostname)
#    - Set hub-port (default: 1194)
#    - Set openvpn-spoke-token

kubectl apply -k .
```

### Scenario 7 & 8: WireGuard Mesh

Same as Scenarios 5 & 6, using WireGuard overlays:

```bash
# Hub
kubectl apply -k overlays/wireguard-hub

# Spoke
kubectl apply -k overlays/wireguard-spoke
```

## Customization

### Change Service Type

Create a patch file in your overlay:

```yaml
# overlays/my-gateway/service-patch.yaml
apiVersion: v1
kind: Service
metadata:
  name: gatekey-openvpn-gateway
spec:
  type: NodePort
  ports:
    - port: 1194
      nodePort: 31194
```

Add to kustomization.yaml:

```yaml
patches:
  - path: service-patch.yaml
```

### Change Resource Limits

```yaml
# overlays/my-gateway/resources-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gatekey-openvpn-gateway
spec:
  template:
    spec:
      containers:
        - name: openvpn-gateway
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 512Mi
```

### Use External Database

For the control-plane overlay, remove postgresql and update the configmap:

```yaml
# overlays/my-control-plane/kustomization.yaml
resources:
  - ../../base/server
  - ../../base/web
  # Remove: - ../../base/postgresql

configMapGenerator:
  - name: gatekey-config
    behavior: replace
    literals:
      - database-host=your-rds-instance.amazonaws.com
      - database-port=5432
      - database-name=gatekey
      - server-url=http://gatekey-server:8080
```

### Add Ingress

```yaml
# overlays/my-control-plane/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gatekey-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: gatekey.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gatekey-web
                port:
                  number: 8080
    - host: gatekey-api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gatekey-server
                port:
                  number: 8080
```

Add to kustomization.yaml:

```yaml
resources:
  - ../../base/server
  - ../../base/web
  - ../../base/postgresql
  - ingress.yaml
```

## GitOps Usage

### With ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gatekey-control-plane
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/gatekey-kustomize
    targetRevision: main
    path: overlays/control-plane
  destination:
    server: https://kubernetes.default.svc
    namespace: gatekey
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### With Flux

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: gatekey-control-plane
  namespace: flux-system
spec:
  interval: 10m
  path: ./overlays/control-plane
  prune: true
  sourceRef:
    kind: GitRepository
    name: gatekey-kustomize
  targetNamespace: gatekey
```

## Secrets Management

For production, use a secrets management solution instead of storing secrets in kustomization.yaml:

### Option 1: Sealed Secrets

```bash
# Create sealed secret
kubectl create secret generic gatekey-secrets \
  --from-literal=database-password=xxx \
  --from-literal=jwt-secret=xxx \
  --from-literal=admin-password=xxx \
  --dry-run=client -o yaml | kubeseal > sealed-secrets.yaml
```

### Option 2: External Secrets Operator

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: gatekey-secrets
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: gatekey-secrets
  data:
    - secretKey: database-password
      remoteRef:
        key: gatekey/database
        property: password
```

### Option 3: SOPS

```bash
# Encrypt secrets file
sops -e secrets.yaml > secrets.enc.yaml
```

## Troubleshooting

### View Generated Manifests

```bash
kubectl kustomize overlays/control-plane
```

### Check Component Status

```bash
kubectl get pods -n gatekey
kubectl logs -n gatekey -l app.kubernetes.io/component=server
```

### Gateway Not Connecting

1. Verify the server URL is accessible from the gateway
2. Check the token is correct
3. Ensure firewall allows outbound HTTPS

```bash
kubectl logs -n gatekey -l app.kubernetes.io/component=openvpn-gateway
```

### Hub/Spoke Connection Issues

1. Verify hub has external IP: `kubectl get svc -n gatekey`
2. Check spoke has correct hub-address and hub-port
3. Ensure firewall allows UDP traffic

## Docker Images

| Image | Description |
|-------|-------------|
| `dyetech/gatekey-server` | Control plane API |
| `dyetech/gatekey-web` | Web UI |
| `dyetech/gatekey-gateway` | OpenVPN gateway |
| `dyetech/gatekey-hub` | OpenVPN mesh hub |
| `dyetech/gatekey-mesh-gateway` | OpenVPN mesh spoke |
| `dyetech/gatekey-wireguard-gateway` | WireGuard gateway |
| `dyetech/gatekey-wireguard-hub` | WireGuard mesh hub |
| `dyetech/gatekey-wireguard-mesh-gateway` | WireGuard mesh spoke |

## License

Apache 2.0 - See [LICENSE](https://github.com/dye-tech/GateKey/blob/main/LICENSE)
