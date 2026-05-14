# crossplane-hub-and-spoke

Spec-driven hub-and-spoke network platform built with [Crossplane](https://crossplane.io) v2.  
Provisions consistent network topologies across **AWS**, **Azure**, and **GCP** from a single unified API.

| Cloud | Hub mechanism | Spoke connection |
|-------|---------------|-----------------|
| AWS   | VPC + Transit Gateway | TGW VPC Attachment + route |
| Azure | VNet + GatewaySubnet  | VNet Peering (bidirectional) |
| GCP   | VPC Network + Cloud NAT | VPC Peering (bidirectional) ⚠ non-transitive |

---

## Repository layout

```
.
├── crossplane.yaml               # Crossplane Configuration package manifest
├── apis/
│   ├── xhubnetwork/
│   │   ├── definition.yaml       # XRD — XHubNetwork
│   │   ├── composition-aws.yaml  # AWS: VPC + IGW + TGW + subnets
│   │   ├── composition-azure.yaml# Azure: VNet + GatewaySubnet
│   │   └── composition-gcp.yaml  # GCP: Network + Subnetwork + Router + NAT
│   └── xspokenetwork/
│       ├── definition.yaml       # XRD — XSpokeNetwork
│       ├── composition-aws.yaml  # AWS: VPC + TGW Attachment + routes
│       ├── composition-azure.yaml# Azure: VNet + bidirectional VNet Peering
│       └── composition-gcp.yaml  # GCP: Network + bidirectional VPC Peering
├── cluster/
│   ├── install/
│   │   ├── crossplane.yaml       # Helm values for crossplane-stable/crossplane
│   │   ├── providers.yaml        # Provider family + sub-provider installs
│   │   └── functions.yaml        # function-patch-and-transform install
│   └── config/
│       ├── providerconfig-aws.yaml
│       ├── providerconfig-azure.yaml
│       └── providerconfig-gcp.yaml
└── examples/
    ├── aws/{hub,spoke}.yaml
    ├── azure/{hub,spoke}.yaml
    └── gcp/{hub,spoke}.yaml
```

---

## Spec-driven development — the core idea

```
XRD (definition.yaml)          ← "the spec" — your platform API contract
    ↓ implements
Composition (composition-*.yaml) ← maps spec → provider-native resources
    ↓ creates
Managed Resources              ← actual cloud objects (VPC, TGW, VNet…)
```

Platform teams own the XRDs and Compositions.  
App teams interact only with `XHubNetwork` / `XSpokeNetwork` composite resources.

---

## Local testing (no cloud credentials)

This validates that XRDs and Compositions parse and are accepted by Crossplane without provisioning any real cloud resources. Resources will be created but remain in `NotReady` state since the credentials are fake.

### Prerequisites

Install the following tools before starting:

| Tool | Install (Windows) | Install (macOS) |
|------|-------------------|-----------------|
| [kind](https://kind.sigs.k8s.io/) | `winget install Kubernetes.kind` | `brew install kind` |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | `winget install Kubernetes.kubectl` | `brew install kubectl` |
| [helm](https://helm.sh/) | `winget install Helm.Helm` | `brew install helm` |

> **Windows note:** After installing with `winget`, reload your PATH before running the tools:
> ```powershell
> $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
> ```

> **PowerShell note:** PowerShell does not support `\` line continuation like bash. All multi-line bash examples below must be written as a single line in PowerShell.

### Step 1 — Create a local cluster

```bash
kind create cluster --name crossplane-local
kubectl cluster-info --context kind-crossplane-local
```

### Step 2 — Install Crossplane

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace --values cluster/install/crossplane.yaml

# Wait until both pods are Running (takes ~30 s)
kubectl get pods -n crossplane-system
```

Expected output:
```
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-xxx                             1/1     Running   0          30s
crossplane-rbac-manager-xxx               1/1     Running   0          30s
```

### Step 3 — Install providers and functions

Wait until Crossplane is `Running` before applying — the webhook is not ready immediately and the first apply may fail. Re-run if it does.

```bash
kubectl apply -f cluster/install/providers.yaml
kubectl apply -f cluster/install/functions.yaml

# Wait until all show INSTALLED=True and HEALTHY=True (takes 2–5 min)
kubectl get providers,functions
```

### Step 4 — Create fake credential secrets

```bash
kubectl create secret generic aws-creds --from-literal=credentials="[default]" --namespace crossplane-system
kubectl create secret generic azure-creds --from-literal=credentials='{"clientId":"fake","clientSecret":"fake","subscriptionId":"fake","tenantId":"fake"}' --namespace crossplane-system
kubectl create secret generic gcp-creds --from-literal=credentials='{"type":"service_account","project_id":"fake"}' --namespace crossplane-system
```

### Step 5 — Apply ProviderConfigs

```bash
kubectl apply -f cluster/config/providerconfig-aws.yaml
kubectl apply -f cluster/config/providerconfig-azure.yaml
kubectl apply -f cluster/config/providerconfig-gcp.yaml
```

### Step 6 — Install XRDs and Compositions

```bash
kubectl apply -f apis/xhubnetwork/
kubectl apply -f apis/xspokenetwork/
```

### Step 7 — Verify XRDs are established

```bash
kubectl get xrd
```

Expected output — both XRDs must show `ESTABLISHED: True`:
```
NAME                                 ESTABLISHED   AGE
xhubnetworks.network.platform.io     True          30s
xspokenetworks.network.platform.io   True          10s
```

Both XRDs established means the schemas were parsed and accepted. Compositions are wired correctly.

### Teardown

```bash
kind delete cluster --name crossplane-local
```

---

## Bootstrap sequence (with real cloud credentials)

### 1 — Install Crossplane

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace --values cluster/install/crossplane.yaml
```

### 2 — Install providers and functions

```bash
kubectl apply -f cluster/install/providers.yaml
kubectl apply -f cluster/install/functions.yaml

# Wait until all providers and functions are healthy
kubectl get providers,functions
```

### 3 — Configure cloud credentials

**AWS**
```bash
kubectl create secret generic aws-creds --from-file=credentials=$HOME/.aws/credentials --namespace crossplane-system
kubectl apply -f cluster/config/providerconfig-aws.yaml
```

**Azure**
```bash
az ad sp create-for-rbac --sdk-auth --role Contributor --scopes /subscriptions/<SUBSCRIPTION_ID> > /tmp/azure-creds.json
kubectl create secret generic azure-creds --from-file=credentials=/tmp/azure-creds.json --namespace crossplane-system
kubectl apply -f cluster/config/providerconfig-azure.yaml
```

**GCP**
```bash
gcloud iam service-accounts keys create /tmp/gcp-creds.json --iam-account=crossplane-sa@<PROJECT_ID>.iam.gserviceaccount.com
kubectl create secret generic gcp-creds --from-file=credentials=/tmp/gcp-creds.json --namespace crossplane-system
# Edit cluster/config/providerconfig-gcp.yaml — set your projectID
kubectl apply -f cluster/config/providerconfig-gcp.yaml
```

### 4 — Install XRDs and Compositions

```bash
kubectl apply -f apis/xhubnetwork/
kubectl apply -f apis/xspokenetwork/
```

### 5 — Provision networks

```bash
# Pick a cloud and apply:
kubectl apply -f examples/aws/hub.yaml    # or azure/ or gcp/
kubectl get xhubnetwork aws-hub -w

# Once the hub is Ready, provision a spoke:
kubectl apply -f examples/aws/spoke.yaml
kubectl get xspokenetwork aws-spoke-workloads -w
```

---

## Selecting a cloud provider

Composite resources use `compositionSelector` labels:

```yaml
spec:
  compositionSelector:
    matchLabels:
      provider: aws      # aws | azure | gcp
      topology: hub-and-spoke
```

---

## Extending this template

| Task | Where |
|------|-------|
| Add a new spoke (same cloud) | Copy `examples/<cloud>/spoke.yaml`, change CIDR + name |
| Add NAT Gateway to AWS hub | Add `NATGateway` + `EIP` resources to `composition-aws.yaml` |
| Add Azure Firewall | Add `Firewall` resource to `composition-azure.yaml` |
| Upgrade GCP to NCC (transitive routing) | Replace `NetworkPeering` resources with `NetworkConnectivityHub` + `NetworkConnectivitySpoke` |
| Version the API | Add `v1beta1` to the XRD `versions` list; set `referenceable: true` on the new version |
| Package as OCI image | `up xpkg build && up xpkg push` |

---

## GCP note — non-transitive peering

GCP VPC Peering does not forward traffic between peered networks (spoke A cannot reach spoke B through the hub). For transitive routing either:

- Deploy a network appliance (NAT VM) in the hub and set custom routes
- Migrate to **Network Connectivity Center (NCC)** using `networkconnectivity.gcp.upbound.io` resources

---

## References

- [Crossplane docs](https://docs.crossplane.io/latest/)
- [upbound/configuration-aws-network](https://github.com/upbound/configuration-aws-network) — official AWS network reference
- [upbound/platform-ref-multi-k8s](https://github.com/upbound/platform-ref-multi-k8s) — multi-cloud reference platform
- [Upbound Marketplace](https://marketplace.upbound.io/providers) — provider & function registry
- [AWS Transit Gateway hub-and-spoke](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-gateway.html)
- [Azure VNet Peering hub-and-spoke](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke-virtual-wan-architecture)
- [Crossplane XR design best practices](https://blog.upbound.io/crossplane-xr-best-practices)
