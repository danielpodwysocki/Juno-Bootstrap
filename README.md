<br />
<p align="center">
    <img src="assets/logo.png"/>
    <h3 align="center">Juno Bootstrap</h3>
    <p align="center">
        Deployment Bootstrap for Juno Deployments on Kubernetes
    </p>
</p>

<br />

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Installation](#installation)
- [Ingress & Envoy Gateway CRDs](#ingress--envoy-gateway-crds)

### Introduction

Juno Bootstrap is a collection of Helm charts that deploys the required services to run Juno on Kubernetes. 


### Prerequisites

- [Helm](https://helm.sh/docs/intro/install/)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/)

## Configuration

Juno uses Helm to bootstrap an existing cluster with the minimum required services. We ship a default `values.yaml` 
file that contains the required fields that are needed to deploy a Juno ready cluster. All fields are required unless
commented out. We recommend copying the default `values.yaml` file to `.values.yaml` and editing it to set the desired 
configuration for your deployment.

1. Copy the default `values.yaml` file to `.values.yaml`:
    ```bash
    cp values.yaml .values.yaml
    ```
2. Edit the `.values.yaml` file to set the desired configuration for your deployment. All fields that are required are commented with `(REQUIRED)`.

## Installation

1. Create the ArgoCD namespace:
    ```bash
    kubectl create namespace argocd
    ```
2. Install ArgoCD:
    ```bash
    kubectl create -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
3. Configure your deployment using the predefined [Juno Deployment Configurations](/deployments/README.md) (optional, but recommended).
4. Install Juno:
   1. If you are using the predefined Juno deployment configurations, run the following command (replace the `<predefined deployment>` with the path to the predefined Juno deployment configuration):
       ```bash
          helm install juno ./chart/ \
           -f <predefined deployment> \
           -f <predefined deployment> \
           -f ./.values.yaml
       ```
    2. If you are using your own configuration, run the following command:
         ```bash
             helm install juno ./chart/ \
              -f ./.values.yaml
         ```

## Ingress & Envoy Gateway CRDs

With the default ingress provider (`orionIngressProvider: envoy-gateway`), the
bootstrap installs [Envoy Gateway](https://gateway.envoyproxy.io/) as a secondary
gateway (controller, `GatewayClass`, `EnvoyProxy`, and a `Gateway`) and the Juno
workload charts attach their `HTTPRoute`/`SecurityPolicy` resources to it.

### CRDs are NOT installed by default

The `gateway-helm` chart **bundles the Gateway API and Envoy Gateway CRDs**. By
default this bootstrap **does not install or manage them** (`envoyGateway.installCRDs:
false`, which sets `skipCrds` on the Argo CD Application). This is deliberate:

> CRDs are cluster-wide and shared. If Argo CD owns them and ever prunes them — a
> failed sync, an app deletion, an ownership change — **every `Gateway`,
> `HTTPRoute`, and `EnvoyProxy` object on the cluster is cascade-deleted**, taking
> down all ingress. Keeping CRDs out of the GitOps prune path avoids this.

You therefore have two options:

**Option A — provision the CRDs out-of-band (recommended).** Install them once,
manually, before deploying Juno (and manage their lifecycle separately from Argo):

```bash
# Gateway API CRDs (pick the channel/version your gateways need)
kubectl apply --server-side --force-conflicts \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/experimental-install.yaml

# Envoy Gateway CRDs (from the gateway-helm chart you deploy)
helm pull oci://docker.io/envoyproxy/gateway-helm --version v1.8.1 --untar
kubectl apply --server-side --force-conflicts -f gateway-helm/charts/crds/crds/generated/
```

**Option B — let this chart install/manage them.** Set the value explicitly:

```yaml
envoyGateway:
  installCRDs: true   # chart installs + manages the CRDs (Argo owns them)
```

Use Option B only if you accept that Argo CD owns these cluster-wide CRDs and you
have prune safeguards in place.
