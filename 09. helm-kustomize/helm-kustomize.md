# Helm and Kustomize - CKAD Essentials

## Helm - Package Manager for Kubernetes

### What is Helm?
**Helm** is a package manager for Kubernetes that simplifies deploying and managing applications using templates and reusable packages called **Charts**.

### Why Use Helm?
- **Templating**: Create reusable Kubernetes manifests
- **Package Management**: Install, upgrade, and rollback applications easily
- **Version Control**: Track application releases
- **Configuration Management**: Separate config from application logic

### Helm Key Concepts
- **Chart**: A Helm package containing Kubernetes templates
- **Release**: An instance of a chart deployed to Kubernetes
- **Values**: Configuration data for charts
- **Repository**: A collection of charts

## Basic Helm Usage

### Installing Helm
```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

### Working with Repositories
```bash
# Add popular repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add nginx https://helm.nginx.com/stable

# Update repositories
helm repo update

# Search for charts
helm search repo nginx
helm search repo mysql
```

### Essential Helm Commands
```bash
# Install a chart
helm install myapp bitnami/nginx

# Install with custom values
helm install myapp bitnami/nginx --set replicaCount=3

# List releases
helm list

# Upgrade release
helm upgrade myapp bitnami/nginx --set image.tag=1.21

# Rollback release
helm rollback myapp 1

# Uninstall release
helm uninstall myapp

# Get release info
helm status myapp
helm get values myapp
```

## Creating Simple Helm Charts

### Chart Structure
```bash
# Create new chart
helm create webapp

# Basic chart structure
webapp/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default values
└── templates/          # Kubernetes templates
    ├── deployment.yaml
    ├── service.yaml
    └── _helpers.tpl
```

### Chart.yaml (Metadata)
```yaml
apiVersion: v2
name: webapp
description: Simple web application
version: 0.1.0
appVersion: "1.0.0"
```

### values.yaml (Configuration)
```yaml
# Simple configuration
replicaCount: 3

image:
  repository: nginx
  tag: "1.21.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

# Application config
app:
  debug: false
  logLevel: "info"
```

### Simple Deployment Template
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
  labels:
    app: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 80
        env:
        - name: DEBUG
          value: "{{ .Values.app.debug }}"
        - name: LOG_LEVEL
          value: "{{ .Values.app.logLevel }}"
```

### Basic Template Functions
```yaml
# Common template functions
{{ .Values.name | upper }}           # Uppercase
{{ .Values.name | lower }}           # Lowercase
{{ .Values.replicas | int }}         # Convert to integer
{{ .Values.tag | default "latest" }} # Default value

# Conditionals
{{- if .Values.ingress.enabled }}
# ingress configuration
{{- end }}

# Loops
{{- range .Values.environments }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}
```

### Working with Your Chart
```bash
# Test template rendering
helm template webapp ./webapp/

# Install your chart
helm install myapp ./webapp/

# Install with custom values
helm install myapp ./webapp/ --set replicaCount=5

# Package chart
helm package webapp/
```

## Kustomize - Kubernetes Native Configuration

### What is Kustomize?
**Kustomize** is a Kubernetes-native tool for customizing YAML configurations without templates. It uses an overlay approach to modify base configurations.

### Why Use Kustomize?
- **Template-free**: Works with standard Kubernetes YAML
- **Overlay approach**: Modify configs without changing originals
- **Built into kubectl**: No extra tools needed
- **GitOps friendly**: Easy version control

### Basic Kustomize Structure
```bash
# Simple project structure
app/
├── base/                    # Base configuration
│   ├── kustomization.yaml   # Base kustomization file
│   ├── deployment.yaml      # Base deployment
│   └── service.yaml         # Base service
└── overlays/               # Environment-specific changes
    ├── development/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

## Basic Kustomize Usage

### Base Configuration
**base/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  app: webapp
  
namePrefix: webapp-
```

**base/deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: nginx:1.20.0
        ports:
        - containerPort: 80
```

**base/service.yaml**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Development Overlay
**overlays/development/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Include base configuration
resources:
  - ../../base

# Development-specific settings
namespace: development
namePrefix: dev-

# Change image tag for development
images:
  - name: nginx
    newTag: latest

# Override replica count
replicas:
  - name: app
    count: 1

# Add development labels
commonLabels:
  environment: development
```

### Production Overlay
**overlays/production/kustomization.yaml**:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

# Production settings
namespace: production
namePrefix: prod-

# Use specific image version
images:
  - name: nginx
    newTag: 1.21.0

# Production replica count
replicas:
  - name: app
    count: 5

# Production labels
commonLabels:
  environment: production
```

## Essential Kustomize Operations

### Basic Commands
```bash
# Preview configuration
kubectl kustomize ./overlays/development/

# Apply configuration
kubectl apply -k ./overlays/development/
kubectl apply -k ./overlays/production/

# Delete resources
kubectl delete -k ./overlays/development/
```

### Common Transformers
```yaml
# Change image tags
images:
  - name: nginx
    newTag: 1.21.0

# Modify replica counts
replicas:
  - name: app-deployment
    count: 5

# Add name prefix/suffix
namePrefix: dev-
nameSuffix: -v1

# Set namespace
namespace: development

# Add common labels
commonLabels:
  app: myapp
  environment: dev

# Add common annotations
commonAnnotations:
  description: "Development deployment"
```

### Simple Patches
```yaml
# overlays/production/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 250m
            memory: 256Mi
```

```yaml
# Include patch in kustomization.yaml
patchesStrategicMerge:
  - patch-resources.yaml
```

## ConfigMaps and Secrets with Kustomize

### Generate ConfigMaps
```yaml
# From literal values
configMapGenerator:
  - name: app-config
    literals:
      - DEBUG=true
      - LOG_LEVEL=info

# From files
configMapGenerator:
  - name: app-config
    files:
      - config.properties
      - app.yaml
```

### Generate Secrets
```yaml
# From literal values
secretGenerator:
  - name: app-secrets
    literals:
      - username=admin
      - password=secret123
    type: Opaque
```

## When to Use Helm vs Kustomize

### Use Helm When:
- Need package management and distribution
- Complex templating requirements
- Want to use existing Helm charts
- Need dependency management
- Require release management features

### Use Kustomize When:
- Managing multiple environments
- Want simple configuration overlays
- Prefer template-free approach
- Working with standard Kubernetes YAML
- Need built-in kubectl integration

## Common Patterns

### Environment Management with Kustomize
```bash
# Development
kubectl apply -k ./overlays/development/

# Staging  
kubectl apply -k ./overlays/staging/

# Production
kubectl apply -k ./overlays/production/
```

### Application Distribution with Helm
```bash
# Add application repository
helm repo add mycompany https://charts.mycompany.com

# Install application
helm install myapp mycompany/webapp

# Upgrade application
helm upgrade myapp mycompany/webapp --version 2.0.0
```

## Key Commands Summary

### Helm Commands
```bash
# Repository management
helm repo add <name> <url>
helm repo update
helm search repo <term>

# Release management
helm install <name> <chart>
helm upgrade <name> <chart>
helm rollback <name> <revision>
helm uninstall <name>
helm list

# Chart development
helm create <name>
helm template <name> <chart>
helm package <chart>
```

### Kustomize Commands
```bash
# Build and apply
kubectl kustomize <directory>
kubectl apply -k <directory>
kubectl delete -k <directory>

# Validation
kubectl apply -k <directory> --dry-run=client
```

## Best Practices

### Helm Best Practices
- Use semantic versioning for charts
- Include resource limits and probes
- Organize values logically
- Test templates before releasing
- Document chart usage

### Kustomize Best Practices
- Keep base configuration minimal
- Use meaningful directory structure
- Apply consistent labeling
- Use overlays for environment differences
- Version control all configurations