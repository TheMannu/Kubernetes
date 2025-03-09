
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
