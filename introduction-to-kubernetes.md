## Introduction to Kubernetes


Kubernetes is a tool for container orchestration and is composed of a set of node machines, which are classified into master node and worker node.

Master node: It is where the control plane is responsible for managing the state of all clusters

• The high availability of the master node must be ensured through replicas
• The persistence in the state of a cluster is done in an etcd (Key-value distributed storage)
• Etcd can be configured on the master node (Stacked topology) or on a dedicated host (External topology)

## Components of the Master:

Kube-apiserver: Coordinates all administrative tasks, intercepts RESTful calls from users, operators and external agents to later validate and process them

• APIServer reads the current state of the cluster through etcd and then makes an execution call, the result is saved in etcd.
• APIServer is the ONLY control plane component to communicate with etcd, it serves as an intermediate interface between etcd and any other control plane agent.

Kube-scheduler: Its role is to assign new workload type objects, such as assigning pods to a node, this is done based on the current state of the cluster and its requirements. This information is obtained from the etcd through the APIServer

• The requirements can be values ​​such as disk == ssd, as well as values ​​for QoS, data locality, affinity, cluster topology, among others.
• The scheduling algorithm filters the nodes, creating pre-candidates to be selected as the nodes with the highest score, that is, the best conditions to host a new workload.
• The output of this decision process is communicated to the API Server and it delegates the deployment of the workload to another agent in the control plane.
• This agent is customizable through scheduling policies, plugins and profiles.

Controller Managers: They are components in the control plane that regulate the state of the clusters, they are a constant cycle that reviews the desired state of the cluster with the current state (obtained through the API Server)

• Kube-controller-manager runs controllers responsible for acting when nodes become unavailable, to ensure that the number of nodes is as expected, to create endpoins, services and API access tokens

etcd is strongly consistent, distributed, and a Key-Value data store to persist the state of a Kubernetes Cluster. Data in etcd is only added, never replaced, obsolete data is periodically compacted

etcdctl (Console Line Interface management tool) provides backup, data snapshots and restore capabilities (It is necessary to replicate these data stores in high availability mode)

