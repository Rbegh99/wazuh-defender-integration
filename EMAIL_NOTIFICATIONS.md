# Email Notifications Setup

Unlike the agent-side `ms-graph` configuration, email notifications are configured on the **Wazuh manager**, not the collector agent. On Wazuh Cloud, this is accessible from the dashboard itself — no separate server access needed.

## 1. Locate the manager configuration editor

1. Dashboard menu (☰) → **Server management** → **Settings**
2. Under **Main configurations**, click **"Edit configuration"** (top right) — this opens a live editor for the manager's `ossec.conf`, with **Save** and **Restart** buttons built into the interface.

This is a different file from the agent's `ossec.conf` covered in `WAZUH_AGENT_SETUP.md` — same filename, different machine, different purpose.

## 2. Configure the global email settings

Find the `<global>` block near the top of the file and set:

```xml
<global>
  <email_notification>yes</email_notification>
  <smtp_server>wazuh-smtp</smtp_server>
  <email_from>no-reply@wazuh.com</email_from>
  <email_to>YOUR_EMAIL</email_to>
  <email_maxperhour>12</email_maxperhour>
  <email_log_source>alerts.json</email_log_source>
</global>
```

`wazuh-smtp` is Wazuh Cloud's built-in relay — no external mail server setup required. It's rate-limited to 100 emails/hour regardless of any higher value set here.

**`email_notification` defaults to `no`.** Missing this one line means nothing will ever be sent, regardless of every other setting below being correct — this was the first thing worth double-checking when notifications didn't seem to fire.

## 3. Set the alert level threshold

```xml
<alerts>
  <log_alert_level>3</log_alert_level>
  <email_alert_level>10</email_alert_level>
</alerts>
```

`email_alert_level` is a **global** filter: any alert reaching this severity level triggers an email, **regardless of which module or rule group produced it** — not just Defender/`ms-graph` alerts. This single setting is enough on its own; no additional per-group configuration is required or recommended (see the warning below).

### Why level 10, specifically

| Level tested | Result |
|---|---|
| 6 | Far too noisy — routine Windows security-log alerts (`EventChannel` rule group, unrelated to Defender) reached this level constantly, flooding the inbox with hundreds of unread messages within days |
| 10 | Filters out routine OS-level noise while still catching genuinely significant Defender alerts, which were observed reaching level 12–15 in real conditions |

Level 10 is a starting point based on the alert volume seen in this specific environment, not a universal constant — adjust based on your own alert distribution (visible under **Threat Hunting → Dashboard → "Top 10 Alert level evolution"**).

## ⚠️ Do not combine with a granular `<email_alerts>` block

An earlier iteration of this configuration also added a granular block scoped to the `ms-graph` group:

```xml
<!-- DO NOT combine this with the global threshold above -->
<email_alerts>
  <level>6</level>
  <group>ms-graph,</group>
</email_alerts>
```

This resulted in **duplicate emails** — every Defender alert crossing both thresholds triggered the global mechanism *and* this granular one independently, producing two separate emails for the same event. The granular block was removed; the global threshold alone is simpler and sufficient. If you want to reintroduce group-based filtering later (e.g. to route different sources to different addresses), do so instead of the global threshold, not alongside it.

## 4. Correct `<email_alerts>` syntax (if you use it)

If you do end up needing the granular form for a more advanced setup, the criteria are **direct children** of `<email_alerts>` — there is no wrapping `<email_alert>` tag around them, despite how the singular/plural naming might suggest otherwise:

**Incorrect** (fails Wazuh's config validation with `Invalid element in the configuration: 'email_alert'`):

![Wrong nested syntax, causes a validation error](../assets/screenshots/06-email-alerts-wrong-syntax-redacted.png)

**Correct**:

![Correct flat syntax](../assets/screenshots/07-email-alerts-correct-syntax-redacted.png)

## 5. Save and restart

Click **Save**. If the syntax is invalid, the interface shows the validation error inline before anything is applied — nothing is written to disk until it parses correctly. Once saved successfully, click **Restart wazuh-manager-master-0** (or your node's name) to apply the change; a banner will note that restarts can take a moment and a manual refresh may be needed to confirm the new status.

## 6. Testing

There's no built-in "send test email" button. The practical way to confirm delivery is to temporarily lower `email_alert_level` and wait for a normal-severity alert to fire naturally, then raise it back to the intended production value — or simply wait for the next genuine high-severity event, since false negatives here are easy to miss if a config error silently prevents sending.
