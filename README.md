# Sensor

A self-contained Docker stack that turns any host on an internal network into a
security **sensor**. It runs:

- **Wazuh manager** as a *worker node* that registers into your central Wazuh
  cluster over your public IP. Local agents on the internal network enroll and
  report to this sensor; the sensor syncs their data up to the cluster master.
- **Fluent Bit**, which tails the Wazuh alert files and forwards them over TCP
  (`json_lines`) to your central log collector (**Graylog / XDR** ingest).

Everything is driven by a single client-editable file, **`sensor.conf`**. You
never touch the Wazuh or Fluent Bit configuration directly.

```
                          Internal network
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  agent   │  │  agent   │  │  agent   │   ...local endpoints
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │ 1514/1515   │             │
        └─────────────┼─────────────┘
                      ▼
             ┌───────────────────┐   TCP json_lines   ┌───────────┐
             │   S E N S O R     │ ────────────────►  │ Collector │
             │                   │  (Fluent Bit)      │ Graylog / │
             │  Wazuh manager    │                    │   XDR     │
             │   (worker node)   │ ──── cluster 1516 ──► Public IP
             │  + Fluent Bit     │                       (Wazuh master)
             └───────────────────┘
```

## Requirements

- Docker Engine 20.10+ and the Docker Compose plugin
- `bash`, `envsubst` (`gettext-base`), and `openssl` on the host
- Outbound reachability from the sensor to:
  - your Wazuh cluster master on the cluster port (default **1516**)
  - your log collector's TCP input (default **55001**)
- Inbound reachability from local agents to the sensor on **1514/1515**

Sizing: `vm.max_map_count` and file/mem limits are set in the compose file; the
Wazuh manager wants a couple of GB of RAM.

## Quick start

```bash
git clone https://github.com/pbassill/Sensor.git
cd Sensor

# 1. Create your config from the template
cp sensor.conf.example sensor.conf

# 2. Edit it for your site (cluster master public IP, cluster key,
#    collector host, enrollment password, ...)
$EDITOR sensor.conf

# 3. Render the Wazuh + Fluent Bit configuration
./configure.sh

# 4. Bring it up
docker compose up -d

# 5. Watch it register with the cluster and connect to Graylog
docker compose logs -f
```

To change anything later: edit `sensor.conf`, re-run `./configure.sh`, then
`docker compose up -d` to apply.

## The configuration file

`sensor.conf` is the single source of truth. See
[`sensor.conf.example`](sensor.conf.example) for the full annotated list. The
key groups:

| Group          | What it sets                                                    |
|----------------|-----------------------------------------------------------------|
| Wazuh cluster  | Cluster name, node name, **cluster key**, **master public IP**, port |
| Agent enroll   | Enrollment password local agents use to register (port 1515)    |
| Log forwarding | Collector host, TCP port, flush interval, forward-archives flag |
| General        | Wazuh version, timezone, whether to forward full archives       |

`./configure.sh` reads `sensor.conf`, validates it, and renders:

- `generated/wazuh/ossec.conf` — the manager config (cluster block filled in)
- `generated/wazuh/authd.pass` — the enrollment password
- `generated/fluent-bit/fluent-bit.conf` — the collector (TCP json_lines) output
- `.env` — image tags / node name / timezone for Docker Compose

The `generated/` directory and `sensor.conf` are git-ignored — the cluster key
and passwords never leave the host.

### Getting the cluster key

The cluster key **must match** the one on your cluster master and every other
node. On the master:

```bash
openssl rand -hex 16     # if you need a new one; then set it on all nodes
```

Put that value in `WAZUH_CLUSTER_KEY`.

## Deploying agents

Once the sensor is up, roll Wazuh agents out to the endpoints on the local
network. They point at the **sensor's** IP, not the central cluster. Full
per-OS instructions (Linux, Windows, macOS) are in
[`docs/agent-installation.md`](docs/agent-installation.md).

Quick Ubuntu example:

```bash
WAZUH_MANAGER="<SENSOR_IP>" \
WAZUH_REGISTRATION_PASSWORD="<enrollment password>" \
apt-get install -y wazuh-agent
systemctl enable --now wazuh-agent
```

## Operating the sensor

```bash
# Status / logs
docker compose ps
docker compose logs -f wazuh-manager
docker compose logs -f fluent-bit

# List agents that have enrolled with this sensor
docker exec sensor-wazuh-manager /var/ossec/bin/agent_control -l

# Check cluster health (worker <-> master sync)
docker exec sensor-wazuh-manager /var/ossec/bin/cluster_control -l

# Apply config changes
./configure.sh && docker compose up -d

# Stop / remove (named volumes persist state)
docker compose down
```

## Ports

| Port | Proto | Direction | Purpose                                   |
|------|-------|-----------|-------------------------------------------|
| 1514 | TCP   | inbound   | Agent event reporting                     |
| 1515 | TCP   | inbound   | Agent enrollment (authd)                  |
| 1516 | TCP   | outbound  | Cluster sync to the master (public IP)    |
| 514  | UDP   | inbound   | Optional syslog input                     |
| 55000| TCP   | inbound   | Optional Wazuh API                        |
| 2020 | TCP   | inbound   | Optional Fluent Bit metrics/health        |
| 55001| TCP   | outbound  | Fluent Bit → collector (json_lines)       |

## Troubleshooting

- **Worker won't join the cluster:** confirm `WAZUH_CLUSTER_KEY`,
  `WAZUH_CLUSTER_NAME`, and Wazuh version match the master, and that the sensor
  can reach the master's public IP on 1516. Check
  `docker exec sensor-wazuh-manager /var/ossec/bin/cluster_control -l`.
- **No logs in the collector:** verify the collector has a raw/TCP input
  listening on `LOG_DEST_PORT` that accepts `json_lines`, that the sensor can
  reach `LOG_DEST_HOST` on that port, and check
  `docker compose logs -f fluent-bit`. Note only alerts at level ≥ 3 are
  written to `alerts.json` by default (set by `log_alert_level` in the manager).
- **Agents won't connect:** see the troubleshooting section in
  [`docs/agent-installation.md`](docs/agent-installation.md).

## License

MIT — see [LICENSE](LICENSE).
