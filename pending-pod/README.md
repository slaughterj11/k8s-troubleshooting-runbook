# Pending Pod

Pod is stuck in Pending state and never gets scheduled to a node.

## Symptoms

- Pod status shows `Pending`
- Pod has been Pending for longer than expected (more than a few seconds for normal workloads)
- `kubectl describe pod` shows `FailedScheduling` events
- Other pods in the same namespace may be running fine

## First Response

```bash
# Check pod status
kubectl get pod <pod-name> -n <namespace>

# Look at scheduling events (this is where the answer usually is)
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "Events"

# Check node availability
kubectl get nodes
```

## Diagnosis

```bash
# Get the exact scheduling failure message
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>,reason=FailedScheduling

# Check pod resource requests
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources.requests}'

# Check available resources on each node
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check if there are node selectors or affinity rules
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeSelector}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.affinity}'

# Check for taints on nodes that might repel the pod
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Check if PVCs the pod depends on are bound
kubectl get pvc -n <namespace>
```

## Common Causes

1. **Insufficient resources**: No node has enough CPU or memory to satisfy the pod's resource requests. The FailedScheduling event will say something like `Insufficient cpu` or `Insufficient memory`.

2. **Node selector or affinity mismatch**: The pod requires specific node labels that no available node has. Check `nodeSelector` and `nodeAffinity` rules.

3. **Taints and tolerations**: All available nodes have taints that the pod does not tolerate. Common when nodes are cordoned or dedicated to specific workloads.

4. **PersistentVolumeClaim not bound**: The pod references a PVC that is in Pending state itself. The pod cannot be scheduled until the volume is available.

5. **ResourceQuota exceeded**: The namespace has a ResourceQuota and the pod would push usage over the limit.

6. **Too many pods on nodes**: Nodes have hit their pod limit (default 110 per node). Happens on small clusters with many small pods.

## Resolution

### Insufficient resources
```bash
# Check what is consuming resources
kubectl top nodes
kubectl top pods -n <namespace> --sort-by=memory

# Option 1: Reduce resource requests if they are overprovisioned
# Option 2: Add more nodes to the cluster
# Option 3: Remove or scale down lower priority workloads
```

### Node selector mismatch
```bash
# Check what labels the pod requires
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.nodeSelector}'

# Check what labels nodes actually have
kubectl get nodes --show-labels

# Add the required label to a node
kubectl label node <node-name> <key>=<value>
```

### Taint and toleration
```bash
# Check node taints
kubectl describe node <node-name> | grep Taints

# Remove a taint from a node
kubectl taint node <node-name> <key>:<effect>-

# Or add a toleration to the pod spec in the deployment
```

### PVC not bound
```bash
# Check PVC status
kubectl get pvc -n <namespace>

# If PVC is Pending, check storage class and available PVs
kubectl describe pvc <pvc-name> -n <namespace>
kubectl get pv
kubectl get storageclass
```

### ResourceQuota exceeded
```bash
# Check quota usage
kubectl describe resourcequota -n <namespace>

# Either increase the quota or free up resources
```

## Prevention

- Set resource requests based on actual usage data, not guesses
- Use `kubectl top` and VPA recommendations to right size requests
- Monitor cluster capacity and set up alerts for when allocatable resources drop below 20%
- Avoid overly specific node selectors unless absolutely necessary
- Use PodDisruptionBudgets to prevent too many pods from being evicted at once
