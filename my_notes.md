Docker:
Dockerfile switched to multi stage. Builder compiles Go binary, and next stage runs it in Alpine as non-root user. Final image doesn't include Go just the compiled binary.
Possible changes:

- Enable CGO if needed
- Alpine is a broad selection. Certificates and user need manual setup. Distroless could be smaller and easier to configure depending on use case.

Kubernetes / Helm:
values.yml:

- stabilize tag used and use image already presnet on node
- create service reachable inside cluster so workers health and metrics endpoints accessible
- remove hardcoded password and expect a Kubernetes secret
- define resources for pod to use
- adjust grace period so Go can drain ~20 seconds for clean termination

deployment.yaml:

- adjust instance names to support multiple releases, Pods stay connected to their instance
- add update strategy so current pod isn't taken down before the replacement is ready
- disables Kubernetes api since no extra functionality or monitoring needed
- enforce non-root user and block unnecessary OS calls
- port 8080 does not need adjusted priviledges, write to root system, linux capabilities
- liveness probe: If process broken and in need of restart, wait 5 seconds after startup, call /healthz every 10 seconds, wait maximum 2 seconds for responses, restarts the container after 3 failures
- readiness probe: Asks if pod should receive traffic. During warmup, healthz will say 200, readyz will be 503. After warmup both 200. Kubernetes then sends traffic. During shutdown, readiness becomes false allowing app to drain with no new traffic. No startup probe needed because service live immediately, predictable startup time, liveness checked separately

service.yaml:

- labels will be used by Prometheus ServiceMonitor
- selector determines which pod will receive traffic
- service port defined for Prometheus Operator

GitOps:
argocd application.yaml:

- default project settings could be altered if more restriction needed, read only, deploy only to specific cluster, deploy only to order-processor namespace.
- sync enabled so no manual sync needed
- prune removed files from Kubernetes, can be destructive so likely make this false in production and rely on Git reviews.
- self heal overrides manual changes outside of the git repo

In a real platform, Helm wouldn't own a cluster wide secret store.
Platform repo owns:

- External Secrets Operator
- ClusterSecretStore
- IAM workload identity

Application repo owns:

- ExternalSecret
- Deployment
- Service

Production ClusterSecretStore intentionally contains no static AWS
credentials. External Secrets Operator is expected to receive a
least-privilege IAM role.

Service Monitor / Prometheus:

- Service Monitor bridging between Kubernetes Service and Prometheus

Prometheus

- selects ServiceMonitor by metadata labels
- ServiceMonitor selects Service by spec.selector
- Service selects Pods
- Prometheus requests /metrics from each selected endpoint

- Prometheus selects ServiceMonitor resources, and each ServiceMonitor selects Kubernetes Services.
- Service and Service Monitor share a namespace so no need to define a selector
- Authentication not needed because metrics are available in the cluster
- No labels need to be rewritten in Prometheus as we aren't mutating results
