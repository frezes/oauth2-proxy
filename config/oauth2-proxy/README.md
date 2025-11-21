Integrating OAuth2-Proxy authentication into the OpenResty reverse proxy allows you to provide unified authentication for services with simple configurations, enhancing application security and ease of access.

## Features

- **Unified Authentication**: Centralized user authentication management for multiple applications using OAuth2.
- **Flexible Configuration**: Easily configure authentication settings and access policies.
- **Scalability**: Leverage OpenResty’s performance and scalability to handle high-traffic scenarios.
- **Customizability**: Extend and customize proxy behavior using OpenResty’s support for Lua scripts.

## Configuration

### 1. Configure OAuth2-Proxy for Unified Authentication Using NodePort

OAuth2-Proxy supports various [OAuth Providers](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/). You can configure the extension to quickly connect to KubeSphere 4.x as your OAuth Provider. By exposing the OpenResty NodePort in the extension, a unified access point is provided for the proxy application. When configuring OAuth2-Proxy, you should 
confirm `openresty.service.nodePort` and modify `global.host` to complete the deployment.

```yaml
global:
  # OAuth2-Proxy service external access address
  # For example, using NodePort, the address is http://172.31.19.4:32080,
  # using Ingress, the hostname is http://172.31.19.4.nip.io:80
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

### 2. Configure OAuth2-Proxy for Unified Authentication Using Ingress

OAuth2-Proxy can be configured to provide unified authentication using Ingress, where Ingress replaces OpenResty as the unified service entry and reverse proxy. In the extension configuration, disable `openresty.enabled`, enable `ingress.enabled`, and modify `global.host` to complete the deployment.

```yaml
global:
  # OAuth2-Proxy service external access address
  # For example, using NodePort, the address is http://172.31.19.4:32080,
  # using Ingress, the hostname is http://172.31.19.4.nip.io:80
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

Additionally, you need to add relevant annotations to the application's Ingress field. Refer to the [ingress-nginx/External OAUTH Authentication](https://kubernetes.github.io/ingress-nginx/examples/auth/oauth-external-auth/) example.

```yaml
...
metadata:
  name: application
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "https://$host/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://$host/oauth2/start?rd=$escaped_request_uri"
...
```

### Note

If you are using KubeSphere 4.x as the OAuth provider, please ensure that the external access address of the KubeSphere Console matches the configuration in the `configmap` kubesphere-config. If there is a mismatch, you can update it with the following steps.

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
        url: "http://172.31.19.4:30880"     # Confirm the issuer address
```

1. Copy the `values.yaml` file from ks-core and create a new file named `custom-kscore-values.yaml`.

    ```bash
    cp ks-core/values.yaml custom-kscore-values.yaml
    ```

2. Modify `portal.hostname` to the actual address.

    ```yaml
    portal:
      ## The IP address or hostname to access ks-console service.
      ## DO NOT use IP address if ingress is enabled.
      hostname: "172.31.19.4"
      http:
        port: 30880
    ```

3. Update ks-core.

    ```bash
    helm upgrade --install -n kubesphere-system --create-namespace ks-core ${kscore_chart_path}  -f ./custom-kscore-values.yaml  --debug --wait
    ```

## Quick Start

Here is an example of how to configure OAuth2 Proxy and access the AlertManager service using NodePort.

In the extension configuration, modify `global.host` and confirm `openresty.service.nodePort`, then modify the `openresty.configs` configuration as follows.

```yaml
openresty:
  configs:
    - name: alertmanager
      description: KubeSphere internal monitoring stack Alertmanager endpoint
      subPath: /alertmanager/
      endpoint: http://whizard-notification-alertmanager.kubesphere-monitoring-system.svc:9093/
      additional:
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
```

> Note:
>
> 1. The connected service must support SubPath access.
> 2. Prometheus requires additional configuration with `routePrefix: /` and `externalUrl: /prometheus/` to support its SubPath.

After configuration, access the external access address of OAuth2-Proxy, such as 172.31.19.4:32080. Once authenticated, you will see the Alertmanager service entry on the homepage, which you can access with a single click.