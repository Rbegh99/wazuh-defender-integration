# wazuh-defender-integration

Real-world SIEM integration: Microsoft Defender alerts → Wazuh via Graph API, with least-privilege Azure AD scoping, tuned email alerting, and a full troubleshooting log.
wazuh siem xdr microsoft-defender microsoft-graph-api azure-ad cybersecurity mssp security-automation


A hands-on SIEM integration connecting Microsoft Defender security alerts to Wazuh via the Microsoft Graph API. Covers Azure AD app registration with least-privilege permission scoping, alert prioritization using Wazuh's native ruleset, and email notification tuning — documented with the real permission errors and configuration mistakes encountered, not just the working end state.

Configuration and documentation for integrating Microsoft Defender security alerts (via Microsoft Graph API) into Wazuh Cloud, with prioritization and email notifications.

Built and documented from a real hands-on implementation, including the actual permission errors, syntax mistakes, and configuration trade-offs encountered along the way — not just the clean end state. See `docs/TROUBLESHOOTING.md` for the full log.

**Skills demonstrated:** SIEM/XDR integration (Wazuh), Azure AD / Microsoft Entra app registration and least-privilege API permission scoping, Microsoft Graph Security API, XML configuration debugging, alert threshold tuning to control notification noise, and technical documentation.

## Goal

Centralize detection and prioritization of Microsoft Defender security alerts in Wazuh, using the native `ms-graph` module and Wazuh's dedicated ruleset (`0995-microsoft-graph_rules.xml`).

## Architecture

```
Microsoft Defender (detection)
        │
        ▼
Microsoft Graph API (security/alerts_v2, security/incidents,
                      deviceManagement/auditEvents)
        │
        ▼
Wazuh Agent — ms-graph module (polls the API every 5 min)
        │
        ▼
Wazuh Manager — ms-graph ruleset (prioritization, levels 0-16)
        │
        ▼
Wazuh Dashboard + email notifications (level ≥ 10)
```

## Repo contents

| File | Description |
|---|---|
| `config/ms-graph-agent.conf.example` | `<ms-graph>` block to add to the collector agent's `ossec.conf` |
| `config/email-alerts-manager.conf.example` | `<global>` / `<alerts>` blocks to add to the manager's `ossec.conf` for email notifications |
| `docs/AZURE_SETUP.md` | Azure AD setup steps (App Registration, permissions, secret) — with screenshots |
| `docs/WAZUH_AGENT_SETUP.md` | Deploying and configuring the Windows collector agent — with screenshots |
| `docs/EMAIL_NOTIFICATIONS.md` | Setting up manager-side email alerts, including the exact syntax pitfalls hit along the way — with screenshots |
| `docs/TROUBLESHOOTING.md` | Full log of every issue encountered during implementation and its fix |
| `assets/screenshots/` | Redacted screenshots referenced by the docs above (all secrets/PII blacked out) |

## What success looks like

Once the pipeline is working end to end, Defender alerts appear under the dashboard's dedicated **Microsoft Graph API** view (not the generic Threat Hunting view, which mixes in every other alert source):

![Working Microsoft Graph dashboard showing real Defender events](assets/screenshots/08-microsoft-graph-dashboard-working.png)

## Prerequisites

- A Wazuh environment (Cloud or self-hosted)
- A dedicated Wazuh agent acting as collector (see note below)
- An Azure AD tenant with Microsoft Defender active
- Global Administrator rights (or equivalent) on the Azure tenant for admin consent

## Important limitation: one tenant per agent

The `ms-graph` module only supports **a single `<api_auth>` block per instance**. To monitor multiple Azure tenants (MSSP multi-client scenario), a dedicated collector agent is required per tenant, each with its own Azure AD App Registration. See `docs/AZURE_SETUP.md` for the procedure to repeat per tenant.

**Alternative evaluated for multi-tenant use cases**: [CIPP (CyberDrain Improved Partner Portal)](https://github.com/CyberDrain/CIPP) natively ingests Microsoft Graph alerts across multiple tenants via GDAP, with direct routing to a PSA (e.g. ConnectWise). For large-scale MSSP use, CIPP may be preferable to duplicating Wazuh agents per client — the two tools are complementary rather than redundant (CIPP for multi-tenant visibility, Wazuh for deeper SIEM correlation on a given tenant).

## Security

- **Never commit real secrets** (`client_id`, `tenant_id`, `secret_value`) — use the `.example` files as templates and keep your real values out of version control (see `.gitignore`).
- Azure AD permissions granted read-only only (least privilege principle).
- Azure AD secret configured with a short expiration (90 days) rather than the default (6 months).

## About this project

This started as a real integration project for an MSSP-oriented environment. Company-identifying details (organization name, tenant identifiers, instance URLs) have been redacted from all screenshots and text; the technical content — configuration, permission scoping, and the troubleshooting log — is unmodified and reflects the actual implementation.

