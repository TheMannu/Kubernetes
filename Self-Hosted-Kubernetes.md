
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
