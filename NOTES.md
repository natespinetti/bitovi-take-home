# Build and test

docker build -t order-processor:dev ./service
docker run --rm -p 8080:8080 order-processor:dev

# Validate chart

helm lint ./charts
helm template production ./charts --namespace order-processor

# Create local cluster and load image

kind create cluster --name bitovi
kind load docker-image order-processor:dev --name bitovi

# Install platform prerequisites

helm upgrade --install argocd argo/argo-cd \
 -n argocd --create-namespace --wait

helm upgrade --install external-secrets \
 external-secrets/external-secrets \
 -n external-secrets --create-namespace \
 --set installCRDs=true --wait

helm upgrade --install monitoring \
 prometheus-community/kube-prometheus-stack \
 -n monitoring --create-namespace --wait \
 --timeout 10m

# Bootstrap GitOps

kubectl apply -f argocd/application.yaml

# Verify

kubectl get application order-processor -n argocd
kubectl get pods -n order-processor
kubectl get externalsecret -n order-processor
kubectl get servicemonitor -n order-processor
