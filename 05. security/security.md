# Security

## Security Primitives

### What is Kubernetes Security?
**Kubernetes Security** encompasses the protection of cluster infrastructure, applications, and data through multiple layers of defense including authentication, authorization, network policies, and encryption.

### Core Security Components
- **Authentication**: Verifying identity of users and services
- **Authorization**: Determining what authenticated entities can do
- **Admission Control**: Validating and potentially modifying requests
- **Network Security**: Controlling traffic flow between components
- **Secrets Management**: Protecting sensitive data

## Authentication Mechanisms

### What is Authentication?
**Authentication** is the process of verifying the identity of users, service accounts, or external systems attempting to access the Kubernetes cluster.

### Authentication Methods

#### 1. Static Token Files
```bash
# Create user-details.csv
password123,user1,u0001,group1
password123,user2,u0002,group1
password123,user3,u0003,group2

# Configure API server
--token-auth-file=user-details.csv
```

#### 2. X.509 Client Certificates
```bash
# Generate user certificate
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=john/O=developers"
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -out user.crt
```

#### 3. Service Account Tokens
- **Automatic**: Created for each namespace
- **JWT-based**: JSON Web Tokens for pod authentication
- **Projection**: Mounted into pods automatically

### API Server Configuration
```yaml
# API Server Authentication Flags
--client-ca-file=/var/lib/kubernetes/ca.pem
--tls-cert-file=/var/lib/kubernetes/apiserver.crt
--tls-private-key-file=/var/lib/kubernetes/apiserver.key
--service-account-key-file=/var/lib/kubernetes/service-account.pem
```

## TLS Certificates

### What are TLS Certificates?
**TLS Certificates** provide cryptographic proof of identity and enable encrypted communication between Kubernetes components.

### Certificate Types

#### Server Certificates
- **API Server**: `apiserver.crt` - Serves HTTPS API endpoints
- **ETCD Server**: `etcdserver.crt` - Encrypts cluster data storage
- **Kubelet**: `kubelet.crt` - Secures node agent communication

#### Client Certificates
- **Admin Users**: `admin.crt` - Administrator access to cluster
- **Scheduler**: `scheduler.crt` - Scheduler to API server communication
- **Controller Manager**: `controller-manager.crt` - Controller authentication
- **Kube-proxy**: `kube-proxy.crt` - Network proxy authentication

### Certificate Generation Process
```bash
# 1. Generate CA Key and Certificate
openssl genrsa -out ca.key 2048
openssl req -new -x509 -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.crt -days 10000

# 2. Generate Server Certificate (API Server)
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr -config openssl.cnf
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -out apiserver.crt -days 365

# 3. Generate Client Certificate (Admin)
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt -days 365
```

### Certificate Requirements
```yaml
# Subject Alternative Names for API Server
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```

### Certificate Inspection
```bash
# View certificate details
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Check certificate expiration
openssl x509 -in apiserver.crt -noout -dates
```

## Authorization

### What is Authorization?
**Authorization** determines what actions an authenticated user or service account can perform within the Kubernetes cluster.[3][4]

### Authorization Modes

#### 1. Node Authorization
- **Purpose**: Authorizes requests from kubelets
- **Scope**: Limited to node-specific resources
- **Certificate Group**: `system:nodes`

#### 2. ABAC (Attribute-Based Access Control)
```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "projectCaribou", "resource": "pods", "readonly": true}}
```

#### 3. RBAC (Role-Based Access Control)
**Most recommended** authorization method for production clusters.[4][3]

#### 4. Webhook Mode
- **External**: Delegates authorization to external services
- **Custom Logic**: Allows complex authorization rules
- **Integration**: Works with external identity providers

### API Server Authorization Configuration
```bash
# Enable multiple authorization modes
--authorization-mode=Node,RBAC,Webhook
```

## Role-Based Access Control (RBAC)

### What is RBAC?
**RBAC** regulates access to Kubernetes resources based on the roles assigned to users, groups, or service accounts.[3][4]

### RBAC Components

#### 1. Role (Namespace-scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
```

#### 2. ClusterRole (Cluster-wide)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

#### 3. RoleBinding (Namespace-scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### 4. ClusterRoleBinding (Cluster-wide)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: User
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Verbs
- **get**: Retrieve specific resource
- **list**: List resources of a type
- **watch**: Watch for resource changes
- **create**: Create new resources
- **update**: Modify existing resources
- **patch**: Partially update resources
- **delete**: Remove resources

### Resource Names (Specific Resource Access)
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "update"]
  resourceNames: ["blue-pod", "green-pod"]
```

### RBAC Management Commands
```bash
# Check user permissions
kubectl auth can-i create deployments --as=dev-user
kubectl auth can-i delete nodes --as=dev-user --namespace=development

# View RBAC resources
kubectl get roles,rolebindings
kubectl get clusterroles,clusterrolebindings
kubectl describe role pod-reader
kubectl describe rolebinding read-pods
```

## KubeConfig Files

### What is KubeConfig?
**KubeConfig** files contain cluster access information including clusters, users, and contexts for kubectl authentication.[2][1]

### KubeConfig Structure
```yaml
apiVersion: v1
kind: Config
current-context: dev-user@development-cluster
clusters:
- name: development-cluster
  cluster:
    certificate-authority: /path/to/ca.crt
    server: https://dev-cluster:6443
- name: production-cluster
  cluster:
    certificate-authority: /path/to/ca.crt
    server: https://prod-cluster:6443
contexts:
- name: dev-user@development-cluster
  context:
    cluster: development-cluster
    user: dev-user
    namespace: development
- name: admin@production-cluster
  context:
    cluster: production-cluster
    user: admin
users:
- name: dev-user
  user:
    client-certificate: /path/to/dev-user.crt
    client-key: /path/to/dev-user.key
- name: admin
  user:
    client-certificate: /path/to/admin.crt
    client-key: /path/to/admin.key
```

### Certificate Data Encoding
```yaml
# Instead of file paths, embed base64-encoded data
clusters:
- name: my-cluster
  cluster:
    certificate-authority-data: LS0tLS1CRUdJTi...
    server: https://cluster:6443
users:
- name: my-user
  user:
    client-certificate-data: LS0tLS1CRUdJTi...
    client-key-data: LS0tLS1CRUdJTi...
```

### What are Contexts?
**Contexts** combine cluster, user, and namespace information into a named configuration that allows easy switching between different Kubernetes environments.[1][2]

### Context Components
- **Cluster**: Which Kubernetes cluster to connect to
- **User**: Which user credentials to use for authentication
- **Namespace**: Default namespace for kubectl commands (optional)

### Context Structure
```yaml
contexts:
- name: dev-user@development-cluster
  context:
    cluster: development-cluster    # Which cluster to use
    user: dev-user                 # Which user credentials to use
    namespace: development         # Default namespace (optional)
- name: admin@production-cluster
  context:
    cluster: production-cluster
    user: admin
    namespace: production
```

### Context Management Commands
```bash
# View all contexts
kubectl config get-contexts

# View current context
kubectl config current-context

# Switch to a different context
kubectl config use-context dev-user@development-cluster

# Switch to production context
kubectl config use-context admin@production-cluster

# View detailed configuration
kubectl config view

# View only current context details
kubectl config view --minify
```

### Creating and Modifying Contexts
```bash
# Create a new context
kubectl config set-context my-context \
  --cluster=my-cluster \
  --user=my-user \
  --namespace=my-namespace

# Modify existing context namespace
kubectl config set-context dev-user@development-cluster --namespace=testing

# Set namespace for current context
kubectl config set-context --current --namespace=production

# Create context with cluster and user
kubectl config set-context staging-context \
  --cluster=staging-cluster \
  --user=staging-user
```

### Context Examples
```bash
# Development environment context
kubectl config set-context development \
  --cluster=dev-cluster \
  --user=dev-user \
  --namespace=development

# Production environment context  
kubectl config set-context production \
  --cluster=prod-cluster \
  --user=admin-user \
  --namespace=production

# Testing environment context
kubectl config set-context testing \
  --cluster=test-cluster \
  --user=test-user \
  --namespace=testing
```

### Context Best Practices
- **Descriptive naming**: Use clear context names like `dev-user@development`
- **Environment separation**: Different contexts for dev, staging, production
- **Namespace defaults**: Set appropriate default namespaces
- **Regular cleanup**: Remove unused contexts periodically
- **Backup configs**: Keep backup copies of important kubeconfig files

### Context Switching Workflows
```bash
# Daily workflow example
# Start with development
kubectl config use-context development
kubectl get pods  # Shows pods in development namespace

# Switch to production for deployment
kubectl config use-context production
kubectl apply -f production-deployment.yaml

# Quick check in staging
kubectl config use-context staging
kubectl get services

# Back to development
kubectl config use-context development
```

### Advanced Context Operations
```bash
# Delete a context
kubectl config delete-context old-context

# Rename context (delete and recreate)
kubectl config delete-context old-name
kubectl config set-context new-name --cluster=cluster --user=user

# Export specific context
kubectl config view --context=production --minify --flatten > prod-config.yaml

# Use temporary context without switching
kubectl --context=production get pods
kubectl --context=development apply -f app.yaml
```

### Multi-Cluster Context Setup
```yaml
# Example: Managing multiple environments
apiVersion: v1
kind: Config
current-context: development

# Development cluster
clusters:
- name: dev-cluster
  cluster:
    server: https://dev-api.company.com:6443
    certificate-authority-data: LS0tLS...

# Staging cluster  
- name: staging-cluster
  cluster:
    server: https://staging-api.company.com:6443
    certificate-authority-data: LS0tLS...

# Production cluster
- name: prod-cluster
  cluster:
    server: https://prod-api.company.com:6443
    certificate-authority-data: LS0tLS...

# Users for different environments
users:
- name: dev-user
  user:
    client-certificate-data: LS0tLS...
    client-key-data: LS0tLS...
- name: staging-user
  user:
    client-certificate-data: LS0tLS...
    client-key-data: LS0tLS...
- name: prod-admin
  user:
    client-certificate-data: LS0tLS...
    client-key-data: LS0tLS...

# Contexts combining clusters and users
contexts:
- name: development
  context:
    cluster: dev-cluster
    user: dev-user
    namespace: development
- name: staging
  context:
    cluster: staging-cluster
    user: staging-user
    namespace: staging
- name: production
  context:
    cluster: prod-cluster
    user: prod-admin
    namespace: production
```

### Context Security Considerations
- **User isolation**: Use different users for different environments
- **Namespace boundaries**: Set appropriate default namespaces
- **Access control**: Ensure users only have access to appropriate clusters
- **Audit trails**: Context switches can be logged for security auditing
- **Credential management**: Rotate certificates and tokens regularly

### Troubleshooting Context Issues
```bash
# Check context configuration
kubectl config view --context=problematic-context

# Verify cluster connectivity
kubectl --context=production cluster-info

# Test authentication
kubectl --context=development auth can-i get pods

# Debug certificate issues
kubectl config view --raw -o jsonpath='{.contexts[?(@.name=="production")].context}'

# Check current context settings
kubectl config get-contexts $(kubectl config current-context)
```

## Image Security

### What is Image Security?
**Image Security** involves securing container images and controlling access to private registries.[1][2]

### Private Registry Authentication
```bash
# Login to private registry
docker login private-registry.io
Username: registry-user
Password: registry-password

# Create secret for registry authentication
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=user@company.com
```

### Using Image Pull Secrets
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  containers:
  - name: app
    image: private-registry.io/apps/internal-app
  imagePullSecrets:
  - name: regcred
```

### Image Security Best Practices
- **Scan images** for vulnerabilities before deployment
- **Use specific tags** instead of `latest`
- **Minimize base images** to reduce attack surface
- **Regular updates** of base images and dependencies

## Security Contexts

### What are Security Contexts?
**Security Contexts** define privilege and access control settings for pods and containers.[2][1]

### Pod Security Context
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
  containers:
  - name: sec-ctx-demo
    image: busybox
    securityContext:
      allowPrivilegeEscalation: false
      runAsUser: 2000
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
        drop: ["ALL"]
      readOnlyRootFilesystem: true
```

### Security Context Options
- **runAsUser**: User ID for running containers
- **runAsGroup**: Primary group ID for containers
- **runAsNonRoot**: Prevents running as root user
- **fsGroup**: Group ownership for mounted volumes
- **allowPrivilegeEscalation**: Controls privilege escalation
- **capabilities**: Linux capabilities to add/drop
- **readOnlyRootFilesystem**: Makes root filesystem read-only

## Network Policies

### What are Network Policies?
**Network Policies** control network traffic flow between pods, providing micro-segmentation at the application layer.[1][2]

### Default Behavior
- **All Allow**: By default, all pods can communicate with all other pods
- **No Isolation**: No network restrictions until policies are applied

### Network Policy Components
- **podSelector**: Which pods the policy applies to
- **policyTypes**: Ingress, Egress, or both
- **ingress**: Rules for incoming traffic
- **egress**: Rules for outgoing traffic

### Basic Network Policy Example
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          name: prod
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
    ports:
    - protocol: TCP
      port: 5978
```

### Network Policy Selectors

#### Pod Selector
```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
```

#### Namespace Selector
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: production
```

#### IP Block
```yaml
ingress:
- from:
  - ipBlock:
      cidr: 192.168.1.0/24
      except:
      - 192.168.1.5/32
```

### Deny All Policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### CNI Plugin Requirements
Network Policies require a compatible CNI plugin:
- **Calico**: Full network policy support
- **Weave**: Supports network policies
- **Flannel**: Does **NOT** support network policies

## ConfigMaps and Secrets Security

### What are ConfigMaps?
**ConfigMaps** store non-confidential configuration data in key-value pairs that can be consumed by pods.[2]

### What are Secrets?
**Secrets** store sensitive information such as passwords, tokens, and keys with additional security measures.[2]

### Security Differences

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Data Type** | Non-sensitive configuration | Sensitive information |
| **Encoding** | Plain text | Base64 encoded |
| **Storage** | etcd (plain text) | etcd (can be encrypted at rest) |
| **Access** | Standard RBAC | Enhanced RBAC controls |

### Secret Types
- **Opaque**: General-purpose secrets (default)
- **kubernetes.io/service-account-token**: Service account tokens
- **kubernetes.io/dockercfg**: Docker registry authentication
- **kubernetes.io/tls**: TLS certificates and keys

### Creating Secrets
```bash
# Imperative creation
kubectl create secret generic app-secret \
  --from-literal=DB_Host=mysql \
  --from-literal=DB_User=root \
  --from-literal=DB_Password=password

# Declarative creation
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_Host: bXlzcWw=     # mysql (base64)
  DB_User: cm9vdA==     # root (base64)
  DB_Password: cGFzc3dvcmQ= # password (base64)
```

### Using Secrets in Pods
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: myapp
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_Password
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
```

### ConfigMap Security Best Practices
- **Separate sensitive data**: Never store passwords in ConfigMaps
- **Use RBAC**: Control access to ConfigMaps
- **Namespace isolation**: Keep ConfigMaps in appropriate namespaces
- **Regular audits**: Review ConfigMap contents regularly

### Secret Security Best Practices
- **Encryption at rest**: Enable etcd encryption
- **RBAC controls**: Strict access controls for secrets
- **Secret rotation**: Regular rotation of sensitive data
- **Avoid logging**: Prevent secrets from appearing in logs
- **Image security**: Don't embed secrets in container images

## Certificate API

### What is the Certificates API?
The **Certificates API** allows you to provision TLS certificates signed by a Certificate Authority (CA) that you control.[1][2]

### Certificate Signing Request (CSR)
```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john-developer
spec:
  request: LS0tLS1CRUdJTi... # base64 encoded CSR
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```

### CSR Workflow
```bash
# 1. Generate private key
openssl genrsa -out john.key 2048

# 2. Create certificate signing request
openssl req -new -key john.key -subj "/CN=john/O=developers" -out john.csr

# 3. Submit CSR to Kubernetes
cat john.csr | base64 | tr -d '\n'

# 4. Create CSR object
kubectl apply -f john-csr.yaml

# 5. Approve CSR
kubectl certificate approve john-developer

# 6. Get certificate
kubectl get csr john-developer -o jsonpath='{.status.certificate}' | base64 -d > john.crt
```

### Controller Manager Configuration
```yaml
# Controller manager handles certificate signing
--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
--cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```

## Security Best Practices

### Cluster Security
- **Regular updates**: Keep Kubernetes and components updated
- **Network segmentation**: Use network policies for micro-segmentation
- **Least privilege**: Apply minimal necessary permissions
- **Audit logging**: Enable comprehensive audit logging
- **Resource limits**: Set appropriate resource quotas and limits

### Authentication Security
- **Strong certificates**: Use proper key lengths and algorithms
- **Certificate rotation**: Regular rotation of certificates
- **Multi-factor authentication**: Where possible, implement MFA
- **Service accounts**: Use dedicated service accounts for applications

### Authorization Security
- **RBAC enforcement**: Use RBAC over other authorization modes
- **Regular reviews**: Audit permissions regularly
- **Namespace isolation**: Use namespaces for logical separation
- **Avoid wildcards**: Be specific in permission grants

### Secret Management
- **External secret management**: Consider tools like HashiCorp Vault
- **Encryption at rest**: Enable etcd encryption
- **Secret scanning**: Scan for secrets in code and images
- **Rotation policies**: Implement automated secret rotation

## Troubleshooting Security Issues

### Common Authentication Problems
```bash
# Check API server configuration
kubectl get pods -n kube-system
kubectl logs kube-apiserver-master -n kube-system

# Verify certificate details
openssl x509 -in client.crt -text -noout

# Test authentication
kubectl auth can-i get pods --as=system:serviceaccount:default:default
```

### Common Authorization Problems
```bash
# Check RBAC configuration
kubectl get roles,rolebindings -A
kubectl describe role developer
kubectl describe rolebinding dev-binding

# Test permissions
kubectl auth can-i create deployments --as=dev-user
kubectl auth can-i list secrets --as=dev-user -n production
```

### Network Policy Debugging
```bash
# Check if network policies are applied
kubectl get networkpolicies
kubectl describe networkpolicy deny-all

# Test connectivity
kubectl exec -it test-pod -- nc -zv target-service 80
kubectl exec -it test-pod -- nslookup target-service
```

## Key Commands Summary

### Authentication & Certificates
```bash
# Certificate operations
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=user"
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -out user.crt

# CSR management
kubectl get csr
kubectl certificate approve user-csr
kubectl certificate deny user-csr
```

### RBAC Management
```bash
# Create RBAC resources
kubectl create role developer --verb=get,list,create --resource=pods
kubectl create rolebinding dev-binding --role=developer --user=dev-user
kubectl create clusterrole cluster-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding cluster-reader-binding --clusterrole=cluster-reader --user=admin
```

### Secrets and ConfigMaps
```bash
# Secret operations
kubectl create secret generic app-secret --from-literal=key=value
kubectl get secrets
kubectl describe secret app-secret

# ConfigMap operations  
kubectl create configmap app-config --from-literal=key=value
kubectl get configmaps
kubectl describe configmap app-config
```

### Network Policies
```bash
# Network policy management
kubectl apply -f network-policy.yaml
kubectl get networkpolicies
kubectl describe networkpolicy deny-all
```
