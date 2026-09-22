# Kubernetes on AWS EKS (Stage B)

Stage B of Project 4 in a hands-on DevOps learning track. Stage A covered core Kubernetes concepts (Pods, Deployments, Services, self-healing, rolling updates, multi-node resilience) using **k3s** on plain EC2 — deliberately kept free of AWS-specific abstractions. This stage takes that same foundation and moves it onto **AWS EKS**, a fully managed Kubernetes control plane, to see exactly what AWS adds on top of vanilla Kubernetes.

Same nginx app, same YAML manifests, same `kubectl` — the goal here isn't to learn Kubernetes again, it's to see the AWS-specific layer added on top of concepts already understood.

---

## What's different from Stage A (k3s)

| | k3s (Stage A) | EKS (Stage B) |
|---|---|---|
| Control plane | Self-managed, runs on your own EC2 instance | Fully managed by AWS — never SSH'd into, AWS handles availability/patching |
| Worker nodes | Same EC2 instance (or a manually joined second one) | Separate managed node group, provisioned via `eksctl` |
| Pod networking | Internal-only fake IPs (`10.42.x.x`) via flannel | Real VPC IP addresses via the AWS VPC CNI plugin |
| `type: LoadBalancer` | Behaves identically to NodePort (no real cloud integration) | Provisions an actual AWS Load Balancer automatically |
| IAM integration | None | IRSA (IAM Roles for Service Accounts) — Pods can get scoped AWS permissions without embedded credentials |

---

## Setup

### Prerequisites
- `eksctl` and standalone `kubectl` installed
- AWS CLI configured with credentials that have permission to create EKS clusters, VPCs, and IAM roles

### 1. Create the cluster

```bash
eksctl create cluster \
  --name learning-cluster \
  --region ap-south-1 \
  --nodegroup-name standard-workers \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed
```

Takes 15-20 minutes — provisions the control plane, VPC networking, IAM roles, and worker nodes, and automatically configures local `kubectl` to point at the new cluster.

**Cost note:** the EKS control plane costs ~$0.10/hour regardless of usage, on top of the worker node EC2 costs. Unlike stopping an EC2 instance, there's no "pause" for the control plane — only full deletion stops the charge. Delete the cluster as soon as you're done with a session (see Teardown below).

### 2. Deploy the app

`nginx-deployment.yaml` — identical to the Stage A manifest.

`nginx-service.yaml` — the one meaningful change, `type: LoadBalancer` instead of `NodePort`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl get svc nginx-service --watch
```

`EXTERNAL-IP` shows `<pending>` for 1-2 minutes while AWS provisions a real Load Balancer, then resolves to a real `*.elb.amazonaws.com` hostname. Visit it directly in a browser — no port number needed.

### 3. See where Pods actually landed
```bash
kubectl get pods -o wide
```

### 4. See everything, not just your app
```bash
kubectl get pods -A
```
EKS runs its own system Pods per node (`aws-node` for VPC networking, `kube-proxy`, `coredns`, `metrics-server`) in the `kube-system` namespace — `kubectl get pods` alone only shows the `default` namespace, so counts will look lower than the AWS console's cluster-wide total until you check across all namespaces.

### Teardown (do this promptly — see cost note above)
```bash
eksctl delete cluster --name learning-cluster --region ap-south-1
```
Confirm it's fully gone:
```bash
eksctl get cluster --region ap-south-1
aws cloudformation describe-stacks --region ap-south-1 --query "Stacks[?contains(StackName, 'learning-cluster')].StackName"
```
Both should return empty.

---

## How `type: LoadBalancer` actually works

It isn't a separate mechanism from NodePort — it's built on top of it. Kubernetes still auto-assigns a NodePort (visible in `kubectl get svc` output, e.g. `80:30219/TCP`), and AWS's `cloud-controller-manager` provisions a real Load Balancer that targets that NodePort on every worker node. `kube-proxy` still does the final hop from Node to Pod, exactly as in Stage A — the only new part is everything *before* the traffic reaches the Node.

## A real mix-up worth documenting

An earlier, already-deleted cluster attempt left behind **stale CloudFormation stacks** stuck in `CREATE_COMPLETE` even though the actual AWS resources (cluster, EC2 instances) were genuinely gone — confirmed via `aws eks list-clusters` and `aws ec2 describe-instances` both returning empty. Deleting the stale stacks directly failed with `DELETE_FAILED`, root cause: **an EKS-created security group** (`eks-cluster-sg-*`) that CloudFormation doesn't track, since EKS creates it outside the stack's management. A VPC can't delete while any non-default security group still exists inside it. Fix: find and delete that security group manually (`aws ec2 describe-security-groups` / `delete-security-group`), then retry the stack deletion.

**Lesson:** always tear down EKS clusters with `eksctl delete cluster`, not manual CloudFormation stack deletion — `eksctl` knows to clean up AWS-created resources like this security group *before* attempting the VPC deletion, avoiding this exact stuck state.

---

## Project Status

- [x] EKS cluster created (managed control plane + managed node group)
- [x] nginx Deployment + real AWS Load Balancer Service, confirmed reachable
- [x] Verified Pod distribution across worker nodes
- [x] Documented the LoadBalancer → NodePort → kube-proxy → Pod chain
- [x] Cluster torn down after the session
