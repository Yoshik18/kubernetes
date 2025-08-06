# Application Lifecycle Management

## Rolling Updates and Rollbacks

### What is a Rollout?
A **Rollout** is the process of deploying a new version of an application in Kubernetes. Each rollout creates a new **revision** that can be tracked and managed.

### Rollout Revisions
- **Revision 1**: Initial deployment (e.g., nginx:1.7.0)
- **Revision 2**: Updated deployment (e.g., nginx:1.7.1)
- **Revision History**: Kubernetes maintains a history of all revisions

### Deployment Strategies

| Strategy | Description | Downtime | Use Case |
|----------|-------------|----------|----------|
| **Recreate** | Kill all old pods, then create new ones | Yes | Development, non-critical apps |
| **RollingUpdate** | Gradually replace old pods with new ones | No | Production applications (default) |

### Rolling Update Process
1. **Create new ReplicaSet** with updated image
2. **Scale up new ReplicaSet** gradually
3. **Scale down old ReplicaSet** simultaneously
4. **Maintain desired replica count** throughout process

### Rollout Commands
```bash
# Check rollout status
kubectl rollout status deployment/myapp-deployment

# View rollout history
kubectl rollout history deployment/myapp-deployment

# Trigger rollout with record
kubectl apply -f deployment-definition.yml --record

# Update image and trigger rollout
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1
```

### Example Rollout Output
```bash
kubectl rollout status deployment/myapp-deployment
# Output:
Waiting for rollout to finish: 0 of 10 updated replicas are available...
Waiting for rollout to finish: 1 of 10 updated replicas are available...
...
deployment "myapp-deployment" successfully rolled out
```

### Rollback Operations
```bash
# Rollback to previous revision
kubectl rollout undo deployment/myapp-deployment

# View ReplicaSets during rollback
kubectl get replicasets

# Before rollback:
# myapp-deployment-67c749c58c    0    0    0    22m  (old)
# myapp-deployment-7d57dbdb8d    5    5    5    20m  (current)

# After rollback:
# myapp-deployment-67c749c58c    5    5    5    22m  (restored)
# myapp-deployment-7d57dbdb8d    0    0    0    20m  (scaled down)
```

### Deployment Definition
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      type: front-end
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.7.1
```

## Commands and Arguments

### Docker Container Behavior
- **Default Command**: Containers run default command specified in Dockerfile
- **Override Command**: Can override default command at runtime
- **Process Lifecycle**: Container exits when main process completes

### Docker Command Examples
```bash
# Default ubuntu behavior (exits immediately)
docker run ubuntu
docker ps -a
# STATUS: Exited (0) 41 seconds ago

# Override with custom command
docker run ubuntu sleep 5
```

### Dockerfile Instructions

#### CMD Instruction
**CMD** specifies the default command to run when container starts:

```dockerfile
FROM Ubuntu
CMD sleep 5
```

```dockerfile
# Shell form
CMD command param1

# Exec form (preferred)
CMD ["command", "param1"]
```

#### ENTRYPOINT Instruction
**ENTRYPOINT** specifies the command that always executes:

```dockerfile
FROM Ubuntu
ENTRYPOINT ["sleep"]
```

```bash
# Usage
docker run ubuntu-sleeper 10
# Command at startup: sleep 10
```

#### ENTRYPOINT + CMD Combination
```dockerfile
FROM Ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```

**Benefits**:
- **Default behavior**: `docker run ubuntu-sleeper` → `sleep 5`
- **Override parameter**: `docker run ubuntu-sleeper 10` → `sleep 10`
- **Override entrypoint**: `docker run --entrypoint sleep2.0 ubuntu-sleeper 10`

### Kubernetes Commands and Arguments

#### Pod Command Override
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]  # Overrides ENTRYPOINT
    args: ["10"]           # Overrides CMD
```

#### Docker vs Kubernetes Mapping

| Docker | Kubernetes | Purpose |
|--------|------------|---------|
| **ENTRYPOINT** | **command** | Main executable |
| **CMD** | **args** | Default arguments |

## Environment Variables

### What are Environment Variables?
**Environment Variables** are key-value pairs that provide configuration data to applications at runtime.

### Flask Application Example
```python
# app.py
import os
from flask import Flask

app = Flask(__name__)
color = os.environ.get('APP_COLOR')  # Read from environment

@app.route("/")
def main():
    print(color)
    return render_template('hello.html', color=color)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port="8080")
```

### Setting Environment Variables

#### Docker
```bash
# Set environment variable
docker run -e APP_COLOR=blue simple-webapp-color
docker run -e APP_COLOR=green simple-webapp-color
docker run -e APP_COLOR=pink simple-webapp-color
```

#### Kubernetes Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    ports:
    - containerPort: 8080
    env:
    - name: APP_COLOR
      value: pink
```

### Environment Variable Value Types

#### 1. Plain Key-Value
```yaml
env:
- name: APP_COLOR
  value: pink
```

#### 2. ConfigMap Reference
```yaml
env:
- name: APP_COLOR
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_COLOR
```

#### 3. Secret Reference
```yaml
env:
- name: APP_COLOR
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: APP_COLOR
```

## ConfigMaps

### What are ConfigMaps?
**ConfigMaps** store configuration data in key-value pairs that can be consumed by pods as environment variables, command-line arguments, or configuration files.

### ConfigMap Use Cases
- **Application settings**: Database URLs, feature flags
- **Configuration files**: Application config, logging config
- **Non-sensitive data**: Any configuration that's not secret

### Creating ConfigMaps

#### Imperative Method
```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=prod

# From file
kubectl create configmap app-config \
  --from-file=app_config.properties
```

#### Declarative Method
```yaml
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
  app.properties: |
    debug=true
    log.level=info
```

```bash
kubectl create -f config-map.yaml
```

### Viewing ConfigMaps
```bash
# List ConfigMaps
kubectl get configmaps

# Detailed view
kubectl describe configmap app-config
```

### Using ConfigMaps in Pods

#### Environment Variables (All Keys)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    envFrom:
    - configMapRef:
        name: app-config
```

#### Single Environment Variable
```yaml
env:
- name: APP_COLOR
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_COLOR
```

#### Volume Mount
```yaml
volumes:
- name: app-config-volume
  configMap:
    name: app-config
```

## Secrets

### What are Secrets?
**Secrets** store sensitive information like passwords, tokens, and keys in base64-encoded format with additional security measures.

### Secret vs ConfigMap

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Data Type** | Non-sensitive configuration | Sensitive information |
| **Encoding** | Plain text | Base64 encoded |
| **Security** | Standard | Enhanced (encryption at rest) |
| **Use Cases** | App config, feature flags | Passwords, tokens, certificates |

### Creating Secrets

#### Imperative Method
```bash
kubectl create secret generic app-secret \
  --from-literal=DB_Host=mysql \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=paswrd
```

#### Declarative Method
```yaml
# secret-data.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=     # mysql (base64)
  DB_User: cm9vdA==     # root (base64)
  DB_Password: cGFzd3Jk # paswrd (base64)
```

### Base64 Encoding/Decoding
```bash
# Encode
echo -n 'mysql' | base64
# Output: bXlzcWw=

# Decode
echo -n 'bXlzcWw=' | base64 --decode
# Output: mysql
```

### Viewing Secrets
```bash
# List secrets
kubectl get secrets

# Describe secret (shows size, not values)
kubectl describe secret app-secret

# View secret YAML (shows base64 values)
kubectl get secret app-secret -o yaml
```

### Using Secrets in Pods

#### Environment Variables (All Keys)
```yaml
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    envFrom:
    - secretRef:
        name: app-secret
```

#### Single Environment Variable
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: DB_Password
```

#### Volume Mount
```yaml
volumes:
- name: app-secret-volume
  secret:
    secretName: app-secret
```

### Secrets as Volumes
When mounted as volumes, each key becomes a file:
```bash
# Inside container
ls /opt/app-secret-volumes
# Output: DB_Host  DB_Password  DB_User

cat /opt/app-secret-volumes/DB_Password
# Output: paswrd
```

## Multi-Container Pods

### What are Multi-Container Pods?
**Multi-Container Pods** contain multiple containers that share the same lifecycle, network, and storage.

### Multi-Container Patterns

#### Sidecar Pattern
- **Main container**: Application
- **Sidecar container**: Supporting functionality (logging, monitoring)
- **Example**: Web server + log agent

#### Ambassador Pattern
- **Main container**: Application
- **Ambassador container**: Proxy for external connections
- **Example**: App + database proxy

#### Adapter Pattern
- **Main container**: Application
- **Adapter container**: Transforms/standardizes output
- **Example**: App + log format converter

### Shared Resources in Multi-Container Pods

#### Lifecycle
- **Start together**: All containers start simultaneously
- **Stop together**: Pod termination affects all containers
- **Restart policy**: Applied to entire pod

#### Network
- **Shared IP**: All containers share same IP address
- **Localhost communication**: Containers communicate via localhost
- **Port sharing**: Containers must use different ports

#### Storage
- **Shared volumes**: Containers can share mounted volumes
- **Data exchange**: File-based communication between containers

### Multi-Container Pod Definition
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 8080
  - name: log-agent
    image: log-agent
```

### Use Cases for Multi-Container Pods
- **Logging**: Main app + log collection sidecar
- **Monitoring**: Application + metrics collection
- **Service mesh**: App + proxy (Istio, Linkerd)
- **Data synchronization**: App + data sync agent
- **Security scanning**: App + security agent

## Application Lifecycle Commands Summary

### Rolling Updates and Rollbacks
```bash
# Deployment management
kubectl create -f deployment-definition.yml
kubectl get deployments
kubectl apply -f deployment-definition.yml
kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1

# Rollout operations
kubectl rollout status deployment/myapp-deployment
kubectl rollout history deployment/myapp-deployment
kubectl rollout undo deployment/myapp-deployment
```

### ConfigMaps and Secrets
```bash
# ConfigMap operations
kubectl create configmap app-config --from-literal=key=value
kubectl get configmaps
kubectl describe configmap app-config

# Secret operations
kubectl create secret generic app-secret --from-literal=key=value
kubectl get secrets
kubectl describe secret app-secret
```

### Pod Management
```bash
# Create and manage pods
kubectl create -f pod-definition.yaml
kubectl get pods
kubectl describe pod 
kubectl logs  -c 
kubectl exec -it  -c  -- /bin/bash
```