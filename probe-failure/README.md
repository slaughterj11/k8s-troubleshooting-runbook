# Probe Failure

Liveness or readiness probes are failing, causing pod restarts or removal from service endpoints.

## Symptoms

- Pod keeps restarting due to liveness probe failure
- Pod is running but not receiving traffic (readiness probe failing)
- Events show `Unhealthy` with `Liveness probe failed` or `Readiness probe failed`
- Intermittent 503 errors from services fronting the pod

## First Response

```bash
# Check pod events for probe failures
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Unhealthy\|probe"

# Check current probe configuration
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].livenessProbe}'
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].readinessProbe}'

# Check container restart count
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

## Diagnosis

```bash
# Test the probe endpoint manually from inside the pod
kubectl exec <pod-name> -n <namespace> -- curl -s localhost:<port><path>

# For TCP probes, test the port
kubectl exec <pod-name> -n <namespace> -- nc -zv localhost <port>

# Check application logs around the time of probe failures
kubectl logs <pod-name> -n <namespace> --since=5m

# Check if the app is under resource pressure when probes fail
kubectl top pod <pod-name> -n <namespace>

# Check probe timing vs actual app startup time
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].startupProbe}'
```

## Common Causes

1. **Probe endpoint returns non 200 status**: The health check endpoint returns an error or unexpected status code. The application may be partially healthy but the probe path reports failure.

2. **Probe timeout too short**: The application takes longer to respond than the probe timeout allows. Under load, a normally fast endpoint may exceed the 1 second default timeout.

3. **initialDelaySeconds too short**: The probe starts checking before the application has finished starting. Common with Java apps, apps that load large datasets, or apps with database migrations.

4. **Wrong port or path**: The probe is configured for a port or path that does not exist or has changed. Often happens after application refactoring.

5. **Application deadlock or thread exhaustion**: The application is running but the health endpoint hangs because all threads are busy or the app is in a deadlocked state.

6. **Resource contention**: CPU throttling or memory pressure causes the health endpoint to respond slowly, exceeding the probe timeout.

## Resolution

### Probe endpoint returning errors
```bash
# Test the exact probe endpoint
kubectl exec <pod-name> -n <namespace> -- curl -sv localhost:<port><path>

# Check what the endpoint expects
# Fix the endpoint or update the probe path
```

### Timeout too short
```bash
# Increase timeout (default is 1 second)
# Update the deployment probe config:
# livenessProbe:
#   httpGet:
#     path: /healthz
#     port: 8080
#   timeoutSeconds: 5
#   periodSeconds: 10
#   failureThreshold: 3
```

### Startup too slow for liveness probe
```bash
# Add a startupProbe to handle slow starts
# The liveness probe will not start until the startup probe succeeds
# startupProbe:
#   httpGet:
#     path: /healthz
#     port: 8080
#   failureThreshold: 30
#   periodSeconds: 10
# This gives the app up to 300 seconds (30 x 10) to start
```

### Wrong port or path
```bash
# Verify the container is listening on the expected port
kubectl exec <pod-name> -n <namespace> -- netstat -tlnp 2>/dev/null || \
kubectl exec <pod-name> -n <namespace> -- ss -tlnp

# Update probe config to match actual endpoint
```

### Resource contention
```bash
# Check if CPU is being throttled
kubectl top pod <pod-name> -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].resources}'

# Increase CPU limits or requests to prevent throttling during health checks
```

## Prevention

- Use startupProbe for any application that takes more than a few seconds to initialize
- Set liveness probe timeoutSeconds to at least 3-5 seconds for production workloads
- Keep health check endpoints lightweight (no database queries, no external calls)
- Set failureThreshold to at least 3 for liveness probes to tolerate transient failures
- Differentiate between liveness and readiness: liveness checks if the process is alive, readiness checks if it can serve traffic
- Never make liveness probes depend on external services (database, cache, etc.)
