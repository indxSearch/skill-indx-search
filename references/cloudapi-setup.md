# Indx — Setup & Deployment

This guide covers running, configuring, and deploying the Indx server. For API endpoints and search usage, see [http-api.md](http-api.md).

## Quick Start

```bash
git clone https://github.com/indxSearch/Indx
cd Indx
dotnet run
```

Requires **.NET 10.0 SDK**. The server starts at `https://localhost:5001`.

## First Run

Navigate to `https://localhost:5001/account/register` and create an account. The **first user to register becomes admin** — no pre-configuration needed.

After logging in, go to **Admin → Settings** to configure registration mode, email provider, and other instance settings through the UI.

## Get an API Key

1. Log in at `/Account/Login`
2. Navigate to `/account/api-key` and click **New API key**
3. Choose the team, the access level and, optionally, the datasets the key may reach:
   - **Search only** — search, document lookup, filters, field lists, status. The only level to use in a browser (every visitor can read the key).
   - **Read only** — every read, including export and configuration. For exports, reporting, agents; keep server-side.
   - **Full access** — everything the owner's team role allows. For your own servers and pipelines.
4. Choose the expiry (30, 90, 180 or 365 days), click **Create key**, and copy it — it is shown once

A key never exceeds its owner's role and cannot be changed later. A key is only valid on the server that issued it (each server signs with its own secret), so create it on the server you will call. See [http-api.md](http-api.md#authentication) for what each level can call.

Send the key in every request:
```bash
curl -H "Authorization: Bearer <your-token>" https://localhost:5001/api/...
```

## Teams

Indx organises access around **teams**. A team owns datasets, and every member has the same role on all of the team's datasets:

| Role | Search | Modify data & fields | Delete / transfer |
|------|:------:|:--------------------:|:-----------------:|
| `Admin` | ✓ | ✓ | ✓ |
| `Editor` | ✓ | ✓ | — |
| `Viewer` | ✓ | — | — |

A user can belong to multiple teams. On registration you get a personal team; create more from the account portal's team pages, where you also add members and assign roles.

Because datasets belong to teams, **every dataset API endpoint is scoped to a team**: `/api/teams/{teamName}/datasets/{dataSetName}/{operation}`. List the datasets you can reach with `GET /api/me/datasets` (each entry carries its `teamName` and your `role`). See [http-api.md](http-api.md).

## Configuration

Indx reads from `appsettings.json` and environment variables. Sensitive values should always be set via environment variables, not committed to the repo.

Most settings can also be changed through the **Admin → Settings** UI after first login, without restarting the app.

### JWT Signing Key

Auto-generated on first startup and saved to `IndxData/jwt.key`. No configuration needed. Set `Jwt__Key` explicitly only if you need to share the key across multiple instances:

```bash
Jwt__Key = your-secret-key-minimum-32-characters
```

### Registration Mode

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
| `EmailDomain` | Only emails from `AllowedDomains` |
| `Closed` | No new registrations |
| `Invite` | Only emails added to the invite list by an admin |

In `Invite` mode, admins add emails via **Admin → Users**. An invite email is sent automatically. The invite is consumed (removed from the list) when the user registers.

### Email Provider

Defaults to `Console` (logs emails to stdout — no SMTP required). Switch to Azure Communication Services to send real emails:

```
Email__Provider = AzureCommunicationServices
Email__AzureCommunicationServices__ConnectionString = endpoint=https://...;accesskey=...
Email__FromAddress = noreply@your-verified-domain.com
```

See [docs/EMAIL_SETUP.md](https://github.com/indxSearch/Indx/blob/main/docs/EMAIL_SETUP.md) for ACS setup.

### OAuth (Optional)

```
Authentication__Microsoft__ClientId = your-client-id
Authentication__Microsoft__ClientSecret = your-client-secret
Authentication__Google__ClientId = your-client-id
Authentication__Google__ClientSecret = your-client-secret
```

When credentials are present, sign-in buttons appear on the login page automatically. Leave empty to use local accounts only.

See [docs/OAUTH_SETUP.md](https://github.com/indxSearch/Indx/blob/main/docs/OAUTH_SETUP.md) for app registration steps.

### Email Confirmation

```
Identity__RequireConfirmedEmail = true
```

OAuth users are auto-confirmed. Requires a real email provider (not Console).

### Database Paths

```json
{
  "ConnectionStrings": {
    "IdentityConnection": "./IndxData/identity.db",
    "SearchDataConnection": "./IndxData/indx.db"
  }
}
```

Two SQLite databases, auto-created on first run. Schema migrations run automatically at startup — no manual steps needed.

### License

```
Indx__LicenseFile = /path/to/indx-developer.license
```

Or place the `.license` file in `./IndxData/` — auto-detected on startup. Without a license, a 100,000 document limit applies per dataset. Free developer licenses available at [indx.co](https://indx.co).

## Local Development Secrets

```bash
dotnet user-secrets set "Authentication:Microsoft:ClientId" "your-client-id"
dotnet user-secrets set "Email:AzureCommunicationServices:ConnectionString" "your-connection-string"
```

Stored outside the project directory, never committed to git.

## Deploy to Azure App Service

### 1. Create the App Service

In the Azure Portal:
- Runtime: **.NET 10** (Linux)
- Plan: 2+ vCPUs recommended if reloading or reindexing while serving search traffic

### 2. Deploy

```bash
dotnet publish -c Release
```

Then deploy via Azure CLI, VS Code Azure extension, GitHub Actions, or the Azure Portal.

### 3. Set Application Settings

In the Azure Portal: **App Service → Settings → Environment variables**

Use `__` (double underscore) as the section separator.

**Optional but recommended:**

| Name | Value |
|------|-------|
| `ASPNETCORE_ENVIRONMENT` | `Production` |
| `Registration__Mode` | `EmailDomain` or `Closed` |
| `Registration__AllowedDomains__0` | `yourcompany.com` |

**Email (if using ACS):**

| Name | Value |
|------|-------|
| `Email__Provider` | `AzureCommunicationServices` |
| `Email__AzureCommunicationServices__ConnectionString` | your ACS connection string |
| `Email__FromAddress` | `noreply@yourdomain.com` |
| `Identity__RequireConfirmedEmail` | `true` |

**OAuth (if needed):**

| Name | Value |
|------|-------|
| `Authentication__Microsoft__ClientId` | your client ID |
| `Authentication__Microsoft__ClientSecret` | your client secret |
| `Authentication__Google__ClientId` | your client ID |
| `Authentication__Google__ClientSecret` | your client secret |

### 4. Database Persistence

The databases are stored at `./IndxData/` relative to the app. On Azure App Service this persists across redeployments (files not in the deployment package are preserved). For slot swaps, store databases on a persistent path outside the swap area (`/home/data/`) and point connection strings there via slot-sticky app settings.

### 5. Configure CORS

**App Service → API → CORS** — add your frontend origins:
- `http://localhost:3000` (local dev)
- `https://yourdomain.com` (production)

### Production Checklist

1. Set `Registration__Mode` to `EmailDomain` or `Closed`
2. Configure CORS
3. Set up ACS if using email confirmation or password reset
4. Place `.license` file in `IndxData/` (or configure `Indx__LicenseFile`)
5. Set up OAuth if needed

## Monitor

A live view of the instance: dataset states, document counts, idle-eviction countdowns, process and
native memory, and an event stream carrying state changes, field configuration changes and anything
the server logs.

- **In the console**, under **Admin → Monitor**. Admin only, and the events are in memory only.
  This is the one that works on a hosted deployment, where there is no terminal.
- **In the terminal**, drawn automatically when the server is started from one. `Ctrl+Q` detaches
  it and **leaves the server running**; `Ctrl+C` stops the server; `F10` shuts the server down
  after a confirmation. Start with `--no-monitor`, or set `Indx__Monitor__Enabled` to `false`, to
  turn it off. A failed startup never draws it, so a port conflict or a bad connection string
  still prints the way it always did.
- **Piped**, where stdout is redirected (Azure log stream, `docker logs`, systemd): a full-screen
  application cannot run there, so it prints a periodic status block instead. **Off by default**;
  set `Indx__Monitor__Enabled` to `true` to get it in an App Service log stream, and
  `Indx__Monitor__StatusSeconds` to change how often (default 30). On App Service the console page
  above is usually the better answer.

## Request Size Limits

The server accepts request bodies up to **2GB** — supports loading large JSON datasets via `POST …/load` and `POST …/load/text`.
