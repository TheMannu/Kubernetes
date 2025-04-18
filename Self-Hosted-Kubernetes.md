
# Self-Hosted Kubernetes Installation Guide

This guide provides a step-by-step process to install a self-hosted Kubernetes cluster on a master node and worker node(s). It includes detailed explanations for each command and step to ensure a smooth setup.

---

## Prerequisites

- **Ubuntu OS** (recommended for both master and worker nodes).
- **sudo privileges** on both master and worker nodes.
- Stable internet connection.

---

## Step 1: Install Docker and Kubernetes Tools

### On Both Master and Worker Nodes

1. **Switch to the root user**:
   ```bash
   sudo su
   ```

2. **Update the package list**:
   ```bash
   sudo apt-get update -y
   ```

3. **Install Docker**:
   Docker is required to run Kubernetes containers.
   ```bash
   sudo apt-get install docker.io -y
   ```

4. **Restart Docker service**:
   Ensure Docker is running after installation.
   ```bash
   sudo service docker restart
   ```

5. **Install Kubernetes dependencies**:
   Install `apt-transport-https`, `ca-certificates`, `curl`, and `gpg` to securely download Kubernetes packages.
   ```bash
   sudo apt-get install -y apt-transport-https ca-certificates curl gpg
   ```

6. **Add Kubernetes repository key**:
   Download and add the Kubernetes GPG key to your system.
   ```bash
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   ```

7. **Add Kubernetes repository**:
   Add the Kubernetes repository to your system's package sources.
   ```bash
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

8. **Update the package list again**:
   Refresh the package list to include the newly added Kubernetes repository.
   ```bash
   sudo apt-get update
   ```

9. **Install Kubernetes tools**:
   Install `kubelet`, `kubeadm`, and `kubectl` (Kubernetes command-line tools).
   ```bash
   sudo apt-get install -y kubelet kubeadm kubectl
   ```

10. **Add Google Cloud's Kubernetes repository**:
    Add the official Kubernetes repository for additional packages.
    ```bash
    sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
    ```

11. **Install specific versions of Kubernetes tools**:
    Install specific versions of `kubeadm`, `kubectl`, and `kubelet` (e.g., version 1.20.0).
    ```bash
    sudo apt-get update
    sudo apt install kubeadm=1.20.0-00 kubectl=1.20.0-00 kubelet=1.20.0-00 -y
    ```

---

## Step 2: Initialize the Kubernetes Cluster

### On the Master Node

1. **Initialize the cluster**:
   Use `kubeadm init` to initialize the Kubernetes master node. Replace the `--pod-network-cidr` value with your desired network range.
   ```bash
   kubeadm init --pod-network-cidr=192.168.0.0/16
   ```

2. **Set up kubeconfig**:
   Configure `kubectl` to interact with the cluster.
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

   Alternatively, if you are the root user, you can set the `KUBECONFIG` environment variable:
   ```bash
   export KUBECONFIG=/etc/kubernetes/admin.conf
   ```

3. **Install a network plugin**:
   Install Calico for networking between pods.
   ```bash
   kubectl apply -f https://docs.projectcalico.org/v3.20/manifests/calico.yaml
   ```

4. **Install Ingress Nginx**:
   Install the Ingress Nginx controller for managing external access to services.
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
   ```

5. **Verify the master node**:
   Check if the master node is ready.
   ```bash
   kubectl get nodes
   ```

---

## Step 3: Join Worker Nodes to the Cluster

### On Worker Nodes

1. **Join the worker node to the cluster**:
   Use the `kubeadm join` command provided after initializing the master node. Replace the IP address, token, and hash with your values.

   ```bash
   kubeadm join 172.31.44.213:6443 --token jr32xy.wd2a9wo7n87bkqn7 --discovery-token-ca-cert-hash sha256:4f432d1258179f953af37a32ce6e386c75dbc548f1d28b636212142050276002
   ```

   Alternatively, you can use a multi-line command:
   ```bash
   kubeadm join 172.31.44.213:6443 \
       --token jr32xy.wd2a9wo7n87bkqn7 \
       --discovery-token-ca-cert-hash sha256:4f432d1258179f953af37a32ce6e386c75dbc548f1d28b636212142050276002
   ```

---

## Step 4: Deploy Applications on Kubernetes

### On the Master Node

1. **Create a directory for MongoDB data**:
   ```bash
   mkdir -p /home/ubuntu/mongo/mongo-vol
   ```

2. **Create a PersistentVolume and PersistentVolumeClaim**:
   Save the following YAML code in a file named `mongo-pv-pvc.yaml`:
   ```yaml
   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: mongo-pv
   spec:
     capacity:
       storage: 1Gi
     accessModes:
       - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     hostPath:
       path: /home/ubuntu/mongo/mongo-vol
   ---
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: mongo-pvc
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
   ```

   Apply the YAML file:
   ```bash
   kubectl apply -f mongo-pv-pvc.yaml
   ```

   Verify the PersistentVolume and PersistentVolumeClaim:
   ```bash
   kubectl get pv
   kubectl get pvc
   ```

3. **Create a Secret for MongoDB credentials**:
   Save the following YAML code in a file named `secret.yaml`:
   ```yaml
   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: mongodb-secret
   type: Opaque
   data:
     username: YWRtaW4=  # "admin" in Base64
     password: MTIz      # "123" in Base64
   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: secret
   type: Opaque
   data:
     nKiuser: YWRtaW4=  # "admin" in Base64
     nKipass: MTIz      # "123" in Base64
     meuser: YWRtaW4=   # "admin" in Base64
     mepass: MT17       # Invalid Base64 (correct this if necessary)
   ```

   Apply the YAML file:
   ```bash
   kubectl apply -f secret.yaml
   ```

   Verify the Secret:
   ```bash
   kubectl get secrets
   kubectl describe secret mongodb-secret
   kubectl describe secret secret
   ```

4. **Deploy MongoDB and Mongo Express**:
   Save the following YAML code in a file named `mongo-deployment.yaml`:
   ```yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mongodb
     labels:
       app: mongodb
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: mongodb
     template:
       metadata:
         labels:
           app: mongodb
       spec:
         containers:
           - name: mongodb
             image: mongo
             ports:
               - containerPort: 27017
             env:
               - name: MONGO_INITDB_ROOT_USERNAME
                 valueFrom:
                   secretKeyRef:
                     name: mongodb-secret
                     key: username
               - name: MONGO_INITDB_ROOT_PASSWORD
                 valueFrom:
                   secretKeyRef:
                     name: mongodb-secret
                     key: password
             volumeMounts:
               - name: mongo-data
                 mountPath: /data/db
         volumes:
           - name: mongo-data
             persistentVolumeClaim:
               claimName: mongo-pvc
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: mongodb-service
   spec:
     selector:
       app: mongodb
     ports:
       - protocol: TCP
         port: 27017
         targetPort: 27017
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: mongodb-configmap
   data:
     db_host: mongodb-service
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mongo-express
     labels:
       app: mongo-express
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: mongo-express
     template:
       metadata:
         labels:
           app: mongo-express
       spec:
         containers:
           - name: mongo-express
             image: mongo-express
             ports:
               - containerPort: 8081
             env:
               - name: ME_CONFIG_BASICAUTH_USERNAME
                 valueFrom:
                   secretKeyRef:
                     name: mongo-express-secret
                     key: meuser
               - name: ME_CONFIG_BASICAUTH_PASSWORD
                 valueFrom:
                   secretKeyRef:
                     name: mongo-express-secret
                     key: mepass
               - name: ME_CONFIG_MONGODB_ADMINUSERNAME
                 valueFrom:
                   secretKeyRef:
                     name: mongodb-secret
                     key: username
               - name: ME_CONFIG_MONGODB_ADMINPASSWORD
                 valueFrom:
                   secretKeyRef:
                     name: mongodb-secret
                     key: password
               - name: ME_CONFIG_MONGODB_SERVER
                 valueFrom:
                   configMapKeyRef:
                     name: mongodb-configmap
                     key: db_host
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: mongo-express-service
   spec:
     selector:
       app: mongo-express
     type: NodePort
     ports:
       - protocol: TCP
         port: 8081
         targetPort: 8081
   ```

   Apply the YAML file:
   ```bash
   kubectl apply -f mongo-deployment.yaml
   ```

5. **Verify the deployment**:
   Check the status of deployments, services, and pods.
   ```bash
   kubectl get deployment
   kubectl get secrets
   kubectl get pv
   kubectl get pvc
   kubectl get pods
   kubectl get services
   ```

---

## Step 5: Clean Up (Optional)

To delete deployments, services, secrets, or persistent volumes, use the following commands:
```bash
kubectl get deployments
kubectl delete deployments <deployment-name>

kubectl get svc
kubectl delete svc <service-name>

kubectl get secrets
kubectl delete secrets <secret-name>

kubectl get pv
kubectl delete pv <persistent-volume-name>
```

---
