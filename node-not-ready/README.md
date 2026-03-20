# Node Not Ready

A node has transitioned to NotReady status, meaning the kubelet is no longer reporting healthy to the control plane.

## Symptoms

- `kubectl get nodes` shows one or more nodes as `NotReady`
- Pods on the affected node may be evicted or stuck in `Terminating`
- New pods cannot be scheduled to the node
- Events show `NodeNotReady` or `NodeStatusUnknown`

## First Response

```bash
# Check node status
kubectl get nodes -o wide

# Get detailed node conditions
kubectl describe node <node-name> | grep -A 20 "Conditions"

# Check if pods on this node are affected
kubectl get pods --all-namespaces --field-selector spec.nodeName=<node-name>
```

## Diagnosis

```bash
# Check kubelet status (requires SSH or node access)
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago" --no-pager

# Check node resource pressure conditions
kubectl describe node <node-name> | grep -E "MemoryPressure|DiskPressure|PIDPressure"

# Check node resource usage
kubectl top node <node-name>

# Check for recent events on the node
kubectl get events --field-selector involvedObject.name=<node-name> --sort-by='.lastTimestamp'

# Check if the container runtime is healthy (requires node access)
crictl ps
# or for Docker: docker ps

# Check disk space on the node (requires node access)
df -h
```

## Common Causes

1. **Kubelet process crashed or stopped**: The kubelet is not running on the node. Check `systemctl status kubelet` and kubelet logs.

2. **Node resource pressure**: Node has hit memory, disk, or PID pressure thresholds. Kubelet marks the node as NotReady to prevent scheduling more workloads.

3. **Network connectivity lost**: The node cannot communicate with the API server. Could be network partition, security group change, or node network interface issue.

4. **Container runtime failure**: Docker, containerd, or CRI-O has crashed or is unresponsive. Kubelet cannot manage containers.

5. **Certificate expiration**: Kubelet certificates have expired and the node can no longer authenticate to the API server.

6. **Node ran out of disk space**: Image layers, container logs, or emptyDir volumes consumed all available disk.

## Resolution

### Kubelet not running
```bash
# SSH to the node
systemctl restart kubelet
systemctl status kubelet

# If kubelet fails to start, check config
journalctl -u kubelet -n 50 --no-pager
```

### Resource pressure
```bash
# Check what is consuming resources
kubectl top pods --all-namespaces --field-selector spec.nodeName=<node-name> --sort-by=memory

# Evict non critical pods to free resources
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Disk pressure: clean up unused images
crictl rmi --prune
```

### Network issues
```bash
# From the node, test API server connectivity
curl -k https://<api-server-ip>:6443/healthz

# Check node network interface
ip addr show
ip route show

# Check security groups / firewall rules for API server port (6443)
```

### Container runtime failure
```bash
# Restart the container runtime
systemctl restart containerd
# or: systemctl restart docker

# Verify it is running
systemctl status containerd
crictl info
```

### Certificate expiration
```bash
# Check certificate expiration dates
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# If expired, rotate kubelet certificates
kubeadm certs renew all
systemctl restart kubelet
```

## Prevention

- Monitor node conditions and set up alerts for MemoryPressure, DiskPressure, and PIDPressure
- Set up log rotation to prevent container logs from filling disk
- Implement image garbage collection policies
- Monitor certificate expiration dates and automate rotation
- Use node problem detector to catch issues early
- Set appropriate eviction thresholds in kubelet config
