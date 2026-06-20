# ComfyUI Helm Chart

A Helm chart for deploying [ComfyUI](https://github.com/Comfy-Org/ComfyUI) on Kubernetes with GPU support.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.10+
- NVIDIA device plugin (if using GPU)

## Installation

```bash
helm repo add comfyui https://YOUR_GITHUB_USERNAME.github.io/comfyui-helm-chart
helm repo update
helm install comfyui comfyui/comfyui
```

## Configuration

| Key | Default | Description |
|-----|---------|-------------|
| `image.repository` | `overseer66/comfyui` | Container image |
| `image.tag` | `""` (Chart.appVersion) | Image tag |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `replicaCount` | `1` | Number of replicas |
| `service.type` | `ClusterIP` | Service type |
| `service.ports` | `8188, 8080` | Service ports (ComfyUI, code-server) |
| `ingress.enabled` | `false` | Enable ingress |
| `ingress.hostname` | `comfyui.example.com` | Ingress hostname |
| `ingress.tls.enabled` | `false` | Enable TLS |
| `route.enabled` | `false` | Enable Gateway API HTTPRoute |
| `route.apiVersion` | `gateway.networking.k8s.io/v1` | Route API version |
| `route.kind` | `HTTPRoute` | Route kind |
| `route.hostnames` | `[]` | Hostnames to match |
| `route.parentRefs` | `[]` | References to parent Gateways |
| `route.httpsRedirect` | `false` | Create a separate HTTP→HTTPS redirect route |
| `route.matches` | `[{path: {type: PathPrefix, value: /}}]` | Route match rules |
| `route.filters` | `[]` | Route filters |
| `route.additionalRules` | `[]` | Extra route rules prepended before the backend rule |
| `route.backendRef.group` | `""` | Backend API group (`""` = core, i.e. Service) |
| `route.backendRef.kind` | `Service` | Backend kind |
| `route.backendRef.port` | `""` (first service port) | Backend port override |
| `route.backendRef.weight` | `1` | Backend weight |
| `route.annotations` | `{}` | Annotations added to the route |
| `route.extraLabels` | `{}` | Extra labels added to the route |
| `gpu.enabled` | `true` | Request NVIDIA GPU |
| `gpu.count` | `1` | Number of GPUs |
| `persistence.enabled` | `true` | Enable persistent storage |
| `persistence.size` | `20Gi` | PVC size |
| `persistence.storageClass` | `""` | StorageClass (empty = cluster default) |
| `persistence.existingClaim` | `""` | Use an existing PVC instead of creating one |
| `persistence.mountPath` | `/home/runner` | Mount path inside the container |
| `env` | `[]` | Extra environment variables |
| `envFrom` | `[]` | Environment from ConfigMaps / Secrets |
| `nodeSelector` | `{}` | Node selector |
| `tolerations` | `[]` | Pod tolerations |
| `affinity` | `{}` | Pod affinity rules |
| `resources` | `{}` | CPU/memory resource requests and limits |
| `podAnnotations` | `{}` | Pod annotations |
| `serviceAccount.create` | `true` | Create a ServiceAccount |

## GPU nodes

GPU nodes are commonly tainted. Add a matching toleration:

```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

## Ingress with basic auth (ingress-nginx)

```bash
htpasswd -c auth myuser
kubectl create secret generic comfyui-basic-auth --from-file=auth
```

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: comfyui-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
  hostname: comfyui.example.com
  tls:
    enabled: true
    secretName: comfyui-tls
```

## Gateway API

As an alternative to `Ingress`, the chart supports [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/) via an `HTTPRoute` (or any other supported route kind).

**Basic HTTPRoute:**

```yaml
route:
  enabled: true
  hostnames:
    - comfyui.example.com
  parentRefs:
    - name: my-gateway
      namespace: gateway-ns
      sectionName: https
```

**With HTTP→HTTPS redirect** (requires the gateway to have both an `http` and an `https` listener):

```yaml
route:
  enabled: true
  httpsRedirect: true
  hostnames:
    - comfyui.example.com
  parentRefs:
    - name: my-gateway
      namespace: gateway-ns
      sectionName: https
  # Optional: override parentRefs for the redirect route only
  # redirect:
  #   parentRefs:
  #     - name: my-gateway
  #       namespace: gateway-ns
  #       sectionName: http
```

The redirect route automatically appends `sectionName: http` to each parentRef when `route.redirect` is not set explicitly.

**Forwarding to the code-server port instead of the default ComfyUI port:**

```yaml
route:
  enabled: true
  hostnames:
    - code.example.com
  parentRefs:
    - name: my-gateway
  backendRef:
    port: 8080
```

## ArtifactHub

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/comfyui)](https://artifacthub.io/packages/search?repo=comfyui)
