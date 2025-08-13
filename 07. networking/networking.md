# Networking

## Networking Fundamentals

### What is Kubernetes Networking?
**Kubernetes Networking** is the mechanism by which different resources within and outside your cluster are able to communicate with each other. It provides the foundation for understanding how containers, pods, and services within Kubernetes communicate seamlessly across the cluster.[1][2]

### Core Networking Principles
Kubernetes networking is built on several fundamental principles that ensure consistent and reliable communication:[3][1]

- **Every pod gets its own IP address**: Each pod receives a unique cluster-wide IP address
- **Containers within a pod share the pod IP address**: All containers in a pod communicate via localhost
- **Pods can communicate with all other pods**: Direct communication without NAT across the entire cluster
- **Network isolation**: Defined using network policies rather than network structure

### Why Kubernetes Networking Matters
Understanding Kubernetes networking is crucial for:[2]
- **Proper environment configuration**: Setting up production-ready clusters
- **Complex networking scenarios**: Implementing advanced traffic management
- **Security implementation**: Controlling communication flows
- **Troubleshooting**: Diagnosing connectivity issues

## The Kubernetes Network Model

### What is the Kubernetes Network Model?
The **Kubernetes Network Model** specifies how different components communicate within a cluster. It provides a flat network structure that makes pod-to-pod communication simple and predictable.[1]

### Network Model Components

#### Pod Network (Cluster Network)
The **Pod Network** handles communication between pods across the cluster:[3]

```yaml
# Every pod gets a unique IP from the cluster CIDR
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

**Key characteristics**:
- **Cluster-wide IP addressing**: All pods can reach each other using IP addresses
- **No NAT required**: Direct pod-to-pod communication
- **Cross-node communication**: Pods on different nodes communicate seamlessly
- **Network namespace sharing**: Containers in same pod share network namespace

#### Service Network
**Services** provide stable network identities for accessing groups of pods:[3]

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### Network Model Implementation
The Kubernetes network model is implemented using:[4]
- **Container Runtime**: Sets up network namespaces for containers
- **CNI Plugin**: Configures networking rules and policies
- **Kube-proxy**: Handles service networking and load balancing

## Container Network Interface (CNI)

### What is CNI?
**Container Network Interface (CNI)** is a standard interface specification for network plugins in container orchestration systems. It defines how container runtimes collaborate with networking plugins to configure networking for containers and pods.[4]

### CNI Responsibilities
CNI plugins handle several critical networking tasks:[4]
- **IP address assignment**: Allocating unique IPs to containers
- **Network interface setup**: Configuring network interfaces
- **Route definition**: Setting up routing tables
- **Network policy enforcement**: Implementing security rules

### Popular CNI Plugins

#### Weave Net
**Weave Net** creates a virtual network that connects containers across multiple hosts:

```bash
# Deploy Weave Net
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

**Features**:
- **Automatic IP allocation**: Dynamic IP address management
- **Encryption**: Secure pod-to-pod communication
- **Network policies**: Traffic filtering capabilities
- **Multi-cloud support**: Works across different cloud providers

#### Calico
**Calico** provides networking and network security for containers:

```bash
# Deploy Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

**Features**:
- **Network policies**: Advanced traffic filtering
- **BGP routing**: Efficient inter-node communication
- **Network security**: Built-in firewall capabilities
- **Performance**: High-performance networking

#### Flannel
**Flannel** is a simple overlay network for Kubernetes:

```bash
# Deploy Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**Features**:
- **Simplicity**: Easy to deploy and manage
- **Overlay networking**: VXLAN-based networking
- **Cross-platform**: Works on various operating systems

### CNI Configuration
CNI is configured through the kubelet:[5]

```bash
# Kubelet CNI configuration
ExecStart=/usr/local/bin/kubelet \
  --network-plugin=cni \
  --cni-bin-dir=/opt/cni/bin \
  --cni-conf-dir=/etc/cni/net.d
```

**Configuration files**:
```json
{
  "cniVersion": "0.2.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

## IP Address Management (IPAM)

### What is IPAM?
**IP Address Management (IPAM)** ensures that each pod receives a unique IP address within the cluster. It prevents IP conflicts and manages IP allocation across nodes.[5]

### IPAM Plugins

#### Host-Local IPAM
**Host-local** maintains IP allocation on each node locally:

```json
{
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

**Characteristics**:
- **Node-specific allocation**: Each node manages its own IP range
- **Persistent storage**: IP allocations stored locally
- **Simple configuration**: Easy to set up and manage

#### DHCP IPAM
**DHCP** uses external DHCP servers for IP allocation:

```json
{
  "ipam": {
    "type": "dhcp"
  }
}
```

### IP Range Planning
Proper IP range planning is crucial for large clusters:[5]

```bash
# Cluster CIDR configuration
kube-apiserver --service-cluster-ip-range=10.96.0.0/12
```

**Common IP ranges**:
- **Pod Network**: `10.244.0.0/16` (65,534 IPs)
- **Service Network**: `10.96.0.0/12` (1,048,574 IPs)
- **Node Network**: `192.168.1.0/24` (254 IPs)

## Services and Service Discovery

### What are Services?
**Services** provide stable network endpoints for accessing groups of pods. They abstract away the dynamic nature of pod IP addresses and provide load balancing.[6]

### Service Types

#### ClusterIP Service
**ClusterIP** exposes the service internally within the cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: internal-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

**Use cases**:
- **Internal communication**: Service-to-service communication
- **Database access**: Backend services accessing databases
- **Microservices**: Communication between microservices

#### NodePort Service
**NodePort** exposes the service on each node's IP at a static port:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

**Characteristics**:
- **External access**: Accessible from outside the cluster
- **Port range**: 30000-32767 by default
- **All nodes**: Service accessible on all nodes

#### LoadBalancer Service
**LoadBalancer** provisions an external load balancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

**Features**:
- **Cloud integration**: Works with cloud provider load balancers
- **Automatic provisioning**: Load balancer created automatically
- **High availability**: Built-in load balancing and failover

### Service Discovery
Kubernetes provides automatic service discovery through:[5]

#### DNS-Based Discovery
Every service gets a DNS name:

```bash
# Service DNS format
..svc.cluster.local

# Examples
web-service.default.svc.cluster.local
database.production.svc.cluster.local
```

#### Environment Variables
Services are exposed as environment variables:

```bash
# Service environment variables
WEB_SERVICE_HOST=10.96.1.100
WEB_SERVICE_PORT=80
WEB_SERVICE_PORT_80_TCP=tcp://10.96.1.100:80
```

## Ingress and Ingress Controllers

### What is Ingress?
**Ingress** manages external access to services in a cluster, typically HTTP and HTTPS. It provides URL-based routing, SSL termination, and load balancing.[5]

### Ingress Components

#### Ingress Resource
**Ingress Resource** defines the routing rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

#### Ingress Controller
**Ingress Controller** implements the ingress rules:

```bash
# Deploy NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

### Advanced Ingress Features

#### SSL/TLS Termination
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

#### Path-Based Routing
```yaml
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
```

## Kubernetes Gateway API

### What is the Gateway API?
The **Kubernetes Gateway API** is a modern, extensible approach to managing ingress and traffic routing in Kubernetes. It provides more expressive, structured routing capabilities compared to traditional Ingress resources.[7]

### Gateway API Benefits
The Gateway API offers several advantages over Ingress:[8]
- **More expressive**: Rich traffic management capabilities
- **Role separation**: Clear separation between infrastructure and application concerns
- **Extensibility**: Built-in extension points for custom functionality
- **Protocol support**: Native support for HTTP, HTTPS, TCP, UDP, and gRPC
- **Implementation agnostic**: Works with different gateway controllers

### Gateway API Architecture

#### GatewayClass
**GatewayClass** defines a set of gateways implemented by a specific controller:[7]

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/gateway-controller
  description: "NGINX Gateway Controller"
```

**Purpose**:
- **Decouples configuration from implementation**: Allows switching between different gateway implementations
- **Multi-controller support**: Multiple gateway implementations can coexist
- **Administrative control**: Platform administrators control available implementations

#### Gateway
**Gateway** defines how traffic enters the cluster:[7]

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: tls-secret
    allowedRoutes:
      namespaces:
        from: All
```

**Key features**:
- **Multiple protocols**: HTTP, HTTPS, TCP, UDP support
- **TLS termination**: Built-in SSL/TLS handling
- **Namespace control**: Fine-grained route permissions

### HTTP Routing with Gateway API

#### Basic HTTP Routing
**HTTPRoute** defines how HTTP traffic is routed to services:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: basic-route
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/v1
    backendRefs:
    - name: api-v1-service
      port: 80
      weight: 100
```

#### Advanced HTTP Routing Features

**Header-Based Routing**:
```yaml
rules:
- matches:
  - headers:
    - name: "version"
      value: "v2"
    path:
      type: PathPrefix
      value: /api
  backendRefs:
  - name: api-v2-service
    port: 80
```

**Method-Based Routing**:
```yaml
rules:
- matches:
  - method: GET
    path:
      type: PathPrefix
      value: /api/users
  backendRefs:
  - name: user-read-service
    port: 80
- matches:
  - method: POST
    path:
      type: PathPrefix
      value: /api/users
  backendRefs:
  - name: user-write-service
    port: 80
```

### Traffic Management with Gateway API

#### HTTP Redirects
**HTTP to HTTPS Redirect**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: https-redirect
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  hostnames:
  - "example.com"
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

**Path Redirects**:
```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /old-api
  filters:
  - type: RequestRedirect
    requestRedirect:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /new-api
      statusCode: 302
```

#### URL Rewriting
**Path Rewriting**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: rewrite-path
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /external-api
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /internal-api
    backendRefs:
    - name: internal-service
      port: 80
```

#### Header Modification
**Request Header Modification**:

```yaml
rules:
- filters:
  - type: RequestHeaderModifier
    requestHeaderModifier:
      add:
      - name: "X-Environment"
        value: "production"
      - name: "X-Request-ID"
        value: "generated-uuid"
      set:
      - name: "Authorization"
        value: "Bearer token"
      remove:
      - "X-Debug"
  backendRefs:
  - name: backend-service
    port: 80
```

**Response Header Modification**:
```yaml
filters:
- type: ResponseHeaderModifier
  responseHeaderModifier:
    add:
    - name: "X-Cache-Status"
      value: "MISS"
    set:
    - name: "Cache-Control"
      value: "max-age=3600"
    remove:
    - "Server"
```

#### Traffic Splitting
**Canary Deployments**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-split
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - backendRefs:
    - name: stable-service
      port: 80
      weight: 90
    - name: canary-service
      port: 80
      weight: 10
```

**A/B Testing**:
```yaml
rules:
- matches:
  - headers:
    - name: "X-User-Group"
      value: "beta"
  backendRefs:
  - name: beta-service
    port: 80
- backendRefs:
  - name: stable-service
    port: 80
```

#### Request Mirroring
**Traffic Mirroring for Testing**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mirror-traffic
  namespace: default
spec:
  parentRefs:
  - name: nginx-gateway
  rules:
  - filters:
    - type: RequestMirror
      requestMirror:
        backendRef:
          name: test-service
          port: 80
    backendRefs:
    - name: production-service
      port: 80
```

### Advanced Gateway API Features

#### TCP and UDP Routing
**TCP Route**:

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: database-route
  namespace: default
spec:
  parentRefs:
  - name: tcp-gateway
  rules:
  - backendRefs:
    - name: postgres-service
      port: 5432
```

**UDP Route**:
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: UDPRoute
metadata:
  name: dns-route
  namespace: default
spec:
  parentRefs:
  - name: udp-gateway
  rules:
  - backendRefs:
    - name: dns-service
      port: 53
```

#### TLS Configuration
**TLS Termination**:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tls-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: example-com-tls
        namespace: default
    allowedRoutes:
      namespaces:
        from: All
```

**TLS Passthrough**:
```yaml
listeners:
- name: tls-passthrough
  protocol: TLS
  port: 443
  tls:
    mode: Passthrough
  allowedRoutes:
    namespaces:
      from: All
```

### Installing NGINX Gateway Fabric

#### Prerequisites
Before installing NGINX Gateway Fabric:[9]

```bash
# Install Gateway API CRDs
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v1.6.2" | kubectl apply -f -

kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/experimental?ref=v1.6.2" | kubectl apply -f -
```

#### Installation via Helm
**Install NGINX Gateway Fabric**:[9]

```bash
# Add NGINX Helm repository
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update

# Install NGINX Gateway Fabric
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace \
  -n nginx-gateway \
  --wait
```

#### Verify Installation
```bash
# Check Gateway Fabric pods
kubectl get pods -n nginx-gateway

# Check GatewayClass
kubectl get gatewayclass

# Check Gateway API resources
kubectl get gateways
kubectl get httproutes
```

### Practical Gateway API Examples

#### Multi-Service Application
**Complete application setup**:

```yaml
# GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/gateway-controller

---
# Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: app-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: app-tls

---
# Frontend Route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: default
spec:
  parentRefs:
  - name: app-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80

---
# API Route with versioning
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
  namespace: default
spec:
  parentRefs:
  - name: app-gateway
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: api-v2-service
      port: 80
```

## Network Policies

### What are Network Policies?
**Network Policies** are Kubernetes resources that control traffic flow between pods, providing network-level security and segmentation. They act as a firewall for your pods, defining which traffic is allowed or denied.[6]

### Network Policy Components

#### Pod Selector
**Pod Selector** determines which pods the policy applies to:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
```

#### Ingress Rules
**Ingress Rules** control incoming traffic to selected pods:

```yaml
spec:
  ingress:
  - from:
    # Allow traffic from pods with specific labels
    - podSelector:
        matchLabels:
          app: api
    # Allow traffic from specific namespaces
    - namespaceSelector:
        matchLabels:
          environment: production
    # Allow traffic from specific IP ranges
    - ipBlock:
        cidr: 192.168.1.0/24
        except:
        - 192.168.1.100/32
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
```

#### Egress Rules
**Egress Rules** control outgoing traffic from selected pods:

```yaml
spec:
  egress:
  - to:
    # Allow traffic to database pods
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    # Allow traffic to external services
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - protocol: TCP
      port: 443  # HTTPS
    - protocol: TCP
      port: 53   # DNS
    - protocol: UDP
      port: 53   # DNS
```

### Network Policy Examples

#### Default Deny All Policy
**Deny all ingress and egress traffic**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # Applies to all pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

#### Allow Specific Traffic
**Three-tier application policy**:

```yaml
# Frontend policy - allows ingress from anywhere, egress to API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {} # Allow all ingress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8080

---
# API policy - allows ingress from frontend, egress to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432

---
# Database policy - allows ingress from API only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 5432
```

### Network Policy Best Practices

#### Namespace Isolation
**Isolate different environments**:

```yaml
# Production namespace policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: production-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          environment: production
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx  # Allow ingress controller
```

#### DNS Access Policy
**Allow DNS resolution**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

## DNS in Kubernetes

### What is Kubernetes DNS?
**Kubernetes DNS** provides service discovery and name resolution within the cluster. It automatically creates DNS records for services and pods, enabling communication using human-readable names instead of IP addresses.[5]

### DNS Architecture

#### CoreDNS
**CoreDNS** is the default DNS server in Kubernetes:[5]

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### DNS Records

#### Service DNS Records
**Services** get DNS records in the following format:[5]

```bash
# Service DNS format
..svc.cluster.local

# Examples
web-service.default.svc.cluster.local        # ClusterIP service
database.production.svc.cluster.local        # Database service
api-gateway.kube-system.svc.cluster.local    # System service
```

**Service record types**:
- **A Record**: Points to service ClusterIP
- **SRV Record**: Includes port information
- **CNAME Record**: For external services

#### Pod DNS Records
**Pods** get DNS records based on their IP addresses:[5]

```bash
# Pod DNS format
..pod.cluster.local

# Examples
10-244-1-5.default.pod.cluster.local     # Pod with IP 10.244.1.5
172-17-0-3.kube-system.pod.cluster.local # System pod
```

#### Headless Service Records
**Headless Services** create DNS records for individual pods:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-service
spec:
  clusterIP: None  # Makes service headless
  selector:
    app: database
  ports:
  - port: 5432
```

```bash
# Headless service DNS records
database-0.headless-service.default.svc.cluster.local
database-1.headless-service.default.svc.cluster.local
database-2.headless-service.default.svc.cluster.local
```

### DNS Configuration

#### Pod DNS Configuration
Pods automatically get DNS configuration:[5]

```bash
# /etc/resolv.conf in pod
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

#### Custom DNS Configuration
**Customize DNS for specific pods**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - custom.local
    options:
    - name: ndots
      value: "2"
  containers:
  - name: app
    image: nginx
```

## Load Balancing and Service Mesh

### Service Load Balancing
Kubernetes provides built-in load balancing through Services:[5]

#### Service Load Balancing Algorithms
**Round Robin** (default):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  sessionAffinity: None  # Round robin
```

**Session Affinity**:
```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

### External Load Balancers

#### Cloud Provider Load Balancers
**AWS Application Load Balancer**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: aws-lb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**Google Cloud Load Balancer**:
```yaml
metadata:
  annotations:
    cloud.google.com/load-balancer-type: "External"
    cloud.google.com/backend-config: '{"default": "my-backendconfig"}'
```

## Imperative Commands for Networking

### Service Management Commands
```bash
# Create services
kubectl expose deployment nginx --port=80 --target-port=8080 --type=ClusterIP
kubectl expose pod nginx --port=80 --type=NodePort
kubectl create service loadbalancer web --tcp=80:8080

# View services
kubectl get services
kubectl get svc -o wide
kubectl describe service web-service

# Edit services
kubectl edit service web-service
kubectl patch service web-service -p '{"spec":{"type":"LoadBalancer"}}'

# Delete services
kubectl delete service web-service
```

### Ingress Management Commands
```bash
# Create ingress
kubectl create ingress web-ingress --rule="example.com/api*=api-service:80"
kubectl create ingress tls-ingress --rule="secure.com/*=web-service:80,tls=tls-secret"

# View ingress
kubectl get ingress
kubectl describe ingress web-ingress

# Edit ingress
kubectl edit ingress web-ingress
kubectl annotate ingress web-ingress nginx.ingress.kubernetes.io/rewrite-target=/

# Delete ingress
kubectl delete ingress web-ingress
```

### Network Policy Management Commands
```bash
# Create network policy (declarative only)
kubectl apply -f network-policy.yaml

# View network policies
kubectl get networkpolicies
kubectl get netpol
kubectl describe networkpolicy web-policy

# Delete network policies
kubectl delete networkpolicy web-policy
kubectl delete netpol --all
```

### DNS and Service Discovery Commands
```bash
# DNS testing
kubectl run test-pod --image=busybox --rm -it -- nslookup web-service
kubectl exec -it test-pod -- dig web-service.default.svc.cluster.local

# Service endpoints
kubectl get endpoints web-service
kubectl describe endpoints web-service

# DNS debugging
kubectl exec -it dns-test -- cat /etc/resolv.conf
kubectl logs -n kube-system deployment/coredns
```

### Gateway API Commands
```bash
# Install Gateway API CRDs
kubectl kustomize "https://github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.0.0" | kubectl apply -f -

# View Gateway API resources
kubectl get gatewayclasses
kubectl get gateways
kubectl get httproutes
kubectl get tcproutes

# Describe Gateway resources
kubectl describe gatewayclass nginx
kubectl describe gateway app-gateway
kubectl describe httproute api-route

# Gateway API debugging
kubectl get gateway app-gateway -o yaml
kubectl get events --field-selector involvedObject.kind=Gateway
```

## Declarative Configurations

### Complete Network Stack Example
```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    environment: production

---
# Network Policy - Default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: ecommerce
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Network Policy - Web tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-policy
  namespace: ecommerce
spec:
  podSelector:
    matchLabels:
      tier: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 8080
  - to: {}  # Allow DNS
    ports:
    - protocol: UDP
      port: 53

---
# ClusterIP Service
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: ecommerce
spec:
  selector:
    app: web
    tier: web
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP

---
# LoadBalancer Service
apiVersion: v1
kind: Service
metadata:
  name: web-external
  namespace: ecommerce
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  selector:
    app: web
    tier: web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer

---
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - shop.example.com
    - api.example.com
    secretName: ecommerce-tls
  rules:
  - host: shop.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
```

## Troubleshooting Networking Issues

### Common Networking Problems

#### Service Discovery Issues
```bash
# Check service endpoints
kubectl get endpoints service-name
kubectl describe service service-name

# Test DNS resolution
kubectl run debug --image=busybox --rm -it -- nslookup service-name
kubectl run debug --image=busybox --rm -it -- wget -qO- http://service-name

# Check CoreDNS
kubectl logs -n kube-system deployment/coredns
kubectl get configmap coredns -n kube-system -o yaml
```

#### Network Policy Problems
```bash
# Check network policies
kubectl get networkpolicies -A
kubectl describe networkpolicy policy-name

# Test connectivity
kubectl exec -it source-pod -- nc -zv target-service 80
kubectl exec -it source-pod -- curl -v http://target-service

# Debug network policy
kubectl get pods --show-labels
kubectl describe pod pod-name | grep -A 5 Labels
```

#### Ingress Issues
```bash
# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Check ingress resources
kubectl get ingress -A
kubectl describe ingress ingress-name

# Test external access
curl -H "Host: example.com" http://ingress-ip/path
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

### Network Diagnostics Tools

#### Network Testing Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
    securityContext:
      capabilities:
        add: ["NET_ADMIN"]
```

#### Common Debugging Commands
```bash
# Inside debug pod
nslookup service-name
dig service-name.namespace.svc.cluster.local
nc -zv service-name port
traceroute service-ip
tcpdump -i any port 80

# Pod connectivity
ping pod-ip
telnet service-name port
wget -qO- http://service-name/health

# Network interface information
ip addr show
ip route show
netstat -tlnp
ss -tlnp
```

## Key Networking Commands Summary

### Essential Service Commands
```bash
# Service lifecycle
kubectl create service clusterip web --tcp=80:8080
kubectl expose deployment app --port=80 --type=NodePort
kubectl get services -o wide
kubectl describe service service-name
kubectl delete service service-name
```

### Essential Ingress Commands
```bash
# Ingress management
kubectl create ingress web --rule="host/path*=service:port"
kubectl get ingress -A
kubectl describe ingress ingress-name
kubectl edit ingress ingress-name
```

### Essential Network Policy Commands
```bash
# Network policy management
kubectl apply -f network-policy.yaml
kubectl get networkpolicies -A
kubectl describe networkpolicy policy-name
kubectl delete networkpolicy policy-name
```

### Essential DNS Commands
```bash
# DNS testing and debugging
kubectl run test --image=busybox --rm -it -- nslookup service-name
kubectl exec pod -- dig service.namespace.svc.cluster.local
kubectl logs -n kube-system deployment/coredns
```

### Essential Gateway API Commands
```bash
# Gateway API resources
kubectl get gatewayclasses
kubectl get gateways -A
kubectl get httproutes -A
kubectl describe gateway gateway-name
kubectl describe httproute route-name
```
[1] https://www.tigera.io/learn/guides/kubernetes-networking/
[2] https://spacelift.io/blog/kubernetes-networking
[3] https://kubernetes.io/docs/concepts/services-networking/
[4] https://www.getambassador.io/blog/kubernetes-networking-guide-top-engineers
[5] https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/36507341/19d1867e-468b-4ab2-b568-f4ac2bb6a59a/Ingress.pdf
[6] https://devopscube.com/ckad-exam-study-guide/
[7] https://devopscube.com/kubernetes-gateway-api/
[8] https://blog.nginx.org/blog/5-reasons-to-try-the-kubernetes-gateway-api
[9] https://blog.nashtechglobal.com/hands-on-kubernetes-gateway-api-with-nginx-gateway-fabric/
[10] https://www.practical-devsecops.com/cka-vs-ckad/
[11] https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/36507341/b7982b4c-143a-4fff-89aa-cfa6c3317581/Kubernetes-CKA-0800-Networking-v1.2.pdf
[12] https://kubernetes.io/docs/concepts/cluster-administration/networking/
[13] https://github.com/nginx/nginx-gateway-fabric
[14] https://www.freecodecamp.org/news/kubernetes-networking-tutorial-for-developers/
[15] https://opensource.com/article/22/6/kubernetes-networking-fundamentals
[16] https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/
[17] https://cloud.google.com/kubernetes-engine/docs/concepts/network-overview
[18] https://kubernetes.io/training/
[19] https://gateway-api.sigs.k8s.io/implementations/
[20] https://kodekloud.com/courses/certified-kubernetes-application-developer-ckad
[21] https://spacelift.io/blog/kubernetes-certification
[22] https://docs.nginx.com/nginx-gateway-fabric/overview/gateway-architecture/