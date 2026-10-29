# code-server Helm Chart

Deploy code-server on Kubernetes to access your development environment through a browser. This guide is based on the current [values.yaml](values.yaml) and Chart templates.

[中文](README.zh.md).

## Versions and Defaults

| Item | Current value |
| --- | --- |
| Chart version | `3.57.0` |
| Chart appVersion | `4.143.1` |
| Container image | `codercom/code-server:4.144.0` |
| Replicas | `1` |
| Service | `ClusterIP`, port `8080` |
| Ingress | Enabled, with IngressClass `nginx` |
| Default URL | `http://code.localhost/` |
| Data directory | `/home/coder` |
| Persistent volume | `10Gi`, `ReadWriteOnce` |
| User ID / filesystem group | `1000` / `1000` |

The container image is determined by `image.repository` and `image.tag`. The Chart's `appVersion` is not automatically used as the image tag.

## Installation

You need a working Kubernetes cluster, Helm 3, and kubectl. The default configuration also requires an available default StorageClass and a matching Ingress Controller.

Run the following command from the repository root:

```bash
helm upgrade --install code-server . \
  --namespace code-server \
  --create-namespace
```

Keep custom settings in a separate override file, such as `custom-values.yaml`:

```yaml
ingress:
  annotations:
    nginx.org/mergeable-ingress-type: null
    nginx.org/path-regex: null
  hosts:
    - host: code.example.com
      paths:
        - /
  tls:
    - secretName: code-server-tls
      hosts:
        - code.example.com

persistence:
  size: 20Gi

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 4Gi
```

```bash
helm upgrade --install code-server . \
  --namespace code-server \
  --create-namespace \
  -f custom-values.yaml
```

Point the domain to your Ingress Controller's entry point and create the TLS Secret in the same namespace before deployment. The example uses `null` to remove the two default NGINX-specific annotations. Adjust other annotations and `ingressClassName` to match your controller.

The default `nginx.org/mergeable-ingress-type: master` annotation belongs to the F5 NGINX Ingress Controller's Mergeable Ingress configuration. This Chart creates only one Ingress and does not create a corresponding minion. Remove this annotation for a standard single-Ingress deployment. The community ingress-nginx controller does not use these `nginx.org/*` annotations.

### Rancher Git Repository

Push the branch containing the Chart to an accessible remote repository. In a Rancher version that supports Git-based Chart repositories, add a repository with these settings:

| Field | Setting |
| --- | --- |
| Repository type | Git Repository |
| Git URL | The URL of the repository hosting this Chart |
| Git branch | `helm-chart`, or the branch containing the Chart at its root |
| Chart path | If the interface provides this field, enter `.` |

Configure credentials for private repositories. Refresh the repository, select `code-server`, choose a namespace, and override values as needed. Git-based Chart repositories differ from HTTP Helm repositories, which require an `index.yaml` file. The exact interface depends on your Rancher version.

## Access and Authentication

The default URL is `http://code.localhost/`. Point this hostname to the Ingress entry point. For remote access, use your own domain and enable HTTPS.

When `existingSecret` is not configured, the Chart creates a Secret with a randomly generated 24-character password during the initial installation. With the release name and namespace shown above, and no resource name overrides, retrieve it with:

```bash
kubectl get secret code-server -n code-server \
  -o jsonpath='{.data.password}' | base64 --decode
```

Alternatively, create a Secret containing a `password` key in the same namespace and configure:

```yaml
existingSecret: code-server-auth
```

When using an existing Secret, retrieve the password from that Secret. The template also accepts `password` to set the initial installation password, but you should not commit plaintext passwords to Git. The generated Secret uses a `pre-install` hook, so changing this value during a normal upgrade does not automatically update the password.

If no Ingress Controller is available, install with `--set ingress.enabled=false`, then use port forwarding:

```bash
kubectl port-forward -n code-server service/code-server 8080:8080
```

Open `http://127.0.0.1:8080/` in your browser. If you change the release name or `fullnameOverride`, adjust resource names in these commands accordingly.

## Configuration Reference

Defaults below reflect the current configuration. "Not set" means that a field is provided only in a commented example or in the templates.

### Workload and Image

| Parameter | Default | Description |
| --- | --- | --- |
| `replicaCount` | `1` | Deployment replica count. The update strategy is `Recreate`. |
| `image.repository` | `codercom/code-server` | Container image repository. |
| `image.tag` | `4.144.0` | Container image tag. |
| `image.pullPolicy` | `IfNotPresent` | Image pull policy. |
| `imagePullSecrets` | `[]` | Image pull Secret references, each containing a `name`. |
| `nameOverride` | `""` | Overrides the Chart name portion of generated names. |
| `fullnameOverride` | `""` | Overrides the generated full resource name. |
| `hostnameOverride` | `""` | Overrides the Pod hostname. |
| `existingSecret` | Not set | Login password Secret name. The Secret must contain a `password` key. |
| `password` | Not set | Initial installation password; randomly generated when omitted. |
| `serviceAccount.create` | `true` | Whether to create a ServiceAccount. |
| `serviceAccount.name` | `""` | ServiceAccount name. Uses `default` when creation is disabled and no name is specified. |
| `serviceAccount.annotations` | `{}` | Not used by the current templates. |
| `annotations` | `{}` | Deployment annotations. |
| `labels` | `{}` | Not used by the current templates. |
| `podAnnotations` | `{}` | Pod annotations. |
| `podSecurityContext` | `{}` | Not used by the current templates. Use the `securityContext` settings below instead. |
| `priorityClassName` | `""` | Pod priority class name. |
| `securityContext.enabled` | `true` | Whether to set the Pod's fsGroup and the main container's runAsUser. |
| `securityContext.fsGroup` | `1000` | Pod filesystem group. |
| `securityContext.runAsUser` | `1000` | Main container user ID. |
| `volumePermissions.enabled` | `true` | Fixes data directory ownership using an init container when persistence is enabled. |
| `volumePermissions.securityContext.runAsUser` | `0` | Permissions init container user ID. The default requires allowing a root init container. |
| `resources` | `{}` | Main container resource requests and limits. Unset by default. |
| `livenessProbe.enabled` | `true` | Enables the `/healthz` liveness probe. |
| `readinessProbe.enabled` | `true` | Enables the `/healthz` readiness probe. |
| `nodeSelector` | `{}` | Node selection labels. |
| `tolerations` | `[]` | Pod tolerations. |
| `affinity` | `{}` | Unset by default. Non-empty values must be a YAML string, rendered using `tpl`. |

### Networking

| Parameter | Default | Description |
| --- | --- | --- |
| `service.type` | `ClusterIP` | Service type. |
| `service.port` | `8080` | Service port. The main container's HTTP port is fixed at `8080`. |
| `service.loadBalancerClass` | `null` | Optional implementation class for a LoadBalancer Service. |
| `service.annotations` | `{}` | Service annotations. |
| `ingress.enabled` | `true` | Whether to create an Ingress. |
| `ingress.annotations` | See below | Includes two NGINX-specific annotations by default. |
| `ingress.ingressClassName` | `nginx` | IngressClass name. |
| `ingress.hosts` | `[{host: code.localhost, paths: [/]}]` | Hosts and paths. Uses the `Prefix` path type on Kubernetes 1.19 and later. |
| `ingress.tls` | Not set | TLS entries, each containing `secretName` and `hosts`. |
| `extraPorts` | `[]` | Adds ports to both the main container and Service. Each entry contains `name`, `port`, and `protocol`; the application must listen on the port. |

Default `ingress.annotations`:

```yaml
nginx.org/mergeable-ingress-type: master
nginx.org/path-regex: case_sensitive
```

### Storage

| Parameter | Default | Description |
| --- | --- | --- |
| `persistence.enabled` | `true` | Persists `/home/coder`. When disabled, uses `emptyDir`, whose data is lost when the Pod is deleted. |
| `persistence.storageClass` | Not set | Uses the cluster's default StorageClass. Set to `"-"` to explicitly disable dynamic provisioning. |
| `persistence.accessMode` | `ReadWriteOnce` | Access mode for a newly created PVC. |
| `persistence.size` | `10Gi` | Storage request for a newly created PVC. Expansion support depends on the StorageClass. |
| `persistence.annotations` | `{}` | Annotations for a newly created PVC. |
| `persistence.existingClaim` | Not set | Uses an existing PVC in the same namespace instead of creating one. |
| `persistence.hostPath` | Not set | Uses an existing directory on the node, taking precedence over existingClaim. Pair with appropriate scheduling constraints. |

The default is a single replica with `ReadWriteOnce` storage. Before increasing replicas, evaluate volume mount restrictions and the safety of sharing data between development instances.

### Extensions

| Parameter | Default | Description |
| --- | --- | --- |
| `extraArgs` | `[]` | Additional main container arguments. An argument name and its value must be separate list items. |
| `extraVars` | `[]` | Additional main container environment variables, using the Kubernetes EnvVar format. |
| `lifecycle.enabled` | `false` | Whether to enable main container lifecycle hooks. |
| `lifecycle.postStart` / `lifecycle.preStop` | Not set | Kubernetes lifecycle handlers. |
| `extraContainers` | Empty string | Additional containers as a YAML list string. Supports Helm template expressions. |
| `extraInitContainers` | Empty string | Additional init containers as a YAML list string. Supports Helm template expressions. |
| `extraSecretMounts` | `[]` | Secret mounts using `name`, `mountPath`, `secretName`, and `readOnly`, with optional `subPath` and `defaultMode`. |
| `extraVolumeMounts` | `[]` | Additional volume mounts using `name`, `mountPath`, and `readOnly`. Volume sources are selected in order: `existingClaim`, `hostPath`, or `emptyDir`. Supports optional `subPath`. |
| `extraConfigmapMounts` | `[]` | ConfigMap mounts using `name`, `mountPath`, `configMap`, and `readOnly`, with optional `subPath` and `defaultMode`. |

For example, add a startup argument and an environment variable:

```yaml
extraArgs:
  - --disable-telemetry
extraVars:
  - name: TZ
    value: Asia/Shanghai
```

Use `|` block strings for `extraContainers` and `extraInitContainers`, not direct YAML arrays. Docker-in-Docker usually requires a privileged container and introduces additional security risks. If you use the example in values, also set the corresponding `DOCKER_HOST`.

## Validation and Uninstallation

```bash
helm lint .
helm template code-server . --namespace code-server
helm test code-server --namespace code-server
helm uninstall code-server --namespace code-server
```

When using an override file, also pass `-f custom-values.yaml` to `helm lint` and `helm template`. The `helm test` command requires an installed release.

Uninstalling may delete the PVC created by the Chart. Whether the underlying data is retained depends on the PV reclaim policy. Existing PVCs and Secrets created by the installation hook are not automatically cleaned up as ordinary release resources. Back up your data before uninstalling and inspect any remaining resources as needed.