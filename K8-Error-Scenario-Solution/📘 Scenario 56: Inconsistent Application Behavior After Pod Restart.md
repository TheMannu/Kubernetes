# 📘 Scenario 56: Inconsistent Application Behavior After Pod Restart

**Category**: Stateful Application Design  
**Environment**: Kubernetes 1.20, GKE, Stateless-by-default expectations  
**Impact**: Data inconsistency across users, revenue loss from corrupted sessions, 4-hour debugging cycle  

---

## Scenario Summary  
An application storing session state and user preferences in ephemeral container storage exhibited unpredictable behavior after pod restarts, resulting in data loss, inconsistent user experiences, and hard-to-reproduce bugs.

---