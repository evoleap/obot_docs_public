# Azure Entra ID Single Sign-On Setup Guide for obot

## Overview

This guide provides step-by-step instructions for configuring Azure Entra ID (formerly Azure AD) single sign-on (SSO) for your obot deployment. By following this guide, you will:

1. Create and configure an Enterprise Application in your Azure tenant
2. Define custom App Roles for user access control
3. Assign users to the appropriate roles
4. Configure environment variables in your Azure Container App deployment to enable SSO

## Prerequisites

- Administrative access to your Azure tenant (Global Administrator or Application Administrator role)
- An existing obot deployment in Azure Container Apps
- Access to configure environment variables in your Container App

## Architecture Overview

obot uses the OmniAuth Entra ID strategy to authenticate users via OpenID Connect. The authentication flow works as follows:

1. Users click "Sign in with Microsoft" on the obot login page
2. They are redirected to Microsoft's authentication endpoint
3. After successful authentication, Microsoft redirects back to obot with an authorization code
4. obot exchanges the code for an access token and retrieves user information
5. obot checks the user's assigned App Roles to determine access level
6. If the user has an appropriate role, they are authenticated and a session is created

## Part 1: Create and Configure the Enterprise Application

### Step 1: Register a New Application

1. Sign in to the [Azure Portal](https://portal.azure.com)
2. Navigate to **Azure Active Directory** (or **Microsoft Entra ID**)
3. Select **Enterprise applications** from the left menu
4. Click **+ New application**
5. Click **+ Create your own application**
6. Enter a name for your application (e.g., "obot Production")
7. Select **Register an application to integrate with Azure AD (App you're developing)**
8. Click **Create**

### Step 2: Configure Basic Application Settings

1. After creation, you'll be redirected to the app registration page
2. Note the following values (you'll need these later):
   - **Application (client) ID**
   - **Directory (tenant) ID**
3. Under **Certificates & secrets**, create a new client secret:
   - Click **+ New client secret**
   - Enter a description (e.g., "obot Production Secret")
   - Choose an expiration period (recommendation: 24 months for production)
   - Click **Add**
   - **IMPORTANT**: Copy the **Value** immediately - it won't be shown again
   - Store this securely - this is your `CLIENT_SECRET`

### Step 3: Configure Redirect URIs

1. In your app registration, go to **Authentication**
2. Click **+ Add a platform**
3. Select **Web**
4. Add the following redirect URI:
   ```
   https://your-obot-domain.azurecontainerapps.io/auth/entra_id/callback
   ```
   Replace `your-obot-domain.azurecontainerapps.io` with your actual obot Container App URL
5. Under **Implicit grant and hybrid flows**, ensure nothing is checked (not needed for authorization code flow)
6. Click **Configure**

### Step 4: Configure API Permissions

1. Go to **API permissions** in your app registration
2. You should see **Microsoft Graph** > **User.Read** already added (delegated)
3. Add the following additional delegated permissions:
   - Click **+ Add a permission**
   - Select **Microsoft Graph**
   - Select **Delegated permissions**
   - Search for and add:
     - `openid`
     - `email`
     - `profile`
4. These permissions typically don't require admin consent for most organizations, but if your organization requires it:
   - Click **Grant admin consent for [Your Organization]**
   - Confirm by clicking **Yes**

## Part 2: Define App Roles

obot supports two levels of access:
- **User**: Standard user access - can submit jobs, view their own jobs, and use the file browser
- **Power User**: Elevated access - can manage users, configure settings, approve upgraded job priorities, etc.

### Step 1: Create App Roles in the Manifest

1. In your app registration, go to **App roles**
2. Click **+ Create app role** to create the first role:

**Regular User Role:**
- **Display name**: `User`
- **Allowed member types**: Select **Users/Groups**
- **Value**: `obot-user` (this is case-sensitive and must match exactly)
- **Description**: `Standard obot user with access to submit and monitor jobs`
- **Enable this app role**: Checked
- Click **Apply**

3. Click **+ Create app role** again for the admin role:

**Power User/Admin Role:**
- **Display name**: `Power User`
- **Allowed member types**: Select **Users/Groups**
- **Value**: `obot-power-user` (this is case-sensitive and must match exactly)
- **Description**: `obot power user with elevated access to system configuration and user management`
- **Enable this app role**: Checked
- Click **Apply**

### Alternative: Edit the Manifest Directly

If you prefer to edit the manifest directly:

1. Go to **Manifest** in your app registration
2. Find the `appRoles` section
3. Add the following JSON (or merge with existing roles):

```json
"appRoles": [
  {
    "allowedMemberTypes": [
      "User"
    ],
    "description": "Standard obot user with access to submit and monitor jobs",
    "displayName": "User",
    "id": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
    "isEnabled": true,
    "lang": null,
    "origin": "Application",
    "value": "obot-user"
  },
  {
    "allowedMemberTypes": [
      "User"
    ],
    "description": "obot power user with elevated access to system configuration and user management",
    "displayName": "Power User",
    "id": "ffffffff-gggg-hhhh-iiii-jjjjjjjjjjjj",
    "isEnabled": true,
    "lang": null,
    "origin": "Application",
    "value": "obot-power-user"
  }
]
```

**Note**: Generate new UUIDs for the `id` fields - don't use the example values above.

4. Click **Save**

## Part 3: Assign Users to App Roles

### Step 1: Navigate to Enterprise Application

1. Go to **Azure Active Directory** > **Enterprise applications**
2. Find and click on your obot application
3. Select **Users and groups** from the left menu

### Step 2: Assign Users

1. Click **+ Add user/group**
2. Under **Users**, click **None Selected**
3. Search for and select the user(s) you want to add
4. Click **Select**
5. Under **Select a role**, click **None Selected**
6. Choose either **User** or **Power User** role
7. Click **Select**
8. Click **Assign**

Repeat this process for each user who needs access to obot.

### Important Notes on Role Assignment

- Users must be assigned to either the **User** or **Power User** role to access obot
- Users without a role assignment will be denied access with the message: "Access denied. You do not have the required app role assigned."
- The **Power User** role provides power user privileges within obot
- The **User** role provides standard user access
- You can change a user's role at any time by removing their current assignment and adding a new one with the desired role

## Part 4: Configure obot Container App Environment Variables

Now that your Entra ID application is configured, you need to provide the connection details to your obot deployment.

### Step 1: Gather Required Information

You'll need the following values:

1. **Application (client) ID** - from your app registration overview page
2. **Client Secret** - the secret value you copied when creating it
3. **Directory (tenant) ID** - from your app registration overview page
4. **Host URL** - your obot application's full URL (e.g., `https://your-obot-domain.azurecontainerapps.io`)

### Step 2: Configure Container App Environment Variables

1. In the Azure Portal, navigate to your **Container Apps** service
2. Select your obot Container App
3. In the left menu, select **Containers** or **Revision management**
4. Click **Create new revision** or **Edit and deploy**
5. Under **Container** or **Environment variables**, add the following environment variables:

| Environment Variable | Value | Description |
|---------------------|-------|-------------|
| `CLIENT_ID` | Your Application (client) ID | The unique identifier for your app registration |
| `CLIENT_SECRET` | Your Client Secret value | The secret used to authenticate your application (mark as secret) |
| `TENANT_ID` | Your Directory (tenant) ID | Your Azure AD tenant identifier |
| `HOST_URL` | `https://your-obot-domain.azurecontainerapps.io` | The full URL of your obot deployment (no trailing slash) |

**Important Security Note**: Mark `CLIENT_SECRET` as a **secret** type environment variable to ensure it's stored securely and not displayed in logs.

### Optional: Configure Custom Role Names

By default, obot looks for roles named "User" and "PowerUser". If you've defined your App Roles with different names, you can override the expected role names using these optional environment variables:

| Environment Variable | Default Value | Description |
|---------------------|---------------|-------------|
| `USER_ROLE` | `obot-user` | The App Role value for standard users |
| `POWER_USER_ROLE` | `obot-power-user` | The App Role value for administrators |

**Example**: If you created roles with values "StandardUser" and "Administrator", add:
- `USER_ROLE` = `StandardUser`
- `POWER_USER_ROLE` = `Administrator`

### Step 3: Deploy the New Revision

1. Review your environment variable configuration
2. Click **Create** or **Deploy** to deploy the new revision with SSO enabled
3. Wait for the revision to be deployed and active

## Part 5: Verify SSO Configuration

### Step 1: Test the Login Flow

1. Navigate to your obot URL in a web browser
2. You should see a "Sign in with Microsoft" button or link
3. Click the sign-in option
4. You'll be redirected to Microsoft's login page
5. Enter your credentials and complete authentication
6. If prompted, consent to the permissions requested by the application
7. You should be redirected back to obot and automatically logged in

### Step 2: Verify Role-Based Access

1. Log in as a user assigned the **User** role
   - Verify they can access job submission and monitoring features
   - Verify they cannot access administrative functions
2. Log in as a user assigned the **PowerUser** role
   - Verify they have elevated access - you should see the + button on the Admin > Projects tab enabled
   - Check that they can access the admin panel and configure settings

### Step 3: Test Access Denial

1. Attempt to log in with a user who has **not** been assigned to any role
2. Verify they see an error message: "Access denied. You do not have the required app role assigned."
3. Verify they are redirected to the login page

## Troubleshooting

### Issue: Users Get "Access Denied" Error

**Possible Causes:**
- User is not assigned to the application in Azure AD
- User is assigned to the application but not to a specific role
- The role value doesn't match what obot expects (default: "User" or "PowerUser")

**Resolution:**
1. Verify the user has been assigned to the Enterprise Application
2. Verify the user has been assigned to either the "User" or "PowerUser" role
3. Check that role values are case-sensitive and match exactly
4. If you've used custom role names, ensure `USER_ROLE` and `POWER_USER_ROLE` environment variables are set correctly

### Issue: Redirect URI Mismatch Error

**Possible Causes:**
- The `HOST_URL` environment variable doesn't match the registered redirect URI
- The redirect URI in Azure AD is incorrect or missing

**Resolution:**
1. Verify the redirect URI in Azure AD is: `https://your-actual-domain.azurecontainerapps.io/auth/entra_id/callback`
2. Verify the `HOST_URL` environment variable is: `https://your-actual-domain.azurecontainerapps.io` (no trailing slash)
3. Ensure both use `https://` (not `http://`)
4. Redeploy the Container App after making changes

### Issue: Client Secret Expired

**Possible Causes:**
- The client secret has expired (they have a maximum lifetime)

**Resolution:**
1. Go to your app registration in Azure AD
2. Navigate to **Certificates & secrets**
3. Create a new client secret
4. Update the `CLIENT_SECRET` environment variable in your Container App
5. Deploy the new revision

### Issue: Users Can't See Their Roles or Get Incorrect Roles

**Possible Causes:**
- Roles are not being included in the token claims
- Microsoft Graph API call is failing to retrieve roles

**Resolution:**
1. Check the Container App logs for error messages related to Graph API calls
2. Verify the application has the required Microsoft Graph permissions
3. If admin consent is required for your organization, ensure it has been granted
4. Contact support if the issue persists - provide relevant log entries from Application Insights

### Issue: User Role Not Updating After Change in Azure AD

**Possible Causes:**
- User has an active session with cached role information
- Role assignment change hasn't propagated yet

**Resolution:**
1. Have the user sign out of obot completely
2. Wait 5-10 minutes for Azure AD changes to propagate
3. Have the user sign in again
4. The new role should be reflected after sign-in

## Advanced Configuration

### Configuring Role Names in the Admin UI

After SSO is enabled and at least one admin user has logged in, you can configure the expected role names directly in the obot admin interface:

1. Log in as an admin user
2. Navigate to the **Admin** section
3. Look for **Application Settings** or **Azure AD Settings**
4. You can configure:
   - **azure_ad_user_role**: The App Role value for standard users (default: "User")
   - **azure_ad_power_user_role**: The App Role value for power users (default: "PowerUser")

These settings override the environment variables `USER_ROLE` and `POWER_USER_ROLE`.

### Using Group Assignments (Optional)

For large deployments, you may want to assign Azure AD groups to roles instead of individual users:

1. In your Enterprise Application, go to **Properties**
2. Ensure **User assignment required?** is set to **Yes**
3. Go to **Users and groups**
4. Click **+ Add user/group**
5. Under **Users**, click **None Selected**
6. Select **Groups** at the top
7. Search for and select the group you want to assign
8. Choose the appropriate role
9. Click **Assign**

All members of the group will inherit the assigned role.

### Automatic User Provisioning

obot automatically provisions users on their first SSO login:

- A new user record is created in the obot database
- The user is marked as an SSO user (`sso_user: true`)
- Username, email, first name, last name, and role are populated from Azure AD
- SSO users do not have a password set in obot (they must authenticate via Azure AD)

On subsequent logins:
- User role is updated if it has changed in Azure AD
- First name and last name are updated if they've changed in Azure AD
- Username and email remain unchanged after initial provisioning

## Security Considerations

1. **Client Secret Protection**: Always mark the `CLIENT_SECRET` as a secret-type environment variable. Never commit it to source control or display it in logs.

2. **Regular Secret Rotation**: Client secrets have expiration dates. Plan to rotate secrets before they expire to avoid service disruption.

3. **Least Privilege**: Assign users the minimum role required for their job function. Use the "User" role by default and only assign "PowerUser" role to those who need elevated access.

4. **Audit Role Assignments**: Regularly review user and group assignments to ensure only authorized users have access.

5. **Monitor Sign-Ins**: Use Azure AD sign-in logs to monitor authentication activity and identify potential security issues.

6. **Conditional Access**: Consider implementing Azure AD Conditional Access policies for additional security (e.g., require MFA, restrict by IP range, etc.).

## Support

For issues or questions regarding SSO configuration:

1. Check the Container App logs in Application Insights for detailed error messages
2. Review this document's troubleshooting section
3. Contact Evoleap support with:
   - Error messages from logs
   - Steps taken to configure SSO
   - Screenshot of the error (if UI-related)
   - Tenant ID and Application ID (never share client secrets)

## Appendix: Environment Variable Reference

Complete list of SSO-related environment variables:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `CLIENT_ID` | Yes | None | Azure AD Application (client) ID |
| `CLIENT_SECRET` | Yes | None | Azure AD client secret (mark as secret) |
| `TENANT_ID` | Yes | None | Azure AD Directory (tenant) ID |
| `HOST_URL` | Yes | None | Full URL of obot deployment (e.g., `https://obot.example.com`) |
| `USER_ROLE` | No | `User` | App Role value for standard users |
| `POWER_USER_ROLE` | No | `PowerUser` | App Role value for power users |

## Appendix: Required API Permissions

The application requires the following Microsoft Graph API permissions (delegated):

- `openid` - Required for OpenID Connect authentication
- `email` - Retrieve user email address
- `profile` - Retrieve user profile information (name, etc.)
- `User.Read` - Read the signed-in user's profile

These permissions are typically granted by the user on first sign-in and don't require admin consent in most organizations.
