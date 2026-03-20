# k8s-troubleshooting-runbook

Field tested troubleshooting runbooks for the most common Kubernetes failure scenarios. Each runbook walks through symptoms, diagnostic commands, root cause analysis, and resolution steps.

Built from real production incidents across enterprise Kubernetes environments.

## Scenarios

| Scenario | Directory | Symptoms |
|----------|-----------|----------|
| OOMKill | `oomkill/` | Container killed with exit code 137, pod restarts unexpectedly |
| CrashLoopBackOff | `crashloopbackoff/` | Pod stuck in restart loop, backoff delay increasing |
| Pending Pod | `pending-pod/` | Pod stuck in Pending state, never gets scheduled |
| ImagePullBackOff | `imagepullbackoff/` | Pod stuck waiting for container image, pull errors in events |
| Node Not Ready | `node-not-ready/` | Node shows NotReady status, workloads evicted or unschedulable |
| Failed Deployment | `failed-deployment/` | Rollout stalled, new ReplicaSet not scaling up |
| DNS Resolution Failure | `dns-resolution-failure/` | Pods cannot resolve service names or external domains |
| Probe Failure | `probe-failure/` | Liveness or readiness probes failing, pod restarts or drops from service |

## How To Use

Each runbook follows the same structure:

1. **Symptoms**: What you see in kubectl output or monitoring
2. **First Response**: Quick commands to assess the situation
3. **Diagnosis**: Deeper investigation to identify root cause
4. **Common Causes**: Most frequent reasons ranked by likelihood
5. **Resolution**: Step by step fixes for each cause
6. **Prevention**: How to avoid this in the future

## Quick Reference

### Get pod status and recent events
\`\`\`bash
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
\`\`\`

### Check logs
\`\`\`bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
\`\`\`

### Check resource usage
\`\`\`bash
kubectl top pods -n <namespace>
kubectl top nodes
\`\`\`

## Prerequisites

- \`kubectl\` configured to point at your cluster
- \`kubectl top\` requires metrics-server installed

## Contributing

If you run into a failure scenario not covered here, open an issue or submit a PR.
