# Troubleshooting notes

## External address is pending

```bash
kubectl describe svc nginx-svc-lb
kubectl get events --sort-by=.lastTimestamp
kubectl get pods -A
```

Check that the EKS cluster has a compatible AWS load-balancer provisioning integration installed and correctly configured. Inspect its controller logs and IAM permissions according to the setup used by your cluster.

## Pod is Running but not Ready

`Running` describes the Pod phase; it does not mean the Pod is eligible to receive Service traffic. Inspect probe events and container logs:

```bash
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
```

Restore the valid readiness path by reapplying:

```bash
kubectl apply -f manifests/00-nginx-deployment.yaml
```

## Inspect Service backends

```bash
kubectl get endpointslices   -l kubernetes.io/service-name=nginx-svc-lb -o yaml
```

Look at endpoint readiness conditions and confirm that ready endpoints correspond to the expected Pods.

## Rollout is stalled

```bash
kubectl rollout status deployment/nginx-deployment
kubectl describe deployment nginx-deployment
kubectl get rs
kubectl get pods -l app=nginx
```

A failed readiness check can prevent a new Pod from becoming available. With `maxUnavailable: 0`, Kubernetes avoids taking an available replica down just to advance the rollout.