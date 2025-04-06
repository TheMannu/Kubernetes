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
