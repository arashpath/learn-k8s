# Kubernetes for the Absolute Beginners
> [Repo:k8s4absbegin](/k8s4absbegin)
## 1. Introduction
## 2. Kubernetes Overview
- Containers Overview
- Container Orchestration 
  - Docker Swarm
  - Kubernetes
  - Mesos
- Kubernetes Architecture
  - K8s Components
    - API Server (kube-apiserver)
    - Scheduler  (kube-scheduler)
    - Controller (kube-controler)
    - etcd - Data Store (etcd)
    - Container Runtime
    - kubelet 
## 3. Setup Kubernetes
- Kubernetes Setup - Introduction and Minikube 
  - Minikube 
  - Kubeadm 
  - GKE
  - https://labs.play-with-k8s.com/
## 4. Kubernetes Concepts
  - pods
    ```sh
    kubectl run nginx --image=nginx
    kubectl get pods
    kubectl describe pods
    kubectl get pods -o wide
    ```
## 5. YAML Introduction
## 6. Kubernetes Concepts - PODs, ReplicaSets, Deployments
- PODs with YAML 
  ```bash
  kubectl create -f c6_pod-definition-k8s.yml
  kubectl get pods
  kubectl describe pod myapp-pod 
  ```
  ```yaml
  apiVersion: v1     # string value
  kind: Pod          # string value
  metadata:          # dict value
    name: myapp-pod
    labels:
      app: myapp
      type: front-end
  spec:              # diffrent every kind          
    containers:
      - name: nginx-container
        image: nginx
  ```
- Replication Controllers and ReplicaSets 
  - Replication Controllers
  ```bash
  kubectl get replicationcontroller
  kubectl get pods
  ```
  ```yaml
  apiVersion: v1
  kind: ReplicationController
  metadata:
    name: myapp-rc
    labels:
      app: myapp
      type: front-end
  spec:
    replicas: 3
    template:
      # contents from pod definition
  ```
  - Replica Sets
  ```bash
  kubectl get replicaset
  # Scale
  kubectl replace -f c6_replicaset-definition.yml
  kubectl scale --replicas=6 -f c6_replicaset-definition.yml
  kubectl scale --replicas=6 -f replicaset myapp-replicaset
  ```
  ```yaml
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name:
    labels:
  spec:
    replicas:
    selector:
    template:
  ```
- Deployments
  ```bash
  kubectl get deployment
  ```
- Deployments - Update and Rollback 
  ```bash
  kubectl create -f c6_deployment-definition-k8s.yml
  kubectl get deployments
  kubectl apply -f c6_deployment-definition-k8s.yml
  kubectl set image deployment/myapp-deployment nginx=nginx:latest
  kubectl rollout status deploymet/myapp-deployment
  kubectl rollout undo deploymet/myapp-deployment
  # --record=true
  ```
## 7. Networking in Kubernetes
- Basics of Networking in Kubernetes 
8. Services/1. Services - NodePort 
8. Services/2. Demo - Services 
8. Services/3. Services - ClusterIP 
9. Microservices Architecture/1. Microservices Application 
9. Microservices Architecture/2. Demo - Deploying Microservices Application on GCP Kubernetes Cluster 
9. Microservices Architecture/3. Demo - Example Voting Application Improvised - v2 
10. Conclusion/1. Conclusion 
10. Conclusion/2. Bonus Lecture Kubernetes Series of Courses 
