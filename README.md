# GitOps Microservices Deployment — Implementation Guide

Deploying the Online Boutique microservices with EKS, ArgoCD, Gateway API, and full observability.

---

## 0. Prerequisites (Local Machine)
- AWS CLI installed + configured (`aws configure`) with IAM access/secret keys
- Terraform installed
- Repo cloned:
  ```bash
  git clone https://github.com/sny17/Microservice-Gitops.git
  cd Microservice-Gitops/terraform/
  ```

---

## 1. Provision Infrastructure (Terraform)
```bash
terraform init
terraform plan
terraform apply
```
- Outputs the bastion host public IP and saves a private key locally.

**Optional — Remote S3 backend:**
```bash
aws s3api create-bucket --bucket <bucket-name> --region us-east-1
aws s3api put-bucket-versioning --bucket <bucket-name> --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket <bucket-name> --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```
Add backend block to `terraform.tf`, then re-run `terraform init` and confirm state migration.

---

## 2. Configure Bastion Host
```bash
ssh -i bastion-key.pem ubuntu@<publicIP>
```
Install: AWS CLI, `kubectl`, Helm, `eksctl` → then:
```bash
aws configure
aws eks update-kubeconfig --region <region> --name <cluster-name>
kubectl get nodes
```

---

## 3. Install AWS Load Balancer Controller
```bash
eksctl utils associate-iam-oidc-provider --region <region> --cluster <cluster-name> --approve

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=<cluster-name> --namespace=kube-system --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts --region <region> --approve

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=<cluster-name> --set region=<region> \
  --set vpcId=<vpc-id> --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set controllerConfig.featureGates.NLBGatewayAPI=true \
  --set controllerConfig.featureGates.ALBGatewayAPI=true --version 3.0.0

kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 4. Set Up Gateway API
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/config/crd/gateway/gateway-crds.yaml
```
Apply (from repo's `gateway-api-manifests/`):
```bash
kubectl apply -f gateway-class.yaml
kubectl apply -f alb-config.yaml   # set your ACM certificate ARN
kubectl apply -f gateway.yaml      # set your domain hostname
kubectl get gateway
```
> `TargetGroupConfiguration` is **only required** for AWS ALB/NLB Gateway API setups (target type `ip`). Not needed for kgateway, Istio, NGINX, or Kong.

---

## 5. Deploy External DNS
```bash
aws iam create-policy --policy-name "AllowExternalDNSUpdates" --policy-document file://policy.json
export POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName==`AllowExternalDNSUpdates`].Arn' --output text)
export EKS_CLUSTER_NAME=<cluster-name>

kubectl create ns external-dns
eksctl create podidentityassociation \
  --cluster $EKS_CLUSTER_NAME --namespace external-dns \
  --service-account-name external-dns --role-name external-dns-pod-identity-role \
  --permission-policy-arns $POLICY_ARN

helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns -n external-dns --version 1.20.0
```
Edit values (`sources: service, ingress, gateway-httproute, gateway-tlsroute, gateway-tcproute, gateway-udproute`), then:
```bash
helm upgrade -i external-dns external-dns/external-dns -f external-dns-values-1.20.0.yaml -n external-dns --version 1.20.0
```

---

## 6. Deploy ArgoCD
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm show values argo/argo-cd --version 9.4.0 > argocd-values-9.4.0.yaml
```
Edit values file:
- `params.server.insecure: true` (TLS terminated at LB)
- `configs.cm.kustomize.buildOptions: "--enable-helm"`
- `httproute.enabled: true`, parentRefs → your gateway, hostnames → `argocd.<yourdomain>`

```bash
helm install argo-cd argo/argo-cd -n argocd -f argocd-values-9.4.0.yaml --version 9.4.0 --create-namespace
kubectl apply -f target-grp-config.yaml
```
Get admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
Login at `https://argocd.<yourdomain>` (user: `admin`) → change password.

---

## 7. Set Up CI (GitHub Actions)
1. Build/push each service's Docker image to GHCR (or use the pre-built ones from the repo's Packages).
2. Link packages to repo → **Package Settings → Add repository → grant Write access**.
3. Add workflow files (already in repo `.github/workflows/`):
   - `microservice-ci.yaml` — reusable workflow: checkout → build → Trivy scan → push to GHCR
   - `ci-trigger.yaml` — detects changed services under `src/**` on push to `main`, triggers matrix build
4. Push a change under `src/` → verify Action runs and image is pushed.

> Tip: Set Trivy `exit-code: 1` in production to fail builds on HIGH/CRITICAL vulnerabilities.

---

## 8. Set Up CD (ArgoCD Application)
Repo layout for GitOps:
- `helm-chart/` — application Helm chart
- `microservices-extra-kube-manifests/` — HTTPRoute + TargetGroupConfiguration (env-specific, kept separate from Helm)

Root `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - microservices-extra-kube-manifests/HTTProute.yaml
  - microservices-extra-kube-manifests/target-grp.yaml
helmCharts:
  - name: boutique-app
    repo: oci://ghcr.io/<you>/onlineboutique
    version: 0.10.4
    releaseName: boutique-app
    namespace: boutique-app
    valuesFile: helm-chart/values.yaml
```

Create ArgoCD Application (`argocd/argocd-apps/boutique-app.yaml`) pointing to repo root, with `syncPolicy.automated: {prune: true, selfHeal: true}` and `CreateNamespace=true`.

```bash
kubectl apply -f boutique-app.yaml
```
Check ArgoCD UI → app should show **Synced**.

---

## 9. Connect CI → CD (Image Auto-Updates)
```bash
helm repo add argo https://argoproj.github.io/argo-helm
```
If repo is **private**, create a GHCR pull secret:
```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io --docker-username=<user> --docker-password=<PAT> -n argocd
```
Install Image Updater:
```bash
helm install argocd-image-updater argo/argocd-image-updater -n argocd --version 1.0.5
```
Apply an `ImageUpdater` CR listing all service images with `updateStrategy: newest-build` and an `allowTags` regex matching your `sha-` tags (see repo `image-updater.yaml` for full example covering all 11 services).

```bash
kubectl apply -f image-updater.yaml
kubectl get imageupdater -n argocd
```
Trigger a CI build → confirm ArgoCD auto-updates the running image.

Access the app: `https://app.<yourdomain>`

---

## 10. Observability — Monitoring (Prometheus/Grafana/Alertmanager)
> Managed **outside** ArgoCD (kept separate from app GitOps for security).

**Slack alerts:**
1. Create a Slack workspace + `#alertmanager` channel.
2. Create a Slack App → enable Incoming Webhooks → add webhook to the channel → copy URL.
3. Store as a K8s secret (never hardcode it):
```bash
kubectl create ns monitoring
kubectl create secret generic alertmanager-slack-webhook \
  --from-literal=slack-webhook-url="<webhook-url>" -n monitoring
```

**Install stack:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm show values prometheus-community/kube-prometheus-stack --version 81.6.3 > kube-prom-stack-81.6.3.yaml
```
Edit values:
- `alertmanager.alertmanagerSpec.secrets: [alertmanager-slack-webhook]`
- Configure `alertmanager.config` routes/receivers to post to Slack via the mounted secret file, filtering on `severity: critical`

```bash
helm upgrade -i kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version 81.6.3 -f kube-prom-stack-81.6.3.yaml -n monitoring
```

**Expose Grafana & Prometheus** (write HTTPRoute + TargetGroupConfiguration manifests per service, referencing `kube-prometheus-stack-grafana` and `kube-prometheus-stack-prometheus` services):
```bash
kubectl apply -f HTTProute-grafana.yaml -f target-grp-grafana.yaml
kubectl apply -f HTTProute-prometheus.yaml -f target-grp-prometheus.yaml
```
Grafana password:
```bash
kubectl --namespace monitoring get secrets kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```
Access: `grafana.<yourdomain>`, `prometheus.<yourdomain>`

---

## 11. Observability — Logging (ELK Stack)

**EBS CSI Driver (required for Elasticsearch PVs):**
```bash
eksctl create iamserviceaccount \
  --cluster <cluster-name> --namespace kube-system --name ebs-csi-controller-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --override-existing-serviceaccounts --approve

eksctl create addon --cluster <cluster-name> --name aws-ebs-csi-driver --version latest \
  --service-account-role-arn <role-arn-from-cfn-output> --force
```

**ECK Operator + Elasticsearch:**
```bash
kubectl create ns logging
helm repo add elastic https://helm.elastic.co
helm install eck-operator elastic/eck-operator --version 3.3.0 -n logging

kubectl apply -f storageclass.yaml   # sets ebs-aws as default StorageClass
helm install eck-elasticsearch elastic/eck-elasticsearch --version 0.18.0 -n logging
```

**Filebeat (log shipping):**
```bash
helm show values elastic/eck-beats --version 0.18.0 > eck-beats-0.18.0.yaml
```
Edit: set `elasticsearchRef.name: eck-elasticsearch`, configure `daemonSet` volume mounts for container logs, and `autodiscover` provider for Kubernetes hints (see repo file for full config).
```bash
helm upgrade -i eck-beats elastic/eck-beats --version 0.18.0 -f eck-beats-0.18.0.yaml -n logging
kubectl get beats -n logging
```

**Kibana:**
```bash
helm show values elastic/eck-kibana --version 0.18.0 > eck-kibana-0.18.0.yaml
# edit: elasticsearchRef.name: eck-elasticsearch
helm install eck-kibana elastic/eck-kibana --version 0.18.0 -f eck-kibana-0.18.0.yaml -n logging
kubectl get kibana -n logging
```

**Expose Kibana:**
```bash
kubectl apply -f HTTProute-kibana.yaml
kubectl apply -f target-grp-kibana.yaml   # include HTTPS health check on /api/status
```
Password:
```bash
kubectl get secret eck-elasticsearch-es-elastic-user -n logging -o go-template='{{.data.elastic | base64decode}}'
```
Access `kibana.<yourdomain>` → Discover → filter `kubernetes.namespace: boutique-app` → view/filter logs by pod, app label, etc.

---

## 12. Scaling — HPA (Horizontal Pod Autoscaler)
```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server --version 3.13.0 -n kube-system
kubectl top nodes   # verify metrics working
```
Confirm CPU requests are set on deployments (`kubectl get deploy frontend -n boutique-app -o yaml | grep -A10 resources`).

Create HPA (example: frontend):
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: boutique-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 1
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50   # lower to 5 only for quick testing
```
```bash
kubectl apply -f frontend-hpa.yaml
kubectl get hpa -n boutique-app -w
```
> **Important:** Since ArgoCD manages the cluster (GitOps), never `kubectl edit` deployments/HPA directly — changes will be reverted. Update values in Git, commit, push, and let ArgoCD sync.

Repeat HPA setup for `cartservice`, `checkoutservice`, `recommendationservice`.

---

## 13. Cleanup
```bash
# Delete load balancer + its security groups in AWS Console first, then:
terraform destroy -auto-approve
```

---

## Architecture Summary
| Layer | Tooling |
|---|---|
| Infra | Terraform, EKS |
| Networking | Gateway API + AWS ALB Controller, External DNS |
| CI | GitHub Actions + Trivy + GHCR |
| CD | ArgoCD + ArgoCD Image Updater + Kustomize |
| Monitoring | kube-prometheus-stack + Alertmanager (Slack) |
| Logging | ECK (Elasticsearch, Filebeat, Kibana) |
| Scaling | HPA + Metrics Server |

Only `cartservice` uses persistent storage (Redis). All other services are stateless by design.
