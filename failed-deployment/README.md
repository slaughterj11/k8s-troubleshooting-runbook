# Failed Deployment

Deployment rollout is stuck or failed. New ReplicaSet is not scaling up or old ReplicaSet is not scaling down.

## Symptoms

- `kubectl rollout status` hangs or reports failure
- New pods are not created or are failing
- Old pods remain running alongside failing new pods
- Deployment shows a mix of ready and not ready replicas

## First Response

```bash
# Check deployment status
kubectl get deployment <deployment-name> -n <namespace>

# Check rollout status
kubectl rollout status deployment/<deployment-name> -n <namespace>

# Check ReplicaSets to see old vs new
kubectl get replicasets -n <namespace> -l app=<app-label>

# Check events
kubectl describe deployment <deployment-name> -n <namespace> | grep -A 15 "Events"
```

## Diagnosis

```bash
# See the rollout history
kubectl rollout history deployment/<deployment-name> -n <namespace>

# Compare current vs previous revision
kubectl rollout history deployment/<deployment-name> -n <namespace> --revision=<current>
kubectl rollout history deployment/<deployment-name> -n <namespace> --revision=<previous>

# Check the new ReplicaSet and its pods
kubectl get rs -n <namespace> -l app=<app-label> --sort-by='.metadata.creationTimestamp'
NEW_RS=$(kubectl get rs -n <namespace> -l app=<app-label> --sort-by='.metadata.creationTimestamp' -o jsonpath='{.items[-1].metadata.name}')
kubectl describe rs $NEW_RS -n <namespace>

# Check pods created by the new ReplicaSet
kubectl get pods -n <namespace> -l app=<app-label> --sort-by='.metadata.creationTimestamp'

# Check if a PDB is blocking the rollout
kubectl get pdb -n <namespace>
```

## Common Causes

1. **New pods are crashing**: The new version has a bug that causes CrashLoopBackOff. The deployment cannot progress because new pods never become ready. Check pod logs.

2. **Insufficient resources**: Cluster does not have enough resources to run both old and new pods during the rolling update. New pods stay Pending.

3. **Failed readiness probe**: New pods start but never pass readiness checks. The deployment waits for them to become ready before continuing the rollout.

4. **Image pull failure**: New pods reference an image that does not exist or cannot be pulled. See the ImagePullBackOff runbook.

5. **PodDisruptionBudget blocking**: A PDB is preventing old pods from being terminated, stalling the rollout.

6. **Deadline exceeded**: The deployment has a `progressDeadlineSeconds` (default 600s) and the rollout did not complete in time.

## Resolution

### Rollback immediately
```bash
# If production is impacted, rollback first, debug later
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Verify the rollback
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

### New pods crashing
```bash
# Get logs from the failing new pods
kubectl logs <new-pod-name> -n <namespace>
kubectl logs <new-pod-name> -n <namespace> --previous

# Fix the issue in the application, rebuild, and redeploy
```

### Insufficient resources
```bash
# Check what the new pods need
kubectl describe pod <pending-pod> -n <namespace> | grep -A 5 "Requests"

# Check cluster capacity
kubectl top nodes

# Increase maxSurge to control how many extra pods are created during rollout
kubectl patch deployment <deployment-name> -n <namespace> \
  -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":1}}}}'
```

### Readiness probe failing
```bash
# Check probe config
kubectl get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'

# Exec into a new pod and test the probe endpoint manually
kubectl exec <new-pod> -n <namespace> -- curl -s localhost:<port><path>
```

### PDB blocking
```bash
# Check PDB status
kubectl get pdb -n <namespace>
kubectl describe pdb <pdb-name> -n <namespace>

# If PDB is overly restrictive, temporarily adjust or delete it
# Then re-trigger the rollout
```

## Prevention

- Always set readiness probes and test them before deploying
- Use `maxUnavailable: 0` and `maxSurge: 1` for zero downtime rollouts (at the cost of needing extra resources)
- Test deployments in staging with the same resource constraints as production
- Set a reasonable `progressDeadlineSeconds` and alert on deployment failures
- Use canary or blue green deployment strategies for critical services
