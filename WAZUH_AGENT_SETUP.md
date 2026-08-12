# Wazuh Collector Agent Setup (Windows)

This covers deploying the Wazuh agent that will run the `ms-graph` module. It doesn't need to be installed on an endpoint you actively want to monitor with traditional Wazuh FIM/log collection — its only job here is to poll Microsoft Graph on a schedule. In testing, it was installed directly on an administrator's workstation; see the note on production placement at the end of this document.

## 1. Deploy the agent from the Wazuh dashboard

1. In the Wazuh dashboard: **Endpoints** (or **Agents management**) → **Summary** → **Deploy new agent**
2. Select your OS package:

![Deploy new agent — package and server address selection](../assets/screenshots/01-deploy-agent-windows.png)

3. The **server address** field is usually pre-filled with your Wazuh Cloud instance's address — leave it as-is.
4. Under **Optional settings**, set an **agent name** that identifies its purpose, not just the machine it runs on — e.g. `collector-defender` rather than the default hostname. In a multi-tenant setup, use `collector-defender-<client>` so each tenant's collector is unambiguous in the dashboard.
5. The wizard generates an install command (PowerShell for Windows) with the manager address and a one-time enrollment password already embedded.

## 2. Run the install command

Open **PowerShell as Administrator** (right-click → Run as administrator), paste the generated command, and run it. It downloads the MSI package and installs silently — there's no visible progress bar during the `msiexec /q` step, that's expected; wait for the prompt to return (usually under a minute).

Then start the service:
```powershell
NET START Wazuh
```

## 3. Verify the agent is active

Back in the dashboard, under **Endpoints**, the new agent should appear with status **active** (green) within about a minute of starting the service.

![Agent shown as active in the Endpoints list](../assets/screenshots/02-agent-active-dashboard.png)

If it stays on **pending** or **never connected**, double-check that the service actually started (`NET START Wazuh` should report success) and that outbound connectivity to the manager's address on port 1514/TCP isn't blocked by a local firewall.

## 4. Locate the agent's configuration files

On a default Windows install, everything lives under:
```
C:\Program Files (x86)\ossec-agent\
```

Key files:

| File | Purpose |
|---|---|
| `ossec.conf` | The agent's configuration — this is where the `<ms-graph>` block goes |
| `ossec.log` (no extension shown in Explorer, type "Text Document") | The agent's live log — check here after any config change |

The install also creates Start Menu shortcuts (search "Manage Agent" or "Edit conf" in the Windows search bar) which are the simplest way to edit the config and restart the service without using the command line directly:

- **Edit conf** — opens `ossec.conf` for editing (must be run, or the resulting Notepad session must be run, with administrator rights to save changes)
- **Manage Agent** — a small GUI to view status and Start/Stop/Restart the service

## 5. Add the ms-graph configuration block

Open `ossec.conf` (via the **Edit conf** shortcut, running as Administrator), find the closing `</ossec_config>` tag near the end of the file, and insert the block from `config/ms-graph-agent.conf.example` (filled in with your real Azure values) just before it — see `AZURE_SETUP.md` for where those values come from.

![ms-graph block structure inside ossec.conf (placeholder values)](../assets/screenshots/05-ossec-conf-ms-graph-structure.png)

Save the file, then restart the service (via **Manage Agent** → Stop, then Start, or `NET STOP Wazuh` / `NET START Wazuh` in an admin PowerShell).

## 6. Confirm the module is running correctly

Open `ossec.log`, jump to the end (Ctrl+End in Notepad), and search (Ctrl+F) for `ms-graph`. Expected healthy sequence right after a restart:

```
wazuh-modulesd:ms-graph: INFO: Started module.
wazuh-modulesd:ms-graph: INFO: Obtaining access token.
wazuh-modulesd:ms-graph: INFO: Scanning tenant '<tenant-id>'
```

No `WARNING` lines should follow immediately after `Scanning tenant`. If one appears, it typically names the exact missing permission or malformed request — see `TROUBLESHOOTING.md` for the specific errors encountered during this setup and their fixes.

The module re-scans on the interval set in the config (`<interval>5m</interval>` by default), so a new `Scanning tenant` line should appear every 5 minutes indefinitely while the service runs.

## Production note: workstation vs. dedicated server

During initial testing, the collector agent ran on a personal Windows workstation. This surfaced a real limitation: sleep mode and network changes on the workstation caused intermittent `Server unavailable` / `agent is offline` warnings in the log, with corresponding gaps in `ms-graph` polling (see `TROUBLESHOOTING.md` #6). The module recovers automatically once connectivity returns, but a workstation is not a reliable 24/7 collection point.

**Before scaling to multiple client tenants**, migrate the collector to a small dedicated server or VM (2 vCPU / 4 GB RAM is more than sufficient for this workload) with stable network connectivity and no sleep/suspend behavior. This also removes the need to keep a personal device powered on and logged in solely to feed this pipeline.
