# msc-de1-distributed-systems-docker-k8s

Containerization, security hardening, publication and local orchestration of the [UBC Flask Sample App](https://github.com/ubc/flask-sample-app).

## 1. Objective and architecture

A minimal Flask REST API (routes `/`, `/items`, `/items/{id}`) with no Dockerfile in the original repository. This repository adds: Dockerfile → Docker Hub image → local `kind` cluster (1 control-plane + 2 workers) → Kubernetes Deployment (2 replicas, probes, security hardening, NetworkPolicy).

## 2. Original starter repository

https://github.com/ubc/flask-sample-app

## 3. Prerequisites

- Python 3.12, Docker Desktop, `kind`, `kubectl`
- A Docker Hub account

## 4. Run the original application (without Docker)

```bash
python -m venv venv
venv\Scripts\Activate.ps1        # Mac/Linux: source venv/bin/activate
pip install -r requirements.txt
python -m unittest discover tests -v
python run.py
# -> http://localhost:5000
```

## 5. Build and run the Docker image

```powershell
docker build -t naomicindy/msc-de1-flask-app:1.0.0 .
docker run -d --name flask-app -p 5000:5000 naomicindy/msc-de1-flask-app:1.0.0
curl.exe http://localhost:5000/
docker logs flask-app
docker stop flask-app
docker rm flask-app
```

## 6. Run with Docker Compose

```powershell
docker compose up --build
curl.exe http://localhost:5000/
docker compose down
```

## 7. Docker Hub

Published image: https://hub.docker.com/r/naomicindy/msc-de1-flask-app
Published tags: `1.0.0`, `latest`
Tag used for the final Kubernetes deployment: **1.0.0**

```powershell
docker push naomicindy/msc-de1-flask-app:1.0.0
docker push naomicindy/msc-de1-flask-app:latest
```

## 8. Create the kind cluster

```bash
kind create cluster --config kind/kind-config.yaml
kubectl get nodes
```

## 9. Deploy to Kubernetes

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/network-policy.yaml

kubectl get pods -n msc-de1-project -o wide
```

## 10. Access and test the application

```bash
kubectl port-forward -n msc-de1-project svc/flask-app 5000:5000
curl.exe http://localhost:5000/
curl.exe http://localhost:5000/items
```

## 11. Distributed systems demonstrations

```bash
# A. Replication and service discovery
kubectl get pods -n msc-de1-project -o wide
# -> 2 pods Running, scheduled on different worker nodes

# B. Self-healing
kubectl delete pod <pod-name> -n msc-de1-project
kubectl get pods -n msc-de1-project -w
# -> the Deployment automatically recreates a replacement pod

# C. Scaling
kubectl scale deployment flask-app --replicas=3 -n msc-de1-project
kubectl get pods -n msc-de1-project
kubectl scale deployment flask-app --replicas=2 -n msc-de1-project

# D. Rolling update and rollback
kubectl set image deployment/flask-app flask-app=naomicindy/msc-de1-flask-app:1.0.1 -n msc-de1-project
kubectl rollout status deployment/flask-app -n msc-de1-project
kubectl rollout history deployment/flask-app -n msc-de1-project
kubectl rollout undo deployment/flask-app -n msc-de1-project
kubectl rollout status deployment/flask-app -n msc-de1-project
```

## 12. Clean up the local cluster

```bash
kind delete cluster --name msc-de1-cluster
```

## Security decisions and known limitations

- The application runs as a non-root user (UID 1000) inside the Docker image (`USER 1000:1000`) and in Kubernetes (`runAsNonRoot`, `runAsUser: 1000`), with `allowPrivilegeEscalation: false`, `capabilities.drop: [ALL]`, `seccompProfile: RuntimeDefault` and `readOnlyRootFilesystem: true` (an `emptyDir` volume on `/tmp` compensates for the few temporary writes gunicorn may need).
- Debian system packages are patched at build time (`apt-get upgrade`), along with `pip` `setuptools`/`wheel`, and the base image was upgraded to `python:3.12-slim` — reducing findings from 131 to 32 detected by `docker scout` (0 CRITICAL remaining). Full detail in `security/vulnerability-scan.txt`.
- gunicorn (added to `requirements.txt`) replaces Flask's development server for production-oriented execution.
- **Known and documented limitation**: item storage is in-memory (no database), so each worker/pod has its own separate memory — an item added on one instance is not visible from another. This applies both to gunicorn workers within a single container and to the 2 Kubernetes replicas. Not fixed, as it is a design constraint of the original application, outside this project's scope — documented here rather than silently worked around.
- **kind's default CNI (kindnetd) does not enforce NetworkPolicy** at the data-plane level — the policy in `k8s/network-policy.yaml` documents the intended network segmentation, but actual enforcement would require a compatible CNI such as Calico.