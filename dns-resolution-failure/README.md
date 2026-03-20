# DNS Resolution Failure

Pods cannot resolve Kubernetes service names or external domain names.

## Symptoms

- Application logs show connection failures or `Name or service not known` errors
- `nslookup` or `dig` from inside a pod returns `NXDOMAIN` or times out
- Services are unreachable by name but work by IP
- Intermittent connectivity issues across the cluster

## First Response

```bash
# Run a debug pod and test DNS
kubectl run debug --rm -it --image=busybox:1.36 --restart=Never -- /bin/sh

# From inside the debug pod:
nslookup kubernetes.default
nslookup <service-name>.<namespace>.svc.cluster.local
nslookup google.com
```

## Diagnosis

```bash
# Check if CoreDNS pods are running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs for errors
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Check the kube-dns service
kubectl get svc -n kube-system kube-dns

# Verify the service has endpoints
kubectl get endpoints -n kube-system kube-dns

# Check the pod's DNS config
kubectl exec <pod-name> -n <namespace> -- cat /etc/resolv.conf

# Check if the CoreDNS configmap has issues
kubectl get configmap coredns -n kube-system -o yaml

# Test DNS from the node level (requires node access)
dig @<coredns-cluster-ip> kubernetes.default.svc.cluster.local
```

## Common Causes

1. **CoreDNS pods are down or crashlooping**: If CoreDNS pods are not running, no DNS resolution works inside the cluster. Check pod status and logs.

2. **CoreDNS overwhelmed**: Under heavy DNS load, CoreDNS may drop queries. Look for `SERVFAIL` responses and high latency in CoreDNS metrics.

3. **Network policy blocking DNS**: A NetworkPolicy is blocking egress to the kube-dns service on UDP port 53.

4. **Incorrect resolv.conf**: The pod's `/etc/resolv.conf` does not point to the correct DNS service IP. Can happen with custom `dnsPolicy` settings.

5. **Upstream DNS issues**: External domain resolution fails because the upstream DNS server configured in CoreDNS is unreachable or slow.

6. **Service does not exist**: The service name being resolved simply does not exist in the expected namespace. Typos or wrong namespace.

## Resolution

### CoreDNS not running
```bash
# Check CoreDNS deployment
kubectl get deployment coredns -n kube-system

# Restart CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# If CoreDNS is crashlooping, check logs
kubectl logs -n kube-system -l k8s-app=kube-dns --previous
```

### Network policy blocking DNS
```bash
# Check network policies in the affected namespace
kubectl get networkpolicies -n <namespace>

# Ensure egress to kube-dns is allowed. Add this to your network policy:
# egress:
#   - to:
#       - namespaceSelector:
#           matchLabels:
#             kubernetes.io/metadata.name: kube-system
#     ports:
#       - protocol: UDP
#         port: 53
#       - protocol: TCP
#         port: 53
```

### Incorrect resolv.conf
```bash
# Check the pod's DNS config
kubectl exec <pod-name> -n <namespace> -- cat /etc/resolv.conf

# Expected content should include:
# nameserver <kube-dns-cluster-ip>
# search <namespace>.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# If wrong, check the pod's dnsPolicy setting
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.dnsPolicy}'
# Should be "ClusterFirst" for most pods
```

### CoreDNS overwhelmed
```bash
# Scale up CoreDNS
kubectl scale deployment coredns -n kube-system --replicas=3

# Check CoreDNS resource usage
kubectl top pods -n kube-system -l k8s-app=kube-dns

# Consider enabling DNS caching with NodeLocal DNSCache
```

## Prevention

- Monitor CoreDNS pod health and query latency
- Use NodeLocal DNSCache for large clusters to reduce load on CoreDNS
- Always allow DNS egress (UDP/TCP 53 to kube-system) in network policies
- Scale CoreDNS based on cluster size (recommended: 1 replica per 100-150 nodes)
- Set appropriate `ndots` value in pod DNS config to reduce unnecessary DNS queries
