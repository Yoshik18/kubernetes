# Definitions & Core Concepts

## Container Runtime Fundamentals

### What is a Container Runtime?
A **Container Runtime** is the software responsible for running containers on a host system. It manages the complete container lifecycle from creation to destruction.

### Docker vs Containerd
- **Docker**: Complete container platform with CLI, API, image management, volumes, networking, and security features
- **Containerd**: Lightweight, industry-standard container runtime focused on core container operations
- **CRI (Container Runtime Interface)**: Kubernetes standard for communicating with container runtimes
- **OCI (Open Container Initiative)**: Industry standards for container formats and runtimes

### CLI Tools Comparison

| Tool | Purpose | Target Runtime | Use Case |
|------|---------|----------------|----------|
| **docker** | General container management | Docker daemon | Development, general use |
| **ctr** | Basic containerd control | Containerd | Limited debugging |
| **nerdctl** | Docker-compatible CLI | Containerd | Docker replacement |
| **crictl** | CRI debugging | Any CRI runtime | Kubernetes troubleshooting |

### Key Commands
```bash
# Docker
docker run --name redis redis:alpine
docker ps -a

# Containerd (ctr)
ctr images pull docker.io/library/redis:alpine
ctr run docker.io/library/redis:alpine redis

# Nerdctl (Docker-like for containerd)
nerdctl run --name redis redis:alpine
nerdctl ps -a

# CRI debugging
crictl pull busybox
crictl images
crictl ps -a
crictl pods
```

## Pod Fundamentals

### What is a Pod?
A **Pod** is the smallest deployable unit in Kubernetes that represents one or more containers sharing the same network and storage.

### Pod Characteristics
- **Shared Network**: All containers in a pod share the same IP address and port space
- **Shared Storage**: Containers can share volumes mounted in the pod
- **Lifecycle Coupling**: All containers start and stop together
- **Ephemeral**: Pods are disposable and replaceable

### Multi-Container Pod Patterns
- **Sidecar**: Helper container alongside main application (logging, monitoring)
- **Ambassador**: Proxy container for external connections
- **Adapter**: Container that standardizes output from main container

### Pod Creation
```bash
# Imperative
kubectl run nginx --image=nginx

# Declarative
kubectl apply -f pod-definition.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: nginx-container
    image: nginx
```

## Controllers and Workload Management

### What is a Controller?
A **Controller** is a Kubernetes component that watches the cluster state and makes changes to move the current state toward the desired state.

### ReplicationController vs ReplicaSet

| Feature | ReplicationController | ReplicaSet |
|---------|----------------------|------------|
| **API Version** | v1 | apps/v1 |
| **Selector** | Equality-based only | Set-based (matchLabels) |
| **Status** | Legacy | Current standard |
| **Used by** | Older deployments | Deployments |

### What is a Deployment?
A **Deployment** provides declarative updates for pods and ReplicaSets, enabling rolling updates, rollbacks, and scaling.

### Deployment Strategies
- **RollingUpdate** (Default): Gradually replace old pods with new ones
- **Recreate**: Kill all old pods before creating new ones

### Key Commands
```bash
# Create deployment
kubectl create deployment nginx --image=nginx

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Update image
kubectl set image deployment/nginx nginx=nginx:1.18

# Rollback
kubectl rollout undo deployment/nginx

# Check rollout status
kubectl rollout status deployment/nginx

### Example Deployment YAML with RollingUpdate Strategy
```
```yaml
# API version for Deployment resource (apps/v1 is the current stable version)
apiVersion: apps/v1
# Resource type - Deployment manages ReplicaSets and provides declarative updates
kind: Deployment
metadata:
  # Unique name for this deployment
  name: nginx-deployment
  # Labels for organizing and selecting resources
  labels:
    app: nginx
spec:
  # Number of replicas (pods) to maintain
  replicas: 3
  
  # Deployment strategy configuration
  strategy:
    # RollingUpdate: Gradually replace old pods with new ones (default)
    # Alternative: Recreate (kills all old pods before creating new ones)
    type: RollingUpdate
    rollingUpdate:
      # maxSurge: Maximum number of pods above desired replicas during update
      # 25% means with 3 replicas, up to 1 additional pod (3 * 0.25 = 0.75, rounded up to 1)
      # Can be absolute number (e.g., 2) or percentage (e.g., 25%)
      maxSurge: 25%
      
      # maxUnavailable: Maximum number of pods unavailable during update
      # 25% means with 3 replicas, minimum 2 pods available (3 * 0.25 = 0.75, rounded down to 0)
      # Can be absolute number (e.g., 1) or percentage (e.g., 25%)
      maxUnavailable: 25%
  
  # Pod selector - determines which pods this deployment manages
  selector:
    matchLabels:
      app: nginx
  
  # Pod template - defines the desired state for pods created by this deployment
  template:
    metadata:
      # Labels that pods created by this deployment will have
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        # Container image to use
        image: nginx:1.18
        
        # Ports that the container exposes
        ports:
        - containerPort: 80
        
        # Resource requirements and limits
        resources:
          # Minimum resources the container needs to run
          requests:
            cpu: "100m"        # 0.1 CPU cores (100 millicores)
            memory: "128Mi"    # 128 megabytes
          # Maximum resources the container can use
          limits:
            cpu: "500m"        # 0.5 CPU cores (500 millicores)
            memory: "512Mi"    # 512 megabytes
        
        # Health check to determine if container is alive
        livenessProbe:
          httpGet:
            path: /            # HTTP path to check
            port: 80          # Port to check
          initialDelaySeconds: 30  # Wait 30s before first probe
          periodSeconds: 10        # Check every 10 seconds
        
        # Health check to determine if container is ready to receive traffic
        readinessProbe:
          httpGet:
            path: /            # HTTP path to check
            port: 80          # Port to check
          initialDelaySeconds: 5   # Wait 5s before first probe
          periodSeconds: 5         # Check every 5 seconds
```

### RollingUpdate Strategy Parameters

| Parameter | Description | Default | Example |
|-----------|-------------|---------|---------|
| **maxSurge** | Maximum number of pods above desired replicas during update. Controls how many extra pods can be created during rollout. | 25% | `maxSurge: 2` or `maxSurge: 25%` |
| **maxUnavailable** | Maximum number of pods unavailable during update. Ensures minimum availability during rollout. | 25% | `maxUnavailable: 1` or `maxUnavailable: 25%` |

**Example Calculation with 3 replicas:**
- **maxSurge: 25%** = Up to 1 extra pod (3 × 0.25 = 0.75, rounded up to 1)
- **maxUnavailable: 25%** = Minimum 2 pods available (3 × 0.25 = 0.75, rounded down to 0 unavailable)
- **Result**: During update, you can have 1-4 pods total, with minimum 2 always available

## Namespaces and Resource Isolation

### What is a Namespace?
A **Namespace** provides virtual clustering within a physical Kubernetes cluster, enabling resource isolation and multi-tenancy.

### Default Namespaces
- **default**: Default namespace for objects with no specified namespace
- **kube-system**: Namespace for Kubernetes system components
- **kube-public**: Publicly readable namespace for cluster information
- **kube-node-lease**: Namespace for node heartbeat objects

### DNS in Namespaces
```bash
# Same namespace
service-name

# Different namespace
service-name.namespace-name.svc.cluster.local
```

### Resource Quotas
**Resource Quota** limits the total resource consumption within a namespace.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

### LimitRange
**LimitRange** sets minimum and maximum resource constraints for individual objects in a namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

## Services and Networking

### What is a Service?
A **Service** is an abstraction that defines a logical set of pods and provides stable network access to them.

### Service Types

#### ClusterIP (Default)
- **Purpose**: Internal cluster communication only
- **IP Range**: Cluster-internal IP addresses
- **Access**: Only accessible from within the cluster

#### NodePort
- **Purpose**: External access via node ports
- **Port Range**: 30000-32767
- **Access**: `:`

#### LoadBalancer
- **Purpose**: Cloud provider load balancer
- **Requirements**: Cloud provider integration
- **Access**: External IP provided by cloud provider

### Service Port Definitions

| Port Type | Definition | Example |
|-----------|------------|---------|
| **targetPort** | Port on the pod where traffic is sent | 80 (nginx container port) |
| **port** | Port on the service where it accepts traffic | 80 (service port) |
| **nodePort** | Port on the node for external access | 30008 (external access) |

### Service Example
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - targetPort: 80    # Pod port
    port: 80          # Service port  
    nodePort: 30008   # Node port
```

### What is a Node IP?
**Node IP** is the IP address of the physical or virtual machine that runs Kubernetes worker nodes. External traffic reaches services through NodeIP:NodePort combination.

## Configuration Management

### What are ConfigMaps?
**ConfigMaps** store non-confidential configuration data in key-value pairs that can be consumed by pods as environment variables, command-line arguments, or configuration files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "postgres"
  database.port: "5432"
  app.properties: |
    debug=true
    log.level=info
```

### What are Secrets?
**Secrets** store sensitive information like passwords, tokens, and keys in base64-encoded format with additional security measures.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: cG9zdGdyZXM=  # postgres (base64)
  password: cGFzc3dvcmQ=  # password (base64)
```

### Using ConfigMaps and Secrets
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:latest
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.host
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

## Kubernetes Control Plane Components

### What is etcd?
**etcd** is a distributed, reliable key-value store that stores all Kubernetes cluster data and state information.

#### Data Stored in etcd
- Nodes, pods, deployments, services
- ConfigMaps, secrets
- Network policies, RBAC rules
- All cluster configuration

### What is the API Server?
**kube-apiserver** is the central management component that exposes the Kubernetes API and handles all cluster communications.

#### API Server Responsibilities
- **Authentication**: Verify user identity
- **Authorization**: Check user permissions (RBAC)
- **Admission Control**: Validate and modify requests
- **Persistence**: Store data in etcd

### What is the Scheduler?
**kube-scheduler** assigns pods to nodes based on resource requirements, constraints, and policies.

#### Scheduling Process
1. **Filtering**: Remove unsuitable nodes
2. **Scoring**: Rank remaining nodes
3. **Binding**: Assign pod to highest-scoring node

### What is the Controller Manager?
**kube-controller-manager** runs various controllers that regulate cluster state.

#### Key Controllers
- **Node Controller**: Monitors node health
- **Replication Controller**: Maintains desired pod replicas
- **Deployment Controller**: Manages rolling updates
- **Service Account Controller**: Creates default service accounts

### What is Kubelet?
**kubelet** is the node agent that manages pod lifecycle on worker nodes.

#### Kubelet Responsibilities
- **Pod Management**: Create, update, delete pods
- **Health Monitoring**: Report node and pod status
- **Resource Reporting**: Send resource usage to API server
- **Container Runtime Interface**: Communicate with container runtime

### What is Kube-proxy?
**kube-proxy** maintains network rules and enables service communication within the cluster.

#### Kube-proxy Functions
- **Service Discovery**: Route traffic to service endpoints
- **Load Balancing**: Distribute traffic across pod replicas
- **Network Rules**: Manage iptables or IPVS rules

## Resource Management

### What are Resource Requests?
**Resource Requests** specify the minimum amount of CPU and memory a container needs to run.

### What are Resource Limits?
**Resource Limits** specify the maximum amount of CPU and memory a container can consume.

```yaml
resources:
  requests:
    cpu: "100m"        # 0.1 CPU core minimum
    memory: "128Mi"    # 128 megabytes minimum
  limits:
    cpu: "500m"        # 0.5 CPU core maximum  
    memory: "512Mi"    # 512 megabytes maximum
```

### Resource Units
- **CPU**: Measured in millicores (1000m = 1 core)
- **Memory**: Measured in bytes (Ki, Mi, Gi, Ti)

## Best Practices

### Declarative vs Imperative

#### Declarative (Recommended)
- **Definition**: Describe desired state in YAML files
- **Command**: `kubectl apply -f file.yaml`
- **Benefits**: Version control, reproducibility, collaboration

#### Imperative
- **Definition**: Direct commands to modify cluster state
- **Commands**: `kubectl run`, `kubectl create`, `kubectl scale`
- **Use Cases**: Quick testing, debugging, learning

### Essential kubectl Commands
```bash
# Resource management
kubectl get 
kubectl describe  
kubectl create -f 
kubectl apply -f 
kubectl delete  

# Debugging
kubectl logs 
kubectl exec -it  -- /bin/bash
kubectl port-forward  8080:80

# Cluster information
kubectl cluster-info
kubectl get nodes
kubectl get events
```
