# IndxCloudApi — Setup & Deployment

This guide covers downloading, running, configuring, and deploying the IndxCloudApi server. For API endpoints and search usage, see [http-api.md](http-api.md).

## Quick Start

```bash
git clone https://github.com/indxSearch/IndxCloudApi
cd IndxCloudApi
dotnet run
```

Requires **.NET 9.0 SDK**. The server starts at `https://localhost:5001`.

## Register a User

1. Open `https://localhost:5001/Account/Register` in a browser
2. Fill in email and password
3. Submit — you're now logged in

**Security note:** By default, registration is open to anyone with any email, and email confirmation is not required. Before deploying to production, review [Registration Control](#registration-control) and [Email Confirmation](#email-confirmation) to restrict access and require verified emails.

## Get an API Key

1. Log in at `https://localhost:5001/Account/Login`
2. Go to `https://localhost:5001/Account/ApiKey`
3. Select token duration from the dropdown: **30**, **90**, **180**, or **360** days
4. Click **Generate API Token**
5. Copy the token using the copy button

The page shows the token, its expiration timestamp, and curl examples for using it. You can regenerate a new token at any time (replaces the previous one).

Use the token in all API requests:
```bash
curl -H "Authorization: Bearer <your-token>" https://localhost:5001/api/...
```

Alternatively, get a short-lived token programmatically:
```bash
curl -X POST https://localhost:5001/api/Login \
  -H "Content-Type: application/json" \
  -d '{"userEmail": "you@example.com", "userPassWord": "YourPass1!"}'
```

## Setting Secrets and Configuration

IndxCloudApi reads configuration from `appsettings.json`, but sensitive values (JWT keys, OAuth secrets, connection strings) should be set via environment variables instead.

### Local Development — dotnet user-secrets

```bash
# Initialize (once per project)
dotnet user-secrets init

# Set secrets
dotnet user-secrets set "Jwt:Key" "your-secret-key-minimum-32-characters"
dotnet user-secrets set "Authentication:Google:ClientId" "your-client-id"
dotnet user-secrets set "Authentication:Google:ClientSecret" "your-client-secret"
dotnet user-secrets set "Authentication:Microsoft:ClientId" "your-client-id"
dotnet user-secrets set "Authentication:Microsoft:ClientSecret" "your-client-secret"
dotnet user-secrets set "Email:AzureCommunicationServices:ConnectionString" "your-connection-string"
```

Secrets are stored outside the project directory and override `appsettings.json` values. They are never committed to git.

### Azure App Service — Application Settings

In the Azure Portal: **App Service → Configuration → Application settings**

Or via CLI:
```bash
az webapp config appsettings set --name your-app --resource-group your-rg --settings \
  Jwt__Key="your-secret-key-minimum-32-characters" \
  Authentication__Google__ClientId="your-client-id" \
  Authentication__Google__ClientSecret="your-client-secret" \
  Registration__Mode="EmailDomain" \
  Registration__AllowedDomains__0="yourcompany.com"
```

Note: Azure uses `__` (double underscore) as the section separator instead of `:`.

### Which values need secrets

| Setting | Local | Azure | Required |
|---------|-------|-------|----------|
| `Jwt:Key` | `dotnet user-secrets` | App Settings | Yes (production) |
| `Authentication:Google:ClientId/Secret` | `dotnet user-secrets` | App Settings | Only if using Google OAuth |
| `Authentication:Microsoft:ClientId/Secret` | `dotnet user-secrets` | App Settings | Only if using Microsoft OAuth |
| `Email:AzureCommunicationServices:ConnectionString` | `dotnet user-secrets` | App Settings | Only if sending real emails |
| `Registration:Mode` | `appsettings.json` | App Settings | No (defaults to Open) |
| `ConnectionStrings:*` | `appsettings.json` | Connection Strings | No (auto-configured) |

## Configuration (appsettings.json)

Non-secret settings can go directly in `appsettings.json`.

### JWT Security

**Required for production** — change the default signing key:

```bash
dotnet user-secrets set "Jwt:Key" "your-secret-key-minimum-32-characters"
```

Or in `appsettings.json` (development only):
```json
{
  "Jwt": {
    "Key": "your-secret-key-minimum-32-characters",
    "Issuer": "IndxCloudApi"
  }
}
```

### Registration Control

Control who can create accounts:

```json
{
  "Registration": {
    "Mode": "Open",
    "AllowedDomains": []
  }
}
```

| Mode | Behavior |
|------|----------|
| `Open` | Anyone can register (default) |
| `EmailDomain` | Only emails from `AllowedDomains` can register |
| `Closed` | No new registrations |

Example — restrict to company emails:
```json
{
  "Registration": {
    "Mode": "EmailDomain",
    "AllowedDomains": ["yourcompany.com"]
  }
}
```

### Database Paths

IndxCloudApi uses two SQLite databases. Default paths:

```json
{
  "ConnectionStrings": {
    "IdentityConnection": "./IndxData/identity.db",
    "SearchDataConnection": "./IndxData/indx.db"
  }
}
```

These auto-create on first run. No setup needed.

### Email Provider

Controls how confirmation and password reset emails are sent:

```json
{
  "Email": {
    "Provider": "Console",
    "FromAddress": "noreply@example.com",
    "FromName": "Indx Search"
  }
}
```

| Provider | Behavior |
|----------|----------|
| `Console` | Logs emails to stdout (default, for development) |
| `AzureCommunicationServices` | Sends real emails via Azure |

For Azure Communication Services:
```json
{
  "Email": {
    "Provider": "AzureCommunicationServices",
    "AzureCommunicationServices": {
      "ConnectionString": "your-azure-connection-string"
    },
    "FromAddress": "noreply@yourdomain.com",
    "FromName": "Your App"
  }
}
```

### Email Confirmation

```json
{
  "Identity": {
    "RequireConfirmedEmail": false
  }
}
```

Set to `true` to require email confirmation before users can log in. Requires a working email provider (not `Console`).

### OAuth (Optional)

Support Google and/or Microsoft login:

```json
{
  "Authentication": {
    "Google": {
      "ClientId": "your-google-client-id",
      "ClientSecret": "your-google-client-secret"
    },
    "Microsoft": {
      "ClientId": "your-microsoft-client-id",
      "ClientSecret": "your-microsoft-client-secret"
    }
  }
}
```

### License

- **No license**: 100,000 document limit per dataset
- **Extended license (free)**: Unlimited — register at [indx.co](https://indx.co)

Place the `.license` file in the `./IndxData/` directory. The server auto-detects it on startup.

## Deploy to Azure App Service

### 1. Create the App Service

In the Azure Portal:

1. Go to **App Services → Create**
2. Select **Windows** as the operating system
3. Choose **.NET 9** as the runtime stack
4. Select a plan — RAM is the main constraint. Any vCPU count works, but **2 vCPUs or more** is recommended if you plan to reload or reindex datasets while serving search traffic

### 2. Deploy the Code

Two recommended approaches:

**Visual Studio:** Right-click the project → Publish → select your App Service.

**Command line + VS Code Azure extension:**
```bash
dotnet publish -c Release
```
Then use the Azure extension in VS Code to deploy the `publish` output folder to your App Service.

### 3. Set Environment Variables

In the Azure Portal: **App Service → Settings → Environment variables**

Add each setting as a new application setting. Azure uses `__` (double underscore) as the section separator instead of `:`.

**Required:**

| Name | Value | Notes |
|------|-------|-------|
| `Jwt__Key` | `your-secret-key-minimum-32-characters` | Must change from default |

**Recommended:**

| Name | Value | Notes |
|------|-------|-------|
| `Registration__Mode` | `EmailDomain` or `Closed` | Restrict who can register |
| `Registration__AllowedDomains__0` | `yourcompany.com` | First allowed domain (add `__1`, `__2` for more) |

**If using OAuth:**

| Name | Value |
|------|-------|
| `Authentication__Google__ClientId` | your Google client ID |
| `Authentication__Google__ClientSecret` | your Google client secret |
| `Authentication__Microsoft__ClientId` | your Microsoft client ID |
| `Authentication__Microsoft__ClientSecret` | your Microsoft client secret |

**If using Azure Communication Services for email:**

| Name | Value |
|------|-------|
| `Email__Provider` | `AzureCommunicationServices` |
| `Email__AzureCommunicationServices__ConnectionString` | your ACS connection string |
| `Email__FromAddress` | `noreply@yourdomain.com` |
| `Email__FromName` | your app name |
| `Identity__RequireConfirmedEmail` | `true` |

### 4. Set Up Email (Azure Communication Services)

If you need email confirmation or password reset, set up Azure Communication Services:

1. In the Azure Portal, create a **Communication Services** resource
2. Go to the resource → **Email → Domains** and configure a sending domain
3. Go to **Keys** and copy the connection string
4. Add the connection string as an environment variable on your App Service (see table above)

This is a standard Azure Communication Services setup — any AI agent can guide you through the specifics.

### 5. Configure CORS

In the Azure Portal: **App Service → API → CORS**

Add the origins that need access to the API:
- `http://localhost:3000` (or your local dev port) for development
- `https://yourdomain.com` for production

This is required for frontend applications calling the API from a browser.

### 6. Database Paths

No action needed. In production, database paths automatically adjust to Azure persistent storage (`D:\home\data\`). SQLite databases are created on first run.

### Production Checklist

1. Change the JWT signing key (`Jwt__Key`)
2. Set registration mode to `EmailDomain` or `Closed`
3. Configure CORS with your frontend origins
4. Set up Azure Communication Services if using email confirmation
5. Place `.license` file in `D:\home\data\` on Azure (or `./IndxData/` locally)
6. Set up OAuth if needed

## Request Size Limits

The server accepts request bodies up to **2GB** (configured via Kestrel). This supports loading large JSON datasets via `LoadString` and `LoadStream`.
