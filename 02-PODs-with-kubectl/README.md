# K8S AND NODES INSTALLATION
#!/bin/bash
# Common stages for both master and worker nodes
# This can be use as user data in launch template or launch configutions
sudo hostname node1
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

sudo apt update -y
sudo apt install -y apt-transport-https -y

sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt update -y
sudo apt install -y kubelet kubeadm  containerd kubectl
# apt-mark hold will prevent the package from being automatically upgraded or removed.

sudo apt-mark hold kubelet kubeadm kubectl containerd

sudo cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# Enable and start kubelet service
sudo systemctl daemon-reload
sudo systemctl start kubelet
sudo systemctl enable kubelet.service
# common for master and worker nodes commands ends
# sudo kubeadm init
  
===============================================================
= == ===========================================================
  

  
  # Landmark Technologies  == Kubernetes  - POD

## Step-01: PODs Introduction
- What is a POD ?
- What is a Multi-Container POD?

## Step-02: PODs Demo
### Get Worker Nodes Status
- Verify if kubernetes worker nodes are ready. 
```
# Get Worker Node Status
kubectl get nodes

# Get Worker Node Status with wide option
kubectl get nodes -o wide
```

### Create a Pod
- Create a Pod
```
# Template
kubectl run <desired-pod-name> --image <Container-Image> --generator=run-pod/v1

# Replace Pod Name, Container Image
kubectl run my-first-pod --image mylandmarktech/hello --generator=run-pod/v1
```
- **Important Note:** Without **--generator=run-pod/v1** it will create a pod with a deployment which is another core kubernetes concept which we will learn in next few minutes. 
- **Important Note:**
  - With **Kubernetes 1.18 version**, there is lot clean-up to **kubectl run** command.
  - The below will suffice to create a Pod as a pod without creating deployment. We dont need to add **--generator=run-pod/v1**
```
kubectl run my-first-pod --image mylandmarktech/hello
```  

### List Pods
- Get the list of pods
```
# List Pods
kubectl get pods

# Alias name for pods is po
kubectl get po
```

### List Pods with wide option
- List pods with wide option which also provide Node information on which Pod is running
```
kubectl get pods -o wide
```

### What happened in the background when the above command is run?
  1. Kubernetes created a pod
  2. Pulled the docker image from docker hub
  3. Created the container in the pod
  4. Started the container present in the pod


### Describe Pod
- Describe the POD, primarily required during troubleshooting. 
- Events shown will be of a great help during troubleshooting. 
```
# To get list of pod names
kubectl get pods

# Describe the Pod
kubectl describe pod <Pod-Name>
kubectl describe pod my-first-pod 
```

### Access Application
- Currently we can access this application only inside worker nodes. 
- To access it externally, we need to create a **NodePort Service**. 
- **Services** is one very very important concept in Kubernetes. 


### Delete Pod
```
# To get list of pod names
kubectl get pods

# Delete Pod
kubectl delete pod <Pod-Name>
kubectl delete pod my-first-pod
```

## Step-03: NodePort Service Introduction
- What are Services in k8s?
- What is a NodePort Service?
- How it works?

## Step-04: Demo - Expose Pod with a Service
- Expose pod with a service (NodePort Service) to access the application externally (from internet)
- **Ports**
  - **port:** Port on which node port service listens in Kubernetes cluster internally
  - **targetPort:** We define container port here on which our application is running.
  - **NodePort:** Worker Node port on which we can access our application.
```
# Create  a Pod
kubectl run <desired-pod-name> --image <Container-Image> --generator=run-pod/v1
kubectl run my-first-pod --image mylandmarktech/hello:4 --generator=run-pod/v1

# Expose Pod as a Service
kubectl expose pod <Pod-Name>  --type=NodePort --port=80 --name=<Service-Name>
kubectl expose pod my-first-pod  --type=NodePort --port=80 --name=my-first-service

# Get Service Info
kubectl get service
kubectl get svc

# Get Public IP of Worker Nodes
kubectl get nodes -o wide
```
- **Access the Application using Public IP**
```
http://<node1-public-ip>:<Node-Port>
```

- **Important Note about: target-port**
  -  If target-port is not defined, by default and for convenience, the **targetPort** is set to the same value as the **port** field.

```
# Below command will fail when accessing the application, as service port (81) and container port (80) are different
kubectl expose pod my-first-pod  --type=NodePort --port=81 --name=my-first-service2     

# Expose Pod as a Service with Container Port (--taret-port)
kubectl expose pod my-first-pod  --type=NodePort --port=81 --target-port=80 --name=my-first-service3

# Get Service Info
kubectl get service
kubectl get svc

# Get Public IP of Worker Nodes
kubectl get nodes -o wide
```
- **Access the Application using Public IP**
```
http://<node1-public-ip>:<Node-Port>
```

## Step-05: Interact with a Pod

### Verify Pod Logs
```
# Get Pod Name
kubectl get po

# Dump Pod logs
kubectl logs <pod-name>
kubectl logs my-first-pod

# Stream pod logs with -f option and access application to see logs
kubectl logs <pod-name>
kubectl logs -f my-first-pod
```
- **Important Notes**
  - Refer below link and search for **Interacting with running Pods** for additional log options
  - Troubleshooting skills are very important. So please go through all logging options available and master them.
  - **Reference:** https://kubernetes.io/docs/reference/kubectl/cheatsheet/

### Connect to Container in a POD
- **Connect to a Container in POD and execute commands**
```
# Connect to Nginx Container in a POD
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it my-first-pod -- /bin/bash

# Execute some commands in Nginx container
ls
cd /usr/share/nginx/html
cat index.html
exit
```

- **Running individual commands in a Container**
```
kubectl exec -it <pod-name> env

# Sample Commands
kubectl exec -it my-first-pod env
kubectl exec -it my-first-pod ls
kubectl exec -it my-first-pod cat /usr/share/nginx/html/index.html
```
## Step-06: Get YAML Output of Pod & Service
### Get YAML Output
```
# Get pod definition YAML output
kubectl get pod my-first-pod -o yaml   

# Get service definition YAML output
kubectl get service my-first-service -o yaml   
```

## Step-07: Clean-Up
```
# Get all Objects in default namespace
kubectl get all

# Delete Services
kubectl delete svc my-first-service
kubectl delete svc my-first-service2
kubectl delete svc my-first-service3

# Delete Pod
kubectl delete pod my-first-pod

# Get all Objects in default namespace
kubectl get all
```
