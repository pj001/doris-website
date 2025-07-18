---
{
    "title": "安全概览",
    "language": "zh-CN"
}
---

Doris 提供以下机制管理数据安全：

身份认证：Doris 支持用户名/密码与 LDAP 认证方式。

- 内置认证：Doris 内置了用户名/密码的认证方式，可以自定义密码策略；
  
- LDAP 认证：Doris 可以通过 LDAP 服务集中管理用户凭证，简化访问控制并增强系统的安全性。

权限管控：Doris 支持基于角色的访问控制或继承 Ranger 实现集中化的权限管理。

- 基于角色的访问控制（RBAC），Doris 可以根据用户角色与权限，限制其对数据库资源的访问与操作；
  
- Ranger 权限管理：Doris 可以通过集成 Ranger 实现集中化的权限管理，允许管理员为不同的用户和组设置细粒度的访问控制策略。
  
审计与日志记录：Doris 可以开启审计日志，记录用户的所有操作行为，包括登录，查询，数据修改等行为，便于事后审计与问题追踪；

数据加密与脱敏：Doris 支持对表中的数据进行加密与脱敏，防止未授权的访问造成敏感数据泄漏；

数据传输加密：Doris 支持 SSL 加密协议，确保客户端与 Doris 服务器之间的数据传输安全，防止数据在传输过程中被窃取或篡改；

细粒度访问控制：Doris 中可以基于规则配置数据行/列管控用户访问权限。
