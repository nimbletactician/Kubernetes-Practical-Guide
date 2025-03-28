# Kubernetes-Practical-Guide

This guide covers essential Kubernetes resources, distribution strategies, migration approaches, and best practices for scalability and high availability.

## Table of Contents
- [Core Kubernetes Resources](#core-kubernetes-resources)
- [Pod Distribution Strategies](#pod-distribution-strategies)
- [Zero Downtime Migration](#zero-downtime-migration)
  - [Stateless Applications](#stateless-applications)
  - [Stateful Applications](#stateful-applications)
- [Scalability and High Availability](#scalability-and-high-availability)

## Core Kubernetes Resources

### Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```
<!-- Pods are the smallest deployable units in Kubernetes. Use standalone pods for one-off tasks or debugging. -->

### ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3                    # Maintains 3 pod replicas
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```
<!-- ReplicaSets ensure the specified number of pods are running. Generally not used directly; Deployments manage ReplicaSets. -->

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate          # Strategy for updating pods
    rollingUpdate:
      maxSurge: 1                # Max pods created above desired number
      maxUnavailable: 0          # Max unavailable pods during update
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        readinessProbe:          # Ensures traffic only goes to ready pods
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```
<!-- Deployments manage ReplicaSets and provide declarative updates to applications. The standard for stateless workloads. -->

### StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"        # Mandatory headless service
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:          # Each pod gets its own PVC
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```
<!-- StatefulSets provide stable network identities and persistent storage for stateful applications. Pods are created/deleted in order. -->

### PersistentVolume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # Keeps data when PVC is deleted
  storageClassName: standard
  hostPath:                               # For testing only!
    path: /data/postgres-data             # Use proper storage in production
```
<!-- PersistentVolumes represent storage resources in the cluster. Generally provisioned by cluster admins or dynamic provisioners. -->

### PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 10Gi
```
<!-- PVCs are requests for storage by users. Pods reference PVCs, not PVs directly. -->

## Pod Distribution Strategies

### PodAntiAffinity
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # Hard requirement
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web
            topologyKey: "kubernetes.io/hostname"  # Spread across nodes
```
<!-- PodAntiAffinity prevents pods from being scheduled on the same node, ensuring higher availability. -->

### NodeAffinity
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-workload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-training
  template:
    metadata:
      labels:
        app: ml-training
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:  # Hard requirement
            nodeSelectorTerms:
            - matchExpressions:
              - key: gpu-type
                operator: In
                values:
                - nvidia-tesla
          preferredDuringSchedulingIgnoredDuringExecution:  # Soft preference
          - weight: 1
            preference:
              matchExpressions:
              - key: ssd
                operator: Exists
```
<!-- NodeAffinity schedules pods on nodes with specific labels. Useful for hardware requirements or zone distribution. -->

## Zero Downtime Migration

### Stateless Applications

#### Blue-Green Deployment
```yaml
# Simplified example - in practice, use a service mesh or dual services
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: v2      # Change this label to switch traffic
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1    # Blue deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: v1
  template:
    metadata:
      labels:
        app: my-app
        version: v1
    spec:
      containers:
      - name: app
        image: my-app:v1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2    # Green deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: v2
  template:
    metadata:
      labels:
        app: my-app
        version: v2
    spec:
      containers:
      - name: app
        image: my-app:v2
```
<!-- Blue-Green: Run both versions simultaneously and switch traffic all at once. Zero downtime but requires double resources. -->

#### Canary Deployment
```yaml
# Using service mesh like Istio for traffic splitting:
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - my-app
  http:
  - route:
    - destination:
        host: my-app-v1
        subset: v1
      weight: 90
    - destination:
        host: my-app-v2
        subset: v2
      weight: 10
```
<!-- Canary: Route a small percentage of traffic to the new version before full rollout. Minimizes impact of potential issues. -->

### Stateful Applications

#### Database Migration with Replicas
```bash
# Overview of process (for PostgreSQL example):
# 1. Set up replication from primary to new version standby
# 2. Monitor replication lag
# 3. When ready, promote standby to new primary
# 4. Update connection strings in application

# Example using StatefulSet with init containers:
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-v2
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
      version: v2
  template:
    metadata:
      labels:
        app: postgres
        version: v2
    spec:
      initContainers:
      - name: init-db
        image: postgres:14
        command:
        - /bin/bash
        - -c
        - |
          # Example: Set up replication from v1 to v2
          pg_basebackup -h postgres-v1-0.postgres -U replicator -D /var/lib/postgresql/data -P -Xs
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 10Gi
```
<!-- For stateful workloads, replication with failover is the key to zero-downtime migrations. -->

#### Backup and Restore
```yaml
# Example using Velero for backup/restore
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: postgres-backup
spec:
  includedNamespaces:
  - database
  storageLocation: default
  hooks:
    resources:
      - name: postgres-backup-hook
        includedNamespaces:
        - database
        includedResources:
        - pods
        labelSelector:
          matchLabels:
            app: postgres
        pre:
          - exec:
              container: postgres
              command:
              - /bin/bash
              - -c
              - "pg_dump -U postgres -d mydb > /tmp/backup.sql"
```
<!-- For some workloads, consistent backups and restores remain the most reliable migration strategy. -->

## Scalability and High Availability

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
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
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:                          # Fine-tune scaling behavior
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
    scaleUp:
      policies:
      - type: Percent
        value: 100                     # Double the pods if needed
        periodSeconds: 60
```
<!-- HPA automatically scales pods based on metrics. Important for handling variable load efficiently. -->

### Cluster Autoscaler
```yaml
# Configure via cloud provider-specific settings
# GKE example:
# gcloud container clusters update my-cluster --enable-autoscaling --min-nodes=3 --max-nodes=10

# Resources should specify limits and requests
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-intensive-app
spec:
  # ...
  template:
    # ...
    spec:
      containers:
      - name: app
        image: app:v1
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
```
<!-- Cluster Autoscaler adjusts the number of nodes based on pod resource requirements. -->

### Multi-Region Deployment
```yaml
# Use context for different clusters
# kubectl config use-context cluster-us-east1

# Deploy to each region with anti-affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-us-east1
spec:
  replicas: 3
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - global-app
            topologyKey: "topology.kubernetes.io/zone"
      # ...

# Global load balancing via cloud provider or service mesh
```
<!-- Multi-region deployments provide the highest level of availability but increase complexity and cost. -->

### Database HA with Operators
```yaml
# Using PostgreSQL operator (simplified)
apiVersion: acid.zalan.do/v1
kind: postgresql
metadata:
  name: postgres-ha-cluster
spec:
  teamId: "db-team"
  postgresql:
    version: "14"
  numberOfInstances: 3
  enableMasterLoadBalancer: true
  enableReplicaLoadBalancer: true
  volume:
    size: 10Gi
  users:
    app_user: []
  databases:
    app_db: app_user
  resources:
    requests:
      cpu: 100m
      memory: 100Mi
    limits:
      cpu: 500m
      memory: 500Mi
```
<!-- Operators automate complex day-2 operations for stateful workloads like databases. -->

---

**Pro Tips:**
1. Always set resource requests/limits for predictable scaling
2. Use readiness/liveness probes for all services
3. Implement proper logging and monitoring
4. Consider GitOps for reliable deployments
5. Test your migration strategies in a staging environment first
