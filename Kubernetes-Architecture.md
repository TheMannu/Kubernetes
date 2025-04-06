1. **KUBECTL**  
   - The **command-line tool** you use to talk to Kubernetes (send commands to the cluster).  

2. **API SERVER**  
   - The **brain** of Kubernetes; it's the front-end that handles all requests (from users, controllers, or other components).  

3. **CONTROLLER MANAGER**  
   - A collection of **controllers** that adjust cluster resources (like Deployments, Nodes, or ReplicaSets) to match the desired state.  

4. **SCHEDULER**  
   - Decides **where to place workloads** (Pods) based on resources, policies, and constraints.  

5. **KUBELET**  
   - An agent that **runs on each worker node**, ensuring Pods are running as expected.  

6. **ETCD**  
   - Kubernetes’ **database**—stores all cluster data (configurations, states, etc.).  

7. **KUBE-PROXY**  
   - Manages **network traffic** (load balancing, routing) to Pods across the cluster.  

8. **POD**  
   - The smallest deployable unit in Kubernetes—a **group of containers** that share resources.  

9. **CONTAINER RUNTIME**  
   - The software (like Docker or containerd) that **runs containers** inside Pods.  


### **Simple Flow:**  
1. You use `kubectl` to send a command (e.g., "Run 3 Nginx Pods").  
2. The **API Server** receives it and updates **etcd**.  
3. The **Scheduler** picks the best nodes for the Pods.  
4. The **Kubelet** on those nodes starts the Pods using the **Container Runtime**. 
