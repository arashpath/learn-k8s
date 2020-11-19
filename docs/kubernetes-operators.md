# Kubernetes Operators
> Automating the Container Orchestration Platform

## C1: Operators Teach Kubernetes New Tricks
```bash
kubectl run --image=nginx staticweb
kubectl scale deployment staticweb --replicas=3
kubectl get pods
anyPod=$(kubectl get pods -l run=staticweb | awk 'END{print$1}')
kubectl delete pod $anyPod
```

## C2: Running Operators
### Setting Up an Operator Lab
- Install Docker
- Install Kubectl
  ```bash
  curl -LO "https://storage.googleapis.com/kubernetes-release/release/v1.16.15/bin/linux/amd64/kubectl"
  chmod +x ./kubectl
  sudo mv ./kubectl /usr/local/bin/kubectl
  kubectl cluster-info
  kubectl get cs
  ```
- Install Kind
  ```bash
  curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64
  chmod +x ./kind
  sudo mv ./kind /usr/local/bin/kind 
  kind version
  kind completion bash > ~/.bash_completion.d/kind
  kind create cluster --config kind-config.yml 
  ```
- Setup Kind cluster
  ```bash
  #Pre-Pull required images
  docker image pull kindest/node:v1.16.15@sha256:a89c771f7de234e6547d43695c7ab047809ffc71a0c3b65aa54eda051c45ed20
  docker image pull kindest/haproxy:v20200708-548e36db
  # Start k8s cluster using kind
  cat <<EOF | kind create cluster --image=kindest/node:v1.16.15 --config -
  kind: Cluster
  apiVersion: kind.x-k8s.io/v1alpha4
  nodes:
  - role: control-plane
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
  EOF
  ```
- Check cluster
  ```bash
  kubectl version
  kubectl describe clusterrole cluster-admin
  ```
- Running Sample Operater (etcd)
  ```bash
  git clone https://github.com/kubernetes-operators-book/chapters.git
  cd chapters/ch03
  # 1 Custom Resource Definition
  kubectl create -f etcd-operator-crd.yaml
  kubectl get crd
  # 2 Operator Service Account
  kubectl create -f etcd-operator-sa.yaml
  kubectl get serviceaccounts
  # 3 Role
  kubectl create -f etcd-operator-role.yaml
  # 4 Role Binding
  kubectl create -f etcd-operator-rolebinding.yaml
  # 5 Deployment
  kubectl create -f etcd-operator-deployment.yaml
  kubectl get deployments
  kubectl get pods
  # 6 Cluster
  kubectl create -f etcd-cluster-cr.yaml
  kubectl get pods -w
  kubectl describe etcdcluster/example-etcd-cluster
  # Exercising etcd
  kubectl get services --selector etcd_cluster=example-etcd-cluster
  kubectl run --rm -i --tty etcdctl --image quay.io/coreos/etcd --restart=Never -- /bin/sh
    # Run in Container
    export ETCDCTL_API=3
    export ETCDCSVC=http://example-etcd-cluster-client:2379
    etcdctl --endpoints $ETCDCSVC put foo bar
    etcdctl --endpoints $ETCDCSVC get foo
  ```
- Scaling the etcd Cluster
  ```bash
  # change cluster size 4
  kubectl apply -f etcd-cluster-cr.yml
  kubectl get pods -l etcd_cluster=example-etcd-cluster
  # delete any on pod
  anyPod=$(kubectl get pods -l etcd_cluster=example-etcd-cluster | awk 'NR == 2{print $1}')
  kubectl delete pods $anyPod
  kubectl describe etcdcluster/example-etcd-cluster
  kubectl get events --field-selector involvedObject.name=example-etcd-cluster
  # check pod health
  kubectl run --rm -i --tty etcdctl --image quay.io/coreos/etcd --restart=Never -- /bin/sh
    # run in container
    etcdctl --endpoints http://example-etcd-cluster-client:2379 cluster-health
  ```
- Upgrading the etcd cluster
  ```bash
  kubectl get pod -l etcd_cluster -o yaml | grep image: | sort | uniq
  # change cluster ver 3.2.13
  kubectl apply -f etcd-cluster-cr.yml
  kubectl describe etcd example-etcd-cluster | grep "Version:"
  kubectl get events --field-selector involvedObject.name=example-etcd-cluster
  kubectl patch etcdcluster example-etcd-cluster --type='json' -p '[{"op": "replace", "path": "/spec/version", "value":3.3.12}]'
  ```

## C3: Operators at the Kubernetes Interface
### Standard Scaling: The ReplicaSet Resource
- `ReplicaSet` is a collection of API objects.
   - key pices: `Selector`, `Replicas` and `Pod Template` fields
```bash
#kubectl create deployment --image=nginx staticweb
kubectl run --image=nginx staticweb
kubectl describe rs -l run=staticweb
```
- Actions the ReplicaSet controller takes are intentionally general and application agnostic.
- An Operator is the application-specific combination of CRs and a custom controller that does know all the details about starting, scaling, recovering, and managing its application. 
### Custom Resources
- An extention of the kubernetes API to represent new collections of objects in the API.
### Custom Controller
- CR are entries in the kubernetes API Database, CR alone is merely a collection of data. to provide a declarative API for a specific application running on a cluster, yum also need active code that captures the processes of managing that application.
- To make an Operator, providing an API for the active management of an application,  We build an instance of the Controller pattern to control your application. 
- This custom controller checks and maintains the application’s desired state, represented in the CR. Every Operator has one or more custom controllers implementing its application-specific management logic.
### Operator Scopes
- A namespace is the boundary for cluster object and resource name.
  - Names must be unique within a single namespace
  - Resource limits and access controls can be applied per namespace.
- A Operator, in turn, can be limited to a namespace, or it can maintain its operand across an entire cluster.
- **Namespace Scope**: 
  - sensible and more flexible for clusters used by multiple teams.
  - can be upgraded independently of other instances.
- **Cluster-Scoped Operators** :
  - In some situations it is desirable for an Operator to watch and manage an application or services throughout a cluster. e.g. `Isto` operator that manages a sercvice mesh or `cert-manager` that issues TLS certificates for application endpoints.
  - To run operator in cluster-Scope
    - it should watch all namespaces
    - run under the auspices of a `ClusterRole` and `ClusterRoleBinding` rather than namespaced `Role` and `RoleBinding` authorization objects.
### Authorization
> Authorization: the power to do things on the cluster through the API, is defined in kubernetes by one of a few available access control systems. **Role-Based Access Control (RBAC)** 
- **Service Accounts:** are managed by Kubernetes and can be created and manipulated through the Kubernetes API, is used to authorizing programs insted of people.
- **Roles:**
  - Kubernetes RBAC denies permission by default, so a role defines granted rights
    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: default
      name: pod-reader
    rules:
    - apiGroups: [""]
      resources: ["pods"]
      verbs: ["get","watch","list"]
    ```
- **RoleBindings:**
  - ties a role to a list of one or more users. 
  - can referrence only those roles in its own namespace.

- **ClusterRoles and ClusterRoleBindings**: 
  - `Roles` and `RoleBindings` are restricted to a namespace.
  - `ClusterRoles` and `ClusterRoleBindings` are their cluster-wide equivalents.
  - `RoleBindings` when refring a `ClusterRole`,  rules are apply to only those specified resources in the binding's own namespace.
  - a set of common roles can be defined once as `ClusterRoles`, but reused and granted to users in just a given namespace.
  - `ClusterRoleBinding` grants capabilites to a user across the entire cluster. in all namespaces.
  - Operators charged with cluster-wide responsibilites will often tie a `ClusterRole` to an Operator service account with a `ClusterRoleBinding`.

## C4: The Operator Framework
- The Red Hat Operator Framwork make it simpler to create and distrubute Operators.
  - SDK: software development kit to build operators
  - OLM: Operator Lifecycle Manager is an operator that installs, manages, and upgrades other Operators.
  - Operator Metering is a metrics system that accounts for Operators use of cluster resource.
### Operator Framework Origins
- `operator-sdk` builds atop the Kubernets `controller-runtime`, a set of libraries providing essential Kubernetes controller routines in the Go programming language provides integration points for distributing and managing Operators with OLM, and measuring them with Operator Metering.
### Operator Maturity Model
- **Phase I** ( Go | Andible | Helm )
  > **Basic Install:** Automated application provisioning and configuration management. 
- **Phase II** ( Go | Andible | Helm )
  > **Seamless Upgrades:** Pach and minor version upgrades supported
- **Phase III** ( Go | Ansible )
  > **Full Lifecycle:** App lifecycle storage lifecycle (backup,failure,recovery)
- **Phase IV** ( Go | Ansible )
  > **Deep Insights:** Metrics, alerts, log processing and workload analysis
- **Phase V** ( Go | Ansible )
  > **Auto Pilot:** Horizontal/Vertical scalling, auto config tuning, abnormal detection, schedule tuning
### Operator SDK
> set of tools for scaffolding, building, and preparing an Operator for deployment. `Go` languange, adapter architecture for `Helm` charts or `Ansible` playbooks.
- Binary Installation
  ```bash
  RELEASE_VERSION=v1.2.0
  for operator in operator-sdk ansible-operator helm-operator 
  do
    curl -LO https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}/${operator}-${RELEASE_VERSION}-x86_64-linux-gnu
    gpg --verify ${operator}-${RELEASE_VERSION}-x86_64-linux-gnu.asc
    chmod +x ${operator}-${RELEASE_VERSION}-x86_64-linux-gnu
    #sudo cp ${operator}-${RELEASE_VERSION}-x86_64-linux-gnu /usr/local/bin/${operator}
    ln -s ${operator}-${RELEASE_VERSION}-x86_64-linux-gnu ${operator}
  done
  ```
### Operator Lifecycle Manager
- OLM defines a schema for Operator metadata - Cluster Service Version (CSV), for describing an Operator and its dependencies.
- Operators with a CSV can be listed as entries in a *catalog* available ot OLM running on a Kubenetes cluster.
- User then *subscribe* to an Operator fromt the catalog to tell OLM to provision ant manage a desired Operator.
- That Operator in turn, provisions and manages its application or service on the cluster.
### Operator Metering
> A system for analyzing the resource usage of the Operators running on Kubernetes clusters. 
- *Budgeting* 
- *Billing*
- *Metrics aggregation*

## C5: Sample Application: Visitors Site
- Application Overview
  - A web frontend, implemented in `React`
  - A REST API, implemented in `Python` using `the Django framework`
  - A database, using `MySQL`
- Installation with Manifests
  - `Deployment`: Contains the information needed to create the containers, including the image name, exposed ports, and specific configuration for a single deployment.
  - `Service`: A network abstraction across all containers in a deployment. If a deployment is scaled up beyond one container, which we will do with the backend, the service sits in front and balances incoming requests across all of the replicas.
- Deploying the Manifests
  ```bash
  kubectl apply -f ch05/database.yml #MySQL
  kubectl apply -f ch05/backend.yml  #Backend
  kubectl apply -f ch05/frontend.yml #Fronend
  ```