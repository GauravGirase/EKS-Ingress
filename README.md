# EKS-Ingress
# Summary


## Tools
| Tool | Version |
| ------ | ------ |
| awscli | aws-cli/2.34.2 |
| eksctl | 0.224.0|
| kubectl Drive | v1.35.2 |
| helm | v4.1.1 |
| docker | 29.2.1 |

## Step 1: EKS cluster provision
eks-config.yaml
```bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: gaurav-ingress-poc
  region: ap-south-1
  tags:
    owner: gaurav
    region: ap-south-1
    project: ingress-poc

availabilityZones:
  - ap-south-1a
  - ap-south-1b

iam:
  withOIDC: true

managedNodeGroups:
  - name: gaurav-poc-eks-private-ng
    instanceType: t3.small
    minSize: 4
    maxSize: 4
    privateNetworking: true
    volumeSize: 20
    iam:
      withAddonPolicies:
        autoScaler: true
        externalDNS: false
        certManager: true
        ebs: false
        efs: false
        albIngress: true
        xRay: false
        cloudWatch: false
    labels:
      lifecycle: ec2-autoscaler
```
### Apply the config usins eksctl
```bash
eksctl create cluster -f eks-config.yaml
```
### verify node distribution
```bash
kubectl get node --show-labels | grep topology.kubernetes.io/zone
```
## Step 2: Install AWS load balancer controller
### step 2.1: Create IAM policy for ALB controller
```bash
# Donwload policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json
# Create policy
aws iam create-policy \
--policy-name AWSLoadBalancerContollerIAMPolicy \
--policy-document file://iam_policy.json
```
### step 2.2: Create IAM-Backed k8s svc account
```bash
eksctl create iamserviceaccount \
--cluster=gaurav-ingress-poc \
--namespace=kube-system \
--name=alb-controller \
--attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region ap-south-1
--approve
```
### verify 
```bash
kubectl get sa -n kube-system alb-controller -o yaml
```
### step 2.3: Install AWS ALB controller using helm
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=gaurav-ingress-poc \
--set serviceAccount.create=false \
--set serviceAccount.name=alb-controller \
--version 1.13.0
```
### Test with the following commands to ensure ALB controller is installed
```bash
helm search repo eks/aws-load-balancer-controller --versions
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl describe pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```
## Step 3: Create landing page deployment & service
manifests/landing.yaml
```bash
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: landing-page
  namespace: app1-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: landing-page
  template:
    metadata:
      labels:
        app: landing-page
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: landing-page
      containers:
        - name: landing-page-container
          image: gauravgirase/ingress:v1.0.0
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: landing-page-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /index.html
spec:
  selector:
    app: landing-page
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
```
## Step 4: Create dashboard deployment, service
manifests/dashboard.yaml
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: app1-ns
  name: dashboard
spec:
  replicas: 2
  selector:
    matchLabels:
      app: dashboard
  template:
    metadata:
      labels:
        app: dashboard
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: dashboard
      containers:
        - name: dashboard-containers
          image: gauravgirase/ingress:v1.0.0
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: dashboard-svc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /dashboard/index.html
spec:
  selector:
    app: dashboard
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
```
## Step 5: Create ingress resource
manifests/ingress.yaml
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-poc
  namespace: app1-ns
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/load-balancer-name: ingress-poc
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    alb.ingress.kubernetes.io/success-codes: '200'
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /landing
            pathType: Prefix
            backend:
              service:
                name: landing-page-svc
                port:
                  number: 80
          - path: /dashboard
            pathType: Prefix
            backend:
              service:
                name: dashboard-svc
                port:
                  number: 80
```
## Step 6: Deploy resources in cluster
```bash
kubectl config set-context --current --namespace app1-ns
kubeclt apply -f manifests/
```
