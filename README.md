# OCI Native Ingress Controller Helm Chart

A Kubernetes Ingress controller for Oracle Cloud Infrastructure (OCI) that enables native OCI Load Balancer integration with your Kubernetes clusters, particularly for OKE (Oracle Kubernetes Engine).

## Overview

The OCI Native Ingress Controller manages OCI Native Load Balancers based on Kubernetes Ingress resources. It automatically provisions and configures load balancers in OCI when you create Ingress objects in your cluster.

**Key Features:**
- Native OCI Load Balancer provisioning
- Automatic SSL/TLS certificate management
- Multi-zone load balancing support
- Pod readiness gate injection for zero-downtime deployments
- Prometheus metrics integration
- RBAC and security hardened by default
- High availability with replica support

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- OKE cluster running on Oracle Cloud Infrastructure
- `cert-manager` 1.1.0+ installed in your cluster
- OCI credentials configured (instance principal or user credentials)

### Install cert-manager (if not already installed)

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

## Installation

### 1. Add the Helm Repository

```bash
helm repo add oci-native-ingress-controller https://github.com/amaanx86/oci-native-ingress-controller-helm
helm repo update
```

### 2. Create Required OCI Configuration

Create a secret with your OCI credentials (if not using instance principal):

```bash
kubectl create namespace native-ingress-controller-system

# If using user credentials:
kubectl create secret generic oci-config \
  --from-file=/path/to/.oci/config \
  --from-file=/path/to/.oci/ocikey.pem \
  -n native-ingress-controller-system
```

### 3. Install the Chart

```bash
helm install oci-native-ingress-controller \
  oci-native-ingress-controller/oci-native-ingress-controller \
  --namespace native-ingress-controller-system \
  --values values.yaml
```

### Example values.yaml

Create a `values.yaml` with your OCI configuration:

```yaml
# OCI Configuration
compartment_id: "ocid1.compartment.oc1..example"
subnet_id: "ocid1.subnet.oc1.region.example"
cluster_id: "ocid1.cluster.oc1.region.example"
region: "us-phoenix-1"

# Controller
replicaCount: 2

# Authentication
authType: instance  # or "user" if using credentials
authSecretName: oci-config

# Enable Kubernetes events for troubleshooting
emitEvents: true

# Pod resources (recommended)
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

# High availability
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - oci-native-ingress-controller
          topologyKey: kubernetes.io/hostname
```

## Configuration

### Required Values

| Parameter | Description | Example |
|-----------|-------------|---------|
| `compartment_id` | OCI Compartment ID for load balancer resources | `ocid1.compartment.oc1..xxx` |
| `subnet_id` | OCI Subnet ID where load balancers will be created | `ocid1.subnet.oc1.xxx` |
| `cluster_id` | OCI Cluster ID | `ocid1.cluster.oc1.xxx` |
| `region` | OCI region identifier | `us-phoenix-1` |

### Optional Values

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `1` | Number of controller replicas |
| `authType` | `instance` | Authentication method: `instance` or `user` |
| `authSecretName` | `oci-config` | Name of secret containing OCI credentials |
| `deploymentNamespace` | `native-ingress-controller-system` | Namespace for deployment |
| `image.tag` | `master` | Controller image tag |
| `emitEvents` | `false` | Emit Kubernetes events for debugging |
| `useLbCompartmentForCertificates` | `false` | Use IngressClassParameters compartment for certificates |
| `certDeletionGracePeriodInDays` | `0` | Grace period for unused certificate cleanup |

For all available options, see `values.yaml`.

## Usage

### Create an Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-app
spec:
  ingressClassName: oci.oraclecloud.com/native-ingress-controller
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: example-service
                port:
                  number: 80
```

### Configure IngressClassParameters

Customize load balancer settings per ingress:

```yaml
apiVersion: ingress.oraclecloud.com/v1beta1
kind: IngressClassParameters
metadata:
  name: custom-lb-params
spec:
  compartmentId: "ocid1.compartment.oc1..xxx"
  subnetId: "ocid1.subnet.oc1.xxx"
  loadBalancerName: "my-custom-lb"
  isPrivate: false
  minBandwidthMbps: 10
  maxBandwidthMbps: 100
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: oci-custom
spec:
  controller: oci.oraclecloud.com/native-ingress-controller
  parameters:
    apiGroup: ingress.oraclecloud.com
    kind: IngressClassParameters
    name: custom-lb-params
```

### Enable Pod Readiness Gates

Pods annotated with `podreadiness.ingress.oraclecloud.com/pod-readiness-gate-inject=enabled` will receive readiness gates, ensuring they're only marked ready after the load balancer is fully configured:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    podreadiness.ingress.oraclecloud.com/pod-readiness-gate-inject: enabled
```

## Monitoring

The controller exposes Prometheus metrics on port `2223` at `/metrics`:

```bash
kubectl port-forward -n native-ingress-controller-system \
  svc/oci-native-ingress-controller 2223:2223

# Then access: http://localhost:2223/metrics
```

Example metrics:
- `ingress_controller_reconcile_total` - Total reconciliations
- `ingress_controller_load_balancer_provisioning_duration_seconds` - Provisioning time
- `ingress_controller_errors_total` - Total errors

## Troubleshooting

### Check Controller Logs

```bash
kubectl logs -n native-ingress-controller-system \
  -l app.kubernetes.io/name=oci-native-ingress-controller \
  -f
```

### Verify Controller Status

```bash
kubectl get deployment -n native-ingress-controller-system
kubectl get pods -n native-ingress-controller-system
kubectl describe pod -n native-ingress-controller-system <pod-name>
```

### Check Ingress Status

```bash
kubectl describe ingress <ingress-name>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### Enable Debug Logging

Update the deployment to increase verbosity:

```bash
kubectl set env deployment/oci-native-ingress-controller \
  -n native-ingress-controller-system \
  -c oci-native-ingress-controller \
  LOGLEVEL=debug
```

### Common Issues

**Issue: Load balancer not being created**
- Verify OCI credentials are correctly configured
- Check that `compartment_id`, `subnet_id`, and `region` are correct
- Ensure the OCI service account has necessary permissions
- Check controller logs for detailed error messages

**Issue: Certificate not being provisioned**
- Verify `cert-manager` is installed and running
- Check cert-manager logs for certificate provisioning errors
- Verify DNS is correctly configured

**Issue: Pods not reaching ready state**
- Check pod readiness gate status
- Verify load balancer health checks are passing
- Review controller logs for webhook-related errors

## Uninstalling

```bash
helm uninstall oci-native-ingress-controller \
  -n native-ingress-controller-system
```

To also delete the namespace:

```bash
kubectl delete namespace native-ingress-controller-system
```
