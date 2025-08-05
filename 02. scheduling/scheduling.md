# Kubernetes Scheduling

## DaemonSets

### What is a DaemonSet?
A **DaemonSet** ensures that a copy of a pod runs on all (or specific) nodes in the cluster.

### DaemonSet Use Cases
- **Node monitoring**: Monitoring agents on every node
- **Log collection**: Log collectors like Fluentd or Filebeat
- **Storage daemons**: Distributed storage solutions
- **Network components**: CNI plugins, kube-proxy

### DaemonSet vs Deployment

| Feature | DaemonSet | Deployment |
|---------|-----------|------------|
| **Replica Logic** | One per node | Total replica count |
| **Scheduling** | Node-based | Cluster-wide |
| **Use Case** | Node-level services | Application services |
| **Updates** | Rolling update per node | Rolling update by replica |

### DaemonSet Definition
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
  labels:
    app: monitoring-daemon
spec:
  selector:
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
    spec:
      containers:
      - name: monitoring-agent
        image: monitoring-agent
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

### DaemonSet Scheduling Evolution
- **Before v1.12**: Used nodeName to bypass scheduler
- **v1.12+**: Uses NodeAffinity with default scheduler

### DaemonSet Commands
```bash
# Create DaemonSet
kubectl create -f daemon-set-definition.yaml

# View DaemonSets
kubectl get daemonsets

# Describe DaemonSet
kubectl describe daemonset monitoring-daemon

# View DaemonSet pods
kubectl get pods -l app=monitoring-agent -o wide
```


## Static Pods

### What are Static Pods?
**Static Pods** are pods managed directly by the kubelet on a specific node, without requiring the Kubernetes API server or any controllers like ReplicaSets or Deployments.

### Key Characteristics
- **Managed by kubelet only**: No involvement of kube-apiserver or controllers
- **Node-specific**: Created and managed on the node where kubelet runs
- **Automatic restart**: kubelet automatically restarts static pods if they fail
- **Mirror pods**: API server creates read-only mirror pods for visibility

### Static Pod Configuration

#### Method 1: Pod Manifest Path
```bash
# Configure kubelet with --pod-manifest-path
ExecStart=/usr/local/bin/kubelet \
--container-runtime=remote \
--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
--pod-manifest-path=/etc/kubernetes/manifests \
--kubeconfig=/var/lib/kubelet/kubeconfig \
--network-plugin=cni \
--register-node=true \
--v=2
```

#### Method 2: Kubelet Configuration File
```yaml
# kubeconfig.yaml
staticPodPath: /etc/kubernetes/manifests
```

```bash
# Use config file
ExecStart=/usr/local/bin/kubelet \
--config=kubeconfig.yaml \
--kubeconfig=/var/lib/kubelet/kubeconfig \
--network-plugin=cni \
--register-node=true \
--v=2
```

### Static Pod Use Cases

#### Control Plane Components
Static pods are commonly used to run control plane components on master nodes:
- **kube-apiserver**: API server pod
- **kube-controller-manager**: Controller manager pod  
- **kube-scheduler**: Scheduler pod
- **etcd**: Database pod

### Static Pod Lifecycle
1. **Creation**: Place YAML file in `/etc/kubernetes/manifests/`
2. **Detection**: kubelet monitors the directory and detects new files
3. **Pod Creation**: kubelet creates the pod directly
4. **Mirror Pod**: API server creates a read-only mirror for visibility
5. **Management**: kubelet handles restarts, updates, and deletion

### Static Pod Example
```yaml
# /etc/kubernetes/manifests/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    app: static-web
spec:
  containers:
  - name: web
    image: nginx
    ports:
    - containerPort: 80
```

### Static Pods vs DaemonSets

| Feature | Static Pods | DaemonSets |
|---------|-------------|------------|
| **Creation** | kubelet directly | kube-apiserver (DaemonSet Controller) |
| **Management** | Node-level | Cluster-level |
| **Use Case** | Control plane components | Monitoring agents, log collectors |
| **Scheduler** | Ignored by scheduler | Uses scheduler |
| **API Visibility** | Mirror pods only | Full API objects |

### Viewing Static Pods
```bash
# View static pods (will show mirror pods)
kubectl get pods -n kube-system

# Example output showing static pods with node names
NAME                             READY   STATUS    RESTARTS   AGE
etcd-master                      1/1     Running   0          15m
kube-apiserver-master            1/1     Running   0          15m
kube-controller-manager-master   1/1     Running   0          15m
kube-scheduler-master            1/1     Running   0          15m
```

## Taints and Tolerations

### What are Taints and Tolerations?
**Taints** are applied to nodes to repel pods, while **tolerations** are applied to pods to allow them to be scheduled on tainted nodes.

### Taint Effects

| Effect | Description |
|--------|-------------|
| **NoSchedule** | Prevents new pods from being scheduled |
| **PreferNoSchedule** | Avoids scheduling if possible |
| **NoExecute** | Evicts existing pods and prevents new ones |

### Applying Taints
```bash
# Taint a node
kubectl taint nodes node1 app=myapp:NoSchedule

# Remove a taint
kubectl taint nodes node1 app=myapp:NoSchedule-

# View node taints
kubectl describe node node1 | grep Taint
```

### Pod Tolerations
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "myapp"
    effect: "NoSchedule"
```

### Master Node Taints
Master nodes are typically tainted to prevent regular workloads:
```bash
kubectl describe node master | grep Taint
# Output: Taints: node-role.kubernetes.io/master:NoSchedule
```

### Toleration Operators
- **Equal**: Key/value must match exactly
- **Exists**: Only key must match (ignores value)

## Node Selectors and Node Affinity

### What is Node Selection?
Node selection mechanisms allow you to constrain pods to run on specific nodes based on node labels.

### Node Selectors (Simple Method)

#### Label Nodes
```bash
# Label a node
kubectl label nodes node-1 size=Large

# View node labels
kubectl get nodes --show-labels
```

#### Use Node Selector
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  nodeSelector:
    size: Large
```

#### Node Selector Limitations
- **Simple equality only**: Cannot handle complex logic
- **No OR conditions**: Cannot select "Large OR Medium"
- **No negation**: Cannot specify "NOT Small"

### Node Affinity (Advanced Method)

#### Required Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: data-processor
    image: data-processor
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
```

#### Preferred Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: size
            operator: In
            values:
            - Large
```

#### Node Affinity Operators
- **In**: Value must be in the list
- **NotIn**: Value must not be in the list
- **Exists**: Key must exist (ignores values)
- **DoesNotExist**: Key must not exist
- **Gt**: Greater than (numeric values)
- **Lt**: Less than (numeric values)

### Node Affinity Types

| Type | DuringScheduling | DuringExecution |
|------|------------------|-----------------|
| **requiredDuringSchedulingIgnoredDuringExecution** | Required | Ignored |
| **preferredDuringSchedulingIgnoredDuringExecution** | Preferred | Ignored |
| **requiredDuringSchedulingRequiredDuringExecution** | Required | Required (Planned) |

#### Lifecycle Phases
- **DuringScheduling**: Rules applied when pod is first scheduled
- **DuringExecution**: Rules applied to running pods when node labels change

## Manual Scheduling

### What is Manual Scheduling?
**Manual Scheduling** is the process of assigning pods to specific nodes without using the Kubernetes scheduler.

### When Manual Scheduling is Needed
- **No scheduler available**: Scheduler is down or not configured
- **Custom placement logic**: Specific node requirements
- **Troubleshooting**: Testing pod placement
- **Static pods**: Control plane components

### Manual Scheduling Methods

#### Method 1: NodeName in Pod Spec
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  nodeName: node02  # Direct node assignment
```

#### Method 2: Binding API (Post-Creation)
```yaml
# Pod without nodeName (will remain Pending)
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

```yaml
# Binding object
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node02
```

```bash
# Create binding via API
curl --header "Content-Type:application/json" \
     --request POST \
     --data '{"apiVersion":"v1", "kind": "Binding", "metadata": {"name": "nginx"}, "target": {"apiVersion": "v1", "kind": "Node", "name": "node02"}}' \
     http://$SERVER/api/v1/namespaces/default/pods/nginx/binding/
```

## Multiple Schedulers

### What are Multiple Schedulers?
Kubernetes supports running **multiple schedulers** simultaneously, allowing different scheduling policies for different types of workloads.

### Use Cases for Multiple Schedulers
- **Specialized workloads**: GPU, high-memory, or compute-intensive tasks
- **Custom scheduling logic**: Business-specific placement rules
- **Multi-tenancy**: Different schedulers for different teams
- **Performance optimization**: Specialized algorithms for specific use cases

### Deploying Custom Scheduler

#### Binary Deployment
```bash
# Download scheduler binary
wget https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler

# Configure custom scheduler service
# my-custom-scheduler.service
ExecStart=/usr/local/bin/kube-scheduler \
--config=/etc/kubernetes/config/kube-scheduler.yaml \
--scheduler-name=my-custom-scheduler
```

#### kubeadm Deployment
```yaml
# my-custom-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --scheduler-name=my-custom-scheduler
    - --lock-object-name=my-custom-scheduler
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
    name: my-custom-scheduler
```

### Using Custom Scheduler
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  schedulerName: my-custom-scheduler  # Specify custom scheduler
```

### Monitoring Multiple Schedulers
```bash
# View all schedulers
kubectl get pods --namespace=kube-system

# View scheduler events
kubectl get events

# View scheduler logs
kubectl logs my-custom-scheduler --namespace=kube-system
```

## Network Policies

### What are Network Policies?
**Network Policies** are Kubernetes resources that control network traffic between pods, providing micro-segmentation at the application level.

### Network Policy Components
- **podSelector**: Selects pods the policy applies to
- **policyTypes**: Ingress, Egress, or both
- **ingress**: Rules for incoming traffic
- **egress**: Rules for outgoing traffic

### Basic Network Policy Example
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    ports:
    - protocol: TCP
      port: 3306
```

### Network Policy Rules

#### Ingress Rules (Incoming Traffic)
```yaml
spec:
  ingress:
  - from:
    # Pod selector - allow from specific pods
    - podSelector:
        matchLabels:
          app: frontend
    # Namespace selector - allow from specific namespaces
    - namespaceSelector:
        matchLabels:
          name: production
    # IP block - allow from specific IP ranges
    - ipBlock:
        cidr: 192.168.1.0/24
        except:
        - 192.168.1.5/32
    ports:
    - protocol: TCP
      port: 80
```

#### Egress Rules (Outgoing Traffic)
```yaml
spec:
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5432
```

## Ingress

### What is Ingress?
**Ingress** is a Kubernetes resource that manages external access to services, typically HTTP/HTTPS, providing load balancing, SSL termination, and name-based virtual hosting.

### Ingress Components
- **Ingress Controller**: Implements ingress functionality (nginx, traefik, etc.)
- **Ingress Resource**: Configuration defining routing rules

### Ingress vs Services

| Feature | Service (NodePort/LoadBalancer) | Ingress |
|---------|-------------------------------|---------|
| **Layer** | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| **Routing** | Simple port-based | Path/host-based routing |
| **SSL/TLS** | External configuration | Built-in SSL termination |
| **Cost** | Multiple load balancers | Single entry point |

### Ingress Controller Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      containers:
      - name: nginx-ingress-controller
        image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
        args:
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-configuration
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
```

### Ingress Resource Examples

#### Simple Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
```

#### Path-based Routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

#### Host-based Routing
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear-watch
spec:
  rules:
  - host: wear.my-online-store.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 80
  - host: watch.my-online-store.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 80
```

## Key Commands

### Static Pods
```bash
# Check static pod directory
ls /etc/kubernetes/manifests/

# View kubelet configuration
systemctl status kubelet
```

### Taints and Tolerations
```bash
# Apply taint
kubectl taint nodes node1 key=value:NoSchedule

# Remove taint
kubectl taint nodes node1 key=value:NoSchedule-

# Check node taints
kubectl describe node node1 | grep Taint
```

### Node Affinity
```bash
# Label nodes
kubectl label nodes node1 size=Large

# View node labels
kubectl get nodes --show-labels
```

### DaemonSets
```bash
# Create DaemonSet
kubectl create -f daemonset.yaml

# View DaemonSets
kubectl get daemonsets

# View DaemonSet pods
kubectl get pods -l app=monitoring-agent -o wide
```

### Multiple Schedulers
```bash
# View schedulers
kubectl get pods -n kube-system

# View scheduler events
kubectl get events

# View scheduler logs
kubectl logs scheduler-name -n kube-system
```

### Network Policies
```bash
# Create network policy
kubectl create -f network-policy.yaml

# View network policies
kubectl get networkpolicies

# Describe network policy
kubectl describe networkpolicy policy-name
```

### Ingress
```bash
# Create ingress
kubectl create -f ingress.yaml

# View ingress
kubectl get ingress

# Describe ingress
kubectl describe ingress ingress-name
```