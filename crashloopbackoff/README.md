# CrashLoopBackOff

Container starts, crashes immediately, and Kubernetes keeps restarting it with exponentially increasing backoff delays.

## Symptoms

- Pod status shows `CrashLoopBackOff`
- Container restarts rapidly, then slows down (10s, 20s, 40s, up to 5 minute backoff)
- `kubectl describe pod` shows high restart count
- Events show `Back-off restarting failed container`

## First Response

```bash
# Check pod status
kubectl get pod <pod-name> -n <namespace>

# Check container exit code and restart count
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 "State\|Last State"

# Get logs from the crashing container
kubectl logs <pod-name> -n <namespace>

# If the current container already crashed, get previous logs
kubectl logs <pod-name> -n <namespace> --previous
```

## Diagnosis

```bash
# Check the exit code (critical for diagnosis)
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'

# Exit code reference:
# 0   = Container exited normally (check if command is meant to be long running)
# 1   = Application error (check logs)
# 126 = Command cannot be invoked (permission issue)
# 127 = Command not found (wrong entrypoint/cmd)
# 137 = OOMKilled or SIGKILL (check memory limits)
# 139 = Segmentation fault
# 143 = SIGTERM (graceful shutdown requested)

# Check if configmaps or secrets the pod depends on exist
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Volumes\|Environment"

# Check if the container image is correct
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].image}'
```

## Common Causes

1. **Application error on startup**: The application throws an unhandled exception or fatal error during initialization. This is the most common cause. Check logs with `--previous`.

2. **Missing configuration**: Required environment variables, config files, or secrets are not mounted or have wrong values. The app fails a startup check and exits.

3. **Wrong command or entrypoint**: The Dockerfile CMD or Kubernetes command override is incorrect. The process either does not exist (exit code 127) or exits immediately (exit code 0).

4. **Dependency not available**: The application tries to connect to a database, message queue, or external service on startup and fails when the dependency is unreachable.

5. **Insufficient resources**: Container hits memory limit on startup (exit code 137). Common for JVM apps with large heap requirements.

6. **Liveness probe too aggressive**: Probe starts checking before the app is ready, kills the container, and the cycle repeats.

## Resolution

### Application error
```bash
# Check logs for stack traces or error messages
kubectl logs <pod-name> -n <namespace> --previous

# If multi container pod, check specific container
kubectl logs <pod-name> -n <namespace> -c <container-name> --previous
```

### Missing configuration
```bash
# Verify all referenced configmaps exist
kubectl get configmap -n <namespace>

# Verify all referenced secrets exist
kubectl get secrets -n <namespace>

# Check environment variable values
kubectl exec <pod-name> -n <namespace> -- env 2>/dev/null || \
  kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].env[*]}'
```

### Wrong entrypoint
```bash
# Check what command the container is running
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].command}'

# Run the container interactively to debug
kubectl run debug --rm -it --image=<image> --restart=Never -- /bin/sh
```

### Dependency not available
```bash
# Test connectivity from within the cluster
kubectl run debug --rm -it --image=busybox --restart=Never -- /bin/sh
# Then from inside: nslookup <service-name> and wget -qO- http://<service-name>:<port>/health
```

### Liveness probe too aggressive
```bash
# Check probe configuration
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].livenessProbe}'

# Increase initialDelaySeconds or switch to startupProbe for slow starting apps
```

## Prevention

- Always include health check endpoints and use startupProbe for apps that take time to initialize
- Use init containers to wait for dependencies before the main container starts
- Set sensible defaults for all required environment variables
- Test container startup locally with `docker run` before deploying to Kubernetes
- Log meaningful error messages on startup failure so the cause is immediately visible in `kubectl logs`
