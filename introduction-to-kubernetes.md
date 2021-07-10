## Introduction to Kubernetes


Kubernetes is a tool for container orchestration and is composed of a set of node machines, which are classified into master node and worker node.

A Master node is where the control plane is responsible for managing the state of the clusters

• The high availability of the master node must be ensured through replicas
• The persistence in the state of a cluster is done in an etcd (Key-value distributed storage)
• Etcd can be configured on the master node (Stacked topology) or on a dedicated host (External topology)

## Master node components

**Kube-apiserver:** Coordinates all administrative tasks, intercepts RESTful calls from users, operators and external agents to later validate and process them

• APIServer reads the current state of the cluster through etcd and then makes an execution call, the result is saved in etcd.
• APIServer is the ONLY control plane component to communicate with etcd, it serves as an intermediate interface between etcd and any other control plane agent.

**Kube-scheduler:** Its role is to assign new workload type objects, such as assigning pods to a node, this is done based on the current state of the cluster and its requirements. This information is obtained from the etcd through the APIServer

• The requirements can be values ​​such as disk == ssd, as well as values ​​for QoS, data locality, affinity, cluster topology, among others.
• The scheduling algorithm filters the nodes, creating pre-candidates to be selected as the nodes with the highest score, that is, the best conditions to host a new workload.
• The output of this decision process is communicated to the API Server and it delegates the deployment of the workload to another agent in the control plane.
• This agent is customizable through scheduling policies, plugins and profiles.

**Controller Managers:** They are components in the control plane that regulate the state of the clusters, they are a constant cycle that reviews the desired state of the cluster with the current state (obtained through the API Server)

• Kube-controller-manager runs controllers responsible for acting when nodes become unavailable, to ensure that the number of nodes is as expected, to create endpoins, services and API access tokens

etcd is strongly consistent, distributed, and a Key-Value data store to persist the state of a Kubernetes Cluster. Data in etcd is only added, never replaced, obsolete data is periodically compacted. etcdctl (Console Line Interface management tool) provides backup, data snapshots and restore capabilities (It is necessary to replicate these data stores in high availability mode)

## Worker node

A worker node provides an execution environment for applications through containerized microservices, these applications are encapsulated in PODS, which are smaller units deployable in a Kubernetes

• In a Multi-Worker cluster the network traffic between users and containerized applications deployed in pods is handled directly by the worker nodes.


## Worker Node components

Container Runtime: To manage the lifecycle of a container, Kubernetes requires a container runtime on the node where a Pod and its containers will be scheduled. Kubernetes supports many container runtimes:

**Docker**
**Containerd**
**CHILD**
**Frakti**

**Kubelet:** It is an agent that executes in each node and communicates with the components of the control panel of the master node, this receives the definitions of a Pod, firstly from the API Server, and interacts with the runtime container in the node to execute containers associated with the pod, it also monitors the health of pod resources by running containers

Kubelet connects to the runtime container through an interface called CRI, this interface consists of a buffer protocol, gRPC API and libraries. To connect to the interchangeable container runtime containers, kubelet uses a compensation application that provides a clear abstraction layer between kubelet and the container runtime

**Kube-proxy** is a network agent that runs on each node, it is responsible for dynamic updates and maintenance of all network rules on each node. Kube proxy summarizes the networking details of the pods (redirects, connection requests, etc.) 
