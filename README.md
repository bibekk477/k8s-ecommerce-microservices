# Online Boutique – Minikube Deployment Guide

A cloud-native microservices demo application deployed locally using Minikube.

---

## Architecture

```
        User
        │ HTTP
        ▼
    frontend (NodePort :30080)
        │
        ├──► adservice          (9555)
        ├──► recommendationservice (8080)
        │        └──► productcatalogservice (3550)
        ├──► productcatalogservice (3550)
        ├──► currencyservice    (7000)
        ├──► cartservice        (7070)
        │        └──► redis-cart (6379)
        ├──► shippingservice    (50051)
        └──► checkoutservice    (5050)
                 ├──► productcatalogservice
                 ├──► shippingservice
                 ├──► paymentservice  (50051)
                 ├──► emailservice    (8080)
                 ├──► currencyservice   (7000)
                 └──► cartservice   (7070)
```

---

## Prerequisites

Make sure the following tools are installed on your machine:

| Tool     | Version | Install                                  |
| -------- | ------- | ---------------------------------------- |
| Minikube | v1.32+  | https://minikube.sigs.k8s.io/docs/start/ |
| kubectl  | v1.28+  | https://kubernetes.io/docs/tasks/tools/  |
| Docker   | 24+     | https://docs.docker.com/get-docker/      |

Verify your installations:

```bash
minikube version
kubectl version --client
docker version
```

---

## Quick Start

### 1. Start Minikube

```bash
minikube start --cpus=4 --memory=8192 --driver=docker
```

> **Minimum recommended:** 4 CPUs and 8 GB RAM. The app runs 11 services simultaneously.

Verify the cluster is up:

```bash
kubectl get nodes
```

### 3. Create a namespace "microservices"

```bash
kubectl create ns microservices
```

### 3. Deploy All Services

Apply all the manifesto at once using:

```bash
kubectl apply -f ./ -n microservices
```

### 3. Wait for Pods to be Ready

```bash
kubectl get pods --watch
```

Wait until all pods show `Running` and `1/1` READY. This may take 2–5 minutes on first run as images are pulled.

```
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-xxxx                           1/1     Running   0          2m
cartservice-xxxx                         1/1     Running   0          2m
checkoutservice-xxxx                     1/1     Running   0          2m
currencyservice-xxxx                     1/1     Running   0          2m
emailservice-xxxx                        1/1     Running   0          2m
frontend-xxxx                            1/1     Running   0          2m
paymentservice-xxxx                      1/1     Running   0          2m
productcatalogservice-xxxx               1/1     Running   0          2m
recommendationservice-xxxx               1/1     Running   0          2m
redis-cart-xxxx                          1/1     Running   0          2m
shippingservice-xxxx                     1/1     Running   0          2m
```

### 4. Access the Application

Since the frontend uses `NodePort`, use Minikube's built-in service tunnel:

```bash
minikube service frontend -n microservices
```

---

## Useful Commands

### Check Pod Status

```bash
kubectl get pods -n microservices
kubectl get pods -o wide -n microservices       # shows which node each pod is on
kubectl describe pod <pod-name>n microservices  # detailed info and events
```

### View Logs

```bash
kubectl logs <pod-name> -n microservices
kubectl logs <pod-name> -f -n microservices         # follow/stream logs
kubectl logs <pod-name> --previous -n microservices # logs from crashed pod
```

Example:

```bash
kubectl logs $(kubectl get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}') -n microservices
```

### Check Services

```bash
kubectl get services -n microservices
kubectl get svc -n microservices
```

### Shell into a Pod

```bash
kubectl exec -it <pod-name> -n microservices -- /bin/sh
```

### Restart a Deployment

```bash
kubectl rollout restart deployment <deployment-name> -n microservices
```

---

## Troubleshooting

### Pods stuck in `Pending`

Minikube may not have enough resources. Try:

```bash
minikube stop
minikube start --cpus=4 --memory=8192
```

### Pods in `CrashLoopBackOff`

Check the logs:

```bash
kubectl logs <pod-name> -n microservices --previous
```

Common causes:

- `redis-cart` not ready before `cartservice` starts — it will auto-recover
- Image pull failures — check your internet connection

### `ImagePullBackOff`

The container images are pulled from `gcr.io`. Make sure you have internet access:

```bash
kubectl describe pod <pod-name> -n microservices
```

Look for `Events` at the bottom for specific error messages.

### Cannot Access the App

Make sure the frontend service is NodePort and Minikube tunnel is working:

```bash
kubectl get svc frontend -n microservices
minikube service frontend --url -n microservices
```

### Email Service Port Warning

The `emailservice` deployment exposes port `8080`, but `checkoutservice` is configured to reach it at `emailservice:5000`. To fix this, update the `emailservice` Service manifest port from `8080` to `5000`, or update the `checkoutservice` env var `EMAIL_SERVICE_ADDR` to `emailservice:8080`.

---

## Stopping and Cleaning Up

### Stop Minikube (preserves state)

```bash
minikube stop
```

### Delete all deployed resources

```bash
kubectl delete -f online-boutique.yaml -n microservices
```

### Delete the Minikube cluster entirely

```bash
minikube delete
```

---

## Resource Summary

| Service               | Image                                           | Port      | Type      |
| --------------------- | ----------------------------------------------- | --------- | --------- |
| frontend              | microservices-demo/frontend:v0.8.0              | 80 → 8080 | NodePort  |
| adservice             | microservices-demo/adservice:v0.8.0             | 9555      | ClusterIP |
| cartservice           | microservices-demo/cartservice:v0.8.0           | 7070      | ClusterIP |
| checkoutservice       | microservices-demo/checkoutservice:v0.8.0       | 5050      | ClusterIP |
| currencyservice       | microservices-demo/currencyservice:v0.8.0       | 7000      | ClusterIP |
| emailservice          | microservices-demo/emailservice:v0.8.0          | 8080      | ClusterIP |
| paymentservice        | microservices-demo/paymentservice:v0.8.0        | 50051     | ClusterIP |
| productcatalogservice | microservices-demo/productcatalogservice:v0.8.0 | 3550      | ClusterIP |
| recommendationservice | microservices-demo/recommendationservice:v0.8.0 | 8080      | ClusterIP |
| shippingservice       | microservices-demo/shippingservice:v0.8.0       | 50051     | ClusterIP |
| redis-cart            | redis:7.2                                       | 6379      | ClusterIP |

---

## Notes

- **Redis** uses `emptyDir` storage — cart data is lost if the `redis-cart` pod restarts. This is fine for local development.
- All internal services use `ClusterIP` and are not accessible outside the cluster.
- Only `frontend` is exposed via `NodePort` on port `30080`.
- There is no `loadgenerator` service defined in the manifests — if you want to simulate traffic, you can deploy it separately or use a tool like `k6` or `hey`.

---
