+++
title = 'K8s Crash Course'
date = 2024-08-18T20:00:11-07:00
draft = false
tags = ['k8s', 'devops', 'containers', 'cloud', 'kubernetes', 'docker']
+++
Over the past week I spent some  time learning Kubernetes. I'm going to share some of the things I learned and some of the things I'm still learning.

> Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized application

# History
## Way back when....there were "Traditional Deployments"
## 2000's
- On Premises, Bare metal hardware
- Teams of sysadmins handle provisioning  and managing of fleets of servers
- Monoliths.....
- Custom, home grown monitoring and tooling
### 2010s
- VM's starting to gain massive adoption
- Clouds services enables VM's to be created and destroyed 
- Configuration management tolling, and overall imporived tooling
### 2020s
- Containers, containers, containers
- Workload orchestrator's (like Kubernetes!) enable treating clusters of machines as a single resource
- These orchestrators included utilities and interfaces to solve many challenges 


# Resources
## Namespace
- Provide a mechanism to group resources within a cluster
- There a 4 initial namespaces: default, kube-system, kube-node-lease, kube-public
- By default, they do not act as network or secutrity boundary 


 Kubernetes is the conductor of a container orchestra
 Key Features
 - Service Discovery/Load Balancing
 - Storage Orchestration
 - Automate Rollouts/Rollbacks
 - Self-healing
 - Secret and Configuration Management
 - Horizontal Scaling

A Kubernetes cluster consists of two types of resources:

- The **Control Plane** coordinates the cluster
- **Nodes** are the workers that run applications

## Control Plane
- The control plane is responsible for managing the xluster. THe control Plane coordinates all activities in the cluster such as:
	- Scheduling applications
	- Maintaining applications desired state
	- Scaling applications
	- rolling out new updates

## Nodes
- **A node is a VM or a physical computer that serves as a worker machine in a Kubernetes cluste**r
- Each node has a Kublet, an agent for managing the node and communicating with the control plane

## Pods
- The "smallest" deployable unit
- Pod can contain multiple containers
	- Init containers
	- Sidecar containers
- Pod containers share the same Network namespace (share IP/port)
- Have same loopback network interface (localhost)
- Ports can be reused within same node (as long as separate pod)
- You will almost never create a pod directly (should be done via Files)
simple pod yaml file
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
    rel: stable
spec:
  containers:
  - name: my-nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
    resources: {}
```

### Probes
- A probe is a diagnostic performed periodically by the kublet on a container 
- Types of probes:
	- Liveness, can be used to determine if a Pod is healthy and running as expected
		- When should a container restart?
	- Readiness probes can be used to determine if a pod is ready to accept requests
		- When should a container start receiving traffic? 
 - These should be set during `.yaml` file creation 
 - Actions you can perform on probes
	 - ExecAction - Executes an action inside the container
	 - TCPSocketAction - TCP check against the containers IP address on a specified port
	 - HTTPGetAction - HTTP GET request against a container
- Probes can have the following results
	- Success
	- Failure
	- Unknown 
simple pod file with probes
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
  labels:
    app: nginx
    rel: stable
spec:
  containers:
  - name: my-nginx
    image: nginx:alpine
    resources:
      limits:
        memory: "128Mi" #128 MB
        cpu: "200m" #200 millicpu (.2 cpu or 20% of the cpu)
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /index.html
        port: 80
      initialDelaySeconds: 15
      timeoutSeconds: 2 # Default is 1
      periodSeconds: 5 # Default is 10
      failureThreshold: 1 # Default is 3
    readinessProbe:
      httpGet:
        path: /index.html
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5 # Default is 10
      failureThreshold: 1 # Default is 3
```

## Replica Set
-  A `ReplicaSet` ensures that a specified number of pod replicas are running at any given time. If a pod crashes or is deleted, the ReplicaSet automatically creates a new pod to replace it, maintaining the desired number of replicas.
-  ReplicaSets act a a Pod controller
	- Self healing mechanism
	- Ensure the requested number of Pods are available
	- Provide fault-tolerance
	- Can be used to scale Pods
	- Relies on a Pod template
- You will almost never create a ReplicaSet directly
- Labels are the link between ReplicaSets and Pods 
- 

## Deployments
- A `Deployment` provides a declarative way to preform updates to applications. It manages the creation and scaling of `ReplicaSets`,  `Deployments` are a higher-level concept that includes many additional features beyond what `ReplicaSets` offer.
-  If you are deploying stateless applications on Kubernetes, a `Deployment` is generally the resource type you should use.
- A `Deployment` ends up deploying a `ReplicaSet` but adds the following functionality:
	1. **Declarative Updates:** Declaratively update your applications. You specify the desired state in the Deployment configuration, and the Deployment controller will ensure that the current state matches the desired state.
	2. **Rolling Updates:** update the pods in a controlled manner with zero downtime. This allows you to update your application to a new version while keeping the old version running until the new version is ready.
	3. **Rollback:** If an update fails, you can easily roll back to a previous revision of the Deployment. Kubernetes keeps track of Deployment revisions for easy rollback.
- A deployment manages Pods, essentially a wrapper over ReplicaSet:
	- Pods are managed using ReplicaSets
	- Scales ReplicaSets, which scale Pods
	- Supports zero-downtime updates by creating and destroying ReplicaSets
	- Creates a unique label that is assigned to the ReplicaSet and generate Pods
	- YAML is very similar to a ReplicaSet
simple deployment file   
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "128Mi" #128 MB
            cpu: "200m" #200 millicpu (.2 cpu or 20% of the cpu)
```
**note**: this will also create ReplicaSet


# Services
- Each pod is assigned an IP address to make it reachable via the network, but the pods are considered ephemeral and may be deleted at any time.
- To provide a stable way to address a set of pods (e.g. from a `Deployment`) we use a `Service`. There are a variety of kinds of services that provide access to pods from within or outside of the cluster.
- A service provides a single point of entry for accessing one or more pods
	> Since pods live and die, can we rely on Pod IP Addresses?
- No! That where services come in 
- Abstract Pod IPS addresses from consumers
- Load balances between pods
- Relies on labels to assosiate a service with a pod
- Nodes kube-proxy creates a virtual IP for services
- Layer 4 (TCP/UDP over IP)
- Services are not ephemeral
## Service Types
- **ClusterIP (default)**: Exposes the Service on a cluster-internal IP. Only accessible from within the cluster.
- **NodePort**: Exposes the Service on each Node’s IP at a static port. External traffic can reach the service via `<NodeIP>:<NodePort>`.
- **LoadBalancer**: Exposes the Service externally using a cloud provider's load balancer.
- **ExternalName**: Maps the Service to a DNS name, useful for external services outside the cluster.
![K8s Service Types](/images/service-types.png)

### Service .yaml file example
```yaml
apiVersion: v1
kind: Service
metadata:
 name: nginx-loadbalancer
spec:
 type: LoadBalancer
 selector:
    app: my-nginx
 ports:
  - name: "80"
    port: 80
    targetPort: 80
```

# Job
- In Kubernetes, a `Job` creates one or more Pods and ensures that a specified number of them complete successfully.
- `Jobs` are used for short-lived, one-time tasks such as batch processing and other short-lived operations. 
- Example `.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: echo-date-better
  namespace: 04--job
spec:
  parallelism: 2
  completions: 2
  activeDeadlineSeconds: 100
  backoffLimit: 1
  template:
    metadata:
      labels:
        app: echo-date
    spec:
      containers:
        - name: echo
          image: busybox:1.36.1
          command: ["date"]
      restartPolicy: Never
```
- Comes with properties to set `parallelism, completions, active deadlines, and backoffLmits`
## CronJob
- Adds the concept of a "schedule" to jobs
- Used for periodic execution of workloads that run to completion 
- This will essentially create a Job that runs on specified schedule, a regualr Job will only run at creation 

# DaemonSet
- A `DaemonSet` ensures that all (or some) nodes run a copy of a `Pod`.
- As nodes are added to the cluster, pods are added to them. As nodes are removed from the cluster, those pods are garbage collected.
- `DaemonSets` are typically used for deploying system-level agents and tools, such as log collectors, monitoring agents, or other utilities that should run on every node.

# StatefulSet
- In Kubernetes, a `StatefulSet` is used to manage stateful applications. Unlike `Deployments`, `StatefulSets` maintain a sticky identity for each of their `Pods`.
- These `Pods` are created from the same spec but are not interchangeable: each has a persistent identifier that it maintains across any rescheduling.  
- Enables configuring workloads that require state management (like primary vs read-replica databases)
# Storage
## Volumes
- Can be used to hold data and state for pods and containers 
-  A Pod can have multiple Volumes attached to it
- Containers rely on a mountPath to access a volume
### Volume Types

- **emptyDir**:
    
    - A volume that is initially empty and is created when a Pod is assigned to a Node. The data in an `emptyDir` volume is stored on the node's filesystem and is deleted when the Pod is removed from the node.
    - **Use Case**: Temporary storage, caching, or when you need a scratch space for a Pod.
- **hostPath**:
    
    - A volume that mounts a file or directory from the host node's filesystem into a Pod. This allows a Pod to access specific files or directories on the host.
    - **Use Case**: Accessing host-level resources, such as logs or Docker sockets. Be cautious as this can tie your Pod to a specific node and pose security risks.
- **nfs**:
    
    - A volume that allows Pods to mount a Network File System (NFS) share. This enables Pods to read and write data on a remote NFS server.
    - **Use Case**: Sharing data between multiple Pods or across nodes, persistent storage with remote access.
- **configMap/secret**:
    
    - Volumes that allow Pods to consume configuration data or sensitive information (like passwords) as files or environment variables. ConfigMaps handle general configuration, while Secrets handle sensitive data.
    - Two primary styles
	    - Property Like (will show as env variables within a container)
	    - File like (will be within a file in the container)
	- Secrets are similar to config maps, however the data is `base64` encoded (to support binary data, NOT a security mechanism) 
	- [Secret Type](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types)
    - **Use Case**: Storing configuration files, environment variables, certificates, or credentials securely.
- **persistentVolumeClaim (PVC)**:
     - Provides API for creating, managing, and consuming storage that lives beyone the life on an individual pod
    - A volume type that abstracts the underlying storage resource and binds to a Persistent Volume (PV). PVCs allow you to request specific storage resources, such as size and access modes, from the cluster.
    - Acess Modes:
	    - ReadWriteOnce (and ReadWriteOncePod)
	    - ReadOnlyMany
	    - ReadWriteMany
	- Reclaim Policy: Retain vs Delete 
    - **Use Case**: Persistent storage for stateful applications, such as databases, where data needs to persist across Pod restarts or rescheduling.
    ![Persistant Volume Claim](/images/pvc.png)
- **Cloud**:
    
    - Cloud provider-specific volumes that integrate with cloud storage services, such as AWS EBS (Elastic Block Store), GCP Persistent Disk, or Azure Disk. These volumes provide managed, scalable, and persistent storage solutions.
    - **Use Case**: Persistent storage in cloud environments, often used for databases, file storage, or any application requiring reliable and durable storage.


# Ingres
- Enables routing traffic to many services via single external LoadBalancer
- Many options to choose from, like Ingress-nginx, HAProxy, Istio...
- Only officially supports layer 7 routing, however layer 4 support is there ([GatewayAPI](https://kubernetes.io/docs/concepts/services-networking/gateway/))
![Ingress](/images/ingress.png)

# Enter Helm....What is it?
- Helm is the de-facto standard for distributing software for Kubernetes 
- It is a combination of:
	- Package manager
	- Templating engine
- Primary use cases:
	- Application deployment
	- Environment management (staging vs prod..)
- Commands
	- `helm install / helm upgrade`
	- `helm rollback`
    ![Helm](/images/helm.png)



### Sources
- https://www.youtube.com/watch?v=2T86xAtR6Fo&t=12477s
- https://www.boot.dev/lessons/02957d87-58be-4fb5-b2b2-768170ed7860
- 
