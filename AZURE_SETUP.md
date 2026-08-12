# Azure AD (Microsoft Entra ID) Setup

Steps to follow **once per monitored tenant** (your own organization, then each client in a multi-tenant scenario).

## 1. Create the App Registration

1. Go to [portal.azure.com](https://portal.azure.com)
2. Search for **"App registrations"** in the top search bar — do **not** use "Enterprise applications", which is a different view (runtime access/roles, not the developer configuration view where secrets and API permissions live). Mixing up these two is a common early mistake.
3. Click **New registration**
4. Suggested name: `wazuh-defender-<tenant-name>` (e.g. `wazuh-defender-acme`) — keep it descriptive if you'll manage several tenants later
5. Account type: **Accounts in this organizational directory only** (single tenant)
6. Click **Register**
7. On the **Overview** page, note down the **Application (client) ID** and **Directory (tenant) ID** — both are shown directly on this page and also in the App registrations list view

These two values are identifiers, not secrets by themselves — they're safe to reference in documentation or tickets, similar to a username. The actual secret is created in the next step.

## 2. Create a client secret

1. In the left menu: **Certificates & secrets** → tab **Client secrets** → **New client secret**
2. Give it a description (e.g. `wazuh-ms-graph`)
3. Expiration: **90 days** recommended, rather than the default 180 days (6 months) — shorter expiration reduces the exposure window if the secret is ever compromised, at the cost of needing to rotate it more often
4. Click **Add**
5. **Copy the Value column immediately** (not the Secret ID column) using the copy icon next to it — the full value is only displayed once, right after creation. If you navigate away before copying it, you'll need to delete this secret and create a new one.

![Client secret created — value redacted](../assets/screenshots/03-azure-client-secret-redacted.png)
*The Value column (redacted above) contains the actual credential. The Secret ID column, unlike the Value, is just an internal reference and not sensitive.*

## 3. Add API permissions

Left menu: **API permissions** → **Add a permission** → **Microsoft Graph** → choose **Application permissions** (not *Delegated permissions* — delegated permissions require an interactively signed-in user, which doesn't apply here since the ms-graph module runs unattended as a background service).

Search for and check each of the following, one at a time:

| Permission | Required for |
|---|---|
| `SecurityAlert.Read.All` | Individual security alerts (`alerts_v2`) |
| `SecurityEvents.Read.All` | Security events |
| `SecurityIncident.Read.All` | Correlated Defender XDR incidents (`incidents`) |
| `DeviceManagementApps.Read.All` | Intune device audit logs (`auditEvents`) — **optional**, only needed if using the `deviceManagement` resource block |

After adding permissions, the list should show them all under **Microsoft Graph**, Type = **Application**:

![Permissions granted, all showing green checkmarks](../assets/screenshots/04-azure-permissions-granted.png)

⚠️ **Common pitfall**: `DeviceManagementApps.Read.All` is easily confused with `DeviceManagementConfiguration.Read.All` — they sound interchangeable but cover different Graph API scopes. Only the former works for the `deviceManagement/auditEvents` relationship. If you grant the wrong one, the agent log will show a 403 error that explicitly names the correct permission — trust that message over guessing. See `TROUBLESHOOTING.md` #2 for the exact error text.

## 4. Grant admin consent

Still on the **API permissions** page: click **"Grant admin consent for [organization name]"** (near the top, above the permissions table) → confirm **Yes** in the popup.

This requires **Global Administrator** or **Privileged Role Administrator** rights on the tenant. Every permission's **Status** column should turn into a green checkmark reading "Granted for [organization]" — a permission still showing an orange warning icon means the consent step hasn't been completed for that specific permission yet (this can happen if you add permissions in two separate batches; each batch needs its own consent click).

## 5. Fill in the values in Wazuh

Copy `config/ms-graph-agent.conf.example` and fill in the 3 values:

```xml
<client_id>YOUR_CLIENT_ID</client_id>
<tenant_id>YOUR_TENANT_ID</tenant_id>
<secret_value>YOUR_AZURE_SECRET</secret_value>
```

Insert the completed block into the **collector agent's** `ossec.conf` (not the manager's), then restart the agent service so it picks up the new configuration. See the main `README.md` for where exactly to place this file on a Windows agent.

**After granting a new permission on an already-running agent**: a simple config reload isn't enough — you must **restart the agent service**. The module caches its Azure access token, and that cached token won't reflect permissions added after it was issued. This applies whether you're adding permissions for the first time or fixing a wrong one.

## Multi-tenant (MSSP) scenario

Each client tenant requires its own App Registration (steps 1-4 above, repeated) and its own dedicated Wazuh collector agent — the `ms-graph` module only supports one tenant per running instance (see the main README for why, and for an alternative approach using CIPP at scale).
