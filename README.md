# Sensor

A self-contained Docker stack that turns any host on an internal network into a
security **sensor**. It runs:

- **Sensor** as a *worker node* that registers into the UKCDL central cluster through the UKCDL load balancers (`CENTRAL_HOST`). Local agents on the internal network enroll and report to this sensor; the sensor syncs their
  data up to the cluster master over the LB's cluster port and reaches port TCP:55000.
- **Fluent Bit**, which tails the Wazuh alert files and forwards them over TCP (`json_lines`) to the **Pulse SIEM** , which sits behind the same load balancer.

A single load balancer (`CENTRAL_HOST`, e.g. `xdr.lon.cyber-defence.io`) fronts the whole central estate — the Pulse cluster master, the Pulse API, and the Pulse SIEM.

Everything is driven by a single client-editable file, **`sensor.conf`**. You never touch the local Wazuh or Fluent Bit configuration directly.

```
                          Internal network
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │  agent   │  │  agent   │  │  agent   │   ...local endpoints
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        │ 1514/1515   │             │
        └─────────────┼─────────────┘
                      ▼
             ┌───────────────────┐                    ┌─────────────────────┐
             │   S E N S O R     │  cluster 55002 ──► │  CENTRAL_HOST        │
             │                   │  API 55000    ───► │  (HAProxy LB)        │
             │  Wazuh manager    │                    │                      │
             │   (worker node)   │                    │  55002 → master 1516 │
             │  + Fluent Bit     │  json_lines 55001► │  55000 → API 55000   │
             └───────────────────┘                    │  55001 → Pulse SIEM  │
                                                       └─────────────────────┘
```

## Requirements

- Docker Engine 20.10+ and the Docker Compose plugin
- `bash`, `envsubst` (`gettext-base`), and `openssl` on the host
- Outbound reachability from the sensor to `CENTRAL_HOST` on:
  - **55002** — Wazuh cluster sync (worker → master; LB forwards to 1516)
  - **55000** — Wazuh API
  - **55001** — Pulse SIEM (Fluent Bit `json_lines`; LB forwards to 5555)
- Inbound reachability from local agents to the sensor on **1514/1515**

Sizing: `vm.max_map_count` and file/mem limits are set in the compose file; the
Wazuh manager wants a couple of GB of RAM.

## Quick start

```bash
git clone https://github.com/pbassill/Sensor.git
cd Sensor

# 1. Create your config from the template
cp sensor.conf.example sensor.conf

# 2. Edit it for your site (CENTRAL_HOST load balancer, cluster key,
#    enrollment password, ...)
$EDITOR sensor.conf

# 3. Render the Wazuh + Fluent Bit configuration
./configure.sh

# 4. Bring it up
docker compose up -d

# 5. Watch it register with the cluster and connect to the Pulse SIEM
docker compose logs -f
```

To change anything later: edit `sensor.conf`, re-run `./configure.sh`, then
`docker compose up -d` to apply.

## The configuration file

`sensor.conf` is the single source of truth. See
[`sensor.conf.example`](sensor.conf.example) for the full annotated list. The
key groups:

| Group           | What it sets                                                    |
|-----------------|-----------------------------------------------------------------|
| Central link    | **`CENTRAL_HOST`** load balancer (Pulse cluster + API + SIEM)   |
| Client identity | **`CLIENTID`** — builds the mandatory endpoint agent groups     |
| Wazuh cluster   | Cluster name, node name, **cluster key**, sync port, API port   |
| Agent enroll    | Enrollment password local agents use to register (port 1515)    |
| Log forwarding  | Pulse SIEM `json_lines` port, flush interval, archives flag     |
| General         | Wazuh version, timezone, whether to forward full archives       |

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
network. They point at the **sensor's** IP, not the Pulse cluster. Full per-OS
instructions are in [`docs/agent-installation.md`](docs/agent-installation.md).

**Every endpoint must join the group for its OS**, built from `CLIENTID`:

- Windows → `<CLIENTID>_windows_azure`
- Linux → `<CLIENTID>_linux_azure`

`./configure.sh` prints the exact names and writes them to
`generated/agent-groups.txt`. Quick Ubuntu example:

```bash
WAZUH_MANAGER="<SENSOR_IP>" \
WAZUH_AGENT_GROUP="<CLIENTID>_linux_azure" \
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

| Port | Proto | Direction | Purpose                                       |
|------|-------|-----------|-----------------------------------------------|
| 1514 | TCP   | inbound   | Local agent event reporting                   |
| 1515 | TCP   | inbound   | Local agent enrollment (authd)                |
| 55002| TCP   | outbound  | Cluster sync to master via `CENTRAL_HOST`     |
| 55000| TCP   | outbound  | Wazuh API on `CENTRAL_HOST`                    |
| 55001| TCP   | outbound  | Fluent Bit → Pulse SIEM via `CENTRAL_HOST`    |
| 55000| TCP   | inbound   | This manager's own Wazuh API (optional)       |
| 514  | UDP   | inbound   | Optional syslog input                         |
| 2020 | TCP   | inbound   | Optional Fluent Bit metrics/health            |

## Troubleshooting

- **Worker won't join the cluster:** confirm `WAZUH_CLUSTER_KEY`,
  `WAZUH_CLUSTER_NAME`, and Wazuh version match the master, and that the sensor
  can reach `CENTRAL_HOST` on the cluster port (55002) — the load balancer's
  `cluster_sync` frontend forwards 55002 to the master's 1516. Check
  `docker exec sensor-wazuh-manager /var/ossec/bin/cluster_control -l`.
- **No logs in the Pulse SIEM:** verify the SIEM has a raw/TCP input behind
  `CENTRAL_HOST` on `LOG_DEST_PORT` that accepts `json_lines`, that the sensor
  can reach it, and check
  `docker compose logs -f fluent-bit`. Note only alerts at level ≥ 3 are
  written to `alerts.json` by default (set by `log_alert_level` in the manager).
- **Agents won't connect:** see the troubleshooting section in
  [`docs/agent-installation.md`](docs/agent-installation.md).

## License

MIT — see [LICENSE](LICENSE).
