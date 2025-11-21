# OAuth2-Proxy 跨集群代理 Prometheus

在多集群环境下，用户可能希望通过 OAuth2-Proxy 代理访问不同集群中的 Prometheus 端点，以实现统一认证和管理。下面介绍如何配置 OAuth2-Proxy 实现跨集群代理 Prometheus。

## 实现原理

如架构所示，用户请求会先到 OAuth2-Proxy 的 OpenRestry 服务， 通过 `/oauth` 接口进行认证，OAauth2-Proxy 收到请求后会向 Auth Provider 进行认证(这里也是KubeSphere), 认证通过后，会根据请求转发到 KubeSphere APIServer， KubeSphere 会根据集群前缀选择集群，通过 ReverseProxy 功能将请求转发到目标集群的 Prometheus 服务，实现跨集群访问。

![](./img/OAuth2-Proxy_cross-cluster_proxy.png)

[ReverseProxy](https://dev-guide.kubesphere.io/extension-dev-guide/zh/feature-customization/extending-api/#reverseproxy) 是 KubeSphere 提供的反向代理功能,提供灵活的 API 反向代理声明，支持 rewrite、redirect、请求头注入、熔断、限流等配置。 通过 ReverseProxy， 我们可以实现跨集群访问。

ReverseProxy 配置示例(WizTelemetry 监控内置)：

```yaml
apiVersion: extensions.kubesphere.io/v1alpha1
kind: ReverseProxy
metadata:
  labels:
    app.kubernetes.io/managed-by: Helm
    kubesphere.io/extension-ref: whizard-monitoring
  name: prometheus
spec:
  directives:
    headerUp:
    - -Authorization
    stripPathPrefix: /proxy/prometheus.io
  matcher:
    method: '*'
    path: /proxy/prometheus.io/*
  upstream:
    url: http://prometheus-k8s.kubesphere-monitoring-system.svc:9090
status:
  state: Available
```

## 配置步骤

### 1. 前提条件

- OAuth2-Porxy 扩展组件版本 》= 7.6.3
- 已按照 OAuth2-Proxy [配置并访问服务](https://docs.kubesphere.com.cn/v4.2.0/11-use-extensions/21-oauth2-proxy/01-config-oauth2-proxy/) 做好基础配置，并验证服务可用
- 登录用户需具体集群访问权限(已验证 cluster-admin 角色用户可访问，由于访问传递了用户 Token，需要用户具备可访问目标集群 ReverseProxy 权限)

### 2. 配置 OAuth2-Proxy 扩展组件

修改 OAuth2-Proxy 扩展组件配置，主要是在 `openresty.configs` 中添加对应集群 Prometheus 端点的配置。例如，假设我们要代理名为 `host` 集群中的 Prometheus 端点，配置如下：

```yaml
    global:
      # OAuth2-Proxy service external access address
      # For example, using NodePort, the address is http://172.31.19.4:32080,
      # using Ingress, the host is http://172.31.19.4.nip.io:80
      host: "http://172.31.19.4:32080"
    openresty:
      configs:
        - name: host prometheus
          description: host 集群 Prometheus 端点
          subPath: /clusters/host/prometheus/               # 注意这里的路径需要和 Prometheus externalUrl 保持一致
          link: /clusters/host/prometheus/query             # 页面访问地址，注意 /clusters/{cluster} 
          endpoint: http://ks-apiserver.kubesphere-system.svc:80/clusters/host/proxy/prometheus.io/  # 借助 KubeSphere ReverseProxy 跨集群访问
          additional: |
            set $kubesphere_token "";
            if ($http_cookie ~* "kubesphere-token=([^;]+)") {
                set $kubesphere_token $1;
            }

            set $auth_header "";
            if ($kubesphere_token != "") {
                set $auth_header "Bearer $kubesphere_token";
            }

            proxy_set_header Authorization $auth_header;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        - name: member prometheus
          description: member 集群 Prometheus 端点
          subPath: /clusters/member/prometheus/
          link: /clusters/member/prometheus/query
          endpoint: http://ks-apiserver.kubesphere-system.svc:80/clusters/member/proxy/prometheus.io/
          additional: |
            set $kubesphere_token "";
            if ($http_cookie ~* "kubesphere-token=([^;]+)") {
                set $kubesphere_token $1;
            }

            set $auth_header "";
            if ($kubesphere_token != "") {
                set $auth_header "Bearer $kubesphere_token";
            }

            proxy_set_header Authorization $auth_header;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
```

## 3. 配置 Prometheus externalUrl

为了确保 Prometheus 正确生成跨集群访问的链接，需要修改 Prometheus 的 `externalUrl` 配置，使其包含集群前缀。

修改 WizTelemetry 监控扩展组件集群 Agent 配置，例如，对于 `host` 集群，配置如下：

```yaml
kube-prometheus-stack:
  prometheus:
    prometheusSpec:
      externalUrl: /clusters/host/prometheus/
```

### 4. 验证访问

完成上述配置后，用户可以通过 OAuth2-Proxy 页面，选择对应集群的 Prometheus 端点进行访问，系统会自动处理认证和跨集群代理请求。


## 扩展

这是一个通用的跨集群代理方案，用户可以根据需要配置其他服务的跨集群访问，只需在 OAuth2-Proxy 中添加相应的配置，并确保目标服务在 KubeSphere 中有对应的 ReverseProxy 配置即可。当前，前提是应用支持 subPath 配置。