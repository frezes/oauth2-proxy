<!---
Please do not delete this line of version tag
RELEASE_MARK v4.2.1 RELEASE_MARK
Please do not delete this line of version tag
-->

## v7.6.3

### 优化

- 优化 OpenResty Helm 模板渲染以支持更多自定义配置


<!---
Please do not delete this line of version tag
RELEASE_MARK v4.1.2 RELEASE_MARK
Please do not delete this line of version tag
-->

## v7.6.2

这是 **OAuth2-Proxy** 扩展组件的第一个版本, 将 OAuth2-Proxy 集成到 OpenResty 代理中，实现强大的认证和访问控制功能，提升应用的安全性和用户管理的便利性。

### 新特性

- 优化 OAuth2-Proxy 的部署流程，降低应用对接的复杂度，使集成更加便捷
- 增加对 NodePort 和 Ingress 的支持，提供了更多的部署选项，提升灵活性
- 引入邮箱白名单认证功能，通过邮箱白名单进行鉴权控制，确保只有被授权的用户可以访问服务，增强安全性