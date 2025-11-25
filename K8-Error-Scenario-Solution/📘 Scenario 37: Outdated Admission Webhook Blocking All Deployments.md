# 📘 Scenario 37: Outdated Admission Webhook Blocking All Deployments

**Category**: Cluster Security & Operations  
**Environment**: Kubernetes 1.25, Self-hosted with Custom Admission Webhooks  
**Impact**: Complete deployment freeze for 3+ hours  

---

## Scenario Summary  
An expired TLS certificate on a mutating admission webhook caused all resource creation operations to fail, effectively freezing cluster operations and preventing any new deployments.

---