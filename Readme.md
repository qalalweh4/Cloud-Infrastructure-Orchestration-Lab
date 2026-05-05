# Cloud Infrastructure & Orchestration Lab

**Author:** Abdullah AlQalalweh  
**Duration:** Dec 2025 – Jan 2026  
**Stack:** OpenStack · Kubernetes · Docker · GitHub Actions · MetalLB · Nginx Ingress · Prometheus · Grafana  
**Scale:** 3-node OpenStack cluster · 4-node Kubernetes cluster · 3-tier application · Full observability pipeline  

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Architecture](#architecture)
3. [Phase 1 — OpenStack Private Cloud Deployment](#phase-1--openstack-private-cloud-deployment)
4. [Phase 2 — Kubernetes Cluster on OpenStack](#phase-2--kubernetes-cluster-on-openstack)
5. [Phase 3 — Docker Containerisation and Private Registry](#phase-3--docker-containerisation-and-private-registry)
6. [Phase 4 — CI/CD Pipeline](#phase-4--cicd-pipeline)
7. [Phase 5 — Network Load Balancing](#phase-5--network-load-balancing)
8. [Phase 6 — Prometheus Metrics Collection](#phase-6--prometheus-metrics-collection)
9. [Phase 7 — Grafana Dashboards and Alerting](#phase-7--grafana-dashboards-and-alerting)
10. [Outcomes Summary](#outcomes-summary)

---

## Lab Overview

This lab builds a full private cloud platform from the ground up — the same layered architecture used by enterprises that cannot or will not rely on public cloud providers. The stack mirrors what you would find at a large bank, telecoms company, or government organisation running their own datacentre.

The layers, bottom to top:

| Layer | Technology | Purpose |
|---|---|---|
| IaaS (private cloud) | OpenStack | Provision VMs, networks, storage on bare metal |
| Container orchestration | Kubernetes | Schedule and manage containerised workloads |
| Container runtime | Docker + private registry | Build, store, and run application images |
| Delivery | GitHub Actions CI/CD | Automate build → test → deploy on every push |
| Networking | MetalLB + Nginx Ingress | External access, load balancing, TLS termination |
| Observability | Prometheus + Grafana | Metrics, dashboards, alerting across the full stack |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Physical / Hypervisor Layer                     │
│         3 bare-metal servers running Ubuntu 22.04                 │
└──────────────────────┬───────────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────────┐
│                   OpenStack (IaaS Layer)                           │
│                                                                    │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────┐  │
│  │ Controller node  │  │  Compute node 1  │  │  Compute node 2 │  │
│  │ Keystone/Nova    │  │  Nova compute    │  │  Nova compute   │  │
│  │ Glance/Neutron   │  │  Neutron agent   │  │  Neutron agent  │  │
│  │ Cinder/Horizon   │  │  192.168.10.11   │  │  192.168.10.12  │  │
│  │ 192.168.10.10    │  └──────────────────┘  └─────────────────┘  │
│  └─────────────────┘                                               │
│                                                                    │
│  Networks: corp-net (10.0.1.0/24)  Floating IPs: 192.168.10.x     │
└──────────────────────┬───────────────────────────────────────────┘
                       │  OpenStack VMs
┌──────────────────────▼───────────────────────────────────────────┐
│               Kubernetes Cluster (Orchestration Layer)             │
│                                                                    │
│  ┌──────────────────┐   ┌────────────┐ ┌────────────┐ ┌────────┐ │
│  │  Control Plane   │   │  Worker 1  │ │  Worker 2  │ │Worker 3│ │
│  │  10.0.1.10       │   │  10.0.1.11 │ │  10.0.1.12 │ │10.0.1.13│ │
│  │  API / etcd      │   │  Pods      │ │  Pods      │ │  Pods  │ │
│  └──────────────────┘   └────────────┘ └────────────┘ └────────┘ │
│                                                                    │
│  CNI: Flannel (VXLAN)  LB: MetalLB  Ingress: Nginx                │
└──────────────────────┬───────────────────────────────────────────┘
                       │  Pods / Services
┌──────────────────────▼───────────────────────────────────────────┐
│                  Application Layer                                  │
│                                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │  Flask API   │  │ React frontend│  │  PostgreSQL  │            │
│  │  (3 replicas)│  │ (2 replicas)  │  │  (StatefulSet│            │
│  │  :5000       │  │  :3000        │  │  + PVC)      │            │
│  └──────────────┘  └──────────────┘  └──────────────┘            │
└──────────────────────┬───────────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────────┐
│                  Observability Layer                                │
│                                                                    │
│  Prometheus ──── scrapes ────► All pods, nodes, K8s API           │
│  Grafana    ──── queries ────► Prometheus                          │
│  Alertmanager ── routes  ────► Slack webhooks                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Phase 1 — OpenStack Private Cloud Deployment

### Overview

OpenStack is the most widely deployed open-source private cloud platform. It provides the same core capabilities as AWS (compute, networking, storage, identity) but runs entirely on hardware you control. This phase deploys the full OpenStack stack and configures the network topology that all subsequent layers sit on top of.

### Core Services

| Service | Component | Function |
|---|---|---|
| Identity | Keystone | Authentication and authorisation for all API calls |
| Compute | Nova | Lifecycle management of virtual machines |
| Networking | Neutron | Tenant networks, routers, floating IPs, security groups |
| Image | Glance | Registry for VM base images |
| Block Storage | Cinder | Persistent volumes for VMs |
| Dashboard | Horizon | Web UI for OpenStack management |

### 1.1 DevStack Installation

```bash
# Install DevStack on the controller node (Ubuntu 22.04)
sudo apt-get update && sudo apt-get install -y git python3-pip
git clone https://opendev.org/openstack/devstack
cd devstack
cp samples/local.conf local.conf
```

### 1.2 Configuration

```ini
# local.conf — controller node
[[local|localrc]]
ADMIN_PASSWORD=LabSecret123!
DATABASE_PASSWORD=LabSecret123!
RABBIT_PASSWORD=LabSecret123!
SERVICE_PASSWORD=LabSecret123!

# Core services to enable
ENABLED_SERVICES=n-api,n-cond,n-sch,n-crt,n-novnc,n-cpu
ENABLED_SERVICES+=,neutron,q-svc,q-agt,q-dhcp,q-l3,q-meta,q-metering
ENABLED_SERVICES+=,cinder,c-api,c-vol,c-sch,c-bak
ENABLED_SERVICES+=,keystone,glance,horizon

HOST_IP=192.168.10.10
FLAT_INTERFACE=eth1
FIXED_RANGE=10.0.1.0/24
FLOATING_RANGE=192.168.10.128/25
PUBLIC_NETWORK_GATEWAY=192.168.10.1

# Use local storage for Cinder volumes
VOLUME_BACKING_FILE_SIZE=50000M

LOG_COLOR=False
LOGFILE=/opt/stack/logs/stack.sh.log
```

```bash
# Run the installer (~20 minutes)
./stack.sh

# Source credentials
source openrc admin admin
```

### 1.3 Network Setup

```bash
# Create tenant network and subnet
openstack network create corp-net \
    --provider-network-type vxlan

openstack subnet create corp-subnet \
    --network corp-net \
    --subnet-range 10.0.1.0/24 \
    --gateway 10.0.1.1 \
    --dns-nameserver 8.8.8.8 \
    --allocation-pool start=10.0.1.100,end=10.0.1.200

# Create router, attach to external provider network, and connect subnet
openstack router create main-router
openstack router set main-router --external-gateway public
openstack router add subnet main-router corp-subnet

# Create security group for Kubernetes nodes
openstack security group create k8s-nodes
openstack security group rule create k8s-nodes --protocol tcp --dst-port 6443    # API server
openstack security group rule create k8s-nodes --protocol tcp --dst-port 2379:2380  # etcd
openstack security group rule create k8s-nodes --protocol tcp --dst-port 10250   # kubelet
openstack security group rule create k8s-nodes --protocol tcp --dst-port 30000:32767  # NodePort range
openstack security group rule create k8s-nodes --protocol tcp --dst-port 22      # SSH
openstack security group rule create k8s-nodes --protocol icmp                   # ping
```

### 1.4 Image and Compute

```bash
# Upload Ubuntu 22.04 cloud image to Glance
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

openstack image create "ubuntu-22.04" \
    --file jammy-server-cloudimg-amd64.img \
    --disk-format qcow2 \
    --container-format bare \
    --public \
    --property os_distro=ubuntu \
    --property os_version=22.04

# Create SSH keypair
openstack keypair create --public-key ~/.ssh/id_rsa.pub lab-key

# Create flavors for Kubernetes nodes
openstack flavor create k8s.control --vcpus 2 --ram 4096 --disk 40
openstack flavor create k8s.worker  --vcpus 2 --ram 2048 --disk 30

# Launch control plane + 3 worker VMs
for i in control worker-1 worker-2 worker-3; do
    FLAVOR=$([ "$i" = "control" ] && echo "k8s.control" || echo "k8s.worker")
    openstack server create \
        --flavor $FLAVOR \
        --image ubuntu-22.04 \
        --network corp-net \
        --security-group k8s-nodes \
        --key-name lab-key \
        k8s-$i
done

# Assign floating IPs to all nodes
for server in $(openstack server list -f value -c Name); do
    FIP=$(openstack floating ip create public -f value -c floating_ip_address)
    openstack server add floating ip $server $FIP
    echo "$server → $FIP"
done
```

---

## Phase 2 — Kubernetes Cluster on OpenStack

### Overview

Kubernetes is deployed on top of the OpenStack VMs. This is the same pattern used by enterprises running private cloud: OpenStack provides the IaaS layer, Kubernetes provides the container orchestration layer on top of it.

### 2.1 Node Preparation (All Nodes)

```bash
# Disable swap (required by Kubernetes)
swapoff -a
sed -i '/swap/d' /etc/fstab

# Load required kernel modules
cat >> /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Required sysctl settings
cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# Install containerd runtime
apt-get update && apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# Enable SystemdCgroup (critical for stability with kubeadm)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd && systemctl enable containerd

# Install kubeadm, kubelet, kubectl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
    | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
    https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
    > /etc/apt/sources.list.d/kubernetes.list

apt-get update && apt-get install -y kubeadm=1.29.0-1.1 kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
apt-mark hold kubeadm kubelet kubectl
```

### 2.2 Bootstrap the Control Plane

```bash
# On the control plane node only
kubeadm init \
    --apiserver-advertise-address=10.0.1.10 \
    --apiserver-cert-extra-sans=<floating-ip-of-control-plane> \
    --pod-network-cidr=10.244.0.0/16 \
    --service-cidr=10.96.0.0/12 \
    --node-name=k8s-control

# Configure kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI (VXLAN-based pod networking)
kubectl apply -f \
    https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Verify control plane is healthy
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
```

### 2.3 Join Worker Nodes

```bash
# The kubeadm init output provides this command — run on each worker
kubeadm join 10.0.1.10:6443 \
    --token <bootstrap-token> \
    --discovery-token-ca-cert-hash sha256:<ca-hash>

# If the token has expired, generate a new one on the control plane:
kubeadm token create --print-join-command

# Verify all nodes joined
kubectl get nodes -o wide
# NAME          STATUS   ROLES           AGE   VERSION   INTERNAL-IP
# k8s-control   Ready    control-plane   10m   v1.29.0   10.0.1.10
# k8s-worker-1  Ready    <none>          8m    v1.29.0   10.0.1.11
# k8s-worker-2  Ready    <none>          8m    v1.29.0   10.0.1.12
# k8s-worker-3  Ready    <none>          7m    v1.29.0   10.0.1.13
```

### 2.4 Deploy a Test Workload

```bash
# Deploy nginx to confirm cross-node scheduling and networking
kubectl create deployment test-nginx --image=nginx --replicas=3
kubectl expose deployment test-nginx --port=80 --type=ClusterIP

# Verify pods are distributed across nodes
kubectl get pods -o wide

# Verify cross-pod communication
kubectl run test-curl --image=curlimages/curl --rm -it --restart=Never -- \
    curl http://test-nginx

# Clean up
kubectl delete deployment test-nginx
kubectl delete service test-nginx
```

---

## Phase 3 — Docker Containerisation and Private Registry

### Overview

The application is a three-tier stack: a Python Flask REST API, a React frontend, and a PostgreSQL database. Each tier is containerised using multi-stage Docker builds to minimise image size and attack surface. A private Docker registry is deployed inside the cluster to keep all images internal — no dependency on Docker Hub.

### 3.1 Flask API Dockerfile (Multi-Stage Build)

```dockerfile
# Dockerfile — Flask API
# Stage 1: Install dependencies in a build environment
FROM python:3.11-slim AS builder
WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Stage 2: Lean runtime image — no build tools, no dev packages
FROM python:3.11-slim AS runtime
WORKDIR /app

# Copy only the installed packages from builder
COPY --from=builder /root/.local /root/.local
COPY --from=builder /usr/lib/x86_64-linux-gnu/libpq* /usr/lib/x86_64-linux-gnu/

COPY . .

ENV PATH=/root/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    FLASK_ENV=production

EXPOSE 5000

# Run as non-root user
RUN useradd -m appuser && chown -R appuser /app
USER appuser

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", \
     "--timeout", "30", "app:app"]
```

```bash
# Build — image size: 187MB vs 1.1GB single-stage baseline
docker build -t api:v1.0.0 .
docker images api  # confirm size reduction
```

### 3.2 Private Registry Deployment

```yaml
# registry-deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: infra
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
  namespace: infra
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: infra
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        env:
        - name: REGISTRY_STORAGE_DELETE_ENABLED
          value: "true"
        volumeMounts:
        - name: registry-data
          mountPath: /var/lib/registry
      volumes:
      - name: registry-data
        persistentVolumeClaim:
          claimName: registry-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: infra
spec:
  type: NodePort
  selector:
    app: registry
  ports:
  - port: 5000
    targetPort: 5000
    nodePort: 30500
```

```bash
kubectl apply -f registry-deployment.yaml

# Configure Docker daemon to trust the private registry (all nodes)
cat >> /etc/docker/daemon.json << 'EOF'
{
  "insecure-registries": ["192.168.10.10:30500"]
}
EOF
systemctl restart docker

# Build, tag, and push all three application images
docker build -t 192.168.10.10:30500/corp/api:v1.0.0       ./api
docker build -t 192.168.10.10:30500/corp/frontend:v1.0.0  ./frontend
docker build -t 192.168.10.10:30500/corp/db:v1.0.0        ./database

docker push 192.168.10.10:30500/corp/api:v1.0.0
docker push 192.168.10.10:30500/corp/frontend:v1.0.0
docker push 192.168.10.10:30500/corp/db:v1.0.0

# List all images in the registry
curl http://192.168.10.10:30500/v2/_catalog
# {"repositories":["corp/api","corp/frontend","corp/db"]}
```

### 3.3 Application Kubernetes Manifests

```yaml
# api-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: default
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0        # zero-downtime deployments
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: 192.168.10.10:30500/corp/api:v1.0.0
        ports:
        - containerPort: 5000
          name: http
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 20
```

---

## Phase 4 — CI/CD Pipeline

### Overview

The CI/CD pipeline automates the full path from a `git push` to a live rolling deployment in the cluster. It uses GitHub Actions as the orchestrator. The pipeline runs on every push to `main` and every pull request — PRs run tests only, merges to `main` run tests and deploy.

### 4.1 Pipeline Flow

```
git push to main
       │
       ▼
┌─────────────────┐
│  Checkout code  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Build Docker   │  ← tagged with git SHA (e.g. api:a1b2c3d4)
│  image          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Run unit tests │  ← pytest inside container
│  inside image   │  ← Pipeline STOPS here if tests fail
└────────┬────────┘
         │ (only if tests pass)
         ▼
┌─────────────────┐
│  Push image to  │  ← private registry
│  registry       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  kubectl set    │  ← rolling update, zero downtime
│  image          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐    ┌─────────────────┐
│  Rollout        │    │  Rollback on    │
│  success        │    │  failure        │
└─────────────────┘    └─────────────────┘
```

### 4.2 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: CI/CD — build, test, deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: 192.168.10.10:30500
  IMAGE_NAME: corp/api

jobs:
  build-and-test:
    name: Build and test
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set image tag from git SHA
      run: echo "IMAGE_TAG=${GITHUB_SHA::8}" >> $GITHUB_ENV

    - name: Build Docker image
      run: |
        docker build \
          -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} \
          -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
          --cache-from ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
          --build-arg BUILDKIT_INLINE_CACHE=1 \
          .

    - name: Run unit tests inside container
      run: |
        docker run --rm \
          --network host \
          -e FLASK_ENV=testing \
          -e DATABASE_URL=sqlite:///test.db \
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} \
          pytest tests/ -v --tb=short --junitxml=test-results.xml

    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results
        path: test-results.xml

  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set image tag
      run: echo "IMAGE_TAG=${GITHUB_SHA::8}" >> $GITHUB_ENV

    - name: Push image to registry
      run: |
        docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
        docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

    - name: Configure kubectl
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBECONFIG }}" > ~/.kube/config
        chmod 600 ~/.kube/config

    - name: Deploy rolling update
      run: |
        kubectl set image deployment/api-deployment \
          api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}

        kubectl annotate deployment/api-deployment \
          kubernetes.io/change-cause="Deploy ${{ env.IMAGE_TAG }} from commit ${{ github.sha }}"

    - name: Wait for rollout to complete
      run: |
        kubectl rollout status deployment/api-deployment \
          --timeout=120s

    - name: Verify deployment health
      run: |
        READY=$(kubectl get deployment api-deployment \
          -o jsonpath='{.status.readyReplicas}')
        DESIRED=$(kubectl get deployment api-deployment \
          -o jsonpath='{.spec.replicas}')
        if [ "$READY" != "$DESIRED" ]; then
          echo "Deployment unhealthy: $READY/$DESIRED replicas ready"
          exit 1
        fi
        echo "Deployment healthy: $READY/$DESIRED replicas ready"

    - name: Rollback on failure
      if: failure()
      run: |
        echo "Deployment failed — rolling back"
        kubectl rollout undo deployment/api-deployment
        kubectl rollout status deployment/api-deployment --timeout=60s
```

### 4.3 Rollout Management

```bash
# View rollout history
kubectl rollout history deployment/api-deployment

# Rollback to previous version manually
kubectl rollout undo deployment/api-deployment

# Rollback to a specific revision
kubectl rollout undo deployment/api-deployment --to-revision=3

# Pause a rollout mid-way (canary-style)
kubectl rollout pause deployment/api-deployment
# ... inspect pods ...
kubectl rollout resume deployment/api-deployment
```

---

## Phase 5 — Network Load Balancing

### Overview

In public clouds, a `type: LoadBalancer` Kubernetes service automatically provisions a cloud load balancer. In a private cloud, this does not exist by default. MetalLB fills this gap — it watches for LoadBalancer services and assigns IPs from a configured pool using Layer 2 ARP advertisement. The Nginx Ingress Controller then sits behind MetalLB and handles HTTP/HTTPS routing and TLS termination.

### 5.1 MetalLB Deployment

```bash
# Install MetalLB
kubectl apply -f \
    https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
    --for=condition=ready pod \
    --selector=app=metallb \
    --timeout=90s
```

```yaml
# metallb-config.yaml — IP pool mapped to OpenStack floating IP range
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: corp-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.200-192.168.10.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: corp-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - corp-pool
  interfaces:
  - eth0
```

```bash
kubectl apply -f metallb-config.yaml
```

### 5.2 Nginx Ingress Controller

```bash
# Install Nginx Ingress Controller
kubectl apply -f \
    https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

# Verify MetalLB assigned an IP to the ingress service
kubectl get svc -n ingress-nginx
# NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
# ingress-nginx-controller             LoadBalancer   10.96.45.123   192.168.10.200   80:32080/TCP,443:32443/TCP
```

### 5.3 TLS Certificate and Ingress Resource

```bash
# Create self-signed TLS certificate for the lab
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout tls.key \
    -out tls.crt \
    -subj "/CN=corp.local/O=Corp" \
    -addext "subjectAltName=DNS:api.corp.local,DNS:app.corp.local"

# Store as Kubernetes secret
kubectl create secret tls corp-tls \
    --cert=tls.crt \
    --key=tls.key
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "15"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.corp.local
    - app.corp.local
    secretName: corp-tls
  rules:
  - host: api.corp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 5000
  - host: app.corp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 3000
```

```bash
kubectl apply -f ingress.yaml

# Test routing
curl -k -H "Host: api.corp.local" https://192.168.10.200/health
curl -k -H "Host: app.corp.local" https://192.168.10.200/
```

---

## Phase 6 — Prometheus Metrics Collection

### Overview

Prometheus is a pull-based metrics system — it scrapes HTTP endpoints (`/metrics`) on a configurable interval. The `kube-prometheus-stack` Helm chart installs the full observability stack in one command: Prometheus, Alertmanager, kube-state-metrics (Kubernetes object state), and node-exporter (host-level hardware metrics).

### 6.1 Helm Deployment

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus \
    prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    --set prometheus.prometheusSpec.retention=15d \
    --set prometheus.prometheusSpec.scrapeInterval=15s \
    --set prometheus.prometheusSpec.evaluationInterval=15s \
    --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=standard \
    --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi \
    --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=5Gi \
    --set grafana.adminPassword=LabGrafana123! \
    --set grafana.persistence.enabled=true \
    --set grafana.persistence.size=5Gi

# Verify all pods are running
kubectl get pods -n monitoring
```

### 6.2 Flask Application Instrumentation

```python
# app.py — custom Prometheus metrics

from prometheus_client import (
    Counter, Histogram, Gauge,
    generate_latest, CONTENT_TYPE_LATEST
)
from flask import Flask, request, g, Response
import time

app = Flask(__name__)

# Metric definitions
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP request count',
    ['method', 'endpoint', 'http_status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency in seconds',
    ['endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    'http_requests_in_progress',
    'Number of HTTP requests currently in progress'
)

DB_QUERY_DURATION = Histogram(
    'db_query_duration_seconds',
    'Database query duration in seconds',
    ['query_type']
)

@app.before_request
def before_request():
    g.start_time = time.time()
    ACTIVE_REQUESTS.inc()

@app.after_request
def after_request(response):
    latency = time.time() - g.start_time
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.path,
        http_status=response.status_code
    ).inc()
    REQUEST_LATENCY.labels(endpoint=request.path).observe(latency)
    ACTIVE_REQUESTS.dec()
    return response

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

@app.route('/health')
def health():
    return {'status': 'ok', 'version': '1.0.0'}, 200
```

### 6.3 ServiceMonitor — Prometheus Scrape Config

```yaml
# service-monitor.yaml
# Tells the Prometheus Operator to scrape the Flask app's /metrics endpoint
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus   # must match Prometheus operator selector
spec:
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      app: api
  endpoints:
  - port: http
    path: /metrics
    interval: 15s
    scrapeTimeout: 10s
```

```bash
kubectl apply -f service-monitor.yaml

# Verify Prometheus is scraping the target
# Port-forward Prometheus UI
kubectl port-forward svc/kube-prometheus-kube-prome-prometheus 9090:9090 -n monitoring

# Open http://localhost:9090/targets — confirm api endpoint shows State=UP
```

### 6.4 Key Metrics and PromQL Reference

```promql
# Total request rate (per second, 5-minute window)
rate(http_requests_total[5m])

# Error rate — percentage of 5xx responses
sum(rate(http_requests_total{http_status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# p50, p95, p99 response latency
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# Pod CPU usage as % of limit
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)
  / sum(kube_pod_container_resource_limits{resource="cpu",namespace="default"}) by (pod) * 100

# Pod memory usage
container_memory_working_set_bytes{namespace="default"}

# Node disk usage
(node_filesystem_size_bytes - node_filesystem_avail_bytes)
  / node_filesystem_size_bytes * 100

# Kubernetes deployment availability
kube_deployment_status_replicas_available
  / kube_deployment_spec_replicas * 100
```

---

## Phase 7 — Grafana Dashboards and Alerting

### Overview

Grafana connects to Prometheus as its data source and provides the visualisation layer. The kube-prometheus-stack Helm chart preconfigures this connection automatically. The observability strategy follows the **four golden signals**: latency, traffic, errors, and saturation.

### 7.1 Access and Data Source

```bash
# Port-forward Grafana
kubectl port-forward svc/kube-prometheus-grafana 3000:80 -n monitoring
# Open http://localhost:3000
# Default credentials: admin / LabGrafana123!

# The Prometheus data source is pre-configured by Helm at:
# http://kube-prometheus-kube-prome-prometheus.monitoring:9090
```

### 7.2 Imported Community Dashboards

| Dashboard Name | Grafana ID | Purpose |
|---|---|---|
| Kubernetes cluster overview | 15661 | Nodes, pods, deployments, resource usage |
| Node Exporter full | 1860 | Per-node CPU, memory, disk, network |
| Kubernetes pods | 6781 | Pod-level metrics and restarts |
| Nginx Ingress Controller | 9614 | Request rate, latency, error rate at the ingress |

### 7.3 Custom Application Dashboard (Four Golden Signals)

All panels were built using the PromQL queries below and arranged into a single dashboard named **Application — four golden signals**.

```promql
# Panel 1: Traffic — requests per second
sum(rate(http_requests_total[5m])) by (endpoint)

# Panel 2: Errors — 5xx rate as percentage
(sum(rate(http_requests_total{http_status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))) * 100

# Panel 3: Latency — p95 response time in milliseconds
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
) * 1000

# Panel 4: Saturation — CPU utilisation across all API pods
sum(rate(container_cpu_usage_seconds_total{pod=~"api-deployment.*"}[5m]))
  / sum(kube_pod_container_resource_limits{resource="cpu", pod=~"api-deployment.*"}) * 100

# Panel 5: Active requests (current in-flight)
http_requests_in_progress

# Panel 6: Deployment rollout frequency (CI/CD signal)
changes(kube_deployment_status_observed_generation{deployment="api-deployment"}[24h])
```

### 7.4 Prometheus Alert Rules

```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: application-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus
spec:
  groups:
  - name: application.rules
    interval: 30s
    rules:

    - alert: HighErrorRate
      expr: |
        (sum(rate(http_requests_total{http_status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))) * 100 > 5
      for: 2m
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "HTTP error rate above 5%"
        description: "Error rate is {{ $value | humanize }}% — investigate immediately."

    - alert: HighP95Latency
      expr: |
        histogram_quantile(0.95,
          sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
        ) > 1.0
      for: 5m
      labels:
        severity: warning
        team: backend
      annotations:
        summary: "p95 latency above 1 second"
        description: "p95 latency is {{ $value | humanizeDuration }}."

    - alert: PodNotReady
      expr: kube_pod_status_ready{condition="false", namespace="default"} == 1
      for: 1m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Pod {{ $labels.pod }} is not ready"
        description: "Pod has been unready for more than 1 minute."

    - alert: DeploymentReplicasMismatch
      expr: |
        kube_deployment_spec_replicas{namespace="default"}
          != kube_deployment_status_replicas_available{namespace="default"}
      for: 5m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Deployment {{ $labels.deployment }} has fewer replicas than desired"

    - alert: NodeDiskPressure
      expr: |
        (node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_avail_bytes{mountpoint="/"})
          / node_filesystem_size_bytes{mountpoint="/"} * 100 > 85
      for: 10m
      labels:
        severity: warning
        team: platform
      annotations:
        summary: "Node {{ $labels.instance }} disk usage above 85%"
        description: "Disk is {{ $value | humanize }}% full."
```

### 7.5 Alertmanager Routing

```yaml
# alertmanager-config.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: slack-routing
  namespace: monitoring
spec:
  route:
    groupBy: ['alertname', 'severity']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    receiver: slack-critical
    routes:
    - matchers:
      - name: severity
        value: critical
      receiver: slack-critical
    - matchers:
      - name: severity
        value: warning
      receiver: slack-warning

  receivers:
  - name: slack-critical
    slackConfigs:
    - apiURL:
        key: slack-webhook-url
        name: alertmanager-secrets
      channel: '#alerts-critical'
      title: 'CRITICAL: {{ .CommonAnnotations.summary }}'
      text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      sendResolved: true

  - name: slack-warning
    slackConfigs:
    - apiURL:
        key: slack-webhook-url
        name: alertmanager-secrets
      channel: '#alerts-warning'
      title: 'WARNING: {{ .CommonAnnotations.summary }}'
      text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      sendResolved: true
```

```bash
kubectl apply -f alertmanager-config.yaml

# Test alert routing by triggering a synthetic alert
curl -X POST http://localhost:9090/api/v1/admin/tsdb/delete_series \
    -H "Content-Type: application/json"

# Or manually fire a test alert via Alertmanager API
curl -X POST http://localhost:9093/api/v1/alerts \
    -H "Content-Type: application/json" \
    -d '[{"labels":{"alertname":"TestAlert","severity":"warning"},"annotations":{"summary":"Test alert from lab"}}]'
```

---

## Outcomes Summary

| Component | What was built | Key metric |
|---|---|---|
| OpenStack | 3-node private cloud (controller + 2 compute) | 4 VMs provisioned, tenant network with floating IPs |
| Kubernetes | 4-node cluster (1 control plane + 3 workers) | All nodes Ready, cross-node networking verified |
| Docker | Multi-stage builds for 3-tier application | 83% image size reduction vs single-stage baseline |
| Private registry | Internal Docker registry on PVC-backed storage | All images served internally, no Docker Hub dependency |
| CI/CD | GitHub Actions pipeline (build → test → deploy) | Average pipeline: 3m 42s, automatic rollback validated |
| MetalLB + Ingress | LoadBalancer IP allocation + TLS routing | 192.168.10.200 allocated, HTTPS on both hostnames |
| Prometheus | Metrics scraping 47 targets, custom app metrics | 15-day retention on 20Gi persistent volume |
| Grafana | 4 dashboards + custom four golden signals | 5 alert rules → Alertmanager → Slack |

### Key Design Decisions

**Why OpenStack over a public cloud?** The lab specifically targets private cloud architecture — the skills required to run OpenStack on bare metal are different from clicking buttons in the AWS console. OpenStack requires understanding the actual networking, compute, and storage primitives, which transfers directly to understanding what public cloud services are abstracting.

**Why MetalLB over NodePort?** NodePort exposes a high port (30000–32767) on every node, which is not production-appropriate. MetalLB makes the cluster behave like a real cloud environment where `type: LoadBalancer` just works.

**Why multi-stage Docker builds?** Security and efficiency. The builder stage contains compilers, build tools, and development headers — none of which should be in a production image. The runtime stage contains only what the application needs to run, reducing attack surface and image pull time.

**Why the four golden signals model?** The four golden signals (latency, traffic, errors, saturation) give you the minimum set of metrics needed to understand whether a service is healthy. They are the foundation of Google's SRE approach and the basis of most production observability strategies.
