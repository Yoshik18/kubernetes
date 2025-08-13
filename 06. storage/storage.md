# Storage

## Volumes

### What are Volumes in Kubernetes?
**Volumes** provide persistent storage for pods that survives container restarts. Unlike the container's ephemeral filesystem, volumes persist data even when containers are recreated or restarted.[1]

### Volume Purpose and Benefits
- **Data persistence**: Maintains data across container restarts
- **Data sharing**: Enables multiple containers in a pod to share data  
- **External storage**: Connects pods to external storage systems
- **Decoupling**: Separates storage concerns from container lifecycle

### Container Storage Behavior
When containers are created, Kubernetes creates a temporary filesystem that gets destroyed when the container is removed. Volumes solve this problem by providing persistent storage that exists independently of container lifecycle.[1]

## Volume Types

### What are Volume Types?
**Volume Types** determine where and how data is stored. Kubernetes supports numerous volume types to accommodate different storage requirements and infrastructure setups.[1]

### Host-Based Volume Types

#### hostPath Volume
**hostPath** mounts a directory from the node's filesystem into the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - image: alpine
    name: alpine
    command: ["/bin/sh","-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: Directory
```

**hostPath Characteristics**:
- **Node-specific**: Data tied to specific node
- **Not portable**: Pods scheduled on different nodes can't access data
- **Development use**: Suitable for single-node testing
- **Production caution**: Not recommended for multi-node clusters

### Network-Based Volume Types

#### NFS (Network File System)
```yaml
volumes:
- name: nfs-volume
  nfs:
    server: nfs-server.example.com
    path: /path/to/nfs/share
```

#### Cloud Provider Volumes

**AWS Elastic Block Store (EBS)**:
```yaml
volumes:
- name: data-volume
  awsElasticBlockStore:
    volumeID: vol-1234567890abcdef0
    fsType: ext4
```

**Google Cloud Persistent Disk**:
```yaml
volumes:
- name: data-volume
  gcePersistentDisk:
    pdName: my-disk
    fsType: ext4
```

**Azure Disk**:
```yaml
volumes:
- name: data-volume
  azureDisk:
    diskName: my-disk.vhd
    diskURI: https://myaccount.blob.core.windows.net/vhds/my-disk.vhd
```

### Distributed Storage Volume Types
- **Ceph**: Distributed storage system
- **GlusterFS**: Scale-out network-attached storage
- **ScaleIO**: Software-defined storage platform
- **Flocker**: Container data volume manager

### Volume Mounting Process
1. **Volume definition**: Specify volume in pod spec
2. **Container mounting**: Mount volume into container filesystem
3. **Path mapping**: Map volume to specific container directory
4. **Data persistence**: Data persists beyond container lifecycle

## Volume Challenges in Large Deployments

### Multi-User Environment Problems
In environments with many users creating pods with different volume requirements:

- **Configuration complexity**: Each user must configure volumes manually
- **Storage management**: Admins must provision storage for every pod
- **Resource conflicts**: Multiple users might try to use same storage
- **Inconsistent setup**: Different volume configurations across teams

### The Need for Abstraction
**Problem**: Direct volume management becomes unmanageable at scale
**Solution**: Kubernetes provides **Persistent Volumes** and **Persistent Volume Claims** to abstract storage management[1]

## Persistent Volumes (PVs)

### What are Persistent Volumes?
**Persistent Volumes (PVs)** are cluster-wide storage resources provisioned by administrators. They abstract the underlying storage implementation and provide a standardized interface for storage consumption.[1]

### PV Benefits
- **Centralized management**: Admins manage storage pool centrally
- **User abstraction**: Users don't need to know storage details
- **Resource pooling**: Create pool of storage resources
- **Standardization**: Consistent storage interface across cluster

### Persistent Volume Definition
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Retain
  awsElasticBlockStore:
    volumeID: vol-1234567890abcdef0
    fsType: ext4
```

### PV Specification Components

#### Access Modes
**Access Modes** define how the volume can be mounted:

| Mode | Description | Use Case |
|------|-------------|----------|
| **ReadWriteOnce (RWO)** | Single node read-write | Databases, single-instance apps |
| **ReadOnlyMany (ROX)** | Multiple nodes read-only | Static content, shared configuration |
| **ReadWriteMany (RWX)** | Multiple nodes read-write | Shared filesystems, collaborative apps |

#### Capacity
```yaml
capacity:
  storage: 1Gi  # Available storage space
```

#### Reclaim Policy
**Reclaim Policy** determines what happens when PV is released:

| Policy | Behavior | Description |
|--------|----------|-------------|
| **Retain** | Manual cleanup | Admin must manually reclaim |
| **Delete** | Automatic deletion | Storage resource deleted automatically |
| **Recycle** | Data scrubbing | Volume scrubbed and made available (deprecated) |

### Creating and Viewing PVs
```bash
# Create PV
kubectl create -f pv-definition.yaml

# List PVs
kubectl get persistentvolume
kubectl get pv

# Detailed PV information
kubectl describe pv pv-vol1
```

### PV Status States
| Status | Description |
|--------|-------------|
| **Available** | Ready for use, not yet claimed |
| **Bound** | Bound to a PVC |
| **Released** | PVC deleted, but not yet reclaimed |
| **Failed** | Reclamation failed |

## Persistent Volume Claims (PVCs)

### What are Persistent Volume Claims?
**Persistent Volume Claims (PVCs)** are requests for storage by users. They specify storage requirements and are matched with available Persistent Volumes.[1]

### PVC Purpose
- **User interface**: Simple way for users to request storage
- **Resource claiming**: Claims available PVs based on requirements
- **Abstraction**: Users don't need to know underlying storage details
- **Automatic binding**: Kubernetes handles PV-PVC binding

### PVC Definition
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### Creating and Managing PVCs
```bash
# Create PVC
kubectl create -f pvc-definition.yaml

# List PVCs
kubectl get persistentvolumeclaim
kubectl get pvc

# Describe PVC
kubectl describe pvc myclaim

# Delete PVC
kubectl delete persistentvolumeclaim myclaim
```

## PV and PVC Binding Process

### What is PV-PVC Binding?
**Binding** is the process where Kubernetes automatically matches PVCs with suitable PVs based on specified criteria.[1]

### Binding Criteria
Kubernetes considers multiple factors when binding PVCs to PVs:

#### 1. Sufficient Capacity
```yaml
# PVC requests 500Mi
resources:
  requests:
    storage: 500Mi

# PV provides 1Gi (sufficient)
capacity:
  storage: 1Gi
```

#### 2. Access Modes Compatibility
```yaml
# Both must have compatible access modes
# PVC
accessModes:
  - ReadWriteOnce

# PV  
accessModes:
  - ReadWriteOnce
```

#### 3. Volume Modes
```yaml
# Block or Filesystem mode compatibility
volumeMode: Filesystem
```

#### 4. Storage Class
```yaml
# Must match storage class (if specified)
storageClassName: fast-ssd
```

#### 5. Selector Labels
```yaml
# PV with labels
metadata:
  labels:
    name: my-pv
    environment: production

# PVC with selector
spec:
  selector:
    matchLabels:
      name: my-pv
      environment: production
```

### Binding Examples

#### Successful Binding
```bash
# Before binding
kubectl get pv,pvc
NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      
persistentvolume/pv-vol1  1Gi        RWO            Retain          Available

NAME                          STATUS    VOLUME   CAPACITY   ACCESS MODES
persistentvolumeclaim/myclaim Pending

# After binding
NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
persistentvolume/pv-vol1  1Gi        RWO            Retain          Bound    default/myclaim

NAME                          STATUS   VOLUME    CAPACITY   ACCESS MODES
persistentvolumeclaim/myclaim Bound    pv-vol1   1Gi        RWO
```

#### Failed Binding (Pending State)
If no suitable PV is available, PVC remains in **Pending** status:
```bash
NAME                          STATUS    VOLUME   CAPACITY   ACCESS MODES
persistentvolumeclaim/myclaim Pending
```

### One-to-One Relationship
- **Exclusive binding**: Each PV can bind to only one PVC
- **Capacity allocation**: Entire PV capacity allocated to PVC (even if PVC requests less)
- **No sharing**: PVs cannot be shared between multiple PVCs

## Using PVCs in Pods

### Mounting PVCs in Pods
Once a PVC is bound to a PV, pods can use the PVC to access persistent storage:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: frontend
    image: nginx
    volumeMounts:
    - mountPath: "/var/www/html"
      name: mypd
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim
```

### PVC Lifecycle in Pods
1. **PVC creation**: User creates PVC
2. **Binding**: Kubernetes binds PVC to suitable PV
3. **Pod usage**: Pod references PVC in volume definition
4. **Mount**: Container mounts PVC at specified path
5. **Data persistence**: Data persists across pod restarts

## Imperative Commands for Storage

### PV Management Commands
```bash
# Create PV from YAML
kubectl create -f pv-definition.yaml

# List all PVs
kubectl get pv
kubectl get persistentvolumes

# Describe specific PV
kubectl describe pv pv-vol1

# Delete PV
kubectl delete pv pv-vol1

# Edit PV
kubectl edit pv pv-vol1

# View PV in different output formats
kubectl get pv -o wide
kubectl get pv -o yaml
kubectl get pv -o json
```

### PVC Management Commands
```bash
# Create PVC imperatively
kubectl create pvc my-pvc --size=1Gi --access-modes=ReadWriteOnce

# Create from YAML
kubectl create -f pvc-definition.yaml

# List PVCs
kubectl get pvc
kubectl get persistentvolumeclaims

# Describe PVC
kubectl describe pvc myclaim

# Delete PVC
kubectl delete pvc myclaim

# View PVC details
kubectl get pvc -o wide
kubectl get pvc myclaim -o yaml
```

### Combined PV/PVC Operations
```bash
# View both PVs and PVCs
kubectl get pv,pvc

# Monitor binding status
kubectl get pv,pvc -w

# Check events related to storage
kubectl get events --field-selector involvedObject.kind=PersistentVolume
kubectl get events --field-selector involvedObject.kind=PersistentVolumeClaim
```

## Declarative Configurations

### Complete PV-PVC-Pod Example
```yaml
# pv-definition.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/data"

---
# pvc-definition.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi

---
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
  - name: task-pv-storage
    persistentVolumeClaim:
      claimName: task-pv-claim
  containers:
  - name: task-pv-container
    image: nginx
    ports:
    - containerPort: 80
      name: "http-server"
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: task-pv-storage
```

### Deployment with PVC
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
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
      - name: webapp
        image: nginx
        volumeMounts:
        - name: webapp-storage
          mountPath: /var/www/html
      volumes:
      - name: webapp-storage
        persistentVolumeClaim:
          claimName: webapp-pvc
```

## Storage Classes (Dynamic Provisioning)

### What are Storage Classes?
**Storage Classes** enable dynamic provisioning of storage resources, automatically creating PVs when PVCs are created.[1]

### Storage Class Benefits
- **Automatic provisioning**: No need to pre-create PVs
- **On-demand storage**: Storage created when needed
- **Different tiers**: Support multiple storage performance tiers
- **Cloud integration**: Seamless integration with cloud storage services

### Storage Class Definition
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  zones: us-west-2a, us-west-2b
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### Using Storage Classes with PVCs
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

## Reclaim Policies in Detail

### Understanding Reclaim Policies
**Reclaim Policies** determine what happens to a PV after its associated PVC is deleted.[1]

### Retain Policy
```yaml
persistentVolumeReclaimPolicy: Retain
```
- **Manual intervention**: Admin must manually handle PV
- **Data preservation**: Data remains intact
- **Status change**: PV status changes to "Released"
- **Reuse restriction**: Cannot be claimed by new PVC until manually cleaned

### Delete Policy
```yaml
persistentVolumeReclaimPolicy: Delete
```
- **Automatic cleanup**: PV and underlying storage deleted automatically
- **Data loss**: All data permanently lost
- **Cloud storage**: Actual cloud storage resources deleted
- **Default**: Often default for dynamically provisioned volumes

### Recycle Policy (Deprecated)
```yaml
persistentVolumeReclaimPolicy: Recycle
```
- **Data scrubbing**: Basic scrub (rm -rf /thevolume/*)
- **Availability**: PV becomes available for new claims
- **Deprecation**: Replaced by dynamic provisioning

## Storage Best Practices

### PV/PVC Design Principles
- **Right-size storage**: Request appropriate storage amounts
- **Access mode selection**: Choose correct access modes for use case
- **Storage class strategy**: Use storage classes for different performance tiers
- **Backup planning**: Implement backup strategies for critical data

### Security Considerations
- **Access controls**: Use RBAC to control PVC creation
- **Namespace isolation**: Keep storage resources in appropriate namespaces
- **Data encryption**: Enable encryption for sensitive data
- **Network policies**: Secure storage network communications

### Performance Optimization
- **Storage location**: Consider data locality for performance
- **IOPS requirements**: Match storage type to performance needs
- **Caching strategies**: Implement appropriate caching layers
- **Monitoring**: Monitor storage performance and usage

## Troubleshooting Storage Issues

### Common PVC Problems
```bash
# PVC stuck in Pending
kubectl describe pvc problematic-pvc
# Look for events and error messages

# Check available PVs
kubectl get pv
# Verify capacity, access modes, storage class

# Check storage class
kubectl get storageclass
kubectl describe storageclass fast-ssd
```

### PV Binding Issues
```bash
# Check PV labels and selectors
kubectl describe pv my-pv
kubectl describe pvc my-pvc

# Verify access modes compatibility
kubectl get pv my-pv -o yaml | grep accessModes
kubectl get pvc my-pvc -o yaml | grep accessModes

# Check storage capacity
kubectl describe pv my-pv | grep Capacity
kubectl describe pvc my-pvc | grep storage
```

### Pod Mount Failures
```bash
# Check pod events
kubectl describe pod my-pod
kubectl get events --field-selector involvedObject.name=my-pod

# Verify PVC status
kubectl get pvc
kubectl describe pvc my-claim

# Check volume mounts
kubectl describe pod my-pod | grep -A 5 "Mounts"
```

## Key Storage Commands Summary

### Essential PV Commands
```bash
# PV lifecycle
kubectl create -f pv-definition.yaml
kubectl get pv
kubectl describe pv 
kubectl delete pv 
```

### Essential PVC Commands
```bash
# PVC lifecycle  
kubectl create -f pvc-definition.yaml
kubectl get pvc
kubectl describe pvc 
kubectl delete pvc 
```

### Storage Monitoring Commands
```bash
# Combined view
kubectl get pv,pvc
kubectl get pv,pvc -o wide

# Storage events
kubectl get events | grep -i volume
kubectl get events | grep -i persistent

# Storage class information
kubectl get storageclass
kubectl describe storageclass 
```