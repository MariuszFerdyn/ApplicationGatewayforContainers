# ApplicationGatewayforContainers

This sample shows how to use **Application Gateway for Containers** not only with **Azure Kubernetes Service (AKS)**, but also with external backends such as:

- **External websites/services** — **working** (`rzetelnekursy.pl`, Step 4), on its own dedicated frontend/FQDN
- **Azure Web Apps** — *(TODO)*
- **Virtual Machines** — *(TODO)*

Run the commands below top to bottom (copy/paste) in a single shell session.

> **Note on the external part (Step 4):** routing AGC to a backend defined via a hand-authored `EndpointSlice` is **not documented by Microsoft for AGC**, but it works — AGC reaches the external public IP and serves it. Two details make it work: a `BackendTLSPolicy` with the correct `sni`, and rewriting the backend Host via **`URLRewrite.hostname`** (a `Host` header set via `RequestHeaderModifier` is *not* programmable on AGC and leaves the route `Programmed=False`).

## Files in this repo

```
alb.yaml                       # ApplicationLoadBalancer CR (subnet id injected at apply time)
deploy-application.yaml        # app + Service + Gateway (agc-gateway) + HTTPRoute
external-rzetelnekursy.yaml    # external backend (Service + EndpointSlice + BackendTLSPolicy)
                               #   + dedicated Gateway + HTTPRoute (IP injected at apply time)
```

## Maintenance instructions

When changing this sample, always review:

- All ARM template `.json` files (none today — cluster is created via `az` CLI below)
- All Kubernetes/application `.yaml` files

And always update `README.md` together with those changes. When implementing a TODO backend (Azure Web Apps, Virtual Machines), add its manifest(s) here and a matching step in this README.

## Prerequisites

- Azure CLI logged in (`az login`) with rights to create resources and role assignments.
- `kubectl` and `dig` available locally.
- A **region where AGC is available** (this guide uses `westeurope`). Full list: <https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview#supported-regions>

```bash
az login
export SUBSCRIPTION_ID='<your-subscription-id>'
az account set --subscription "$SUBSCRIPTION_ID"

az extension add --name alb
az extension update --name alb

az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.NetworkFunction
az provider register --namespace Microsoft.ServiceNetworking

# Register the AGC add-on preview features (one-time per subscription).
# Both are required for the REST API path; skipping returns PreviewFeatureNotRegistered.
az feature register --namespace Microsoft.ContainerService --name ApplicationLoadBalancerPreview
az feature register --namespace Microsoft.ContainerService --name ManagedGatewayAPIPreview

# Wait until BOTH report "Registered" (can take a few minutes)
az feature show --namespace Microsoft.ContainerService --name ApplicationLoadBalancerPreview --query properties.state -o tsv
az feature show --namespace Microsoft.ContainerService --name ManagedGatewayAPIPreview --query properties.state -o tsv

# Once Registered, refresh the provider
az provider register --namespace Microsoft.ContainerService
```

## Variables

```bash
export RESOURCE_GROUP='rg-agc-lab'
export LOCATION='westeurope'              # must be AGC-supported (see link above)
export AKS_NAME='k8s-agc-lab'
export VM_SIZE='Standard_DS2_v2'

export VNET_NAME='vnet-agc-lab'
export NODE_SUBNET='subnet-nodes'         # 10.10.1.0/24
export ALB_SUBNET='subnet-alb'            # 10.10.2.0/24 (delegated to AGC)

export EXTERNAL_FQDN='rzetelnekursy.pl'
```

## 1. Create the VNet and the cluster (Azure CNI Overlay, in the new VNet)

```bash
az group create -n "$RESOURCE_GROUP" -l "$LOCATION"

# VNet + node subnet
az network vnet create -g "$RESOURCE_GROUP" -n "$VNET_NAME" -l "$LOCATION" \
  --address-prefixes 10.10.0.0/16 \
  --subnet-name "$NODE_SUBNET" --subnet-prefixes 10.10.1.0/24

# AGC association subnet: /24, delegated to the AGC service
az network vnet subnet create -g "$RESOURCE_GROUP" --vnet-name "$VNET_NAME" \
  -n "$ALB_SUBNET" --address-prefixes 10.10.2.0/24 \
  --delegations 'Microsoft.ServiceNetworking/trafficControllers'

NODE_SUBNET_ID=$(az network vnet subnet show -g "$RESOURCE_GROUP" --vnet-name "$VNET_NAME" -n "$NODE_SUBNET" --query id -o tsv)

az aks create -g "$RESOURCE_GROUP" -n "$AKS_NAME" -l "$LOCATION" \
  --node-vm-size "$VM_SIZE" --node-count 1 \
  --network-plugin azure --network-plugin-mode overlay \
  --pod-cidr 10.244.0.0/16 \
  --vnet-subnet-id "$NODE_SUBNET_ID" \
  --enable-oidc-issuer --enable-workload-identity --no-ssh-key

az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_NAME" --overwrite-existing
```

## 2. Add Application Gateway for Containers

> Make sure the preview features from **Prerequisites** (`ApplicationLoadBalancerPreview`, `ManagedGatewayAPIPreview`) show `Registered` before running this — otherwise the REST call returns `PreviewFeatureNotRegistered`.

Enable the Gateway API + ALB Controller add-ons via the AKS REST API (works regardless of `az aks` CLI version):

```bash
AKS_ID=$(az aks show -g "$RESOURCE_GROUP" -n "$AKS_NAME" --query id -o tsv)
AKS_REGION=$(az aks show -g "$RESOURCE_GROUP" -n "$AKS_NAME" --query location -o tsv)

# NOTE: double-quoted --body so $AKS_REGION is interpolated (single quotes would send it literally)
az rest --method put \
  --uri "https://management.azure.com${AKS_ID}?api-version=2025-09-02-preview" \
  --body "{
    \"location\": \"${AKS_REGION}\",
    \"properties\": {
      \"ingressProfile\": {
        \"applicationLoadBalancer\": { \"enabled\": true },
        \"gatewayAPI\": { \"installation\": \"Standard\" }
      }
    }
  }" --verbose
```

Wait for the cluster update to finish and the controller + CRDs to install **before** applying `alb.yaml` (the PUT is async; applying too early gives `no matches for kind "ApplicationLoadBalancer" ... ensure CRDs are installed first`):

```bash
az aks show -g "$RESOURCE_GROUP" -n "$AKS_NAME" --query provisioningState -o tsv   # wait for "Succeeded"
kubectl get pods -n kube-system | grep alb-controller                              # expect Running
kubectl get crd | grep alb.networking.azure.io                                     # ApplicationLoadBalancer CRD present
```

The add-on creates its own managed identity named `applicationloadbalancer-<cluster-name>` in the **node (MC_) resource group** and auto-assigns it Network Contributor + AppGw for Containers Configuration Manager + Reader **on the MC resource group** (plus a federated credential for `kube-system/alb-controller-sa`). Because this repo uses a **bring-your-own VNet**, the auto-granted Network Contributor is on the MC RG, not on your subnet — so additionally grant Network Contributor on your own AGC subnet:

```bash
MC_RG=$(az aks show -g "$RESOURCE_GROUP" -n "$AKS_NAME" --query nodeResourceGroup -o tsv)
az identity list -g "$MC_RG" -o table     # confirm name: applicationloadbalancer-<cluster-name>

PRINCIPAL_ID=$(az identity show -g "$MC_RG" -n "applicationloadbalancer-${AKS_NAME}" --query principalId -o tsv)
echo "ALB identity principalId: $PRINCIPAL_ID"

export ALB_SUBNET_ID=$(az network vnet subnet show -g "$RESOURCE_GROUP" --vnet-name "$VNET_NAME" -n "$ALB_SUBNET" --query id -o tsv)

# Network Contributor (subnet join) on YOUR (BYO) AGC subnet
az role assignment create --assignee-object-id "$PRINCIPAL_ID" --assignee-principal-type ServicePrincipal \
  --scope "$ALB_SUBNET_ID" --role "4d97b98b-1d4f-4787-a291-c67834d212e7"
```

> If no `applicationloadbalancer-*` identity exists yet, the add-on hasn't finished provisioning — check `kubectl get pods -n kube-system | grep alb-controller`.

Provision the AGC resource — inject the subnet id into `alb.yaml` and apply (the `ApplicationLoadBalancer` CR is created in `default`; both Gateways reference it via the `alb-namespace`/`alb-name` annotations):

```bash
sed "s#__ALB_SUBNET_ID__#${ALB_SUBNET_ID}#" alb.yaml | kubectl apply -f -

# Wait until it reports a provisioned state (re-run to re-check)
kubectl get applicationloadbalancer -n default -o yaml | grep -iE "name:|state:|message:"
```

## 3. Deploy the application and expose it via AGC

```bash
kubectl apply -f deploy-application.yaml

kubectl get pods -l app=app -o wide

# The frontend FQDN is NOT assigned immediately — the Gateway must reach "Programmed" first
kubectl wait --for=condition=Programmed gateway/agc-gateway -n default --timeout=600s

export FQDN=$(kubectl get gateway agc-gateway -n default -o jsonpath='{.status.addresses[0].value}')
echo "$FQDN"
curl -I http://$FQDN/
```

If this returns the app, AGC is correctly wired to the cluster.

## 4. External website/service backend — `rzetelnekursy.pl` on a dedicated frontend

`external-rzetelnekursy.yaml` contains everything for the external backend: the selector-less `Service` + manual `EndpointSlice` (the external public IP), a `BackendTLSPolicy` (SNI), and a **dedicated `Gateway` + `HTTPRoute`** so it gets its own FQDN and never collides with the app's `/` route. In the managed model the ALB Controller auto-provisions a new frontend per Gateway — no `az network alb` command needed.

Resolve the public IP, write it into the manifest, and apply:

```bash
IP=$(dig +short "$EXTERNAL_FQDN" A | grep -E '^[0-9]+\.' | head -n1)
echo "$EXTERNAL_FQDN -> $IP"

# Put the resolved IP into the file in place (idempotent: replaces placeholder or a prior IP)
sed -i -E "s/addresses: \[\"[^\"]*\"\]/addresses: [\"${IP}\"]/" external-rzetelnekursy.yaml

kubectl apply -f external-rzetelnekursy.yaml
```

Get the dedicated FQDN (different from the app's) and test:

```bash
kubectl wait --for=condition=Programmed gateway/ext-gateway -n default --timeout=600s
export FQDN_EXT=$(kubectl get gateway ext-gateway -n default -o jsonpath='{.status.addresses[0].value}')
echo "$FQDN_EXT"
curl -i http://$FQDN_EXT/      # serves rzetelnekursy.pl
```

### If it doesn't work, check

```bash
# Endpoint pushed to AGC?  Expect: "Successfully sent endpoint update request" / OPERATION_STATUS_SUCCESS
kubectl -n kube-system logs deploy/alb-controller | grep -iE "endpoint update|OPERATION_STATUS"

# Route must be Programmed=True (Accepted but Programmed=False usually means a non-programmable filter,
# e.g. a "Host" header set via RequestHeaderModifier instead of URLRewrite.hostname):
kubectl get httproute route-rzetelnekursy -n default -o jsonpath='{range .status.parents[*].conditions[*]}{.type}={.status} | {.message}{"\n"}{end}'

# BackendTLSPolicy must be Accepted:
kubectl get backendtlspolicy rzetelnekursy-tls -n default -o jsonpath='{range .status.conditions[*]}{.type}={.status} | {.message}{"\n"}{end}'
```

## TODO backends

- **Azure Web Apps** — App Service is a multitenant origin; reaching it from AGC needs the right SNI/Host (`BackendTLSPolicy.sni` + `URLRewrite.hostname`) and, for private access, a Private Endpoint. Add the manifest + a step here.
- **Virtual Machines** — a VM reachable in the VNet works the same way: point the `EndpointSlice` at its IP. Add the manifest + a step here.

## Gotchas

- **Backend Host rewrite must use `URLRewrite.hostname`**, not a `Host` header via `RequestHeaderModifier` — the latter is not programmable on AGC and leaves the route `Accepted` but `Programmed=False` (frontend then returns 404).
- **BackendTLSPolicy**: omit the `verify` block for public-CA origins (AGC trusts well-known CAs by default); adding `verify` requires `verify.caCertificateRef`.
- **Dedicated frontends are free in the managed model** — each Gateway gets its own frontend/FQDN automatically; you don't run `az network alb frontend create` (that's the BYO-deployment model).
- **Path vs hostname routing** — two routes on the *same* Gateway (`/` and `/something`) collide unless precedence is exact; giving the external backend its own Gateway/FQDN avoids that entirely.
- **Region must be AGC-supported**; **kubenet is not supported** (use Azure CNI Overlay); **same VNet** for AGC + nodes, no peering; AGC subnet must be a **/24**.
- The add-on creates the identity `applicationloadbalancer-<cluster-name>` in the **MC_ (node) resource group** and runs the controller in **kube-system**. For a BYO VNet you must add Network Contributor on your own AGC subnet (Step 2).
- **Built-in WAF** on AGC uses Azure WAF policies — it is **not** ModSecurity/OWASP CRS; rules don't port 1:1 from an nginx setup.
- The external backend IP can change; for anything non-throwaway, drive the `EndpointSlice` from an operator/pipeline rather than a one-shot edit.

## Sources

- AGC ALB Controller — AKS add-on quickstart (incl. REST API enablement): https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-deploy-application-gateway-for-containers-alb-controller-addon
- Supported regions / overview: https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/overview#supported-regions
- BYO deployment (subnet delegation + role assignments): https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/quickstart-create-application-gateway-for-containers-byo-deployment
- Container networking (no kubenet, /24, same VNet, >=1.7.9): https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/container-networking
- Backend mTLS / BackendTLSPolicy: https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/how-to-backend-mtls-gateway-api
- Selector-less Service + EndpointSlice pattern (generic, different controller): https://gateway.envoyproxy.io/latest/tasks/traffic/routing-outside-kubernetes/
