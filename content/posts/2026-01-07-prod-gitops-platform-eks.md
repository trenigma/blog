---
title: "Building a Production-Ready GitOps Platform on AWS EKS"
date: 2026-01-07
draft: false
tags: ["aws", "eks", "kubernetes", "argocd", "gitops", "portfolio"]
description: "I needed to prove I could work with Kubernetes on AWS. So I built a production-ready EKS cluster with ArgoCD, RDS, multi-AZ networking, and 8 debugging issues that taught me the most."
hero_image: "/images/hero/k8s-box.jpg"
hero_alt: "Kubernetes architecture diagram"
---

The [ECS project](https://blog.trenigma.dev/posts/2025-10-27-aws-project-reboot-career/) was done. But every job I was looking at wanted Kubernetes, and I hadn't touched EKS in a while. So I needed to go build a thing.

Full GitOps on AWS EKS: ArgoCD watching the repo, RDS for persistence, multi-AZ networking. Here's how it went, including 8 debugging issues that each cost me a different amount of sanity.

## What I Built

A production-grade Kubernetes platform on AWS with:

- **Amazon EKS cluster** with managed node groups across multiple availability zones
- **GitOps workflow** using ArgoCD for declarative, Git-driven deployments
- **RDS PostgreSQL** for actual data persistence (no sqlite this time!)
- **Helm charts** for packaging and deploying the application
- **Multi-AZ VPC** with public/private subnet architecture
- **Infrastructure as Code** using Terraform modules
- **LoadBalancer service** for external access

The same Flask TODO API from my ECS project, but now running on Kubernetes with a proper GitOps deployment model.

[**Repository**](https://github.com/treehousepnw/eks-todo-gitops)

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│              GitHub Repository                   │
│  (Single source of truth for all config)        │
└─────────────┬────────────────────────────────────┘
              │
              ▼
┌────────────────EKS Cluster──────────────────────┐
│                                                  │
│  Control Plane (Managed by AWS)                 │
│  ┌────────────────────────────────┐            │
│  │ • Kubernetes API Server         │            │
│  │ • etcd                          │            │
│  │ • Scheduler                     │            │
│  └────────────────────────────────┘            │
│                                                  │
│  Worker Nodes (EC2 in Private Subnets)         │
│  ┌────────────────────────────────┐            │
│  │ ArgoCD:                         │            │
│  │ • Watches GitHub repo           │            │
│  │ • Auto-syncs every 30 seconds   │            │
│  │ • Applies changes automatically │            │
│  │                                 │            │
│  │ Application Workloads:          │            │
│  │ • TODO API (2 pods)             │            │
│  │ • LoadBalancer service          │            │
│  │ • ConfigMaps & Secrets          │            │
│  └────────────────────────────────┘            │
└──────────────────────────────────────────────────┘
          │
          ▼
    AWS Services
    • RDS PostgreSQL (Multi-AZ)
    • Classic Load Balancer
    • ECR (Container Registry)
```

## Key Design Decisions

### Why EKS Over Self-Managed Kubernetes?

I chose managed EKS for the control plane because:

- AWS handles Kubernetes upgrades and patches
- Built-in high availability for the control plane
- Better security posture with AWS integration
- This is what most companies actually use in production

### Why Managed Node Groups?

Rather than Fargate or self-managed EC2:

- More cost-effective than Fargate for always-on workloads
- Simpler than managing EC2 instances directly
- AWS handles node updates and security patches
- Can still SSH to nodes for debugging if needed

### Why ArgoCD Over Manual Deployment?

GitOps was the main learning goal here:

- **Single source of truth:** All config lives in Git
- **Automated deployments:** Push to main, ArgoCD handles the rest
- **Easy rollbacks:** Just revert the Git commit
- **Audit trail:** Git history shows who changed what and when
- **Declarative:** Describe desired state, ArgoCD makes it happen

This is a huge step up from my ECS project where I was manually running `terraform apply` and managing deployments by hand.

## The Build Process

### Phase 1: Infrastructure with Terraform

Starting with the foundation, I created Terraform modules for:

**VPC Module:**

- 3 public subnets across us-west-2a, us-west-2b, us-west-2c
- 3 private subnets for worker nodes
- NAT Gateways for private subnet internet access
- Proper tagging for EKS (kubernetes.io/cluster/[name] = shared)

**EKS Module:**

- Managed control plane with public endpoint access
- Managed node group with t3.medium instances (2 min, 5 max)
- IAM roles for cluster and node groups
- Security groups allowing node-to-node and control-plane-to-node communication

**RDS Module:**

- PostgreSQL 15 instance (db.t3.micro for cost savings)
- Multi-AZ deployment for production resilience
- Automated backups with 7-day retention
- Security group allowing traffic from EKS nodes

One lesson from my ECS project: I saved ~$70/month in the dev environment by using a single NAT Gateway instead of three. For production, you'd want three for high availability, but for a portfolio project, one is fine.

### Phase 2: Helm Chart Creation

I created a Helm chart to package the TODO API with proper Kubernetes manifests:

```
helm/todo-api/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── secret.yaml
```

The Helm chart includes:

- **Deployment:** 2 replicas with health checks and resource limits
- **Service:** LoadBalancer type for external access
- **Secret:** Database credentials from Kubernetes secrets
- **ConfigMap:** Database connection configuration

### Phase 3: ArgoCD Setup

Installing ArgoCD was straightforward:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Then I created an ArgoCD Application pointing to my GitHub repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/treehousepnw/eks-todo-gitops
    targetRevision: main
    path: helm/todo-api
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: apps
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

This is the magic of GitOps - ArgoCD watches the GitHub repo and automatically applies any changes I push.

## The 8 Debugging Issues (And How I Solved Them)

This is where the real learning happened. Here are the issues I hit and how I solved them:

### Issue #1: Port Conflict with `kubectl port-forward`

Running `kubectl port-forward svc/argocd-server 8080:443` threw "address already in use." I had a port-forward open in another terminal I'd forgotten about.

```bash
# Find what's using port 8080
lsof -i :8080

# Kill the old process
kill -9 <PID>

# Or just use a different port
kubectl port-forward svc/argocd-server 9090:443 -n argocd
```

Clean up your port-forwards when you're done, or just use different port numbers for each service.

---

### Issue #2: Docker Image Not Updating

Made changes, rebuilt the image, pods were still on the old version. The `:latest` tag was the problem. Kubernetes cached it because the tag didn't change.

```bash
# Use unique tags with version numbers
docker build -t <account>.dkr.ecr.us-west-2.amazonaws.com/todo-api:v1.0.1 app/
docker push <account>.dkr.ecr.us-west-2.amazonaws.com/todo-api:v1.0.1

# Update values.yaml with the new tag
image:
  tag: "v1.0.1"

# Commit and push - ArgoCD handles the rest!
git add helm/todo-api/values.yaml
git commit -m "Update to v1.0.1"
git push
```

Don't use `:latest` in production. Tag your images.

---

### Issue #3: Pods Couldn't Reach RDS Database

Pods crashing with "could not connect to server." The RDS security group wasn't allowing traffic from the EKS nodes.

```hcl
# In terraform/modules/rds/main.tf
resource "aws_security_group_rule" "rds_from_eks" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.rds.id
  source_security_group_id = var.eks_node_security_group_id
}
```

Draw a mental map of source → destination before you apply. Security groups will get you every time.

---

### Issue #4: Missing Environment Variables in Pods

Application logs showed the database connection variables weren't set. The Helm chart deployment template was missing the `envFrom` section to load the secret.

```yaml
# helm/todo-api/templates/deployment.yaml
spec:
  containers:
  - name: todo-api
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    envFrom:
    - secretRef:
        name: db-credentials
    env:
    - name: DB_HOST
      value: "{{ .Values.database.host }}"
    - name: DB_PORT
      value: "{{ .Values.database.port }}"
    - name: DB_NAME
      value: "{{ .Values.database.name }}"
```

Kubernetes won't inject secrets on its own. You have to reference them explicitly in the deployment spec.

---

### Issue #5: THE INFAMOUS PASSWORD SPECIAL CHARACTERS BUG 🐛

Still failing after fixing security groups and env vars. "password authentication failed." I'd been staring at this for an hour.

Turned out the password had `{`, `}`, and `$` in it. Those curly braces were being interpreted as template variables. Two hours I will never get back.

```bash
# This is what I saw in the logs:
FATAL:  password authentication failed for user "todoadmin"

# The actual password was: MyP@ss{w0rd}$123
# But it was being interpreted as a template variable!
```

Fix:

```yaml
# Create the secret properly in Kubernetes
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  DB_USER: todoadmin
  DB_PASSWORD: "MyP@ss{w0rd}$123"  # Quoted properly!
  DB_NAME: tododb
```

Then reference individual keys instead of building connection strings:

```python
# app.py
import os

db_user = os.getenv('DB_USER')
db_password = os.getenv('DB_PASSWORD')
db_host = os.getenv('DB_HOST')
db_name = os.getenv('DB_NAME')

# Build connection string safely
db_url = f"postgresql://{db_user}:{db_password}@{db_host}:5432/{db_name}"
```

Use simple passwords in dev, or quote them properly. Better yet, use Secrets Manager (that's in the "What's Next" section).

When you see "authentication failed" and everything else looks right, check for special characters getting mangled through multiple layers of templating.

---

### Issue #6: ArgoCD Not Syncing Changes

Pushed changes, ArgoCD wasn't picking them up. Auto-sync was on. The files were staged but not pushed. Did a `git add`, forgot the `git commit` and `git push`.

```bash
# Always verify your changes are actually in GitHub
git status
git log --oneline -3

# Check remote repo
curl https://raw.githubusercontent.com/treehousepnw/eks-todo-gitops/main/helm/todo-api/values.yaml | grep "tag"

# Force ArgoCD to sync if needed
kubectl patch application todo-api -n argocd --type merge -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {"revision": "HEAD"}}}'
```

Always verify changes are actually in the remote before wondering why ArgoCD isn't syncing.

---

### Issue #7: Health Check Failing After Deployment

Pods running but `/health` returning 503. The database connection pool wasn't initializing on startup.

```python
# app.py - add proper initialization
from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
db = SQLAlchemy(app)

# Add a health check that actually verifies DB connection
@app.route('/health')
def health():
    try:
        # Try to execute a simple query
        db.session.execute('SELECT 1')
        return jsonify({
            'status': 'healthy',
            'database': 'connected'
        }), 200
    except Exception as e:
        return jsonify({
            'status': 'unhealthy',
            'database': str(e)
        }), 503
```

Health checks should actually check something. Returning a static 200 is lying to yourself.

---

### Issue #8: LoadBalancer Taking Forever to Provision

After deploying the service, LoadBalancer stuck in "Pending" for 10+ minutes. AWS was provisioning a Classic Load Balancer, which just takes a while. Also didn't have the AWS Load Balancer Controller installed.

The "fix" for now: wait. Classic LB provisioning takes 5-10 minutes. Go make coffee.

Better long-term: install the AWS Load Balancer Controller and use ALBs via Ingress instead:

```bash
# This will be in my "What's Next" section
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=todo-eks-cluster
```

Then use Ingress resources instead of LoadBalancer services for faster, more cost-effective load balancing.

Classic LBs are slow. ALBs via Ingress are the move for EKS.

---

## Testing the Final Result

Once everything was working, testing was straightforward:

```bash
# Get the LoadBalancer URL
export LB_URL=$(kubectl get svc todo-api -n apps -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Check health
curl http://$LB_URL/health | jq .
# Response:
# {
#   "status": "healthy",
#   "database": "connected"
# }

# Create a TODO
curl -X POST http://$LB_URL/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title":"Built with GitOps!","completed":true}'

# List all TODOs
curl http://$LB_URL/api/todos | jq .
```

Success! A working Kubernetes application deployed via GitOps.

## What I Learned

GitOps actually feels as good as everyone says. Push to main, watch ArgoCD sync, no SSH, no manual `kubectl apply`. That part clicked fast.

Helm took a bit longer but it's worth it. Raw YAML across a whole cluster is a mess. Charts make it manageable and reusable.

Security groups will get you on EKS the same way they get you on ECS. Always draw out source → destination before you apply.

That password bug was genuinely embarrassing but I probably learned more about Kubernetes secrets and templating in those two hours than in any tutorial.

Doing this after the ECS project made it way faster. Having a reference for "how do I structure Terraform modules" or "what does a working security group look like" saves a lot of time. Build the first thing, then build the second thing. They compound.


## Cost Breakdown

Here's what this infrastructure costs to run:

|Service|Configuration|Monthly Cost|
|---|---|---|
|EKS Control Plane|Managed|$73|
|EC2 Nodes (t3.medium)|2 nodes|$60|
|RDS PostgreSQL (t3.micro)|Single instance|$15|
|NAT Gateway|1 gateway|$35|
|Classic Load Balancer|1 LB|$20|
|EBS Volumes|40GB total|$4|
|**Total**||**~$207/month**|

**Cost optimization tips for dev environments:**

- Use SPOT instances for nodes (70% savings)
- Scale down to 1 node when not actively using
- Use RDS db.t3.micro (free tier eligible)
- Delete the NAT Gateway when not in use ($35/month savings)

**Optimized dev cost:** ~$100-120/month

## What's Next

This project has a solid foundation, but there are several enhancements I want to add:

### Immediate Next Steps (Week 2):

- **AWS Load Balancer Controller** - Replace Classic LB with ALB via Ingress
- **External Secrets Operator** - Pull secrets from AWS Secrets Manager instead of storing them in Git
- **Metrics Server** - Enable `kubectl top` commands and prepare for HPA

### Phase 2 Enhancements:

- **Horizontal Pod Autoscaler (HPA)** - Auto-scale pods based on CPU/memory
- **Cluster Autoscaler** - Auto-scale nodes based on pod demand
- **Prometheus + Grafana** - Full observability stack with dashboards
- **Network Policies** - Lock down pod-to-pod communication
- **Pod Security Standards** - Enable restricted security context

### Phase 3 (Production-Ready):

- **Multi-environment setup** - Separate staging and production clusters
- **CI/CD Pipeline** - GitHub Actions to build, test, and push images automatically
- **Backup strategy** - Velero for cluster state backup
- **Disaster recovery plan** - Test RDS restore and cluster recovery

## Resources & Links

[**Project Repository**](https://github.com/treehousepnw/eks-todo-gitops)

**Key Documentation I Used:**

- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [ArgoCD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Helm Chart Template Guide](https://helm.sh/docs/chart_template_guide/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)

**Related Project:**

- [My ECS + Terraform Project](https://blog.trenigma.dev/posts/2025-10-27-aws-project-reboot-career/)

## Final Thoughts

Kubernetes is actually fine once you get past the initial "what the hell is any of this" phase. The primitives make sense. ArgoCD, Helm, Prometheus: they all solve real problems instead of just creating new ones.

That password bug was infuriating. Two hours of "authentication failed" when everything else looked correct. But I know way more about Kubernetes secrets and templating now than I would have from just reading docs.

If I was doing it again: ECS first, then EKS. The first project gives you a mental model. The second one builds on it. Starting cold with EKS would have taken twice as long.

---

**Questions or feedback?** Find me on [LinkedIn](https://linkedin.com/in/trenigma) or check out more projects on my [GitHub](https://github.com/treehousepnw).

_Built with ☕ and a healthy dose of `kubectl describe pod`_