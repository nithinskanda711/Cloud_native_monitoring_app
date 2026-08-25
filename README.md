# Cloud Native Monitoring App

A Flask application that reports live CPU and memory utilisation using `psutil`, rendered as Plotly gauges. Containerised with Docker, published to Amazon ECR, and deployed to Amazon EKS.

Built as a hands-on DevOps project to work through the full path from local application to running pod on a managed Kubernetes cluster.

![Dashboard](docs/dashboard.png)

---

## Architecture

```
app.py (Flask + psutil)
    │
    ├── Docker image  ──►  Amazon ECR
    │                          │
    │                          ▼
    └── deployment.yaml ──► Amazon EKS ──► Pod (gunicorn :5001)
        service.yaml                          │
        cluster.yaml (eksctl)                 ▼
                                          ClusterIP Service
```

---

## Prerequisites

| Tool | Install |
|---|---|
| Docker Desktop | `brew install --cask docker` |
| kubectl | `brew install kubectl` |
| eksctl | `brew install eksctl` |
| AWS CLI | `brew install awscli` |
| Python 3.12 | `brew install python@3.12` |

An AWS account with programmatic access configured via `aws configure`.

> **Cost warning.** The EKS control plane bills roughly **$0.10/hour (~$73/month)** whether or not any workload is running, plus EC2 node costs. Delete the cluster when you are not using it — see [Teardown](#teardown).

---

## Quick start (local)

```bash
git clone https://github.com/nithinskanda711/Cloud_native_monitoring_app.git
cd Cloud_native_monitoring_app

python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

python app.py
```

Open <http://localhost:5001>.

To run the way the container does:

```bash
gunicorn --bind 127.0.0.1:5001 app:app
```

---

## Step 1 — Containerise

```bash
docker build --platform linux/amd64 -t my-flask-app .
docker run --rm -p 5001:5001 my-flask-app
```

Open <http://localhost:5001> to confirm the image works before pushing.

**`--platform linux/amd64` is not optional on Apple Silicon.** A Mac builds `arm64` by default. EKS nodes are x86_64, and an architecture mismatch produces a pod that starts and immediately dies with `exec format error` — a confusing failure, because the manifest is correct and the image pulls successfully.

Verify with:

```bash
docker inspect my-flask-app --format '{{.Architecture}}'
```

---

## Step 2 — Push to Amazon ECR

Create the repository:

```bash
python ecr.py
```

Authenticate Docker to the registry, tag, and push. Replace `<account-id>` with your own:

```bash
export ACCOUNT_ID=<account-id>
export REGION=eu-west-1
export REPO=$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/my-cloud-native-app

aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

docker tag my-flask-app:latest $REPO:v1
docker push $REPO:v1
```

The ECR login token expires after 12 hours — rerun the login line when pushes start failing with an authorisation error.

Tag with a version rather than `:latest`. `:latest` makes rollbacks impossible and leaves untagged layers accumulating in the repository. Set a lifecycle policy on the repo to expire untagged images.

---

## Step 3 — Create the EKS cluster

```bash
eksctl create cluster -f cluster.yaml
```

Takes about 15 minutes unattended. This builds a dedicated VPC with correctly tagged public and private subnets, a node IAM role with the right trust policy, the cluster access entries, and a single managed node group.

**Use eksctl rather than the console.** A cluster created through the AWS console in the default VPC will fail in ways that are slow to diagnose:

- Default VPC subnets lack the `kubernetes.io/cluster/<name>` tags the CNI needs for network discovery. Nodes register but sit at `NotReady` with `cni plugin not initialized`.
- The console will accept the *cluster* IAM role where the *node* IAM role belongs. Nodes then launch as EC2 instances but never register, because the cluster role trusts `eks.amazonaws.com` while kubelet needs a role trusting `ec2.amazonaws.com`.
- EKS Auto Mode requires access entries of type `EC2`. A leftover `EC2_LINUX` entry from a managed node group attempt will block the compute config update.

None of these are patchable in place with any confidence. A declarative cluster config avoids all three.

Wire up kubectl:

```bash
aws eks update-kubeconfig --name cloud-native-cluster --region eu-west-1
kubectl get nodes
```

You want one node in `Ready` state. Cluster names are case-sensitive.

---

## Step 4 — Deploy

Update the image URI in `deployment.yaml` to point at your own ECR repository, then:

```bash
kubectl apply -f deployment.yaml -f service.yaml
kubectl rollout status deployment my-flask-app
kubectl get pods
```

Reach the app:

```bash
kubectl port-forward svc/my-flask-service 8080:5001
```

Open <http://localhost:8080>.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| Pod `Pending`, `no nodes available` | No node group, or nodes not yet registered. Check `kubectl get nodes`. |
| Node `NotReady`, `cni plugin not initialized` | Subnets missing EKS discovery tags. Rebuild with eksctl. |
| NodeClaim `Registered: NodeNotFound` | Compute config using the cluster IAM role instead of the node role. |
| Pod `CrashLoopBackOff`, `exec format error` | arm64 image on x86_64 nodes. Rebuild with `--platform linux/amd64`. |
| Pod `Running` but Service refuses connections | `targetPort` does not match the port gunicorn binds. |
| `Service is invalid: spec.ports[0].name Required` | Resource was created imperatively, so `kubectl apply` merged instead of replacing. `kubectl delete svc` then reapply. |
| Node group `AsgInstanceLaunchFailures` | Instance type unavailable in the AZ. `t2.micro` also caps at 4 pods per node, which CoreDNS alone will nearly fill. |

The diagnosis almost always sits in the Events or Conditions section:

```bash
kubectl describe pod <pod-name>
kubectl describe node <node-name>
kubectl describe nodeclaim <claim-name>
kubectl get events --sort-by=.lastTimestamp
```

---

## Security

The repository is scanned by SonarQube Cloud on every push. The following hardening was applied in response to findings:

- **gunicorn instead of Flask's development server.** The dev server is not built for production and exposes an interactive debugger when `debug=True`.
- **Non-root container user.** `useradd appuser` plus `USER appuser` in the Dockerfile.
- **Explicit `COPY` instructions** rather than `COPY . .`, so only `app.py` and `templates/` enter the image.
- **Resource requests and limits** on the pod, and `automountServiceAccountToken: false` since the app makes no Kubernetes API calls.
- **Pinned dependencies** via `pip freeze`.
- **Versioned image tags** rather than `:latest`.

Two findings were reviewed and accepted rather than fixed:

- **CSRF protection** — the single route is a read-only `GET` with no forms, no session state, and no state-changing requests.
- **Dependency hash verification** — dependencies are pinned to exact versions; hash pinning adds little for a single-source dependency chain in a personal project.

---

## Teardown

```bash
eksctl delete cluster -f cluster.yaml --wait
```

Then confirm nothing survives, since orphaned load balancers and NAT gateways bill independently of the cluster:

```bash
export AWS_PAGER=""
aws eks list-clusters --region eu-west-1
aws ec2 describe-instances --region eu-west-1 \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId'
aws elbv2 describe-load-balancers --region eu-west-1 --query 'LoadBalancers[].LoadBalancerName'
aws ec2 describe-volumes --region eu-west-1 \
  --filters "Name=status,Values=available" --query 'Volumes[].VolumeId'
aws ec2 describe-nat-gateways --region eu-west-1 \
  --query 'NatGateways[?State==`available`].NatGatewayId'
aws ec2 describe-addresses --region eu-west-1 \
  --query 'Addresses[?AssociationId==null].PublicIp'
```

All should return empty. The ECR repository costs pennies and can be left in place.

---

## Files

| File | Purpose |
|---|---|
| `app.py` | Flask application |
| `templates/index.html` | Plotly gauge dashboard |
| `Dockerfile` | Container build |
| `requirements.txt` | Pinned Python dependencies |
| `ecr.py` | Creates the ECR repository via boto3 |
| `eks.py` | Creates Deployment and Service via the Kubernetes Python client (superseded by the YAML manifests) |
| `cluster.yaml` | eksctl cluster definition |
| `deployment.yaml` | Kubernetes Deployment |
| `service.yaml` | Kubernetes Service |
