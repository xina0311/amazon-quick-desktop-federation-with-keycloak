# 使用指南：为 Amazon Quick 配置 Keycloak SSO

本指南帮助你将部署好的 Keycloak IdP 对接 Amazon Quick Web 和 Desktop。

## 前提条件

- CloudFormation Stack 状态为 `CREATE_COMPLETE`
- 你有 AWS 控制台访问权限，可管理 Amazon Quick
- 已创建 Amazon Quick 应用（或即将创建）

## 步骤 1：获取 Stack 输出

从 CloudFormation 控制台（或 CLI）记录以下值：

```bash
aws cloudformation describe-stacks \
  --stack-name keycloak-quick-idp \
  --query 'Stacks[0].Outputs' \
  --output table
```

你需要的关键值：
- `IssuerURL` — 如 `https://<IP>.nip.io:8443/realms/aws-realm`
- `SAMLMetadataUrl` — 如 `https://<IP>.nip.io:8443/realms/aws-realm/protocol/saml/descriptor`
- `SAMLProviderArn` — 如 `arn:aws:iam::<account>:saml-provider/Keycloak`
- `QuickSightRoleArn` — 如 `arn:aws:iam::<account>:role/QuickSight-Keycloak-SSO-Role`
- `ClientID` — `amazon-quick-desktop`
- `AuthEndpoint`、`TokenEndpoint`、`JWKSURI`

## 步骤 2：验证 Keycloak 运行正常

1. 浏览器打开 `AdminConsoleURL`（如 `https://<IP>.nip.io:8443/admin/`）
2. 如有证书警告，点击继续访问
3. 登录：
   - 用户名：`admin`
   - 密码：部署时设置的 `AdminPwd`

4. 确认 `aws-realm` 存在且包含：
   - OIDC Client：`amazon-quick-desktop`
   - SAML Client：`urn:amazon:webservices`
   - 用户：`ws-lab-7f3k`

---

## 步骤 3：配置 Amazon Quick Web（SAML）

### 3.1 创建或选择 Amazon Quick 应用

1. 进入 **Amazon Quick 控制台** → **Applications**
2. 创建新应用或选择已有应用
3. 在应用设置中找到 **Authentication / Identity Provider** 部分

### 3.2 配置 SAML 联合

在 Amazon Quick 应用的 IAM Identity Provider 设置中：

1. **SAML Provider ARN**：使用 Stack 输出的 `SAMLProviderArn`
2. **IAM Role ARN**：使用 Stack 输出的 `QuickSightRoleArn`

### 3.3 配置 QuickSight（如使用 QuickSight 集成）

1. 进入 **QuickSight 控制台** → **Manage QuickSight** → **Security & permissions**
2. 在 **Identity providers** 下配置：
   - **Type**：SAML
   - **IAM Role**：`QuickSightRoleArn`
   - **SAML Provider**：`SAMLProviderArn`

### 3.4 测试 Amazon Quick Web 登录

1. 访问你的 Amazon Quick Web 应用 URL
2. 应被重定向到 Keycloak 登录页面
3. 输入：
   - 用户名：`ws-lab-7f3k`
   - 密码：你的 `AdminPwd`
4. 认证成功后，应被重定向回 Amazon Quick Web

---

## 步骤 4：配置 Amazon Quick Desktop（OIDC）

### 4.1 启用 Amazon Quick Desktop 扩展访问

1. 进入 **Amazon Quick 控制台** → **Applications** → 你的应用
2. 找到 **Extensions** 或 **Desktop access** 设置
3. 为该应用启用 Amazon Quick Desktop 访问

### 4.2 在 Amazon Quick 管理端配置 OIDC

在 Amazon Quick 应用的 Desktop 身份设置中：

| 字段 | 值 |
|------|------|
| Issuer URL | `https://<IP>.nip.io:8443/realms/aws-realm` |
| Authorization Endpoint | `https://<IP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/auth` |
| Token Endpoint | `https://<IP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/token` |
| JWKS URI | `https://<IP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/certs` |
| Client ID | `amazon-quick-desktop` |
| Scopes | `openid email profile offline_access` |

> **注意**：OIDC Client 使用 PKCE（S256）— 不需要 Client Secret。

### 4.3 配置 Amazon Quick Desktop 客户端

1. 打开 Amazon Quick Desktop 应用
2. 进入 **Settings** → **Account** 或 **Sign In**
3. 选择 **Use custom identity provider** 或企业 SSO 选项
4. 客户端应自动从 Amazon Quick 应用配置中发现设置

### 4.4 测试 Amazon Quick Desktop 登录

1. 在 Amazon Quick Desktop 中点击 **Sign In**
2. 浏览器打开 Keycloak 登录页面
3. 输入凭据：
   - 用户名：`ws-lab-7f3k`
   - 密码：你的 `AdminPwd`
4. 登录后，Amazon Quick Desktop 应显示 "Connected" 状态

---

## 步骤 5：创建更多用户（可选）

### 通过 Keycloak 管理控制台

1. 打开 `AdminConsoleURL` → 以 admin 登录
2. 从 Realm 下拉菜单选择 **aws-realm**
3. 进入 **Users** → **Add user**
4. 填写：
   - Username（必填）
   - Email（SAML 必需）
   - First name / Last name
   - Email verified：ON
5. 点击 **Create**
6. 进入 **Credentials** 标签 → **Set password**
   - 输入密码
   - Temporary：OFF
   - 点击 **Save**

### 通过 Keycloak API

```bash
# 获取 admin token
KC="https://<IP>.nip.io:8443"
TOKEN=$(curl -sk -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d "username=admin&password=YOUR_ADMIN_PWD&grant_type=password&client_id=admin-cli" \
  | jq -r '.access_token')

# 创建用户
curl -sk -X POST "$KC/admin/realms/aws-realm/users" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john.doe",
    "email": "john.doe@example.com",
    "emailVerified": true,
    "enabled": true,
    "firstName": "John",
    "lastName": "Doe",
    "credentials": [{"type": "password", "value": "SecurePass123", "temporary": false}]
  }'
```

---

## 故障排查

### Keycloak 无法访问

1. 检查 EC2 实例是否运行：
   ```bash
   aws ec2 describe-instances --filters "Name=tag:Name,Values=keycloak-quick-desktop" \
     --query 'Reservations[].Instances[].State.Name'
   ```
2. 检查安全组是否允许 8443 端口入站
3. 检查 CloudFormation 事件中的部署错误：
   ```bash
   aws cloudformation describe-stack-events --stack-name keycloak-quick-idp \
     --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
   ```

### 证书问题

- TLS 证书由 Let's Encrypt 为 `<IP>.nip.io` 颁发
- 如看到证书警告，确保通过 nip.io 域名访问，而非裸 IP
- 证书每月自动续期，但需要 80 端口可访问

### SAML 登录失败

1. 验证 SAML metadata 可访问：浏览器打开 `SAMLMetadataUrl`
2. 检查 IAM SAML Provider 是否使用正确的 metadata：
   ```bash
   aws iam get-saml-provider --saml-provider-arn <SAMLProviderArn>
   ```
3. 确保用户在 Keycloak 中设置了 email 属性（SAML NameID 必需）

### OIDC 登录失败

1. 验证 OIDC 发现端点：
   ```
   https://<IP>.nip.io:8443/realms/aws-realm/.well-known/openid-configuration
   ```
2. 确认 Client ID 匹配：`amazon-quick-desktop`
3. 检查 `offline_access` scope 是否在 client 的默认 scope 中

### 实例重启

如果 EC2 实例被停止后重新启动：
- 弹性 IP 保持绑定（IP 不变）
- Keycloak Docker 容器自动重启（`--restart unless-stopped`）
- Let's Encrypt 证书保存在 EBS 卷上
- 无需重新配置

---

## 生产环境安全建议

1. **收紧安全组**：限制 8443 端口仅允许已知 IP 范围访问
2. **使用自定义域名**：用正式域名 + Route53 替代 nip.io
3. **收窄 IAM 权限**：将 `quicksight:*` 替换为最小必要权限
4. **启用 MFA**：在 Keycloak 中为用户启用多因素认证
5. **定期更新**：保持 Keycloak Docker 镜像为最新版本
6. **备份**：定期对 EBS 卷做快照
