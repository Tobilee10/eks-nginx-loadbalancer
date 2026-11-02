# NGINX on Amazon EKS with a Kubernetes LoadBalancer Service

A hands-on project demonstrating how to deploy NGINX to Amazon EKS, expose it through a Kubernetes `Service` of type `LoadBalancer`, investigate readiness-probe failures, and scale the Deployment.

## Architecture

![Amazon EKS NGINX LoadBalancer architecture](diagrams/architecture-diagram.png)

> **AWS implementation note:** On EKS, a `Service` of type `LoadBalancer` asks the configured AWS cloud integration/controller to provision an external load balancer. The exact load-balancer type and behavior depend on the cluster's controller and configuration. This project does not assume ALB or NLB.

## Repository layout

```text
eks-nginx-loadbalancer/
├── manifests/
│   ├── 00-nginx-deployment.yaml
│   ├── 01-nginx-service-loadbalancer.yaml
│   └── 02-readiness-broken.yaml
├── diagrams/
│   └── architecture-diagram.png
├── docs/
│   └── troubleshooting.md
├── .gitignore
└── README.md
```

## Prerequisites

- An AWS account with permissions to create and manage EKS-related resources.
- AWS CLI configured with credentials and a default/selected region.
- `eksctl` and `kubectl` installed.
- An EKS cluster with a supported AWS load-balancer provisioning integration configured.

## 1. Create an EKS cluster

If you already created the cluster for the lab, skip this step. Example:

```bash
eksctl create cluster   --name nginx-lb-lab   --region <AWS_REGION>   --nodes 2   --node-type t3.medium
```

Check access:

```bash
aws sts get-caller-identity
kubectl get nodes
```

## 2. Deploy NGINX

```bash
kubectl apply -f manifests/00-nginx-deployment.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx -o wide
```

The Deployment starts with two replicas. Readiness probes determine whether a Pod is eligible to receive Service traffic; liveness probes help Kubernetes detect and restart an unhealthy container.

## 3. Expose NGINX

```bash
kubectl apply -f manifests/01-nginx-service-loadbalancer.yaml
kubectl get svc nginx-svc-lb
```

Wait for the external address to appear. Depending on the AWS integration, the value may be a hostname rather than an IP address:

```bash
kubectl get svc nginx-svc-lb   -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}'

kubectl get svc nginx-svc-lb   -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

Use the value that is populated as the endpoint:

```bash
curl http://<EXTERNAL-ENDPOINT>
```

If neither field is populated, inspect the Service events and your cluster's load-balancer controller/integration:

```bash
kubectl describe svc nginx-svc-lb
kubectl get events --sort-by=.lastTimestamp
```

## 4. Simulate a readiness failure

Apply the deliberately broken readiness-probe manifest:

```bash
kubectl apply -f manifests/02-readiness-broken.yaml
kubectl get pods -l app=nginx -w
```

The readiness probe requests `/does-not-exist`, which NGINX does not serve as a successful health endpoint. The new Pod may be Running but not Ready. With `maxUnavailable: 0`, the rollout should preserve the existing available replica(s) rather than intentionally reducing availability to make progress.

Inspect the rollout and endpoints:

```bash
kubectl rollout status deployment/nginx-deployment --timeout=30s
kubectl describe pods -l app=nginx
kubectl get endpointslices   -l kubernetes.io/service-name=nginx-svc-lb
```

A not-ready Pod should not be included as a ready backend endpoint. Confirm actual behavior in your cluster rather than assuming all controller implementations behave identically.

## 5. Restore the healthy configuration

```bash
kubectl apply -f manifests/00-nginx-deployment.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
```

## 6. Scale from two to four replicas

```bash
kubectl scale deployment/nginx-deployment --replicas=4
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
kubectl get endpointslices   -l kubernetes.io/service-name=nginx-svc-lb
```

The Service selector remains `app: nginx`; it can route to the additional ready Pods without changing the Service manifest. Re-check the same external endpoint with `curl`.

## 7. Clean up

Delete the Kubernetes resources:

```bash
kubectl delete -f manifests/01-nginx-service-loadbalancer.yaml
kubectl delete -f manifests/00-nginx-deployment.yaml
```

If this cluster was created only for this lab, delete it to avoid ongoing AWS charges:

```bash
eksctl delete cluster --name nginx-lb-lab --region <AWS_REGION>
```

Confirm the cluster name and region before running the deletion command.

## What this project demonstrates

- Provisioning and interacting with Amazon EKS using `eksctl`, AWS CLI, and `kubectl`.
- Deploying a containerized workload with replicas, probes, resource requests/limits, and a rolling-update strategy.
- Exposing an application through a Kubernetes `LoadBalancer` Service.
- Troubleshooting readiness failures using Pod status, rollout status, events, and EndpointSlices.
- Scaling a Deployment while keeping the Service configuration stable.

## Evidence to add before publishing

Add your own screenshots to the repository or blog:
1. `kubectl get nodes` showing the EKS worker nodes.
2. `kubectl get svc nginx-svc-lb` showing the assigned external hostname/IP.
3. Successful `curl` output from the external endpoint.
4. Pod status during the failed readiness probe.
5. EndpointSlice output during the failure and after recovery.
6. Four ready Pods after scaling.

