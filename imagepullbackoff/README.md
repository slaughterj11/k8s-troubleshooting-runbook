# ImagePullBackOff

Pod cannot pull the container image from the registry.

## Symptoms

- Pod status shows `ImagePullBackOff` or `ErrImagePull`
- `kubectl describe pod` shows events with `Failed to pull image`
- Container never starts
- Error messages reference authentication, image not found, or timeout

## First Response

```bash
# Check pod status
kubectl get pod <pod-name> -n <namespace>

# Get the exact error message
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Events"

# Check which image it is trying to pull
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].image}'
```

## Diagnosis

```bash
# Check if image pull secrets are configured
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.imagePullSecrets}'

# Check if the referenced pull secret exists
kubectl get secrets -n <namespace> | grep docker

# Verify the secret contains valid credentials
kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# Test pulling the image manually from a debug pod
kubectl run debug --rm -it --image=busybox --restart=Never -- /bin/sh
# From inside: wget -qO- https://<registry>/v2/<image>/tags/list
```

## Common Causes

1. **Image tag does not exist**: The specified tag (or `latest`) does not exist in the registry. Typos in image names or tags are extremely common.

2. **Missing or wrong image pull secret**: Private registries require authentication. The secret is either missing, in the wrong namespace, or contains expired credentials.

3. **Registry is unreachable**: Network policies, firewall rules, or DNS issues prevent the node from reaching the container registry.

4. **Rate limiting**: Docker Hub and other registries enforce pull rate limits. Anonymous pulls from Docker Hub are limited to 100 per 6 hours per IP.

5. **Wrong registry URL**: Image reference points to the wrong registry or uses an incorrect path.

## Resolution

### Image tag does not exist
```bash
# Verify the image and tag exist
# For Docker Hub:
curl -s https://hub.docker.com/v2/repositories/<namespace>/<image>/tags | jq '.results[].name'

# For private registries, check your CI/CD pipeline to confirm the image was pushed
# Fix the image tag in the deployment
kubectl set image deployment/<deployment-name> <container-name>=<correct-image:tag> -n <namespace>
```

### Missing pull secret
```bash
# Create a pull secret
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password> \
  -n <namespace>

# Patch the deployment to use it
kubectl patch deployment <deployment-name> -n <namespace> \
  -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"regcred"}]}}}}'
```

### Rate limiting (Docker Hub)
```bash
# Check your current rate limit status
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/alpine:pull" | jq -r .token)
curl -sI -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/library/alpine/manifests/latest | grep ratelimit

# Solutions:
# 1. Authenticate to Docker Hub (increases limit to 200/6hrs)
# 2. Use a pull through cache or registry mirror
# 3. Switch to a different registry (ECR, GCR, ACR)
```

### Registry unreachable
```bash
# Test connectivity from a node or debug pod
kubectl run debug --rm -it --image=busybox --restart=Never -- /bin/sh
# From inside: nslookup <registry-host> && wget -qO- https://<registry-host>/v2/

# Check if network policies are blocking egress
kubectl get networkpolicies -n <namespace>
```

## Prevention

- Always use specific image tags, never `latest`
- Set up image pull secrets at the namespace level using a service account default
- Use a registry mirror or pull through cache to avoid rate limits
- Include image existence checks in your CI/CD pipeline before deploying
- Monitor pull secret expiration dates and rotate before they expire
