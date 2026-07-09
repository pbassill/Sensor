# Deploying Wazuh agents to the local network

Once the Sensor is running, deploy Wazuh agents to the endpoints on this
internal network. Agents enroll and report to **this sensor**, not to the
central cluster - the sensor forwards their data upstream.

## Before you start

You need:

- The **sensor host IP** on the local network (the machine running this Docker
  stack). In these examples it is written as `SENSOR_IP`.
- The **enrollment password** you set as `WAZUH_REGISTRATION_PASSWORD` in
  `sensor.conf`. Written below as `ENROLL_PASSWORD`.
- Agents should match the manager's `WAZUH_VERSION` (major.minor).

Network requirements from each agent to the sensor:

| Port | Protocol | Purpose            |
|------|----------|--------------------|
| 1514 | TCP      | Event reporting    |
| 1515 | TCP      | Enrollment (authd) |

---

## Linux (Debian / Ubuntu)

```bash
# 1. Install the agent (set the manager address at install time)
WAZUH_MANAGER="SENSOR_IP" \
WAZUH_AGENT_GROUP="default" \
WAZUH_REGISTRATION_PASSWORD="ENROLL_PASSWORD" \
apt-get install -y wazuh-agent=4.9.2-1
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
WAZUH_REGISTRATION_PASSWORD="ENROLL_PASSWORD" \
yum install -y wazuh-agent-4.9.2

systemctl daemon-reload
systemctl enable --now wazuh-agent
```

## Windows

Run in an elevated PowerShell:

```powershell
# Download the installer
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi `
  -OutFile $env:tmp\wazuh-agent.msi

# Install pointing at the sensor
msiexec.exe /i $env:tmp\wazuh-agent.msi /q `
  WAZUH_MANAGER="SENSOR_IP" `
  WAZUH_REGISTRATION_PASSWORD="ENROLL_PASSWORD" `
  WAZUH_AGENT_GROUP="default"

# Start the service
NET START WazuhSvc
```

## macOS

```bash
sudo installer -pkg wazuh-agent.pkg -target /
sudo /Library/Ossec/bin/agent-auth -m SENSOR_IP -P "ENROLL_PASSWORD"
sudo /Library/Ossec/bin/wazuh-control start
```

---

## Verify enrollment on the sensor

From the sensor host, list the agents that have registered:

```bash
docker exec sensor-wazuh-manager /var/ossec/bin/manage_agents -l
# or
docker exec sensor-wazuh-manager /var/ossec/bin/agent_control -l
```

A freshly enrolled agent shows up as **Active** once it has connected on 1514.

## Troubleshooting

- **Agent stuck "Never connected":** the agent enrolled (1515) but can't reach
  1514. Check firewalls between the agent and `SENSOR_IP`.
- **Enrollment rejected:** the `WAZUH_REGISTRATION_PASSWORD` on the agent does
  not match `sensor.conf`. Re-run `./configure.sh` and
  `docker compose up -d` on the sensor after changing it.
- **Version mismatch warnings:** align the agent package version with the
  manager's `WAZUH_VERSION`.
