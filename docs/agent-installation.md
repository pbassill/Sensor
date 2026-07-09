# Deploying Wazuh agents to the local network

Once the Sensor is running, deploy Wazuh agents to the endpoints on this
internal network. Agents enroll and report to **this sensor**, not to the
Pulse cluster - the sensor forwards their data upstream to the UK Cyber
Defence SOC.

## Mandatory agent groups

Every endpoint **must** be enrolled into the agent group that matches its OS,
built from the `CLIENTID` in `sensor.conf`:

| Endpoint OS | Agent group              |
|-------------|--------------------------|
| Windows     | `<CLIENTID>_windows_azure` |
| Linux       | `<CLIENTID>_linux_azure`   |

Run `./configure.sh` on the sensor and it prints the exact names for your
client and writes them to `generated/agent-groups.txt`. For example, with
`CLIENTID=acme` the groups are `acme_windows_azure` and `acme_linux_azure`.

These groups are created centrally in the Pulse cluster by UK Cyber Defence and
sync down to this sensor. If you enroll an agent before the group exists, it
lands in `default` until the group syncs - re-check it after a few minutes.

Replace `<CLIENTID>` in every command below with your client ID.

## Before you start

You need:

- The **sensor host IP** on the local network (the machine running this Docker
  stack). In these examples it is written as `SENSOR_IP`.
- The **enrollment password** you set as `WAZUH_REGISTRATION_PASSWORD` in
  `sensor.conf`. Written below as `ENROLL_PASSWORD`.
- Agents should match the sensor's `WAZUH_VERSION` (see `sensor.conf`).

Network requirements from each agent to the sensor:

| Port | Protocol | Purpose            |
|------|----------|--------------------|
| 1514 | TCP      | Event reporting    |
| 1515 | TCP      | Enrollment (authd) |

---

## Linux (Debian / Ubuntu)

```bash
# 1. Install the agent (set the manager address and Linux group at install time)
WAZUH_MANAGER="SENSOR_IP" \
WAZUH_AGENT_GROUP="<CLIENTID>_linux_azure" \
WAZUH_REGISTRATION_PASSWORD="ENROLL_PASSWORD" \
apt-get install -y wazuh-agent=4.14.6-1
# If the package isn't found, add the Wazuh apt repo first - see:
# https://documentation.wazuh.com/current/installation-guide/wazuh-agent/wazuh-agent-package-linux.html

# 2. Enable and start
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent

# 3. Confirm it connected
systemctl status wazuh-agent
tail -f /var/ossec/logs/ossec.log     # look for "Connected to the server"
```

## Linux (RHEL / CentOS / Rocky / Alma)

```bash
WAZUH_MANAGER="SENSOR_IP" \
WAZUH_AGENT_GROUP="<CLIENTID>_linux_azure" \
WAZUH_REGISTRATION_PASSWORD="ENROLL_PASSWORD" \
yum install -y wazuh-agent-4.14.6

systemctl daemon-reload
systemctl enable --now wazuh-agent
```

## Windows

Run in an elevated PowerShell:

```powershell
# Download the installer
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.6-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi

# Install pointing at the sensor, with the Windows group
msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
  WAZUH_MANAGER="SENSOR_IP" `
  WAZUH_REGISTRATION_PASSWORD="ENROLL_PASSWORD" `
  WAZUH_AGENT_GROUP="<CLIENTID>_windows_azure"

# Start the service
NET START WazuhSvc
```

> Other platforms (macOS, etc.) are not covered by the standard Azure groups.
> Contact the UK Cyber Defence SOC for the correct group before enrolling them.

---

## Verify enrollment on the sensor

From the sensor host, list the agents that have registered and their group:

```bash
docker exec sensor-wazuh-manager /var/ossec/bin/agent_control -l
# Confirm the group assignment for a specific agent id (e.g. 001):
docker exec sensor-wazuh-manager /var/ossec/bin/agent_groups -s -i 001
```

A freshly enrolled agent shows up as **Active** once it has connected on 1514,
and should list the `<CLIENTID>_windows_azure` or `<CLIENTID>_linux_azure` group.

## Troubleshooting

- **Agent stuck "Never connected":** the agent enrolled (1515) but can't reach
  1514. Check firewalls between the agent and `SENSOR_IP`.
- **Enrollment rejected:** the `WAZUH_REGISTRATION_PASSWORD` on the agent does
  not match `sensor.conf`. Re-run `./configure.sh` and
  `docker compose up -d` on the sensor after changing it.
- **Agent landed in `default`, not its client group:** the group had not synced
  from the Pulse cluster yet, or the name was mistyped. Confirm the exact name
  in `generated/agent-groups.txt` and that UK Cyber Defence has created it
  centrally; the agent moves once the group syncs.
- **Version mismatch warnings:** align the agent package version with the
  sensor's `WAZUH_VERSION`.
