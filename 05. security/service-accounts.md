# Service Accounts

## What are Service Accounts?

**Service Accounts** are special types of accounts in Kubernetes that provide an identity for processes running in pods. Unlike user accounts (which are for humans), service accounts are designed for applications and system components.[1][2]

### Key Characteristics
- **Pod Identity**: Every pod runs with a service account
- **API Access**: Service accounts enable pods to interact with the Kubernetes API
- **Namespace Scoped**: Each service account belongs to a specific namespace
- **Automatic Creation**: Every namespace gets a default service account
- **Token-Based**: Authentication uses JWT tokens

## Default Service Account

### Automatic Behavior
```bash
# Every namespace gets a default service account
kubectl get serviceaccounts
kubectl get sa  # Short form

# Output shows:
# NAME      SECRETS   AGE
# default   1         5d
```

### Default Service Account Properties
- **Name**: `default`
- **Automatic Mounting**: Token automatically mounted to all pods
- **Limited Permissions**: Can only access basic API discovery endpoints
- **Location**: `/var/run/secrets/kubernetes.io/serviceaccount/`

### Default Token Structure
```bash
# Inside a pod, check mounted service account
ls /var/run/secrets/kubernetes.io/serviceaccount/
# ca.crt    namespace    token

# View the token
cat /var/run/secrets/kubernetes.io/serviceaccount/token

# View the namespace
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace
```

## Creating Service Accounts

### Imperative Creation
```bash
# Create a service account
kubectl create serviceaccount my-service-account
kubectl create sa my-app-sa  # Short form

# Create in specific namespace
kubectl create serviceaccount my-sa -n development

# List service accounts
kubectl get serviceaccounts -A
kubectl get sa --all-namespaces
```

### Declarative Creation
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-service-account
  namespace: production
  labels:
    app: my-application
    tier: backend
automountServiceAccountToken: true  # Default: true
```

### Service Account with Secrets
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
secrets:
- name: build-robot-secret
imagePullSecrets:
- name: private-registry-secret
```

## Service Account Tokens

### What are Service Account Tokens?
**Service Account Tokens** are JWT (JSON Web Token) credentials that pods use to authenticate with the Kubernetes API server.[2][1]

### Token Types

#### 1. Automatic Tokens (Legacy)
```bash
# Kubernetes < 1.24: Automatic secret creation
kubectl create serviceaccount token-test
kubectl get secrets
# Shows: token-test-token-xxxxx

# View token secret
kubectl describe secret token-test-token-xxxxx
```

#### 2. Projected Tokens (Current)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  serviceAccountName: my-service-account
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-xxxxx
      readOnly: true
  volumes:
  - name: kube-api-access-xxxxx
    projected:
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
```

### Manual Token Creation (Kubernetes 1.24+)
```bash
# Create long-lived token secret
kubectl create token my-service-account

# Create token with expiration
kubectl create token my-service-account --duration=1h

# Create token for specific audience
kubectl create token my-service-account --audience=https://my-api.example.com

# Create permanent token secret (legacy style)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-sa-token
  annotations:
    kubernetes.io/service-account.name: my-service-account
type: kubernetes.io/service-account-token
EOF
```

## Using Service Accounts in Pods

### Specifying Service Account
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  containers:
  - name: app
    image: my-app:latest
```

### Deployment with Service Account
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app-service-account
      containers:
      - name: app
        image: my-app:latest
```

### Disabling Automatic Token Mounting
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-token-pod
spec:
  serviceAccountName: my-service-account
  automountServiceAccountToken: false
  containers:
  - name: app
    image: my-app:latest
```

```yaml
# Disable at service account level
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-auto-mount-sa
automountServiceAccountToken: false
```

## Service Accounts and RBAC

### What is Service Account RBAC?
Service accounts integrate with Kubernetes RBAC to define what API operations pods can perform.[3][4]

### Creating Role for Service Account
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
```

### Binding Service Account to Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: ServiceAccount
  name: my-app-service-account
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole for Service Account
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes-global
subjects:
- kind: ServiceAccount
  name: node-reader-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### Service Account Subject Format
```yaml
# In RoleBinding/ClusterRoleBinding
subjects:
- kind: ServiceAccount
  name: service-account-name
  namespace: namespace-name  # Required for service accounts
```

## Advanced Service Account Usage

### Multiple Service Accounts Example
```yaml
# Development service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dev-app-sa
  namespace: development
---
# Production service account  
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prod-app-sa
  namespace: production
---
# Different permissions for each environment
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: dev-permissions
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps", "secrets"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: prod-permissions
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list"]
```

### Service Account with Image Pull Secrets
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: registry-sa
imagePullSecrets:
- name: private-registry-secret
---
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  serviceAccountName: registry-sa
  containers:
  - name: app
    image: private-registry.io/my-app:latest
```

### Service Account Labels and Annotations
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: labeled-sa
  namespace: production
  labels:
    app: my-application
    environment: production
    team: backend
  annotations:
    description: "Service account for backend application"
    owner: "backend-team@company.com"
    compliance.requirement: "high-security"
```

## Service Account Security Best Practices

### Principle of Least Privilege
```yaml
# ❌ BAD: Too broad permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bad-binding
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin  # Too much access!
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# ✅ GOOD: Specific permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: specific-permissions
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
  resourceNames: ["app-config"]  # Specific resource
```

### Namespace Isolation
```bash
# Create dedicated namespaces
kubectl create namespace app-production
kubectl create namespace app-development

# Create service accounts per namespace
kubectl create serviceaccount prod-app-sa -n app-production
kubectl create serviceaccount dev-app-sa -n app-development
```

### Disable Default Service Account
```yaml
# Patch default service account to disable auto-mounting
kubectl patch serviceaccount default -p '{"automountServiceAccountToken":false}'

# Or create pod without service account
apiVersion: v1
kind: Pod
metadata:
  name: no-service-account-pod
spec:
  automountServiceAccountToken: false
  containers:
  - name: app
    image: my-app:latest
```

### Token Security
```bash
# Use short-lived tokens
kubectl create token my-sa --duration=1h

# Avoid long-lived tokens in secrets
# Use projected service account tokens instead
```

## Service Account Troubleshooting

### Common Issues and Solutions

#### 1. Permission Denied Errors
```bash
# Check service account permissions
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa

# Check what the service account can do
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa

# Debug RBAC
kubectl get rolebindings,clusterrolebindings -A -o wide | grep my-sa
```

#### 2. Token Not Found
```bash
# Check if service account exists
kubectl get serviceaccount my-sa

# Check token mounting
kubectl describe pod my-pod | grep -A5 -B5 "service-account"

# Manually create token
kubectl create token my-sa
```

#### 3. Wrong Namespace
```bash
# Check service account namespace
kubectl get serviceaccount my-sa -n correct-namespace

# Service accounts are namespace-scoped
kubectl get sa -A | grep my-sa
```

### Debugging Commands
```bash
# View service account details
kubectl describe serviceaccount my-sa

# Check mounted token in pod
kubectl exec -it my-pod -- ls -la /var/run/secrets/kubernetes.io/serviceaccount/

# View token content (be careful with sensitive data)
kubectl exec -it my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Test API access from within pod
kubectl exec -it my-pod -- curl -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" https://kubernetes.default.svc/api/v1/namespaces/default/pods --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

## CKA Exam Scenarios

### Scenario 1: Create Service Account and Bind to Role
```bash
# 1. Create service account
kubectl create serviceaccount app-service-account -n production

# 2. Create role
kubectl create role app-role --verb=get,list,create --resource=pods,configmaps -n production

# 3. Create role binding
kubectl create rolebinding app-binding --role=app-role --serviceaccount=production:app-service-account -n production

# 4. Test permissions
kubectl auth can-i get pods --as=system:serviceaccount:production:app-service-account -n production
```

### Scenario 2: Fix Pod Service Account Issue
```bash
# Problem: Pod cannot access API
kubectl describe pod failing-pod

# Solution: Check and fix service account
kubectl get serviceaccount
kubectl create serviceaccount correct-sa
kubectl patch deployment my-app -p '{"spec":{"template":{"spec":{"serviceAccountName":"correct-sa"}}}}'
```

### Scenario 3: Security Hardening
```bash
# Disable default service account auto-mount
kubectl patch serviceaccount default -p '{"automountServiceAccountToken":false}'

# Create minimal permission service account
kubectl create serviceaccount minimal-sa
kubectl create role minimal-role --verb=get --resource=configmaps --resource-name=app-config
kubectl create rolebinding minimal-binding --role=minimal-role --serviceaccount=default:minimal-sa
```

## Service Account Commands Summary

### Basic Operations
```bash
# Create service account
kubectl create serviceaccount NAME
kubectl create sa NAME -n NAMESPACE

# List service accounts
kubectl get serviceaccounts
kubectl get sa
kubectl get sa -A

# Describe service account
kubectl describe serviceaccount NAME
kubectl describe sa NAME

# Delete service account
kubectl delete serviceaccount NAME
kubectl delete sa NAME
```

### Token Operations
```bash
# Create token (Kubernetes 1.24+)
kubectl create token SERVICE_ACCOUNT_NAME
kubectl create token SA_NAME --duration=1h
kubectl create token SA_NAME --audience=my-audience

# View mounted token in pod
kubectl exec POD_NAME -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

### RBAC Integration
```bash
# Create role and binding for service account
kubectl create role ROLE_NAME --verb=VERBS --resource=RESOURCES
kubectl create rolebinding BINDING_NAME --role=ROLE_NAME --serviceaccount=NAMESPACE:SA_NAME

# Create cluster role and binding
kubectl create clusterrole CLUSTER_ROLE_NAME --verb=VERBS --resource=RESOURCES
kubectl create clusterrolebinding BINDING_NAME --clusterrole=CLUSTER_ROLE_NAME --serviceaccount=NAMESPACE:SA_NAME

# Test service account permissions
kubectl auth can-i VERB RESOURCE --as=system:serviceaccount:NAMESPACE:SA_NAME
kubectl auth can-i --list --as=system:serviceaccount:NAMESPACE:SA_NAME
```

### Debugging
```bash
# Check service account in pod
kubectl describe pod POD_NAME | grep -i "service account"

# View service account tokens
kubectl get secrets | grep service-account-token

# Check RBAC bindings
kubectl get rolebindings,clusterrolebindings -A | grep SA_NAME
```

## Key Points for CKA Exam

### Must Know Concepts
1. **Default Behavior**: Every pod gets the default service account if none specified
2. **Namespace Scope**: Service accounts are namespace-scoped resources
3. **RBAC Integration**: Service accounts work with RBAC for authorization
4. **Token Mounting**: Tokens automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/`
5. **Security**: Follow least privilege principle

### Common Exam Tasks
- Create service accounts
- Bind service accounts to roles/cluster roles
- Configure pods to use specific service accounts
- Troubleshoot permission issues
- Disable automatic token mounting
- Create manual tokens

### Important File Paths
- **Token**: `/var/run/secrets/kubernetes.io/serviceaccount/token`
- **CA Certificate**: `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
- **Namespace**: `/var/run/secrets/kubernetes.io/serviceaccount/namespace`

### Critical Commands
```bash
kubectl create serviceaccount NAME
kubectl create role NAME --verb=VERBS --resource=RESOURCES
kubectl create rolebinding NAME --role=ROLE --serviceaccount=NAMESPACE:SA
kubectl auth can-i VERB RESOURCE --as=system:serviceaccount:NAMESPACE:SA
```
