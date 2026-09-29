# Order Processor

Production-ready container and Kubernetes deployment for the provided Go worker service.

## Prerequisites

Install and start:

- Docker Desktop
- Git
- `kubectl`
- Helm
- kind

On macOS:

```bash
brew install git kubectl helm kind
```

## 1. Clone the repository

```bash
git clone https://github.com/natespinetti/bitovi-take-home.git
cd bitovi-take-home
```

## 2. Build and test the container

```bash
docker build \
  --tag order-processor:dev \
  ./service

docker run \
  --rm \
  --publish 8080:8080 \
  order-processor:dev
```

Test it from another terminal:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:8080/metrics
```

`/readyz` may return an error during the first five seconds.

## 3. Validate the Helm chart

```bash
helm lint ./charts

helm template production ./charts \
  --namespace order-processor
```

## 4. Create the Kubernetes cluster

```bash
kind create cluster \
  --name bitovi

kind load docker-image \
  order-processor:dev \
  --name bitovi
```

## 5. Add the Helm repositories

```bash
helm repo add argo \
  https://argoproj.github.io/argo-helm

helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

## 6. Install the platform components

### Argo CD

```bash
helm upgrade --install argocd \
  argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --wait
```

### External Secrets Operator (optional AWS integration)

Skip this component for local review. Install it only when enabling
`externalSecrets.enabled`.

```bash
helm repo add external-secrets \
  https://charts.external-secrets.io

helm repo update

helm upgrade --install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true \
  --wait
```

### Prometheus Operator

```bash
helm upgrade --install monitoring \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --wait \
  --timeout 10m
```

## 7. Configure the database secret

The chart defaults to `externalSecrets.enabled: false`, so local review requires
no AWS credentials or External Secrets Operator. Create the Kubernetes secret
referenced by `db.existingSecret` before deploying:

```bash
kubectl create namespace order-processor

kubectl create secret generic order-processor-db \
  --namespace order-processor \
  --from-literal=password=local-review-only
```

This placeholder is only for the supplied worker, which simulates processing
and does not connect to Postgres. Real database credentials should be provisioned
separately and kept out of Git.

For AWS integration, set `externalSecrets.enabled: true` in the values used by
Argo CD (or pass `--set externalSecrets.enabled=true` when deploying with Helm),
install the optional operator above, and omit the local secret creation. The
chart then expects AWS Secrets Manager to contain:

```text
Region:        us-east-2
Secret name:   Bitovi-Order-Processor-PW
JSON property: order-processor-db
```

Example value:

```json
{
  "order-processor-db": "example-password"
}
```

AWS credentials are intentionally not included in this repository. The cluster must provide AWS access to External Secrets Operator using workload identity, such as EKS Pod Identity or IRSA.

## 8. Deploy with Argo CD

```bash
kubectl apply \
  --filename argocd/application.yaml
```

Argo CD then deploys and manages the application from Git. Do not manually apply the manifests inside `charts/`.

## 9. Verify the deployment

```bash
kubectl get application order-processor \
  --namespace argocd

kubectl get pods \
  --namespace order-processor

kubectl get servicemonitor \
  --namespace order-processor
```

Expected status:

```text
Argo CD:          Synced / Healthy
Application pod:  1/1 Running
```

When AWS integration is enabled, also check `kubectl get externalsecret
--namespace order-processor` for `SecretSynced` / `Ready`.

## 10. Test the deployed service

```bash
kubectl port-forward \
  service/production-order-processor \
  18080:80 \
  --namespace order-processor
```

Test from another terminal:

```bash
curl http://localhost:18080/healthz
curl http://localhost:18080/readyz
curl http://localhost:18080/metrics
```

## 11. Verify Prometheus

```bash
kubectl port-forward \
  service/monitoring-kube-prometheus-prometheus \
  9090:9090 \
  --namespace monitoring
```

Open:

```text
http://localhost:9090
```

Query:

```promql
up{namespace="order-processor"}
```

A value of `1` confirms that Prometheus is scraping the service.

## Cleanup

```bash
kind delete cluster \
  --name bitovi
```

## Notes

- Application resources are managed by Argo CD.
- Local review uses a manually created placeholder Kubernetes secret; AWS
  deployments load secrets through External Secrets Operator.
- AWS credentials are never stored in Git.
- The submitted `ClusterSecretStore` expects workload identity.
- The local image uses `IfNotPresent` because it is loaded directly into kind.
- See `DESIGN.md` for production tradeoffs and proposed future improvements.
