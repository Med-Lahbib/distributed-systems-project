# Distributed Systems Project – Flask, Docker and Kubernetes

This repository contains my individual Distributed Systems project for MSc DE1.

The project is based on the provided Flask sample application and demonstrates how an existing application can be containerized, secured, published to Docker Hub, and deployed as multiple replicas on a local Kubernetes cluster.

## Architecture

The application is deployed using the following architecture:

```text
Client
  |
  v
Kubernetes Service
  |
  +-------------------+
  |                   |
  v                   v
Flask Pod 1        Flask Pod 2
Worker Node 1      Worker Node 2
```

The local Kubernetes environment is created with kind and contains:

- 1 control-plane node
- 2 worker nodes
- 2 application replicas in the final deployment

## Application

The Flask application provides the following REST endpoints:

| Method | Endpoint | Description |
| --- | --- | --- |
| GET | `/` | Returns a greeting |
| GET | `/items` | Returns all items |
| GET | `/items/{item_id}` | Returns a specific item |
| POST | `/items` | Adds an item |

The application uses an in-memory list for item storage. Therefore, data is not persistent and is not shared between replicas. The purpose of this project is the containerization and distributed deployment of the supplied application rather than redesigning its storage architecture.

## Project Structure

```text
.
├── app/                    # Flask application
├── tests/                  # Application tests
├── kind/
│   └── kind-config.yaml    # 3-node kind cluster
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── network-policy.yaml
├── security/
│   ├── vulnerability-scan.txt
│   └── sbom.spdx
├── evidence/               # Verification evidence
├── Dockerfile
├── .dockerignore
├── compose.yaml
├── requirements.txt
├── requirements-dev.txt
├── run.py
└── README.md
```

## Local Application Setup

Create and activate a Python virtual environment.

### Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install the development dependencies:

```powershell
pip install -r requirements-dev.txt
```

Run the tests:

```powershell
pytest -v
```

The current test suite contains four tests covering the application routes.

Run the application locally:

```powershell
python run.py
```

The application is available at:

```text
http://localhost:5000
```

## Docker

### Build

```powershell
docker build -t msc-de1-flask-app:1.0.0 .
```

### Run

```powershell
docker run -d --name flask-app -p 5000:5000 msc-de1-flask-app:1.0.0
```

Test the application:

```powershell
curl.exe http://localhost:5000/
```

### Docker Compose

The application can also be built and started with:

```powershell
docker compose up -d --build
```

Check its status:

```powershell
docker compose ps
```

Stop it with:

```powershell
docker compose down
```

The Compose configuration applies additional runtime restrictions including dropped Linux capabilities, no-new-privileges, and a read-only root filesystem.

## Container Security

The Docker image uses `python:3.12-slim` and applies several security measures:

- Dedicated non-root user with UID 10001
- Minimal slim Python base image
- Dependency installation without pip cache
- `.dockerignore` to reduce build context
- Only the application port is exposed
- Exec-form container command
- Container health check
- No embedded secrets
- Dropped Linux capabilities in Compose
- `no-new-privileges`
- Read-only root filesystem in Compose

The container user can be verified with:

```powershell
docker exec flask-app id
```

## Vulnerability Scanning

Docker Scout was used to scan image version `1.0.0`.

The final recorded scan reported:

```text
CRITICAL      0
HIGH          1
MEDIUM        6
LOW          26
UNSPECIFIED   1
```

The remaining HIGH finding is associated with the Debian `zlib` package. At the time of scanning, Docker Scout reported that no fixed version was available.

An attempt was made to upgrade the bundled pip version. A rescan produced a worse security profile because additional HIGH findings appeared in packages introduced or detected after the change. The change was therefore reverted and the better-scoring image was retained.

The complete scan output is stored in:

```text
security/vulnerability-scan.txt
```

## SBOM

An SPDX 2.3 Software Bill of Materials was generated with Docker Scout:

```powershell
docker scout sbom msc-de1-flask-app:1.0.0 --format spdx --output security\sbom.spdx
```

The result is stored in:

```text
security/sbom.spdx
```

## Docker Hub

The public image is available from Docker Hub as:

```text
m0h4med/msc-de1-flask-app:1.0.0
m0h4med/msc-de1-flask-app:latest
```

Pull the versioned image with:

```powershell
docker pull m0h4med/msc-de1-flask-app:1.0.0
```

Run the published image with:

```powershell
docker run -d --name flask-app -p 5000:5000 m0h4med/msc-de1-flask-app:1.0.0
```

## Kubernetes with kind

### Create the cluster

The kind configuration creates one control-plane and two worker nodes.

```powershell
kind create cluster --name msc-de1 --config kind\kind-config.yaml
```

Verify the nodes:

```powershell
kubectl get nodes
```

### Deploy the application

Apply the namespace first:

```powershell
kubectl apply -f k8s\namespace.yaml
```

Then deploy the application resources:

```powershell
kubectl apply -f k8s\deployment.yaml
kubectl apply -f k8s\service.yaml
kubectl apply -f k8s\network-policy.yaml
```

Verify the Pods:

```powershell
kubectl get pods -n msc-de1-project -o wide
```

The final configuration runs two replicas. The topology spread constraint encourages the replicas to be scheduled across different worker nodes.

### Access the application

Forward the Kubernetes Service to the local machine:

```powershell
kubectl port-forward -n msc-de1-project service/flask-app-service 8080:80
```

Then access:

```text
http://localhost:8080/
```

## Kubernetes Security

The Deployment applies:

- `runAsNonRoot: true`
- UID 10001 and GID 10001
- `allowPrivilegeEscalation: false`
- All Linux capabilities dropped
- `seccompProfile: RuntimeDefault`
- Read-only root filesystem
- CPU and memory requests
- CPU and memory limits
- Readiness probe
- Liveness probe
- Controlled rolling-update strategy

A NetworkPolicy is also included to restrict ingress to the application Pods on TCP port 5000 from Pods in the same namespace.

NetworkPolicy enforcement depends on the CNI implementation. The default kind networking environment may not enforce NetworkPolicy rules, so the manifest demonstrates the intended policy while this limitation should be considered when reproducing the project.

## Distributed Behaviour Demonstrations

### Self-Healing

A running Pod was manually deleted:

```powershell
kubectl delete pod <pod-name> -n msc-de1-project
```

The Deployment automatically created a replacement Pod to restore the desired replica count of two.

### Scaling

The application was scaled from two replicas to three:

```powershell
kubectl scale deployment flask-app --replicas=3 -n msc-de1-project
```

It was then returned to the final state:

```powershell
kubectl scale deployment flask-app --replicas=2 -n msc-de1-project
```

### Rolling Update

A second image version was created and published:

```text
m0h4med/msc-de1-flask-app:1.1.0
```

The Deployment was updated from `1.0.0` to `1.1.0`, which returned a visible version string from the root endpoint.

The rollout was monitored with:

```powershell
kubectl rollout status deployment/flask-app -n msc-de1-project
```

### Rollback

The Deployment was rolled back with:

```powershell
kubectl rollout undo deployment/flask-app -n msc-de1-project
```

The final running image was verified as:

```text
m0h4med/msc-de1-flask-app:1.0.0
```

## Evidence

Command output collected during verification is stored under `evidence/`, including:

- Kubernetes node topology
- Running Pods and worker placement
- Kubernetes resources
- Deployment rollout history

Security evidence is stored separately under `security/`.

## Cleanup

Delete the local Kubernetes cluster with:

```powershell
kind delete cluster --name msc-de1
```

Stop the Docker Compose environment with:

```powershell
docker compose down
```

## Repository and Image

GitHub:

```text
https://github.com/Med-Lahbib/distributed-systems-project
```

Docker Hub:

```text
https://hub.docker.com/r/m0h4med/msc-de1-flask-app
```

## Acknowledgment

The Flask application was provided as starter code for the project. The Docker containerization, security configuration, Docker Compose setup, vulnerability analysis, SBOM generation, Docker Hub publication, kind cluster configuration, Kubernetes deployment, and distributed-system demonstrations were completed as part of this individual project.