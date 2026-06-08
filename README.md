# windows-sysmon-ansible

Ansible project to install and configure **Sysmon** (System Monitor) on
Windows Server hosts over WinRM, for **defensive endpoint monitoring and
detection**.

Sysmon is a Sysinternals tool that logs detailed system activity —
process creation, network connections, file/registry changes, image
loads — to the Windows event log channel
`Microsoft-Windows-Sysmon/Operational`. Pairing it with a config like the
Wazuh process-injection ruleset gives a SIEM rich, structured telemetry
for threat detection and incident response.

> This project is strictly for **defensive blue-team monitoring**. It
> deploys logging/visibility tooling. It contains no offensive content.

## What it does

1. Downloads the official Sysmon ZIP from Sysinternals.
2. Downloads a Sysmon configuration XML (defaults to the Wazuh
   process-injection config).
3. Extracts Sysmon to `C:\Program Files\Sysmon`.
4. Installs Sysmon (accepting the EULA) if it isn't already installed,
   or refreshes the running config if it is. Idempotent on re-runs.
5. Ensures the Sysmon service is running and set to auto-start.
6. (Optional, opt-in) Points the Wazuh agent at the Sysmon eventchannel.

## Requirements

- Ansible control node (Linux/macOS) with:
  - `ansible.windows` collection — `ansible-galaxy collection install ansible.windows`
  - `community.windows` collection — `ansible-galaxy collection install community.windows`
  - `pywinrm` Python package — `pip install pywinrm`
- Windows Server target(s) with **WinRM enabled** and reachable
  (HTTPS/5986 recommended). Sysinternals' Ansible setup guide or
  `ConfigureRemotingForAnsible.ps1` can bootstrap WinRM.
- Local administrator credentials on the targets.

## Layout

```
.
├── inventory.example.ini            # copy -> inventory.ini
├── group_vars/
│   └── windows.example.yml          # copy -> windows.yml (creds/conn)
├── playbooks/
│   └── install-sysmon.yml
├── roles/
│   └── sysmon/
│       ├── defaults/main.yml        # URLs, paths, toggles
│       └── tasks/main.yml
└── README.md
```

## Setup

```bash
# 1. Install required collections
ansible-galaxy collection install ansible.windows community.windows
pip install pywinrm

# 2. Create your inventory and connection vars from the examples
cp inventory.example.ini inventory.ini
cp group_vars/windows.example.yml group_vars/windows.yml

# 3. Edit inventory.ini (hostnames/IPs) and group_vars/windows.yml
#    (user, password, transport). Encrypt the creds with Vault:
ansible-vault encrypt group_vars/windows.yml
```

## Configuration variables

Defined in `roles/sysmon/defaults/main.yml`; override in `group_vars/`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `sysmon_zip_url` | `https://download.sysinternals.com/files/Sysmon.zip` | Sysmon download |
| `sysmon_config_url` | Wazuh sysmonconfig.xml | Sysmon config XML |
| `sysmon_install_dir` | `C:\Program Files\Sysmon` | Install location |
| `sysmon_binary` | `Sysmon64.exe` | Binary to run (64-bit) |
| `sysmon_service_name` | `Sysmon64` | Service used to detect installs |
| `sysmon_configure_wazuh` | `false` | Opt-in: edit Wazuh `ossec.conf` |

The default `sysmon_config_url` is the Wazuh-maintained config:
`https://wazuh.com/resources/blog/detecting-process-injection-with-wazuh/sysmonconfig.xml`

For production, point `sysmon_config_url` at a config you have reviewed
and version-controlled (e.g. a fork of
[`SwiftOnSecurity/sysmon-config`](https://github.com/SwiftOnSecurity/sysmon-config)
or [`olafhartong/sysmon-modular`](https://github.com/olafhartong/sysmon-modular)).

## Running

```bash
# Dry run — shows intended changes without applying them
ansible-playbook -i inventory.ini playbooks/install-sysmon.yml --check

# Apply
ansible-playbook -i inventory.ini playbooks/install-sysmon.yml --ask-vault-pass

# Limit to one host
ansible-playbook -i inventory.ini playbooks/install-sysmon.yml --limit win-server-01
```

## Safe testing / verification

First validate connectivity and authentication without changing anything:

```bash
# Confirm WinRM auth works
ansible -i inventory.ini windows -m ansible.windows.win_ping

# Run the playbook in check mode (no changes)
ansible-playbook -i inventory.ini playbooks/install-sysmon.yml --check --diff
```

On a target host (PowerShell, as admin) confirm the install:

```powershell
# Service is present and running
Get-Service Sysmon64

# Show the active configuration / schema version
& 'C:\Program Files\Sysmon\Sysmon64.exe' -c

# Confirm events are being written
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 10
```

A non-destructive way to generate a benign Sysmon event for testing is to
open Notepad or run `whoami` — both produce a normal process-create
(Event ID 1) entry you can see in the log above.

## Wazuh integration (collect the Sysmon channel)

By default this role does **not** modify your Wazuh agent. The recommended
approach is to add the following block to the agent's `ossec.conf`
(`C:\Program Files (x86)\ossec-agent\ossec.conf`) inside `<ossec_config>`
and restart the agent:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

```powershell
Restart-Service WazuhSvc
```

If you would rather have this project append that block automatically,
set `sysmon_configure_wazuh: true` in `group_vars/windows.yml`. The role
will back up `ossec.conf`, insert the `localfile` entry before
`</ossec_config>`, and restart the Wazuh service.

## Uninstall

```powershell
# On the target host (PowerShell, as admin)
& 'C:\Program Files\Sysmon\Sysmon64.exe' -u force
```

Or via Ansible ad-hoc:

```bash
ansible -i inventory.ini windows -m ansible.windows.win_command \
  -a '"C:\Program Files\Sysmon\Sysmon64.exe" -u force'
```

`-u force` stops and removes the Sysmon driver and service. Remove the
`C:\Program Files\Sysmon` directory afterward if you want a clean state.

## Notes

- Re-running the playbook is safe: it refreshes the Sysmon config on hosts
  where Sysmon is already installed rather than reinstalling.
- Always review a Sysmon config before deploying it fleet-wide — overly
  broad rules can generate high event volume.
