# Design and Install a Kubernetes Cluster HA (High Availability)

## High Availability Overview

### What is Kubernetes High Availability?
**High Availability (HA)** in Kubernetes ensures that the cluster remains operational even when individual components fail. HA Kubernetes clusters eliminate single points of failure and provide continuous service availability for production workloads.[1]

### Why HA is Critical for Production
- **Business continuity**: Prevents costly downtime and service disruptions
- **Data protection**: Maintains cluster state and application data integrity
- **Scalability**: Supports enterprise-scale deployments up to 5,000 nodes[2]
- **Fault tolerance**: Automatic recovery from component failures
- **Regulatory compliance**: Meets enterprise availability requirements

## HA Topology Options

### Stacked Control Plane Topology
**Stacked topology** co-locates etcd members with control plane nodes:[3]

```yaml
# Stacked HA Configuration
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "loadbalancer.example.com:6443"
etcd:
  local:
    dataDir: "/var/lib/etcd"
```

**Advantages**:
- **Simpler setup**: Fewer infrastructure components to manage
- **Lower cost**: Requires fewer nodes than external etcd topology
- **Easier management**: Co-located components simplify administration
- **Default approach**: Standard kubeadm topology for HA clusters[3]

**Disadvantages**:
- **Coupled failure risk**: Loss of one node affects both etcd and control plane
- **Resource contention**: etcd and API server compete for resources
- **Limited scalability**: Not ideal for very large clusters

### External etcd Topology
**External etcd topology** separates etcd cluster from control plane nodes:[3]

```yaml
# External etcd Configuration
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "loadbalancer.example.com:6443"
etcd:
  external:
    endpoints:
    - "https://10.100.0.10:2379"
    - "https://10.100.0.11:2379"
    - "https://10.100.0.12:2379"
    caFile: "/etc/kubernetes/pki/etcd/ca.crt"
    certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
    keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
```

**Advantages**:
- **Reduced risk**: Control plane and etcd failures are decoupled
- **Better performance**: Dedicated resources for etcd operations
- **Higher scalability**: Better suited for large-scale deployments
- **Independent maintenance**: Can maintain etcd and control plane separately

**Disadvantages**:
- **Complex setup**: Requires additional infrastructure and configuration
- **Higher cost**: More nodes required for full HA setup
- **Management overhead**: Multiple clusters to monitor and maintain

## Design Considerations

### Infrastructure Requirements
Based on cluster size and workload demands:[4]

| Nodes | GCP Instance | AWS Instance | Use Case |
|-------|--------------|--------------|----------|
| 1-5 | N1-standard-1 (1 vCPU, 3.75GB) | M3.medium (1 vCPU, 3.75GB) | Development/Testing |
| 6-10 | N1-standard-2 (2 vCPU, 7.5GB) | M3.large (2 vCPU, 7.5GB) | Small Production |
| 11-100 | N1-standard-4 (4 vCPU, 15GB) | M3.xlarge (4 vCPU, 15GB) | Medium Production |
| 101-250 | N1-standard-8 (8 vCPU, 30GB) | M3.2xlarge (8 vCPU, 30GB) | Large Production |
| 251-500 | N1-standard-16 (16 vCPU, 60GB) | C4.4xlarge (16 vCPU, 30GB) | Enterprise |
| >500 | N1-standard-32 (32 vCPU, 120GB) | C4.8xlarge (36 vCPU, 60GB) | Hyperscale |

### Network Architecture Planning
**Load Balancer Configuration**:

```yaml
# HAProxy Load Balancer Configuration
global
    daemon

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend kubernetes
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 192.168.5.11:6443 check fall 3 rise 2
    server master2 192.168.5.12:6443 check fall 3 rise 2
    server master3 192.168.5.13:6443 check fall 3 rise 2
```

### Node Distribution Strategy
**Minimum HA Requirements**:
- **Control Plane**: 3 master nodes for quorum
- **Worker Nodes**: Minimum 2 workers for workload distribution
- **Load Balancer**: 1-2 load balancers for API access
- **etcd Cluster**: 3 or 5 nodes for data consistency

### Storage Considerations
**High-Performance Storage Requirements**:[2]
- **SSD-backed storage**: For high-performance applications
- **Network-based storage**: For multiple concurrent connections
- **Persistent volumes**: Shared access across multiple pods
- **Backup strategy**: Regular etcd and application data backups

## etcd High Availability

### etcd Cluster Fundamentals
**etcd** is a distributed key-value store that maintains cluster state:[4]

```bash
# etcd Service Configuration
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/kubernetes.pem \
  --peer-key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380,controller-2=https://${CONTROLLER2_IP}:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
```

### etcd Quorum and Fault Tolerance
**Quorum Requirements**:[4]

| Instances | Quorum | Fault Tolerance | Recommendation |
|-----------|--------|-----------------|----------------|
| 1 | 1 | 0 | Development only |
| 2 | 2 | 0 | Not recommended |
| 3 | 2 | 1 | **Minimum for production** |
| 4 | 3 | 1 | Same as 3, higher cost |
| 5 | 3 | 2 | **Recommended for critical workloads** |
| 6 | 4 | 2 | Same as 5, higher cost |
| 7 | 4 | 3 | Maximum practical size |

### etcd Best Practices
**Optimal Configuration**:
- **Odd numbers**: Always use odd number of etcd instances
- **Fast storage**: Use SSD storage for etcd data directory
- **Network latency**: Keep etcd nodes in same data center
- **Resource isolation**: Dedicated CPU and memory for etcd
- **Regular backups**: Automated etcd snapshot backups

## Control Plane High Availability

### API Server HA Configuration
**Multiple API Server Setup**:[4]

```yaml
# API Server Service Configuration
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=${INTERNAL_IP} \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.pem \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \
  --event-ttl=1h \
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \
  --runtime-config='api/all=true' \
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
```

### Controller Manager HA with Leader Election
**Leader Election Configuration**:[4]

```yaml
# Controller Manager Service
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --leader-elect=true \
  --leader-elect-lease-duration=15s \
  --leader-elect-renew-deadline=10s \
  --leader-elect-retry-period=2s \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --v=2
```

### Scheduler HA with Leader Election
**Scheduler Leader Election**:[4]

```yaml
# Scheduler Service
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/config/kube-scheduler.yaml \
  --leader-elect=true \
  --leader-elect-lease-duration=15s \
  --leader-elect-renew-deadline=10s \
  --leader-elect-retry-period=2s \
  --v=2
```

## Installation Process

### Prerequisites and Planning
**Infrastructure Preparation**:

1. **Node Requirements**:[4]
   - **Operating System**: Linux x86_64 architecture
   - **Container Runtime**: Docker 1.11.1, 1.12.1, 1.13.1, 17.03, 17.06, 17.09, 18.06
   - **Network**: All nodes must communicate on required ports
   - **Storage**: SSD storage recommended for etcd

2. **Network Configuration**:
   - **Pod Network**: `10.244.0.0/16` (default)
   - **Service Network**: `10.96.0.0/12` (default)
   - **DNS**: CoreDNS for service discovery

### Step 1: Provision Infrastructure
**Using Vagrant for Lab Setup**:[4]

```ruby
# Vagrantfile for HA Cluster
Vagrant.configure("2") do |config|
  # Load Balancer
  config.vm.define "loadbalancer" do |lb|
    lb.vm.box = "ubuntu/bionic64"
    lb.vm.hostname = "loadbalancer"
    lb.vm.network "private_network", ip: "192.168.5.30"
    lb.vm.provider "virtualbox" do |v|
      v.name = "loadbalancer"
      v.memory = 1024
      v.cpus = 1
    end
  end

  # Master Nodes
  (1..3).each do |i|
    config.vm.define "master-#{i}" do |master|
      master.vm.box = "ubuntu/bionic64"
      master.vm.hostname = "master-#{i}"
      master.vm.network "private_network", ip: "192.168.5.1#{i}"
      master.vm.provider "virtualbox" do |v|
        v.name = "master-#{i}"
        v.memory = 2048
        v.cpus = 2
      end
    end
  end

  # Worker Nodes
  (1..2).each do |i|
    config.vm.define "worker-#{i}" do |worker|
      worker.vm.box = "ubuntu/bionic64"
      worker.vm.hostname = "worker-#{i}"
      worker.vm.network "private_network", ip: "192.168.5.2#{i}"
      worker.vm.provider "virtualbox" do |v|
        v.name = "worker-#{i}"
        v.memory = 2048
        v.cpus = 2
      end
    end
  end
end
```

### Step 2: Install Container Runtime
**Docker Installation on All Nodes**:

```bash
# Install Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker-ce=5:18.09.7~3-0~ubuntu-bionic

# Configure Docker daemon
sudo tee /etc/docker/daemon.json  apiserver-csr.conf  kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.21.0
controlPlaneEndpoint: "192.168.5.30:6443"
networking:
  podSubnet: "10.244.0.0/16"
etcd:
  external:
    endpoints:
    - "https://192.168.5.11:2379"
    - "https://192.168.5.12:2379"
    - "https://192.168.5.13:2379"
    caFile: "/etc/kubernetes/pki/etcd/ca.crt"
    certFile: "/etc/kubernetes/pki/apiserver-etcd-client.crt"
    keyFile: "/etc/kubernetes/pki/apiserver-etcd-client.key"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.5.11"
  bindPort: 6443
EOF

# Initialize first master node
sudo kubeadm init --config kubeadm-config.yaml --upload-certs
```

### Step 6: Configure Load Balancer
**HAProxy Configuration for API Server**:

```bash
# Install HAProxy
sudo apt-get update
sudo apt-get install -y haproxy

# Configure HAProxy
cat  \
  --discovery-token-ca-cert-hash  \
  --control-plane \
  --certificate-key 
```

### Step 8: Configure Networking
**Install CNI Plugin (Flannel)**:

```bash
# Apply Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Step 9: Join Worker Nodes
**Adding Worker Nodes to Cluster**:

```bash
# On worker nodes
sudo kubeadm join 192.168.5.30:6443 \
  --token  \
  --discovery-token-ca-cert-hash 
```

## Validation and Testing

### Cluster Health Verification
**Basic Health Checks**:

```bash
# Check node status
kubectl get nodes -o wide

# Check control plane pods
kubectl get pods -n kube-system

# Check etcd cluster health
kubectl -n kube-system exec etcd-master-1 -- etcdctl \
  --ca-file=/etc/kubernetes/pki/etcd/ca.crt \
  --cert-file=/etc/kubernetes/pki/etcd/server.crt \
  --key-file=/etc/kubernetes/pki/etcd/server.key \
  --endpoints=https://127.0.0.1:2379 \
  endpoint health

# Test API server accessibility
curl -k https://192.168.5.30:6443/healthz
```

### Application Deployment Testing
**Deploy Test Application**:

```bash
# Create test deployment
kubectl create deployment test-nginx --image=nginx --replicas=3

# Expose deployment
kubectl expose deployment test-nginx --port=80 --type=NodePort

# Scale deployment
kubectl scale deployment test-nginx --replicas=6

# Check pod distribution
kubectl get pods -o wide
```

### Failure Testing
**Simulate Node Failures**:

```bash
# Stop one master node
sudo systemctl stop kubelet
sudo systemctl stop docker

# Verify cluster continues to operate
kubectl get nodes

# Test application availability
kubectl get pods

# Restart failed node
sudo systemctl start docker
sudo systemctl start kubelet
```

## Production Best Practices

### Security Hardening
**Essential Security Measures**:
- **RBAC**: Enable role-based access control
- **Network Policies**: Implement pod-to-pod communication rules
- **TLS**: Use TLS for all component communication
- **Encryption**: Enable etcd encryption at rest
- **Regular Updates**: Keep Kubernetes components updated

### Monitoring and Alerting
**Monitoring Stack Setup**:

```yaml
# Prometheus monitoring configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

### Backup and Disaster Recovery
**etcd Backup Strategy**:

```bash
# Create etcd backup
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshot.db

# Schedule automated backups
cat << EOF | sudo tee /etc/cron.d/etcd-backup
0 2 * * * root /usr/local/bin/backup-etcd.sh
EOF
```

## Troubleshooting Common Issues

### etcd Cluster Issues
**Common etcd Problems**:

```bash
# Check etcd cluster status
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://192.168.5.11:2379,https://192.168.5.12:2379,https://192.168.5.13:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check etcd member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# View etcd logs
sudo journalctl -u etcd -f
```

### API Server Connectivity Issues
**Load Balancer Troubleshooting**:

```bash
# Test load balancer connectivity
nc -zv 192.168.5.30 6443

# Check HAProxy status
sudo systemctl status haproxy

# View HAProxy logs
sudo tail -f /var/log/haproxy.log

# Test direct API server access
curl -k https://192.168.5.11:6443/healthz
curl -k https://192.168.5.12:6443/healthz
curl -k https://192.168.5.13:6443/healthz
```

### Control Plane Recovery
**Master Node Recovery Procedures**:

```bash
# Recover from control plane failure
sudo kubeadm init phase control-plane all --config kubeadm-config.yaml

# Recreate admin kubeconfig
sudo kubeadm init phase kubeconfig admin --config kubeadm-config.yaml

# Restart kubelet
sudo systemctl restart kubelet
```

## Key Commands Summary

### Installation Commands
```bash
# Initialize HA cluster
sudo kubeadm init --config kubeadm-config.yaml --upload-certs

# Join control plane node
sudo kubeadm join LOAD_BALANCER:6443 --token TOKEN --control-plane --certificate-key KEY

# Join worker node
sudo kubeadm join LOAD_BALANCER:6443 --token TOKEN --discovery-token-ca-cert-hash HASH

# Generate new join token
kubeadm token create --print-join-command
```

### Health Check Commands
```bash
# Cluster status
kubectl cluster-info
kubectl get nodes
kubectl get componentstatuses

# Control plane health
kubectl get pods -n kube-system
kubectl top nodes

# etcd health
kubectl -n kube-system exec etcd-master-1 -- etcdctl endpoint health
```

### Maintenance Commands
```bash
# Drain node for maintenance
kubectl drain node-name --ignore-daemonsets --delete-emptydir-data

# Uncordon node
kubectl uncordon node-name

# Upgrade cluster
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.21.1
```
[1] https://trilio.io/kubernetes-disaster-recovery/kubernetes-high-availability/
[2] https://notes.kodekloud.com/docs/CKA-Certification-Course-Certified-Kubernetes-Administrator/Design-and-Install-a-Kubernetes-Cluster/Design-a-Kubernetes-Cluster
[3] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/
[4] https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/36507341/674ceec9-7144-40f7-bafa-afa541d8aea1/Kubernetes-CKA-0900-Install-v1.4.pdf
[5] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
[6] https://blog.kubesimplify.com/ha-kubernetes
[7] https://kubesphere.io/docs/v3.4/installing-on-linux/high-availability-configurations/set-up-ha-cluster-using-keepalived-haproxy/
[8] https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke1-for-rancher
[9] https://www.xcubelabs.com/blog/high-availability-kubernetes-architecting-for-resilience/
[10] https://www.cisco.com/c/en/us/td/docs/net_mgmt/cisco_container_platform/1-5/User_Guide/CCP-User-Guide-1-5-0/CCP-User-Guide-1-5-0_chapter_010.pdf
[11] https://docs.redhat.com/it/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.9/pdf/install/index
[12] https://www.youtube.com/watch?v=JXbTyz1QIHI
[13] https://www.spectrocloud.com/blog/architecting-high-availability-applications-on-kubernetes
[14] https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-cluster-setup/k3s-for-rancher
[15] https://ubuntu.com/tutorials/getting-started-with-kubernetes-ha
[16] https://learn.microsoft.com/en-us/azure-stack/user/pattern-highly-available-kubernetes?view=azs-2501
[17] https://iamondemand.com/wp-content/uploads/2019/11/Kubernetes-eBook.pdf
[18] https://kubeops.net/blog/achieving-high-availability-in-kubernetes-clusters
[19] https://kubernetes.io/docs/concepts/architecture/
[20] https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
[21] https://platform9.com/media/Operating-Kubernetes.pdf