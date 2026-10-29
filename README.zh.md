# code-server Helm Chart

在 Kubernetes 中部署 code-server，通过浏览器使用开发环境。本文基于当前 [values.yaml](values.yaml) 和 Chart 模板编写。

## 版本与默认配置

| 项目 | 当前值 |
| --- | --- |
| Chart 版本 | `3.57.0` |
| Chart appVersion | `4.143.1` |
| 实际运行镜像 | `codercom/code-server:4.144.0` |
| 副本数 | `1` |
| Service | `ClusterIP`，端口 `8080` |
| Ingress | 已启用，IngressClass 为 `nginx` |
| 默认访问地址 | `http://code.localhost/` |
| 数据目录 | `/home/coder` |
| 持久卷 | `10Gi`，`ReadWriteOnce` |
| 运行用户 / 文件系统组 | `1000` / `1000` |

实际镜像由 `image.repository` 和 `image.tag` 决定，不会自动使用 Chart 的 `appVersion`。

## 安装

需要可用的 Kubernetes 集群、Helm 3 和 kubectl。默认配置还需要可用的默认 StorageClass 和匹配的 Ingress Controller。

在仓库根目录执行：

```bash
helm upgrade --install code-server . \
  --namespace code-server \
  --create-namespace
```

建议将自定义配置保存在单独的覆盖文件中，例如 `custom-values.yaml`：

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

将域名解析到 Ingress Controller 的入口地址，并提前在同一命名空间创建 TLS Secret。上述示例用 `null` 移除了默认的两项 NGINX 专用注解；其他注解和 `ingressClassName` 请按实际控制器调整。

默认注解 `nginx.org/mergeable-ingress-type: master` 属于 F5 NGINX Ingress Controller 的 Mergeable Ingress 配置；本 Chart 只创建一个 Ingress，不创建对应的 minion。普通单 Ingress 部署应移除该注解。社区 ingress-nginx 不使用这些 `nginx.org/*` 注解。

### Rancher Git Repository

将 Chart 所在分支推送到你可访问的远程仓库后，在支持 Git 类型 Chart 仓库的 Rancher 版本中添加：

| 字段 | 配置 |
| --- | --- |
| 仓库类型 | Git Repository |
| Git URL | 实际托管此 Chart 的仓库地址 |
| Git 分支 | `helm-chart`，或实际包含根目录 Chart 的分支 |
| Chart 路径 | 若界面提供该字段，填写 `.` |

私有仓库需要配置访问凭据。刷新仓库后选择 `code-server`，设置命名空间并覆盖所需 values。Git Chart 仓库与需要 `index.yaml` 的 HTTP Helm 仓库不同；具体入口取决于 Rancher 版本。

## 访问与认证

默认访问 `http://code.localhost/`，需要让该域名指向 Ingress 入口。远程访问应使用实际域名并启用 HTTPS。

未配置 `existingSecret` 时，Chart 在首次安装时创建包含 24 位随机密码的 Secret。使用上面的 release 名称、命名空间且未覆盖资源名称时，可执行：

```bash
kubectl get secret code-server -n code-server \
  -o jsonpath='{.data.password}' | base64 --decode
```

也可预先在同一命名空间创建包含 `password` 键的 Secret，并设置：

```yaml
existingSecret: code-server-auth
```

使用已有 Secret 时，请从该 Secret 获取密码。模板也支持通过 `password` 指定首次安装的密码，但不建议将明文密码提交到 Git；自动生成的 Secret 使用 `pre-install` hook，修改此值不会在普通升级时自动更新密码。

没有 Ingress Controller 时，安装时设置 `--set ingress.enabled=false`，然后通过端口转发访问：

```bash
kubectl port-forward -n code-server service/code-server 8080:8080
```

浏览器打开 `http://127.0.0.1:8080/`。若修改 release 名称或 `fullnameOverride`，请同步调整命令中的资源名称。

## 配置参数

以下默认值以当前配置为准；“未设置”表示该字段仅在注释示例或模板中提供。

### 工作负载与镜像

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `replicaCount` | `1` | Deployment 副本数；更新策略为 `Recreate`。 |
| `image.repository` | `codercom/code-server` | 镜像仓库。 |
| `image.tag` | `4.144.0` | 镜像标签。 |
| `image.pullPolicy` | `IfNotPresent` | 镜像拉取策略。 |
| `imagePullSecrets` | `[]` | 镜像拉取 Secret 列表，元素包含 `name`。 |
| `nameOverride` | `""` | 覆盖 Chart 名称部分。 |
| `fullnameOverride` | `""` | 覆盖生成的完整资源名称。 |
| `hostnameOverride` | `""` | 覆盖 Pod 主机名。 |
| `existingSecret` | 未设置 | 登录密码 Secret 名称，必须包含 `password` 键。 |
| `password` | 未设置 | 首次安装时使用的密码；未设置则随机生成。 |
| `serviceAccount.create` | `true` | 是否创建 ServiceAccount。 |
| `serviceAccount.name` | `""` | 指定 ServiceAccount；不创建且未指定时使用 `default`。 |
| `serviceAccount.annotations` | `{}` | 当前模板未使用此参数。 |
| `annotations` | `{}` | Deployment 注解。 |
| `labels` | `{}` | 当前模板未使用此参数。 |
| `podAnnotations` | `{}` | Pod 注解。 |
| `podSecurityContext` | `{}` | 当前模板未使用此参数；请使用下面的 `securityContext` 配置。 |
| `priorityClassName` | `""` | Pod 优先级类名称。 |
| `securityContext.enabled` | `true` | 是否设置 Pod 的 fsGroup 和主容器的 runAsUser。 |
| `securityContext.fsGroup` | `1000` | Pod 文件系统组。 |
| `securityContext.runAsUser` | `1000` | 主容器运行用户。 |
| `volumePermissions.enabled` | `true` | 持久化开启时，通过初始化容器修正数据目录所有权。 |
| `volumePermissions.securityContext.runAsUser` | `0` | 权限初始化容器运行用户；默认需要允许 root 初始化容器。 |
| `resources` | `{}` | 主容器资源 requests / limits，默认不设置。 |
| `livenessProbe.enabled` | `true` | 启用 `/healthz` 存活探针。 |
| `readinessProbe.enabled` | `true` | 启用 `/healthz` 就绪探针。 |
| `nodeSelector` | `{}` | 节点选择标签。 |
| `tolerations` | `[]` | Pod 容忍配置。 |
| `affinity` | `{}` | 默认不设置；非空时模板要求 YAML 字符串，通过 `tpl` 渲染。 |

### 网络

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `service.type` | `ClusterIP` | Service 类型。 |
| `service.port` | `8080` | Service 对外端口；主容器 HTTP 端口固定为 `8080`。 |
| `service.loadBalancerClass` | `null` | 使用 LoadBalancer 类型时可指定实现类。 |
| `service.annotations` | `{}` | Service 注解。 |
| `ingress.enabled` | `true` | 是否创建 Ingress。 |
| `ingress.annotations` | 见下文 | 默认包含两项 NGINX 专用注解。 |
| `ingress.ingressClassName` | `nginx` | IngressClass 名称。 |
| `ingress.hosts` | `[{host: code.localhost, paths: [/]}]` | 域名和路径；Kubernetes 1.19 及以上使用 `Prefix` 路径类型。 |
| `ingress.tls` | 未设置 | TLS 配置列表，每项包含 `secretName` 和 `hosts`。 |
| `extraPorts` | `[]` | 同时向主容器和 Service 添加端口，每项包含 `name`、`port`、`protocol`；应用仍需自行监听。 |

默认 `ingress.annotations` 为：

```yaml
nginx.org/mergeable-ingress-type: master
nginx.org/path-regex: case_sensitive
```

### 存储

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `persistence.enabled` | `true` | 持久化 `/home/coder`；关闭后使用 `emptyDir`，Pod 删除时数据丢失。 |
| `persistence.storageClass` | 未设置 | 使用集群默认 StorageClass；设为 `"-"` 时显式禁用动态供给。 |
| `persistence.accessMode` | `ReadWriteOnce` | 新建 PVC 的访问模式。 |
| `persistence.size` | `10Gi` | 新建 PVC 的存储请求；扩容能力取决于 StorageClass。 |
| `persistence.annotations` | `{}` | 新建 PVC 的注解。 |
| `persistence.existingClaim` | 未设置 | 使用同命名空间已有 PVC，不创建新 PVC。 |
| `persistence.hostPath` | 未设置 | 使用节点上已存在的目录，优先于 existingClaim；应配合节点调度约束。 |

默认单副本配合 `ReadWriteOnce`。增加副本前，需要评估存储的挂载限制和多个开发实例共享数据的安全性。

### 扩展配置

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `extraArgs` | `[]` | 主容器额外启动参数；参数名和值必须分别作为列表元素。 |
| `extraVars` | `[]` | 主容器额外环境变量，支持 Kubernetes EnvVar 格式。 |
| `lifecycle.enabled` | `false` | 是否启用主容器生命周期钩子。 |
| `lifecycle.postStart` / `lifecycle.preStop` | 未设置 | Kubernetes 生命周期处理器。 |
| `extraContainers` | 空字符串 | 额外容器的 YAML 列表字符串，支持 Helm 模板表达式。 |
| `extraInitContainers` | 空字符串 | 额外初始化容器的 YAML 列表字符串，支持 Helm 模板表达式。 |
| `extraSecretMounts` | `[]` | Secret 挂载列表，使用 `name`、`mountPath`、`secretName`、`readOnly`，可选 `subPath`、`defaultMode`。 |
| `extraVolumeMounts` | `[]` | 附加卷挂载，使用 `name`、`mountPath`、`readOnly`；卷来源依次选择 `existingClaim`、`hostPath` 或 `emptyDir`，可选 `subPath`。 |
| `extraConfigmapMounts` | `[]` | ConfigMap 挂载列表，使用 `name`、`mountPath`、`configMap`、`readOnly`，可选 `subPath`、`defaultMode`。 |

例如添加启动参数和环境变量：

```yaml
extraArgs:
  - --disable-telemetry
extraVars:
  - name: TZ
    value: Asia/Shanghai
```

`extraContainers` 和 `extraInitContainers` 应使用 `|` 块字符串，而不是直接填写 YAML 数组。Docker-in-Docker 通常需要特权容器，并存在额外安全风险；若使用 values 中的示例，还需设置对应的 `DOCKER_HOST`。

## 检查与卸载

```bash
helm lint .
helm template code-server . --namespace code-server
helm test code-server --namespace code-server
helm uninstall code-server --namespace code-server
```

使用覆盖文件时，为 `helm lint` 和 `helm template` 同样添加 `-f custom-values.yaml`。`helm test` 需要已部署的 release。

卸载可能删除 Chart 创建的 PVC，底层数据是否保留取决于 PV 回收策略；已有 PVC 和安装 hook 创建的 Secret 不会作为普通 release 资源自动清理。卸载前请备份数据，按实际情况检查遗留资源。