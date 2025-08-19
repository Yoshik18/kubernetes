# Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA)

## Horizontal Pod Autoscaler (HPA)

### What is HPA?
**Horizontal Pod Autoscaler (HPA)** automatically scales the number of pods in a deployment, replica set, or stateful set based on observed CPU utilization, memory usage, or custom metrics.[1][2]

### Key Concepts
- **Horizontal Scaling**: Adding/removing pod replicas
- **Metrics-based**: Decisions based on resource utilization
- **Target Utilization**: Desired resource usage percentage
- **Scale Up/Down**: Automatic pod count adjustment
- **Control Loop**: Continuous monitoring and adjustment

### HPA Architecture
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Metrics API   │◄───│  HPA Controller │────►│   Deployment    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       ▼
         ▼                       ▼              ┌─────────────────┐
┌─────────────────┐    ┌─────────────────┐     │      Pods       │
│ Metrics Server  │    │  Scale Decision │     └─────────────────┘
└─────────────────┘    └─────────────────┘
```

## HPA Prerequisites

### 1. Metrics Server
```bash
# Check if metrics server is running
kubectl get pods -n kube-system | grep metrics-server

# If not installed, deploy metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify metrics are available
kubectl top nodes
kubectl top pods
```

### 2. Resource Requests
```yaml
# Pods MUST have resource requests defined
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:latest
        resources:
          requests:
            cpu: 100m        # Required for CPU-based HPA
            memory: 128Mi    # Required for memory-based HPA
          limits:
            cpu: 200m
            memory: 256Mi
```

## Creating HPA

### 1. Imperative Creation
```bash
# CPU-based HPA (simple)
kubectl autoscale deployment web-app --cpu-percent=70 --min=2 --max=10

# View HPA
kubectl get hpa
kubectl describe hpa web-app
```

### 2. Declarative Creation (CPU-based)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

### 3. Memory-based HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 4. Multi-metric HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: multi-metric-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## HPA Behavior Configuration

### Scaling Policies
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: advanced-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
      policies:
      - type: Percent
        value: 10                      # Scale down by max 10% of current replicas
        periodSeconds: 60              # Per minute
      - type: Pods
        value: 2                       # Or max 2 pods per minute
        periodSeconds: 60
      selectPolicy: Min                # Use the policy that removes fewer pods
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 1 minute before scaling up
      policies:
      - type: Percent
        value: 50                      # Scale up by max 50% of current replicas
        periodSeconds: 60              # Per minute
      - type: Pods
        value: 4                       # Or max 4 pods per minute
        periodSeconds: 60
      selectPolicy: Max                # Use the policy that adds more pods
```

### Behavior Parameters
- **stabilizationWindowSeconds**: Time to wait before making scaling decisions
- **selectPolicy**: 
  - `Max`: Use policy that allows highest scaling
  - `Min`: Use policy that allows lowest scaling
  - `Disabled`: Disable scaling in this direction

## Custom Metrics HPA

### Prerequisites for Custom Metrics
```bash
# Install custom metrics API server (example with Prometheus)
# 1. Deploy Prometheus
kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml

# 2. Deploy Prometheus Adapter
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter
```

### Custom Metrics HPA Example
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # CPU metric
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  # Custom metric: requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  # External metric: SQS queue length
  - type: External
    external:
      metric:
        name: sqs_queue_length
        selector:
          matchLabels:
            queue: worker-queue
      target:
        type: Value
        value: "30"
```

### Object Metrics
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: object-metrics-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: main-route
      target:
        type: Value
        value: "2k"
```

## Vertical Pod Autoscaler (VPA)

### What is VPA?
**Vertical Pod Autoscaler (VPA)** automatically adjusts CPU and memory requests for containers in pods to optimize resource utilization and improve cluster efficiency.[1][2]

### VPA vs HPA
| Feature | HPA | VPA |
|---------|-----|-----|
| **Scaling Type** | Horizontal (replica count) | Vertical (resource requests) |
| **Pod Recreation** | No | Yes (in most cases) |
| **Use Case** | Handle load spikes | Optimize resource allocation |
| **Compatibility** | Works with HPA | Can conflict with HPA |

### VPA Components
- **VPA Recommender**: Analyzes resource usage and provides recommendations
- **VPA Updater**: Applies recommendations by evicting pods
- **VPA Admission Controller**: Sets resource requests on new pods

## Installing VPA

### VPA Installation
```bash
# Clone VPA repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/

# Install VPA
./hack/vpa-install.sh

# Verify installation
kubectl get pods -n kube-system | grep vpa
```

### Check VPA Components
```bash
# VPA components should be running
kubectl get pods -n kube-system
# vpa-admission-controller-xxx
# vpa-recommender-xxx
# vpa-updater-xxx
```

## VPA Modes

### 1. Off Mode (Recommendation Only)
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-off-mode
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Off"  # Only provide recommendations
```

### 2. Initial Mode (Set on Pod Creation)
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-initial-mode
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Initial"  # Set requests only on pod creation
```

### 3. Auto Mode (Automatic Updates)
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-auto-mode
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"  # Automatically update running pods
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1000m
        memory: 500Mi
      controlledResources: ["cpu", "memory"]
      controlledValues: "RequestsAndLimits"
```

## VPA Resource Policies

### Comprehensive Resource Policy
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: advanced-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 1Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: "RequestsAndLimits"
      mode: Auto
    - containerName: sidecar
      mode: "Off"  # Don't adjust sidecar container
```

### Resource Policy Options
- **controlledResources**: `["cpu"]`, `["memory"]`, `["cpu", "memory"]`
- **controlledValues**: 
  - `RequestsOnly`: Only adjust requests
  - `RequestsAndLimits`: Adjust both requests and limits
- **mode**: `Auto`, `Off`

## VPA Recommendations

### Viewing VPA Recommendations
```bash
# Get VPA status and recommendations
kubectl describe vpa vpa-name

# View VPA recommendations in detail
kubectl get vpa vpa-name -o yaml
```

### Sample VPA Output
```yaml
status:
  conditions:
  - status: "True"
    type: RecommendationProvided
  recommendation:
    containerRecommendations:
    - containerName: web-app
      lowerBound:
        cpu: 50m
        memory: 262144k
      target:
        cpu: 250m
        memory: 262144k
      uncappedTarget:
        cpu: 250m
        memory: 262144k
      upperBound:
        cpu: 366m
        memory: 262144k
```

## HPA + VPA Combination

### Safe Combination (Different Metrics)
```yaml
# HPA for horizontal scaling based on custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
---
# VPA for vertical scaling based on resource usage
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: resource-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"
```

### Avoiding Conflicts
- **Don't use HPA with CPU/Memory + VPA together**
- **Use VPA in "Off" mode for recommendations only when using HPA**
- **Use different metrics for HPA (custom) and VPA (resources)**

## Monitoring and Observability

### HPA Monitoring
```bash
# Watch HPA in real-time
kubectl get hpa --watch

# Check HPA events
kubectl describe hpa web-app-hpa

# View scaling events
kubectl get events --sort-by=.metadata.creationTimestamp
```

### VPA Monitoring
```bash
# Check VPA recommendations
kubectl describe vpa web-app-vpa

# Monitor VPA events
kubectl get events | grep VerticalPodAutoscaler

# Check resource usage
kubectl top pods
```

### Metrics to Monitor
```bash
# CPU and memory usage
kubectl top nodes
kubectl top pods

# Pod resource requests vs usage
kubectl describe pod pod-name | grep -A 10 "Requests:"
```

## Troubleshooting

### Common HPA Issues

#### 1. HPA Not Scaling
```bash
# Check metrics server
kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system deployment/metrics-server

# Check resource requests
kubectl describe deployment web-app | grep -A 5 "Requests:"

# Check HPA status
kubectl describe hpa web-app-hpa
```

#### 2. Unable to get Resource Metrics
```bash
# Verify metrics API
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

# Check metrics server logs
kubectl logs -n kube-system -l k8s-app=metrics-server

# Test metrics availability
kubectl top nodes
kubectl top pods
```

#### 3. Thrashing (Rapid Scaling)
```yaml
# Add stabilization windows
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
  scaleUp:
    stabilizationWindowSeconds: 60
```

### Common VPA Issues

#### 1. VPA Not Updating Pods
```bash
# Check VPA components
kubectl get pods -n kube-system | grep vpa

# Check VPA status
kubectl describe vpa web-app-vpa

# Verify admission controller
kubectl logs -n kube-system -l app=vpa-admission-controller
```

#### 2. Pods Being Evicted Too Often
```yaml
# Use "Initial" mode instead of "Auto"
updatePolicy:
  updateMode: "Initial"

# Or use "Off" mode for recommendations only
updatePolicy:
  updateMode: "Off"
```

#### 3. Resource Recommendations Too High/Low
```yaml
# Set resource limits in VPA
resourcePolicy:
  containerPolicies:
  - containerName: web-app
    minAllowed:
      cpu: 100m
      memory: 128Mi
    maxAllowed:
      cpu: 1000m
      memory: 512Mi
```

## Real-World Examples

### E-commerce Application Scaling
```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecommerce
  template:
    metadata:
      labels:
        app: ecommerce
    spec:
      containers:
      - name: web-server
        image: nginx:latest
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
      - name: app-server
        image: myapp:latest
        resources:
          requests:
            cpu: 300m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
---
# HPA for handling traffic spikes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "200"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 60
---
# VPA for resource optimization (recommendations only)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ecommerce-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-app
  updatePolicy:
    updateMode: "Off"  # Only recommendations to avoid conflicts with HPA
```

### Batch Processing Application
```yaml
# For batch processing, VPA is more appropriate than HPA
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: batch-processor-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: batch-processor
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: processor
      minAllowed:
        cpu: 500m
        memory: 1Gi
      maxAllowed:
        cpu: 4000m
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: "RequestsAndLimits"
```

## CKA Exam Scenarios

### Scenario 1: Create HPA for Deployment
```bash
# Given: deployment 'web-app' with resource requests
# Task: Create HPA to scale between 2-8 replicas at 80% CPU

# Solution:
kubectl autoscale deployment web-app --cpu-percent=80 --min=2 --max=8

# Verify:
kubectl get hpa
kubectl describe hpa web-app
```

### Scenario 2: Multi-metric HPA with Custom Behavior
```yaml
# Task: Create HPA with CPU and memory metrics, custom scaling behavior
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: advanced-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 3
        periodSeconds: 60
```

### Scenario 3: VPA for Resource Optimization
```bash
# Task: Create VPA to optimize resources for 'database' deployment

# Solution:
kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: database-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: database
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: mysql
      minAllowed:
        cpu: 500m
        memory: 1Gi
      maxAllowed:
        cpu: 2000m
        memory: 4Gi
EOF

# Check recommendations:
kubectl describe vpa database-vpa
```

## Best Practices

### HPA Best Practices
1. **Always set resource requests** on containers
2. **Use stabilization windows** to prevent thrashing
3. **Monitor scaling events** and adjust policies
4. **Test scaling behavior** under load
5. **Use multiple metrics** for better decision making
6. **Set appropriate min/max replicas**

### VPA Best Practices
1. **Start with "Off" mode** to analyze recommendations
2. **Set resource limits** to prevent runaway resource allocation
3. **Use "Initial" mode** for predictable workloads
4. **Monitor eviction frequency** in "Auto" mode
5. **Avoid VPA + HPA conflicts** on same resources
6. **Review recommendations regularly**

### General Autoscaling Best Practices
1. **Monitor resource utilization** patterns
2. **Set up proper monitoring** and alerting
3. **Test autoscaling** in non-production environments
4. **Document scaling policies** and decisions
5. **Regular review** of autoscaling configurations
6. **Capacity planning** for maximum scale scenarios

## Commands Summary

### HPA Commands
```bash
# Create HPA
kubectl autoscale deployment NAME --cpu-percent=70 --min=2 --max=10

# Manage HPA
kubectl get hpa
kubectl describe hpa NAME
kubectl delete hpa NAME

# Monitor scaling
kubectl get hpa --watch
kubectl top pods
kubectl get events --sort-by=.metadata.creationTimestamp
```

### VPA Commands
```bash
# Install VPA (if needed)
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
./hack/vpa-install.sh

# Manage VPA
kubectl get vpa
kubectl describe vpa NAME
kubectl delete vpa NAME

# Monitor VPA
kubectl get vpa NAME -o yaml
kubectl describe vpa NAME
```

### Metrics and Debugging
```bash
# Check metrics server
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
kubectl top pods

# Debug autoscaling
kubectl describe hpa NAME
kubectl describe vpa NAME
kubectl get events | grep -E "(HorizontalPodAutoscaler|VerticalPodAutoscaler)"
```

## Key Points for CKA Exam

### Must Know Concepts
1. **HPA scales replicas** horizontally based on metrics
2. **VPA adjusts resource requests** vertically
3. **Resource requests are required** for HPA/VPA to work
4. **Metrics server is prerequisite** for resource-based scaling
5. **Behavior policies control** scaling rate and timing

### Common Exam Tasks
- Create HPA for deployments
- Configure multi-metric HPA
- Set up VPA for resource optimization
- Troubleshoot scaling issues
- Monitor autoscaling behavior

### Critical Configuration
- **minReplicas/maxReplicas** for HPA
- **updateMode** for VPA (Off/Initial/Auto)
- **Resource requests** in deployments
- **Stabilization windows** for smooth scaling
- **Resource policies** for VPA boundaries
