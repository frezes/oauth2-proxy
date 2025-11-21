将 OAuth2-Proxy 身份认证功能集成到 OpenResty 反向代理中，通过简易配置为服务提供统一认证，提升应用的安全性和接入的便利性。

## 核心特性

- **统一认证**：通过 OAuth2 对多个应用的用户认证进行集中管理。
- **灵活配置**：轻松配置认证设置和访问策略。
- **可扩展性**：利用 OpenResty 的性能和可扩展性来应对高流量场景。
- **自定义功能**：借助 OpenResty 对 Lua 脚本的支持来扩展和定制代理行为。

## 配置

### 1. 配置 OAuth2-Proxy 以 NodePort 方式为应用提供统一认证入口

OAuth2-Proxy 支持多种 [OAuth Providers](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/)。扩展组件可快速配置接入 KubeSphere 4.x 作为您的 OAuth Provider。通过将扩展组件中的 OpenResty NodePort 对外暴露，为代理应用提供统一访问入口。配置时，主要需要确认 `openresty.service.nodePort` 和修改 `global.host`，以完成扩展组件的部署。

```yaml
global:
  # OAuth2-Proxy service external access address
  # For example, using NodePort, the address is http://172.31.19.4:32080,
  # using Ingress, the host is http://172.31.19.4.nip.io:80
  host: "http://<oauth2-proxy-service-external-access-address>"

  # Kubesphere portal address. For example, http://172.31.19.4:30880
  # No need to set this explicitly, KubeSphere's portal address will be auto-injected.
  portal.url: "http://<kubesphere-console-address>"

openresty:
  enabled: true

  service:
    type: NodePort
    portNumber: 80
    nodePort: 32080
    annotations: {}

oauth2-proxy:
  extraArgs:
    provider: oidc
    provider-display-name: "kubesphere"
    # Issuer address
    # The KubeSphere portal URL is filled by default, but if you use another OAuth Provider, change it
    oidc-issuer-url: "{{ .Values.global.portal.url }}"
```

### 2. 配置 OAuth2-Proxy 以 Ingress 方式为应用提供统一认证入口

OAuth2-Proxy 支持通过 Ingress 方式为应用配置统一认证，在这种场景下，Ingress 将替代 OpenResty 提供统一的服务入口和反向代理功能。在扩展组件配置中，禁用 `openresty.enabled`，启用 `ingress.enabled`，并修改 `global.host`，即可完成扩展组件的部署。

```yaml
global:
  # OAuth2-Proxy service external access address
  # For example, using NodePort, the address is http://172.31.19.4:32080,
  # using Ingress, the host is http://172.31.19.4.nip.io:80
  host: "http://<oauth2-proxy-service-external-access-address>"

  # Kubesphere portal address. For example, http://172.31.19.4:30880
  # No need to set this explicitly, KubeSphere's portal address will be auto-injected.
  portal.url: "http://<kubesphere-console-address>"

openresty:
  enabled: false

oauth2-proxy:
  extraArgs:
    provider: oidc
    provider-display-name: "kubesphere"
    # Issuer address
    # The KubeSphere portal URL is filled by default, but if you use another OAuth Provider, change it
    oidc-issuer-url: "{{ .Values.global.portal.url }}"

  ingress:
    enabled: true
    className: nginx
```

此外，您还需要在应用的 Ingress 字段中添加相关注解，请参考 [ingress-nginx/External OAUTH Authentication](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/) 示例。

```yaml
...
metadata:
  name: application
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "https://$host/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://$host/oauth2/start?rd=$escaped_request_uri"
...
```

### 注意事项

如果您使用 KubeSphere 4.x 作为 OAuth Provider，请确保 KubeSphere Console 外部访问地址与 `configmap` kubesphere-config 中的配置一致，如不一致，需按照以下步骤进行更新。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubesphere-config
  namespace: kubesphere-system
data:
  kubesphere.yaml: |
    authentication:
      issuer:
        url: "http://172.31.19.4:30880"     # 确认 issuer 地址
```

1. 复制 ks-core 的 values.yaml 文件，新建为 `custom-kscore-values.yaml`。

    ```bash
    cp ks-core/values.yaml custom-kscore-values.yaml
    ```

2. 修改 `portal.hostname`，配置为实际地址。

    ```yaml
    portal:
      ## The IP address or hostname to access ks-console service.
      ## DO NOT use IP address if ingress is enabled.
      hostname: "172.31.19.4"
      http:
        port: 30880
    ```

3. 更新 ks-core。

    ```bash
    helm upgrade --install -n kubesphere-system --create-namespace ks-core ${kscore_chart_path}  -f ./custom-kscore-values.yaml  --debug --wait
    ```

### 快速开始

下面以 NodePort 方式为例，介绍如何配置 OAuth2-Proxy 并访问 AlertManager 服务。

在扩展组件配置中，修改 `global.host`，并确认 `openresty.service.nodePort`，然后修改 `openresty.configs` 配置如下。

```yaml
openresty:
  configs:
    - name: alertmanager
      description: KubeSphere 内部监控栈的 Alertmanager 端点
      subPath: /alertmanager/
      endpoint: http://whizard-notification-alertmanager.kubesphere-monitoring-system.svc:9093/
      additional:
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
```

> 注:
>
> 1. 接入的服务必须支持 SubPath 访问。
> 2. Prometheus 需要增加 `routePrefix: /` 和 `externalUrl: /prometheus/` 配置，以支持其 SubPath。

配置完成后，访问 OAuth2-Proxy 的外部地址，例如 172.31.19.4:32080，通过认证登录后，您将在首页看到 Alertmanager 服务的入口，点击即可访问。
