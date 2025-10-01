# 📘 Substrate Environment Deployment Runbook

This runbook provides a step-by-step guide to setting up and deploying a
new environment in **Epic Games Substrate**. It links together all
moving parts: workstation setup, infrastructure provisioning,
application deployment, and day-2 operations.

------------------------------------------------------------------------

## 🔸 0. Substrate Environment --- Mental Model (How Things Link)

-   **Accounts & Access:** Okta → AWS SSO for console/CLI; Teleport for
    DB/EC2; GitHub over HTTPS for code.
-   **Two Lanes Meet in Cluster:** Infrastructure via Terraform (HCP
    workspace per service+env). Applications via CI/CD (build → push →
    Helm deploy).
-   **Pod-to-AWS Access:** IRSA (EKS OIDC provider + IAM role bound to a
    ServiceAccount) --- no static keys.
-   **Networking:** Ingress/ALB for north‑south traffic;
    service‑to‑service via internal DNS/networking conventions; prefer
    private links where available.
-   **Secrets & Config:** Vault for app secrets; SSM Parameter Store for
    OTEL client secret; Kubernetes Secrets for non‑credential config
    where appropriate.

> See your repo reference docs inside `epic-games/` for Codefresh
> deploys, deployment strategies, Terraform patterns, networking
> guidance, and access guides.

------------------------------------------------------------------------

## 🔹 1A. Workstation Setup --- Deep Dive & Verification

### A. VPN & DNS

-   Install GlobalProtect (macOS/Windows) or OpenConnect (Linux). Ensure
    **split‑tunnel** is enabled so Epic routes/DNS are injected.
-   macOS: allow GP system extensions; Linux: install `estd` for
    split‑DNS/routing. Disable DoH in browser.

**Verify**

``` bash
scutil --dns | grep -A2 "Resolver #"
ip route
nslookup <internal-hostname>
```

### B. AWS SSO & CLI

-   From Okta, choose target account/role → open console **and** obtain
    CLI creds.
-   Use one helper per shell session (don't mix `aop` and
    `aws-sso/k8s-sso`).

**Verify**

``` bash
aws sts get-caller-identity
aws ec2 describe-vpcs --region <REGION>
```

### C. GitHub over HTTPS

``` bash
git clone https://github.com/Majormekzy/epic-games.git
```

### D. Teleport (DB/EC2)

``` bash
tsh login --proxy <substrate-proxy>
tsh db ls && tsh ls
tsh db connect <dbname>
```

------------------------------------------------------------------------

## 🔹 2A. Environment Request & Naming

-   Ensure the service exists in Backstage (capture EUID) and the
    environment name follows standards (keep ALB names ≤31 chars).
-   Decide resources up‑front: RDS, Redis, SQS, certificates, SGs.

------------------------------------------------------------------------

## 🔹 3A. Terraform Implementation --- Practical Steps

-   Create HCP workspace per service+env.
-   Link VCS repo, set WD, tfvars, execution agent.
-   Seed env tfvars and tfbackend.sh scripts.
-   Plan & apply via HCP or locally.

------------------------------------------------------------------------

## 🔹 4A. Kubernetes Access to AWS (IRSA)

-   Create IAM policy/role, trust with OIDC provider.

-   Annotate ServiceAccount.

-   Verify with `aws sts get-caller-identity` inside pod.

------------------------------------------------------------------------

## 🔹 5A. Application Delivery

-   Build multi-arch images.

-   CI/CD: build → test → push → Helm upgrade.

-   Configure ingress, resources, probes.

------------------------------------------------------------------------

## 🔹 6A. Networking Model

-   Ingress via ALB/NLB.

-   East-west traffic via DNS/NetworkPolicies.

-   Keep ALB names short.

------------------------------------------------------------------------

## 🔹 7A. Observability

-   Add OTEL client secret to SSM.

-   Confirm metrics in Grafana/Loki.

------------------------------------------------------------------------

## 🔹 8A. End-to-End Example

1.  VPN on, AWS SSO role assumed.

2.  Create workspace `svc-lt`.

3.  Add `env/lt.tfvars`, `scripts/lt.tfbackend.sh`.

4.  Apply infra.

5.  Secrets in Vault, DB users validated.

6.  IRSA role bound.

7.  Deploy app via CI/CD.

8.  Add OTEL secret.

9.  Smoke tests.

------------------------------------------------------------------------

## 🔹 9A. Troubleshooting

-   DNS: check split-tunnel.

-   HCP ELB error: shorten ALB name.

-   Pods fail AWS: check IRSA annotation.

-   Image fails on ARM: rebuild multi-arch.

-   Ingress 502: adjust timeouts.

------------------------------------------------------------------------

## 🔹 Appendix --- Handy Commands

``` bash
kubectl -n <ns> get deploy,po,svc,ingress
kubectl -n <ns> describe ingress <name>
kubectl -n <ns> logs deploy/<svc> --tail=200
kubectl -n <ns> get events --sort-by=.lastTimestamp | tail -n 50
```
