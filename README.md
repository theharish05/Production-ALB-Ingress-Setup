# Production ALB Ingress Setup on AWS EKS

##  Objective

The goal of this exercise is to configure an AWS Application Load Balancer (ALB) using the Kubernetes Ingress resource to securely route public traffic to three separate internal applications based on specific URL paths.

### Routing Requirements

* `/api/*` $\rightarrow$ `api-service`
* `/admin/*` $\rightarrow$ `admin-service`
* `/dashboard/*` $\rightarrow$ `dashboard-service`

### Infrastructure Requirements

*  **SSL Termination** configured on the ALB.
*  **HTTP to HTTPS Redirection** (301 Redirect).
*  **Target Group Health Checks** to monitor pod readiness.

---

##  Architecture & Component Workflow

To handle external routing natively in AWS EKS, multiple systems work together dynamically:

```
[ Public Traffic ] ──> [ AWS ALB (Port 80/443) ] 
                              │
         ┌────────────────────┼────────────────────┐ (Path Rules)
         ▼                    ▼                    ▼
   /api Target Group    /admin Target Group   /dashboard Target Group
         │                    │                    │
         ▼                    ▼                    ▼
     [api-pod]           [admin-pod]         [dashboard-pod]

```

1. **Kubernetes Ingress Manifest:** Declares the human-readable routing paths and maps them to cluster services.
2. **AWS Load Balancer Controller:** A pod running via Helm that monitors the cluster API. The moment an Ingress is created, it commands the AWS API to provision a physical ALB.
3. **IRSA (IAM Roles for Service Accounts):** Securely bridges EKS to AWS IAM using short-lived tokens, eliminating the need to hardcode dangerous static root AWS keys inside pods.
4. **Target Groups:** The ALB routes traffic directly to individual Pod private IPs (`target-type: ip`) synced by the controller, bypassing extra kube-proxy hops.

---

##  Step-by-Step Implementation Guide

### Step 1: Deploy the Applications and Services

Create a manifest named `apps.yaml` to spin up three distinct test applications using a lightweight plaintext mirroring container.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api-container
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: api
# Repeat this structure for admin-deployment/admin-service and dashboard-deployment/dashboard-service

```

Apply the applications to your cluster:

```bash
kubectl apply -f apps.yaml

```

Verify the pods are running and internal services are initialized:

```bash
kubectl get pods,svc

```

### Step 2: Configure AWS Permissions using IRSA

1. **Associate IAM OIDC Provider:**
```bash
eksctl utils associate-iam-oidc-provider \
  --cluster=<YOUR_CLUSTER_NAME> \
  --region=<YOUR_REGION> \
  --approve

```


2. **Download and Create the IAM Policy:**
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

```


3. **Create the IAM Service Account mapping:**
```bash
eksctl create iamserviceaccount \
  --cluster=<YOUR_CLUSTER_NAME> \
  --region=<YOUR_REGION> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

```



### Step 3: Install the AWS Load Balancer Controller via Helm

1. Add and sync the official EKS repository:
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

```


2. Deploy the controller into the system namespace:
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<YOUR_CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<YOUR_REGION>

```


3. Confirm the backend system engine is running successfully:
```bash
kubectl get pods -n kube-system | grep aws-load-balancer

```



### Step 4: Deploy the Ingress Resource

Create an `ingress.yaml` configuration. The annotations drive the cloud controller to set up public infrastructure, enforce SSL standards, handle target group parameters, and redirect unencrypted traffic natively.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-alb-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:<REGION>:<ACCOUNT_ID>:certificate/<CERTIFICATE_UUID>
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
spec:
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
          - path: /dashboard
            pathType: Prefix
            backend:
              service:
                name: dashboard-service
                port:
                  number: 80

```

Apply the routing manifest:

```bash
kubectl apply -f ingress.yaml

```

---

##  Verification and Testing

1. **Monitor ALB Provisioning:**
Wait 2–5 minutes for AWS to assign a public DNS endpoint.
```bash
kubectl get ingress production-alb-ingress -w

```


2. **Verify Path-Based Routing via Terminal:**
Execute queries against the newly populated `ADDRESS` URL endpoint to verify that separate pod groups process traffic contextually:
```bash
curl http://<YOUR-ALB-DNS-ADDRESS>.amazonaws.com/api
curl http://<YOUR-ALB-DNS-ADDRESS>.amazonaws.com/admin
curl http://<YOUR-ALB-DNS-ADDRESS>.amazonaws.com/dashboard

```


3. **Check Target Group Health Status:**
Inspect routing events directly within Kubernetes to confirm Target registration:
```bash
kubectl describe ingress production-alb-ingress

```


*(Alternatively, navigate to **EC2 -> Target Groups** in the AWS console to view the bright green `Healthy` status flags of your underlying Pod IPs).*
