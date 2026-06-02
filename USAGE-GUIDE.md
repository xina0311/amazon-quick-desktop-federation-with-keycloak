# Usage Guide: Configure Amazon Quick with Keycloak SSO

This guide walks you through connecting your deployed Keycloak IdP to Amazon Quick Web and Desktop.

## Prerequisites

- CloudFormation stack is in `CREATE_COMPLETE` status
- You have access to the AWS Console with permissions to manage Amazon Quick
- Amazon Quick application created (or you will create one)

## Step 1: Gather Stack Outputs

From the CloudFormation console (or CLI), note these values:

```bash
aws cloudformation describe-stacks \
  --stack-name keycloak-quick-idp \
  --query 'Stacks[0].Outputs' \
  --output table
```

Key values you'll need:
- `IssuerURL` — e.g., `https://<IP>.nip.io:8443/realms/aws-realm`
- `SAMLMetadataUrl` — e.g., `https://<IP>.nip.io:8443/realms/aws-realm/protocol/saml/descriptor`
- `SAMLProviderArn` — e.g., `arn:aws:iam::<account>:saml-provider/Keycloak-Quick-IdP`
- `QuickSightRoleArn` — e.g., `arn:aws:iam::<account>:role/QuickSight-Keycloak-Role`
- `ClientID` — `amazon-quick-desktop`
- `AuthEndpoint`, `TokenEndpoint`, `JWKSURI`

## Step 2: Verify Keycloak is Running

1. Open `AdminConsoleURL` in your browser (e.g., `https://<IP>.nip.io:8443/admin/`)
2. Accept the self-signed certificate warning if prompted
3. Login with:
   - Username: `admin`
   - Password: The `AdminPwd` you set during deployment

4. Verify the `aws-realm` exists and contains:
   - OIDC Client: `amazon-quick-desktop`
   - SAML Client: `urn:amazon:webservices`
   - User: `ws-lab-7f3k`

---

## Step 3: Configure Amazon Quick Web (SAML)

### 3.1 Create or Select Amazon Quick Application

1. Go to **Amazon Quick Console** → **Applications**
2. Create a new application or select an existing one
3. In the application settings, find **Authentication / Identity Provider** section

### 3.2 Configure SAML Federation

In the Amazon Quick application IAM Identity Provider settings:

1. **SAML Provider ARN**: Use the `SAMLProviderArn` from stack outputs
2. **IAM Role ARN**: Use the `QuickSightRoleArn` from stack outputs

### 3.3 Configure QuickSight (if using QuickSight integration)

1. Go to **QuickSight Console** → **Manage QuickSight** → **Security & permissions**
2. Under **Identity providers**, configure:
   - **Type**: SAML
   - **IAM Role**: `QuickSightRoleArn`
   - **SAML Provider**: `SAMLProviderArn`

### 3.4 Test Amazon Quick Web Login

1. Navigate to your Amazon Quick Web application URL
2. You should be redirected to Keycloak login
3. Login with:
   - Username: `ws-lab-7f3k`
   - Password: Your `AdminPwd`
4. After successful authentication, you should be redirected back to Amazon Quick Web

---

## Step 4: Configure Amazon Quick Desktop (OIDC)

### 4.1 Enable Amazon Quick Desktop Extension Access

1. Go to **Amazon Quick Console** → **Applications** → Your app
2. Navigate to **Extensions** or **Desktop access** settings
3. Enable Amazon Quick Desktop access for the application

### 4.2 Configure OIDC in Amazon Quick Admin

In the Amazon Quick application's identity settings for Desktop:

| Field | Value |
|-------|-------|
| Issuer URL | `https://<IP>.nip.io:8443/realms/aws-realm` |
| Authorization Endpoint | `https://<IP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/auth` |
| Token Endpoint | `https://<IP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/token` |
| JWKS URI | `https://<IP>.nip.io:8443/realms/aws-realm/protocol/openid-connect/certs` |
| Client ID | `amazon-quick-desktop` |
| Scopes | `openid email profile offline_access` |

> **Note**: The OIDC client uses PKCE (S256) — no client secret is needed.

### 4.3 Configure Amazon Quick Desktop Client

1. Open Amazon Quick Desktop application
2. Go to **Settings** → **Account** or **Sign In**
3. Select **Use custom identity provider** or enterprise SSO option
4. The client should auto-discover settings from the Amazon Quick application configuration

### 4.4 Test Amazon Quick Desktop Login

1. Click **Sign In** in Amazon Quick Desktop
2. A browser window opens with the Keycloak login page
3. Enter credentials:
   - Username: `ws-lab-7f3k`
   - Password: Your `AdminPwd`
4. After login, Amazon Quick Desktop should show "Connected" status

---

## Step 5: Create Additional Users (Optional)

### Via Keycloak Admin Console

1. Open `AdminConsoleURL` → Login as admin
2. Select **aws-realm** from the realm dropdown
3. Go to **Users** → **Add user**
4. Fill in:
   - Username (required)
   - Email (required for SAML)
   - First name / Last name
   - Email verified: ON
5. Click **Create**
6. Go to **Credentials** tab → **Set password**
   - Enter password
   - Temporary: OFF
   - Click **Save**

### Via Keycloak API

```bash
# Get admin token
KC="https://<IP>.nip.io:8443"
TOKEN=$(curl -sk -X POST "$KC/realms/master/protocol/openid-connect/token" \
  -d "username=admin&password=YOUR_ADMIN_PWD&grant_type=password&client_id=admin-cli" \
  | jq -r '.access_token')

# Create user
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

## Troubleshooting

### Keycloak Not Accessible

1. Check EC2 instance is running:
   ```bash
   aws ec2 describe-instances --filters "Name=tag:Name,Values=keycloak-quick-desktop" \
     --query 'Reservations[].Instances[].State.Name'
   ```
2. Check Security Group allows inbound on port 8443
3. Check CloudFormation events for deployment errors:
   ```bash
   aws cloudformation describe-stack-events --stack-name keycloak-quick-idp \
     --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
   ```

### Certificate Issues

- The TLS certificate is issued by Let's Encrypt for `<IP>.nip.io`
- If you see certificate warnings, ensure you're accessing via the nip.io domain, not the raw IP
- Certificate auto-renews but requires port 80 to be accessible

### SAML Login Fails

1. Verify SAML metadata is accessible: open `SAMLMetadataUrl` in browser
2. Check IAM SAML Provider is using the correct metadata:
   ```bash
   aws iam get-saml-provider --saml-provider-arn <SAMLProviderArn>
   ```
3. Ensure the user has an email attribute set in Keycloak (required for SAML NameID)

### OIDC Login Fails

1. Verify OIDC discovery endpoint:
   ```
   https://<IP>.nip.io:8443/realms/aws-realm/.well-known/openid-configuration
   ```
2. Ensure Client ID matches: `amazon-quick-desktop`
3. Check that `offline_access` scope is in the client's default scopes

### Instance Restart

If the EC2 instance is stopped and restarted:
- The Elastic IP remains attached (IP doesn't change)
- Keycloak Docker container auto-restarts (`--restart unless-stopped`)
- Let's Encrypt certificate persists on the EBS volume
- No reconfiguration needed

---

## Security Recommendations for Production

1. **Restrict Security Group**: Limit port 8443 access to known IP ranges
2. **Use a custom domain**: Replace nip.io with a proper domain + Route53
3. **Scope down IAM Role**: Replace `quicksight:*` with minimum required permissions
4. **Enable MFA**: Configure Keycloak to require MFA for users
5. **Regular updates**: Keep Keycloak Docker image updated
6. **Backup**: Snapshot the EBS volume periodically
