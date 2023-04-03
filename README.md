# Run MySQL and PHPMyAdmin with Kubernetes 

### Uninstall Docker and Kubernetes and clean install
This installation script will remove all the Docker containers, images, and volumes from your system, 
uninstall Docker and Kubernetes, and then set up a new Kubernetes cluster on your machine. 
It will also configure necessary ports, network settings, and install container runtime.

To uninstall Docker and Kubernetes, run the following commands:
```
sudo docker stop $(sudo docker ps -a -q)
sudo docker rm $(sudo docker ps -a -q)
sudo docker rmi $(sudo docker images -q)
sudo docker volume rm $(sudo docker volume ls -q)

sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get purge docker-ce docker-ce-cli containerd.io
sudo rm -rf /var/lib/docker /etc/docker

sudo apt-get remove kubeadm kubectl kubelet
sudo apt-get purge kubeadm kubectl kubelet
sudo apt-get autoremove

sudo rm -rf /etc/kubernetes
```
### Remove remaining configuration files
To remove any remaining Kubernetes configuration files, run:
```
sudo rm -rf /etc/kubernetes
```
To remove any remaining Docker configuration files, run:
```
sudo rm -rf /etc/containerd
sudo rm -rf /var/lib/dockershim
sudo rm -rf /var/lib/docker
sudo rm -rf /var/run/docker.sock
```
To remove any remaining Kubernetes files, run:
```
sudo rm -rf ~/.kube/
sudo rm -rf /etc/kubernetes/
sudo rm -rf  /var/lib/etcd
```
To disable the Kubernetes apt repository, run:
```
sudo rm /etc/apt/sources.list.d/kubernetes.list
```
### Configure necessary ports
To configure necessary ports for Kubernetes, run:
```
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10259/tcp
sudo ufw allow 10257/tcp

sudo ufw status
```
### Add machine IP addresses to hosts file
To add machine IP addresses to the hosts file, run:
```
echo $(hostname -I) $(hostname) | sudo tee -a /etc/hosts
nano /etc/hosts
<vm-ip> master-ext
<vm-ip> worker-1
<vm-ip> worker-2
<vm-ip> alibaba
```
### Set up Kubernetes runtime and enable forwarding features
To configure Kubernetes runtime and enable forwarding features, run:
```
sudo swapoff -a
sudo grep -i swap /etc/fstab

sudo tee /etc/modules-load.d/kubernetes.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo lsmod | grep netfilter

sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

export -p NEEDRESTART_MODE="a"
echo "\$nrconf{restart} = 'a';" | sudo tee -a /etc/needrestart/needrestart.conf
```
### Install container runtime
To install container runtime, run:
```
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

```

Next, we install the required packages - curl, gnupg2, software-properties-common, apt-transport-https, and ca-certificates. 
We also add the Docker repository, and install containerd.io.
```
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update
sudo apt install -y containerd.io
```
Then, we configure containerd by creating the /etc/containerd/config.toml file and restarting the containerd service.
```
sudo su -
mkdir -p /etc/containerd
containerd config default>/etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd
```
Finally, we enable the systemd cgroup driver.
```
systemd cgroup driver
```

### Installing kubeadm, kubelet and kubectl on the control-plane node
Now that we have installed the container runtime, we can move on to installing the Kubernetes components - kubeadm, kubelet and kubectl.

We add the Kubernetes apt keyring and repository to our system.
```
sudo curl -sLo /etc/apt/trusted.gpg.d/kubernetes-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

sudo apt-add-repository -y "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```
We then install kubeadm, kubelet and kubectl.
```
sudo apt install -y kubelet kubeadm kubectl

sudo systemctl enable kubelet
```
After installing these packages, we can join the node to an existing Kubernetes cluster by running the appropriate command provided by the cluster administrator.

Alternatively, we can create a new Kubernetes cluster by running the kubeadm init command. 
We provide the --pod-network-cidr flag to specify the Pod network CIDR, and the --cri-socket flag to specify the socket file used by the containerd runtime. 
We also provide the --upload-certs flag to upload certificates to the Kubernetes API server, and the --control-plane-endpoint flag to specify the external endpoint of the control plane. 
We ignore all preflight checks by providing the --ignore-preflight-errors=all flag.

```
sudo kubeadm config images pull
```

```
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket /run/containerd/containerd.sock \
  --upload-certs \
  --control-plane-endpoint=<master> \
  --ignore-preflight-errors=all --v=5
```
***
```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join <master>:6443 --token qorbai.bsyt1ygjp1z2ywgc \
        --discovery-token-ca-cert-hash sha256:18fa68750af99a3767fd599f82b97ade28ad20472d87c0765b61b774fcd6b0e7 \
        --control-plane --certificate-key a4a817053c1b12625fc09b2017b6910d4d6574f5d2da3a6742d26c316189f631

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <master>:6443 --token mrs1bm.6mh7cryi1nga8xwt --discovery-token-ca-cert-hash \
		sha256:18fa68750af99a3767fd599f82b97ade28ad20472d87c0765b61b774fcd6b0e7
```
***

Check cluster status:
```
kubectl cluster-info
```
### Download installation manifest.
```
wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

!If your Kubernetes installation is using custom podCIDR (not 10.244.0.0/16) you need to modify the network to match your one in downloaded manifest.
```
nano kube-flannel.yml

net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

### Then install Flannel by creating the necessary resources.
```
kubectl apply -f kube-flannel.yml
```
Confirm flannel pods are ready.
```
kubectl get pods -n kube-flannel
```
Check nodes;
```
kubectl get nodes -o wide
```
### Generating join token
If the join token is expired;
```
 kubeadm token create --print-join-command
 kubeadm init phase upload-certs --upload-certs
```

## Deploy App( MySQL and Phpmyadmin) on cluster

To run MySQL and PHPMyAdmin on an existing Kubernetes cluster, you will need to create the following YAML files:

1. pv.yaml (Persistent Volume)
2. pvc.yaml (Persistent Volume Claim)
3. storageclass.yaml (Storage Class)
4. mysql.yaml (MySQL deployment and service)
5. phpmyadmin.yaml (PHPMyAdmin deployment and service)

Here is an overview of each of these files:

### 1. pv.yaml (Persistent Volume)
A Persistent Volume (PV) is a storage resource that is provisioned by an administrator and can be used by a Kubernetes cluster. It can be thought of as a network-attached hard drive that can be shared across multiple pods. 
The PV.yaml file is used to create a Persistent Volume for your MySQL database.

Here is the pv.yaml file:

***
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /data/mysql-pv
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-2

```
***

This YAML file defines a Persistent Volume named mysql-pv with a storage capacity of 10GB. It uses a Storage Class named local-storage and the access mode is set to ReadWriteMany, 
meaning that only one pod can use the volume at a time. The hostPath is set to /data/mysql-pv, which is the location on the node where the PV will be mounted.

### 2. pvc.yaml (Persistent Volume Claim)

A Persistent Volume Claim (PVC) is a request for storage resources by a pod. It is used to bind a pod to a Persistent Volume. 
The PVC.yaml file is used to create a Persistent Volume Claim for your MySQL database.

Here is the pvc.yaml file:
***
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

```
***

This YAML file defines a Persistent Volume Claim named mysql-pvc with a storage capacity of 10GB. It uses the same Storage Class named local-storage and access mode ReadWriteOnce as the PV. 
The PVC will request the storage resources from the PV and bind the pod to the PV.

### 3. storageclass.yaml (Storage Class)

A Storage Class is used to define the type of storage that is available to your Kubernetes cluster. It defines the provisioner that will be used to provision the storage resources, 
as well as any other parameters that are needed for storage provisioning. The storageclass.yaml file is used to create a Storage Class for your MySQL database.

Here is the storageclass.yaml file:

***
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

```
***

This YAML file defines a Storage Class named local-storage. It uses a provisioner named kubernetes.io/no-provisioner, which means that the storage resources will be provisioned by the administrator. 
The volumeBindingMode is set to WaitForFirstConsumer, meaning that the PV will not be bound until a pod requests storage resources.

### 4. mysql.yaml (MySQL deployment and service)

The MySQL deployment and service YAML file will create a deployment and service for the MySQL database. 
This will allow multiple replicas of the pod to be created, providing high availability and scalability for your database.

Before creating mysql.yaml create secret for mysql 
```
kubectl create secret generic mysql-secrets --from-literal=MYSQL_ROOT_PASSWORD=example --from-literal=MYSQL_DATABASE=example --from-literal=MYSQL_USER=example --from-literal=MYSQL_PASSWORD=example
```

Here is the mysql.yaml file:

***
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
spec:
  selector:
    matchLabels:
      app: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      nodeSelector:
        kubernetes.io/hostname: worker-2
      tolerations:
      - key: "node-role.kubernetes.io/worker"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
        - name: mysql
          image: mysql:8
          envFrom:
          - secretRef:
              name: mysql-secrets
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-persistent-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  type: NodePort
  selector:
    app: mysql
  ports:
  - name: mysql
    port: 3306
    targetPort: 3306
    nodePort: 31000
  clusterIP: "10.101.99.126"

```
***


This YAML file defines a Deployment for MySQL with a single replica. It uses the latest version of the MySQL image and sets the `MYSQL_ROOT_PASSWORD` environment variable to `mypassword`. 
The container listens on port 3306 and mounts the `mysql-persistent-storage` volume to `/var/lib/mysql`.
A volume named `mysql-persistent-storage` is also defined in the Deployment, which is linked to the `mysql-pvc` PVC.
A Service is also defined for the MySQL deployment, which exposes the MySQL port 3306 within the cluster.
Also nodePort exposes the MySQL port 31000 for remote connection.

### 5. phpmyadmin.yaml (PHPMyAdmin deployment and service)

The PHPMyAdmin deployment and service YAML file will create a deployment and service for the PHPMyAdmin interface.

Here is the phpmyadmin.yaml file:

***
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
spec:
  selector:
    matchLabels:
      app: phpmyadmin
  replicas: 1
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec: # Add this line  dnsPolicy: ClusterFirstWithHostNet
      nodeSelector:
        kubernetes.io/hostname: worker-2
      tolerations:
      - key: "node-role.kubernetes.io/worker"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: phpmyadmin
        image: phpmyadmin/phpmyadmin
        env:
        - name: PMA_HOST
          value: "10.101.99.126" # MySQL clusterIP
        - name: PMA_PORT
          value: "3306"
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: my-registry-key

---
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin-service
spec:
  selector:
    app: phpmyadmin
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30000 # You can specify any available port number

```
***


This YAML file defines a Deployment for PHPMyAdmin with a single replica. It uses the latest version of the PHPMyAdmin image and sets the `PMA_HOST` environment variable to `10.101.99.126`, which is the `clusterIP` of the MySQL service. 
The `PMA_USER` environment variable is set to `root`. The container listens on port 80. A Service is also defined for the PHPMyAdmin deployment, which is of type `NodePort` and exposes port 30000 to the outside world. 
The selector is set to `app: phpmyadmin`, which will match the label defined in the Deployment.
