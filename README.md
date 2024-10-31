# n8n Helm Chart Deployment in Kubernetes

This guide explains how to deploy [n8n](https://github.com/n8n-io/n8n), an open-source workflow automation tool, in a Kubernetes cluster using Helm and FluxCD. n8n allows you to automate tasks by integrating different services and applications with a visual interface. This guide also includes instructions for both standard installation via Helm as well as a more automated approach using FluxCD.

In the FluxCD deployment, the lifecycle of both the helm repository and the Helm release is managed entirely by FluxCD. This approach enables continuous synchronization between the Helm chart and the Kubernetes cluster, ensuring that your n8n deployment stays in sync with any updates.


## Standard Deployment of n8n Helm Chart

Below are the standard steps to deploy the n8n Helm chart directly using `helm` commands with your actual addresses.

### Step 1: Add Helm Repository

First, add the n8n Helm repository:

```bash
helm repo add n8n-repo https://dreemix.github.io/helm-charts-n8n
helm repo update
```
### Step 2: Install n8n Using Helm
Install the n8n chart, specifying the required values for your deployment:
```bash
helm install n8n n8n-repo/n8n \
  --namespace n8n-ns \
  --create-namespace \
  --set postgresql.enabled=true \
  --set postgresql.postgresqlUsername=n8n \
  --set postgresql.postgresqlPassword=N8nPassW0r$ \
  --set postgresql.postgresqlPostgresPassword=N8nPassW0r$ \
  --set postgresql.postgresqlDatabase=n8n \
  --set persistence.enabled=true \
  --set persistence.size=1Gi \
  --set storageClass=standard \
  --set image.repository=n8nio/n8n \
  --set image.tag=1.64.3 \
  --set env.DB_TYPE=postgresdb \
  --set env.DB_POSTGRESDB_USER=n8n \
  --set env.DB_POSTGRESDB_DATABASE=n8n \
  --set env.DB_POSTGRESDB_HOST=n8n-postgresql.n8n-ns.svc.cluster.local \
  --set env.DB_POSTGRESDB_PASSWORD=N8nPassW0r$ \
  --set env.WEBHOOK_URL=https://n8n.seosmart.ua/ \
  --set env.N8N_ENCRYPTION_KEY=p4b4U93Cjc/M01rUaunRXgSMrM4ggGby \
  --set service.type=ClusterIP \
  --set service.port=5678 \
  --set service.targetPort=5678
```

## Requirements

- Kubernetes 1.12+
- Helm 3+
- FluxCD installed on the cluster

## FluxCD HelmRepository and HelmRelease Configuration for n8n

We will leverage FluxCD's `HelmRepository` and `HelmRelease` custom resources to deploy n8n from a Helm chart.

### Step 1: Add Helm Repository via FluxCD

Use the following `HelmRepository` resource YAML in FluxCD to point to the n8n Helm chart repository. This resource will ensure FluxCD pulls updates from the specified Helm repository.

For example `helmrepository-n8n.yaml`:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: n8n-repo
  namespace: flux-system
spec:
  interval: 10m0s
  url: https://dreemix.github.io/helm-charts-n8n
```

### Step 2: Define HelmRelease for n8n

Below is the `HelmRelease` configuration that will instruct FluxCD to deploy the n8n service in the `n8n-ns` namespace. The release will automatically configure and deploy PostgreSQL as the database for n8n.

**Important:** Ensure to keep your passwords and secret keys safe and secure.
`helmrelease-n8n.yaml`
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: n8n
  namespace: n8n-ns
spec:
  interval: 10m0s
  chart:
    spec:
      chart: n8n
      version: 0.137.1
      sourceRef:
        kind: HelmRepository
        name: n8n-repo
        namespace: flux-system
  values:
    image:
      repository: n8nio/n8n
      tag: 1.64.3
    postgresql:
      enabled: true
      postgresqlUsername: n8n
      postgresqlPassword: "********"
      postgresqlPostgresPassword: "********"
      postgresqlDatabase: n8n
      persistence:
        enabled: true
        size: 1Gi
      image:
        registry: docker.io
        repository: bitnami/postgresql
        tag: 15
    env:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_HOST: n8n-postgresql.n8n-ns.svc.cluster.local
      DB_POSTGRESDB_PASSWORD: "********"
      WEBHOOK_URL: https://your-domain.com/
      N8N_ENCRYPTION_KEY: "********"
    persistence:
      enabled: true
      size: 1Gi
      storageClass: standard
    replicaCount: 1
    service:
      type: ClusterIP
      port: 5678
      targetPort: 5678
```

### Step 3: Set Up Ingress for n8n

To access n8n externally, you must set up an Ingress resource. Below is an example configuration for Ingress that allows access to n8n via a specific domain name and sets up TLS with Let's Encrypt.

Replace `your-domain.com` with your actual domain name.
`n8n-ingress.yaml`
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: n8n-ingress
  namespace: n8n-ns
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    nginx.ingress.kubernetes.io/client-max-body-size: "100m"
spec:
  ingressClassName: nginx
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: n8n
            port:
              number: 5678
  tls:
  - hosts:
    - your-domain.com
    secretName: n8n-tls
```

## Usage

Once the `HelmRepository`, `HelmRelease`, and `Ingress` resources are created in your Kubernetes cluster under `flux-system` and `n8n-ns` namespaces respectively, FluxCD will automatically manage the lifecycle and ensure that the application is deployed.

To monitor the deployment status, you can use the following commands:

```bash
# Check the status of the Helm release
kubectl get helmrelease -n n8n-ns

# Check the Ingress resource 
kubectl get ingress -n n8n-ns
```
After the successful synchronization, you should be able to access n8n at [https://your-domain.com](https://your-domain.com).

For more on n8n configurations and environment variables, refer to the [official n8n documentation](https://docs.n8n.io/reference/environment-variables.html).