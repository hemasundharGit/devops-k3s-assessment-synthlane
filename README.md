# Infrastructure DevOps Intern Assessment – Hetzner

## Overview
This repository contains the complete infrastructure setup, Kubernetes deployment, application rollout, debugging process, and production ownership considerations for running Open WebUI on a Hetzner Cloud VM.

---

## Environment
- Cloud Provider: Hetzner Cloud
- OS: Ubuntu 24.04
- Kubernetes: k3s (single-node)
- Container Runtime: containerd
- Helm: v3
- Application: Open WebUI

---

## VM, Docker & Kubernetes Setup

The virtual machine was prepared by installing Docker and setting up a single-node k3s Kubernetes cluster. This provides a lightweight, cost-efficient, and production-like environment suitable for an early-stage startup.

```bash
apt update && apt upgrade -y

apt install -y ca-certificates curl gnupg lsb-release

mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
tee /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

systemctl enable docker
systemctl start docker

curl -sfL https://get.k3s.io | sh -
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

Ownership & Production Readiness
Production Risks

Single-node failure

No automated backups

No monitoring or alerting

Insecure secret management

No autoscaling

Failure Scenario (10x Traffic Spike)

First failure: CPU and memory exhaustion

Recovery: Restart node and scale resources

Improvement: Move to multi-node cluster with autoscaling

Security & Secrets

Secrets managed via Kubernetes Secrets or Vault

Secrets must never be committed to Git

Rotate SSH keys, API tokens, and OIDC credentials

Backups & Recovery

Backup persistent volumes and configuration

Daily backups

Regular recovery testing

Cost Ownership (Hetzner)

Keep infrastructure minimal early

Avoid over-provisioning

Move away from k3s when high availability is required

Validation Output
kubectl get nodes
kubectl get all -n openwebui

