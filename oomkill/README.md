# OOMKill

Container terminated by the kernel for exceeding its memory limit.

## Symptoms

- Pod restarts unexpectedly
- `kubectl describe pod` shows `OOMKilled` as last state reason
- Exit code 137
- Container `restartCount` keeps increasing

## First Response

```bash
# Check pod status and restart count
kubectl get pod <pod-name> -n <namespace> -o wide

# Look for OOMKilled in last state
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Last State"

# Check events for OOM related messages
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name> --sort-by='.lastTimestamp'
```

## Diagnosis

```bash
# Check current memory limits vs requests
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'

# Check actual memory usage (requires metrics-server)
kubectl top pod <pod-name> -n <namespace>

# Check previous container logs for memory related errors
kubectl logs <pod-name> -n <namespace> --previous

# Check node level memory pressure
kubectl describe node <node-name> | grep -A 5 "Conditions"
```

## Common Causes

1. **Memory limit set too low**: The application legitimately needs more memory than the limit allows. This is the most common cause.

2. **Memory leak in application**: The application gradually consumes more memory over time until it hits the limit. Look for steadily increasing memory usage before the kill.

3. **JVM heap misconfiguration**: For Java apps, the JVM heap size is not aligned with the container memory limit. The JVM can consume more memory than the heap (metaspace, thread stacks, native memory).

4. **Burst traffic**: Application memory spikes under load. Normal usage fits within limits but traffic spikes push it over.

5. **Sidecar memory not accounted for**: Istio, Linkerd, or other sidecar containers share the pod memory budget but were not included in sizing calculations.

## Resolution

### Memory limit too low
```bash
# Check current limits
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources.limits.memory}'

# Update the deployment with a higher limit
kubectl patch deployment <deployment-name> -n <namespace> --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "512Mi"}]'
```

### Memory leak
```bash
# Watch memory usage over time
watch kubectl top pod <pod-name> -n <namespace>

# Get a heap dump (Java example)
kubectl exec <pod-name> -n <namespace> -- jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp <namespace>/<pod-name>:/tmp/heap.hprof ./heap.hprof
```

### JVM specific
Set JVM flags to respect container limits:
```
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0
```
This tells the JVM to use at most 75% of the container memory limit for heap, leaving room for metaspace and native memory.

## Prevention

- Set memory requests equal to limits for predictable behavior
- Run load tests to establish actual memory requirements before setting limits
- Monitor memory usage trends over time, not just point in time snapshots
- For JVM apps, always set `-XX:MaxRAMPercentage` and leave 25% headroom for non heap memory
- Use VPA (Vertical Pod Autoscaler) in recommendation mode to get data driven sizing suggestions
