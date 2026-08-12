# Troubleshooting Log

## 1. 403 error on the `incidents` relationship

**Symptom** (`ossec.log`):
```
WARNING: Received unsuccessful status code when attempting to get relationship 'incidents'
logs: Status code was '403' & response was '{"error":{"code":"Forbidden","message":"Missing application roles..."
```

**Cause**: the `SecurityIncident.Read.All` permission had not been granted initially (only `SecurityAlert.Read.All` and `SecurityEvents.Read.All` were).

**Fix**: add the missing permission in Azure AD, grant admin consent, then **restart the agent** to force a fresh token (the cached token doesn't reflect permissions added after the fact).

---

## 2. 403 error on `deviceManagement`/`auditEvents`

**Symptom**:
```
WARNING: ... relationship 'auditEvents' ... Status code was '403'
"message":"Application must have one of the following scopes: DeviceManagementApps.Read.All, DeviceManagementApps.ReadWrite.All"
```

**Cause**: wrong permission granted by mistake (`DeviceManagementConfiguration.Read.All` instead of `DeviceManagementApps.Read.All`). The two names look similar but cover different scopes.

**Fix**: read the exact error message returned by the API — it names the precise permission required. Add the correct permission, grant admin consent, restart the agent.

---

## 3. XML validation error — `email_alert`

**Symptom** (Wazuh Cloud interface, editing manager `ossec.conf`):
```
Error found saving the file.
Could not update configuration in specified node - Invalid element in the configuration: 'email_alert'.
```

**Cause**: incorrect syntax. The `<email_alerts>` block was written with the criteria nested inside a wrapper `<email_alert>` tag:
```xml
<!-- WRONG -->
<email_alerts>
  <email_alert>
    <group>ms-graph</group>
    <level>6</level>
  </email_alert>
</email_alerts>
```

**Fix**: criteria (`level`, `group`, `email_to`, etc.) are **direct children** of `<email_alerts>`, with no wrapper:
```xml
<!-- CORRECT -->
<email_alerts>
  <level>6</level>
  <group>ms-graph,</group>
</email_alerts>
```

---

## 4. Duplicate emails for a single alert

**Symptom**: receiving 2 separate emails for one Defender alert.

**Cause**: two notification mechanisms active simultaneously and overlapping:
1. The global threshold `<alerts><email_alert_level>` (fires an email for **any** alert reaching that level, across all sources)
2. The granular `<email_alerts>` block targeting the `ms-graph` group

A Defender alert exceeding both thresholds triggers both mechanisms independently.

**Fix**: pick a single mechanism. The global threshold alone was kept (simpler, sufficient once tuned to the right level); the granular block was removed.

---

## 5. Excessive notification volume (threshold too low)

**Symptom**: 392+ unread emails within a few days, including routine Windows alerts (`EventChannel`, levels 6-9), drowning out the important Defender alerts.

**Cause**: `email_alert_level` initially set to 6 — too low, captures every alert source on the system (not just Defender), including standard Windows logs collected by the agent.

**Fix**: threshold raised to **10**. Trade-off accepted: minor-severity Defender alerts (level 6-9, e.g. false positives, auto-resolved items) no longer generate an email; only genuinely significant alerts (10+, observed up to level 15 in real conditions) do.

---

## 6. Collector network instability

**Symptom** (`ossec.log`):
```
WARNING: Server unavailable. Setting lock.
WARNING: Process locked due to agent is offline. Waiting for connection...
```
repeated several times over roughly an hour, causing temporary failures in the `ms-graph` module (`No response received`).

**Cause**: sleep mode / network dropouts on the workstation used as the collector (personal PC rather than a dedicated server).

**Fix**: no immediate fix applied — the module resumes automatically after reconnection, with no data loss beyond the downtime window. **Recommendation for production**: migrate the collector to a dedicated server (VM with stable network connectivity, 24/7 availability) before scaling to multiple tenants.
