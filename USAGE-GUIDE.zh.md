# 使用指南：为 Amazon Quick 配置 Keycloak SSO

中文 | [English](USAGE-GUIDE.md)

本指南帮助你将部署好的 Keycloak IdP 对接 Amazon Quick Web（SAML）和 Amazon Quick Desktop（OIDC）。

## 前提条件

- CloudFormation Stack 状态为 `CREATE_COMPLETE`
- 你有 AWS 控制台管理员权限
- 已准备好 Stack 输出值（运行以下命令获取）

```bash
aws cloudformation describe-stacks \
  --stack-name keycloak-quick-idp \
  --query 'Stacks[0].Outputs' \
  --output table
```

---

## Phase 0：注册 Amazon Quick 账户

如果你还没有 Amazon Quick 账户：

1. 打开 Amazon Quick 控制台，点击 **Sign up for Amazon Quick**

   ![注册](images/01-signup-amazon-quick.png)

2. 填写账户信息：
   - **Account name**：选择一个名称（如 `anycompany-quick`）
   - **Email for account notifications**：你的管理员邮箱
   - **Default region**：US East (N. Virginia)
   - **Authentication method**：选择 **Password-based or Single-Sign On (Recommended)**
   - **Encryption**：Use AWS-managed key (Default)
   - 点击 **Create account**

   ![认证方式](images/02-signup-auth-method.png)

> **重要**：必须选择 "Password-based or Single-Sign On" 才能使用 Keycloak 的 SAML/OIDC 联合登录。

---

## Phase 1：配置 Quick Web SAML SSO

### 步骤 1：打开 SSO 配置页面

1. 登录 Amazon Quick Admin Console
2. 左侧菜单 → **Single Sign-on (SSO)**
3. URL 格式：`https://us-east-1.quicksight.aws.amazon.com/sn/account/<ACCOUNT_NAME>/admin`

### 步骤 2：开启 Email Syncing for Federated Users

在 SSO 页面顶部：
- 将 **Email Syncing for Federated Users** 设为 **ON**
- 这允许 Quick 使用 IdP 传来的预配置邮箱地址

### 步骤 3：配置 Service Provider Initiated SSO

填写 Configuration 部分：

| 字段 | 值 |
|------|------|
| IdP URL | `https://<EIP>.nip.io:8443/realms/aws-realm/protocol/saml/clients/amazon-aws` |
| IdP redirect URL parameter | `RelayState` |

点击 **Save**，等待字段变为不可编辑状态（出现 Edit/Delete 按钮）。

> **警告**：必须先点 Save 再开启 SSO。如果先点 ON 会报错 "Enter a configuration first"。

### 步骤 4：开启 SSO

1. 在 **Status** 下，点击 **ON**
2. 确认对话框 "Turn on SSO?" → 点击 **Turn on SSO**

   ![SSO 配置](images/03-sso-configuration.png)

### 步骤 5：测试 Quick Web SSO

打开无痕浏览器窗口，访问：

```
https://us-east-1.quicksight.aws.amazon.com/sn/auth/signin?account=<ACCOUNT_NAME>&enable-sso=1
```

应跳转到 Keycloak 登录页面，输入：
- 用户名：`ws-lab-7f3k`
- 密码：你的 `AdminPwd`

认证成功后，将重定向回 Amazon Quick Web。

---

## Phase 2：配置 Quick Desktop OIDC（Extension Access）

### 步骤 1：打开 Extension Access 页面

在 Amazon Quick Admin Console 中：
- 左侧菜单 → **Extension access**（在 Permissions 分类下）

### 步骤 2：添加 Extension Access

1. 点击 **Add extension access**
2. 选择 **Amazon Quick — Desktop application for Quick**
3. 点击 **Next** 进入详情页

### 步骤 3：填写 OIDC 配置

   ![添加 Extension Access](images/04-add-extension-access.png)

| 字段 | 值 |
|------|------|
| Name | `QuickDesktop-access`（预填，保持不变） |
| Description | `Desktop application for Quick`（预填） |
| Issuer URL | `https://<EIP>.nip.io:8443/realms/aws-realm` |
| Authorization Endpoint | `https://<EIP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/auth` |
| Token Endpoint | `https://<EIP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/token` |
| JWKS URI | `https://<EIP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/certs` |
| Client ID | `amazon-quick-desktop` |

> **警告**：OIDC 配置字段创建后不可编辑，只能删除重建。请在点击 Add 前仔细核对所有值。

### 步骤 4：提交

点击 **Add**，看到绿色提示 "QuickDesktop-access added successfully"。

---

## Phase 3：创建并激活 Desktop Extension

### 步骤 1：进入 Extensions 页面

切换到 **Amazon Quick Console**（非 Admin Console）：
- 左侧菜单 → **Extensions**

### 步骤 2：创建 Extension

1. 点击右上角 **Create extension**
2. 选择 **QuickDesktop-access**
3. 点击 **Next**

   ![创建 Extension](images/05-create-extension.png)

4. 确认，确保状态显示为 **Active**

### 步骤 3：下载 Desktop 客户端

在 QuickDesktop-extension 行：
- 点击右侧 **...** 菜单
- 选择 **Download for Mac** 或 **Download for Windows**

> **重要**：此步骤不可跳过！没有激活的 Extension，Desktop 客户端无法发现 OIDC 配置。

---

## Phase 4：测试验证

### Quick Web SSO 测试

1. 打开无痕浏览器
2. 访问：`https://us-east-1.quicksight.aws.amazon.com/sn/auth/signin?account=<ACCOUNT_NAME>&enable-sso=1`
3. 跳转到 Keycloak → 输入 `ws-lab-7f3k` 登录 → 进入 Quick Web

### Quick Desktop SSO 测试

1. 启动 Amazon Quick Desktop
2. 选择 **Enterprise sign-in**
3. 输入你的 account name
4. 浏览器弹出 Keycloak 登录页面
5. 输入 `ws-lab-7f3k` / 你的密码
6. 返回 Desktop → 显示 "Connected" 状态

### 快速验证命令

```bash
# OIDC Discovery — 应返回 issuer URL
curl -sk https://<EIP>.nip.io:8443/realms/aws-realm/.well-known/openid-configuration | jq .issuer

# SAML Metadata — 应返回 XML
curl -sk https://<EIP>.nip.io:8443/realms/aws-realm/protocol/saml/descriptor | head -1

# Keycloak 健康检查
curl -sk https://<EIP>.nip.io:9000/health/ready
```

---

## 创建更多用户

### 通过 Keycloak 管理控制台

1. 打开 `https://<EIP>.nip.io:8443/admin/` → 以 `admin` 登录
2. 从 Realm 下拉菜单选择 **aws-realm**
3. 进入 **Users** → **Add user**
4. 填写：
   - Username（必填）
   - Email（SAML 联合登录必需）
   - First name / Last name
   - Email verified：**ON**
5. 点击 **Create**
6. 进入 **Credentials** 标签 → **Set password**
   - 输入密码
   - Temporary：**OFF**
   - 点击 **Save**

### 通过 Keycloak API

```bash
# 获取 admin token
KC="https://<EIP>.nip.io:8443"
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

## 重要说明

- **EIP 已固定**：公网 IP 不会因实例停止/启动而改变，所有配置持久有效。
- **TLS 证书**：每月通过 cron job 自动续期，无需手动干预。
- **首次 SAML 登录**：Quick Web 会要求确认邮箱地址来注册为联合用户。
- **双端共享 Realm**：`aws-realm` 中的同一用户可同时登录 Quick Web 和 Quick Desktop。
- **Keycloak 容器自动重启**：实例重启后 Keycloak 自动恢复运行。

---

## 故障排查

### Keycloak 无法访问

1. 确认实例运行中：检查 EC2 控制台或 `aws ec2 describe-instances`
2. 检查安全组是否允许 8443 端口入站
3. 检查 CloudFormation Stack 事件中是否有错误

### SSO 报错 "Enter a configuration first"

你在保存配置前就尝试开启 SSO。先点 Save，再点 ON。

### OIDC Extension Access 创建失败

- 确认所有 endpoint URL 从浏览器可访问
- URL 末尾不要有多余的斜杠
- Client ID 必须完全匹配：`amazon-quick-desktop`

### Quick Desktop 无法发现 OIDC 配置

- 确保已完成 Phase 3，Extension 状态为 **Active**
- Extension 是将 OIDC 配置关联到 Desktop 客户端的桥梁

### 证书警告

- 通过 `https://<EIP>.nip.io:8443` 访问（不要用裸 IP）
- Let's Encrypt 证书被所有主流浏览器和客户端信任

---

## 生产环境安全建议

1. **收紧安全组**：限制 8443 端口仅允许已知 IP 范围访问
2. **使用自定义域名**：用正式域名 + Route53 替代 nip.io
3. **收窄 IAM 权限**：将 `quicksight:*` 替换为最小必要权限
4. **启用 MFA**：在 Keycloak 中为用户启用多因素认证
5. **定期更新**：保持 Keycloak Docker 镜像为最新版本
6. **备份**：定期对 EBS 卷做快照
