## Application Deployment Using Helm

Open WebUI was deployed into the k3s cluster using Helm.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add open-webui https://helm.openwebui.com/
helm repo update

kubectl create namespace openwebui

helm install webui open-webui/open-webui \
  --namespace openwebui \
  --set service.type=ClusterIP \
  --dry-run

helm install webui open-webui/open-webui \
  --namespace openwebui \
  --set service.type=ClusterIP
Validation
kubectl get all -n openwebui

Authentication Configuration (OIDC)

OIDC authentication was enabled using Helm values.

oidc:
  clientId: "test"
  clientSecret: ""
  issuer: "https://<DOMAIN>/auth/realms/hyperplane/.well-known/openid-configuration"
  scopes:
    - openid
    - profile
    - email


Applied using:

helm upgrade webui open-webui/open-webui \
  --namespace openwebui \
  --values values-oidc.yaml

Debugging (Intentional Failure)

After enabling OIDC, the Open WebUI pod entered a CrashLoopBackOff state.

kubectl get pods -n openwebui
kubectl describe pod <pod-name> -n openwebui
kubectl logs <pod-name> -n openwebui

Root Cause

The OIDC provider uses a self-signed TLS certificate

TLS verification failed during application startup

OIDC client secret was not configured

Fix Applied
extraEnv:
  - name: NODE_TLS_REJECT_UNAUTHORIZED
    value: "0"


This allowed the application to start successfully. In production, the correct solution would be to trust a custom CA instead of disabling TLS verification.
Validation Output
kubectl get nodes
kubectl get all -n openwebui
