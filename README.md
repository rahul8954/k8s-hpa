# Kubernetes HA & Scalable Demo (minikube)

This repository demonstrates a **production-like Kubernetes environment** with high availability, scalability, and TLS-secured ingress. It uses **Minikube on macOS** with Docker driver.

---

## What this demo contains

* Minikube 3-node cluster
* NGINX Ingress controller
* Sample web app (`hello-deployment`) with 3 replicas, readiness/liveness probes
* Horizontal Pod Autoscaler (HPA)
* PodDisruptionBudget (PDB)
* Ingress with TLS (self-signed)
* Optional: Postgres StatefulSet

---

## Prerequisites

* macOS
* Docker Desktop running
* kubectl (`brew install kubectl`)
* minikube (`brew install minikube`)
* openssl (for TLS certificate)

---

## Step 1: Start Minikube Cluster

```bash
minikube start --driver=docker --nodes=3 --cpus=2 --memory=4096
```

Verify nodes:

```bash
kubectl get nodes -o wide
```

---

## Step 2: Enable Required Addons

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

Verify ingress controller pods:

```bash
kubectl get pods -n ingress-nginx
```

---

## Step 3: Create Namespace

```bash
kubectl create ns myapp
```

---

## Step 4: Create TLS Secret

Generate a self-signed TLS certificate:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.local/O=myapp"
```

Create the secret in the namespace:

```bash
kubectl -n myapp create secret tls myapp-tls --key tls.key --cert tls.crt
```

---

## Step 5: Apply Manifests

Apply all Kubernetes manifests:

```bash
kubectl apply -f manifests/
```

Verify deployments, services, and ingress:

```bash
kubectl get pods -n myapp -o wide
kubectl get svc -n myapp
kubectl get ingress -n myapp
kubectl get hpa -n myapp
kubectl get pdb -n myapp
```

---

## Step 6: Map `myapp.local` to Minikube

Add entry to `/etc/hosts`:

```bash
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts
```

---

## Step 7: Access the Application

### Option 1: Using NodePort (quick test)

Get ingress controller NodePort URL:

```bash
minikube service ingress-nginx-controller -n ingress-nginx --url
```

Test with curl:

```bash
curl -k -H "Host: myapp.local" https://127.0.0.1:<HTTPS_PORT>/
```

Expected output:

```
Hello, world!
Version: 1.0.0
Hostname: hello-deployment-xxxxx
```

---

### Option 2: Using Minikube Tunnel (bonus)

Run tunnel in a separate terminal:

```bash
minikube tunnel
```

Update `/etc/hosts`:

```
127.0.0.1 myapp.local
```

Now you can access:

```bash
curl -k https://myapp.local/
```

Multiple curls will show **different pod hostnames**, proving load balancing and HA.

---

## Step 8: Horizontal Pod Autoscaler (HPA)

Check HPA status:

```bash
kubectl get hpa -n myapp -w
```

To generate CPU load (simulate scaling):

```bash
kubectl run -n myapp load-generator --rm -it --image=busybox -- /bin/sh
```

Inside the pod:

```sh
while true; do wget -q -O- http://hello-service.myapp.svc.cluster.local/; done
```

HPA will scale your deployment automatically up to the max replicas defined.

---

## Step 9: Zero-Downtime Rolling Updates

Update the deployment image:

```bash
kubectl set image deployment/hello-deployment hello=gcr.io/google-samples/hello-app:2.0 -n myapp
kubectl rollout status deployment/hello-deployment -n myapp
```

Verify pods update without downtime.

---



---

## Step 11: Verification / Screenshots to include

* `kubectl get nodes -o wide`
* `kubectl get pods -n myapp -o wide`
* `kubectl get svc -n myapp`
* `kubectl get ingress -n myapp`
* `kubectl get hpa -n myapp`
* `curl -k https://myapp.local/`

---

## Step 12: Cleanup

```bash
minikube delete
```

---
