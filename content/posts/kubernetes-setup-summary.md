---
title: "Kubernetes Setup Summary"
date: 2026-05-09
draft: false
categories: ["docker"]
tags: []
---

# Kubernetes Setup on Raspberry Pi 5

## What We Covered

### Docker vs Kubernetes
- Docker runs containers on a single machine. Kubernetes orchestrates containers across multiple machines.
- Kubernetes is **reactive**, not preventative — it doesn't stop failures, it recovers from them automatically.
- Key advantage over Docker: reduced mean time to recovery. Pod crashes, it restarts. Node dies, workloads reschedule — no manual intervention.
- Kubernetes natively handles: scheduling, self-healing, rolling deployments, scaling, service discovery, secrets and config distribution. Monitoring, logging, and storage are still separate installs.

### Why k3s
- Lightweight, production-grade Kubernetes stripped of cloud-provider bloat.
- Runs well on a Pi 5.
- Uses `containerd` as its runtime, not Docker — so existing Docker Compose stacks are completely unaffected.

---

## Installation

### Install k3s (Traefik disabled to avoid port conflicts with NPM)
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

### Allow non-root kubectl access
Edit the k3s systemd service:
```bash
sudo nano /etc/systemd/system/k3s.service
```

Add `--write-kubeconfig-mode=644` to the ExecStart block:
```
ExecStart=/usr/local/bin/k3s \
    server \
        '--disable=traefik' \
        '--write-kubeconfig-mode=644' \
```

Restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

### Copy kubeconfig
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown pi:pi ~/.kube/config
```

Set in `/etc/environment`:
```
KUBECONFIG=/home/pi/.kube/config
```

---

## Verify Cluster

```bash
kubectl get nodes
kubectl get pods -A
```

Expected output — single node `Ready`, system pods `Running` in `kube-system` namespace:
- `coredns`
- `local-path-provisioner`
- `metrics-server`

---

## First Deployment — Nginx Load Balancing

File: `~/kubes/nginx-test.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
  namespace: default
spec:
  type: NodePort
  selector:
    app: nginx-test
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

NodePort range `30000-32767` is reserved for external access and doesn't conflict with existing Docker services.

```bash
kubectl apply -f ~/kubes/nginx-test.yaml
kubectl get pods -w
```

### Demonstrated load balancing
Write each pod's hostname to its index page:
```bash
kubectl exec -it <pod-name> -- sh -c 'echo $HOSTNAME > /usr/share/nginx/html/index.html'
```

Curl repeatedly to see round-robin across pods:
```bash
curl http://192.168.0.166:30080
```

### Self-healing
Delete a pod and watch Kubernetes immediately replace it:
```bash
kubectl delete pod <pod-name>
kubectl get pods -w
```

### Cleanup
```bash
kubectl delete -f ~/kubes/nginx-test.yaml
```

---

## Headlamp — Web UI

### Install
```bash
kubectl apply -f https://raw.githubusercontent.com/headlamp-k8s/headlamp/main/kubernetes-headlamp.yaml
```

### Expose via NodePort
```bash
kubectl patch svc headlamp -n kube-system -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 4466, "nodePort": 30090}]}}'
```

Access at `http://192.168.0.166:30090`

### Generate authentication token
```bash
kubectl create serviceaccount headlamp-admin -n kube-system
kubectl create clusterrolebinding headlamp-admin --clusterrole=cluster-admin --serviceaccount=kube-system:headlamp-admin
kubectl create token headlamp-admin -n kube-system
```

Paste the token output into the Headlamp login screen. Note: token expires in 1 hour by default.

---

## Fixes Applied

### metrics-server failing readiness probe on ARM64
```bash
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

---

## Useful Commands

| Command | Purpose |
|---|---|
| `kubectl get pods -A` | All pods across all namespaces |
| `kubectl get pods -n kube-system` | Pods in a specific namespace |
| `kubectl get pods -w` | Watch pods live |
| `kubectl describe pod <name> -n <ns>` | Detailed pod state and events |
| `kubectl logs <pod-name> -n <ns>` | Pod logs |
| `kubectl exec -it <pod> -- sh` | Shell into a container |
| `kubectl apply -f file.yaml` | Create or update resources |
| `kubectl delete -f file.yaml` | Delete resources from a manifest |
| `kubectl get events -A --field-selector type=Warning` | Cluster warnings only |
| `kubectl top nodes` | Node CPU and memory usage |
| `kubectl top pods -A` | Pod resource usage |
| `kubectl scale deployment <name> --replicas=5` | Scale a deployment |

---

## Working Directory
All Kubernetes manifests stored in `~/kubes/`
