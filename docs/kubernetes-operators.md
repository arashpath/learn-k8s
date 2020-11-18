# Kubernetes Operators
> Automating the Container Orchestration Platform

## C1: Operators Teach Kubernetes New Tricks

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
  kubectl delete pods $(kubectl get pods -l etcd_cluster=example-etcd-cluster | awk 'END{print $1}')
  kubectl describe etcdcluster/example-etcd-cluster
  kubectl delete pods $(kubectl get pods -l etcd_cluster=example-etcd-cluster | awk 'NR == 2{print $1}')
  kubectl get events --field-selector involvedObject.name=example-etcd-cluster
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
