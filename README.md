# UDP Service Discovery with SDN (Mininet + Ryu)

A service discovery system built on **UDP**, where application availability drives **network forwarding policy**. Servers advertise themselves to a registry, clients discover services by name, and when a server crashes the registry notifies a **Ryu SDN controller**, which installs an OpenFlow rule on the switch to drop traffic to the dead host. When the server comes back, the rule is removed automatically.

Built and tested on Mininet + Open vSwitch (OpenFlow 1.3) + Ryu.

---

## Features

- **UDP service advertisements** with periodic heartbeats (`REGISTER` every 2 s, TTL 6 s)
- **In-memory service registry** with TTL-based expiry
- **Service discovery** by name, or list everything; clients then call the service directly (`REQUEST` / `RESPONSE`)
- **Timeout handling** in the registry (TTL), client (socket timeouts) and server (missed ack)
- **Failure detection**: crash (TTL expiry) and graceful stop (`UNREGISTER`)
- **SDN traffic control**: dead host gets a priority-100 drop rule; recovery removes it
- **Dynamic registration**: no static service list; any service name can appear at any time
- Reproducible Mininet topology (1 registry, 2 servers, 2 clients, 1 switch)
- Protocol unit tests and a discovery latency benchmark

## Architecture

```text
 server1 (10.0.0.2:10000) ----\
                               \
 server2 (10.0.0.3:10000) ------ s1 (OpenFlow 1.3) ------ registry (10.0.0.1:9999/UDP)
                               /                              ^
 client1 (10.0.0.4) ----------/                               | UDP DISCOVER / results
 client2 (10.0.0.5) -----------------------------------------+

 registry --(append JSON event to shared file)--> Ryu controller --(OpenFlow FlowMod)--> s1
```

| Node | IP | Role |
|---|---|---|
| `registry` | 10.0.0.1 | UDP registry, port 9999 |
| `server1` | 10.0.0.2 | Example service (`weather`), port 10000 |
| `server2` | 10.0.0.3 | Example service (`maps`), port 10000 |
| `client1`, `client2` | 10.0.0.4, 10.0.0.5 | Discovery clients |
| `s1` | n/a | Open vSwitch, OpenFlow 1.3 |

All application traffic is UDP (one JSON object per datagram). The only TCP in the setup is the OpenFlow control channel between the switch and Ryu.

## How it works

**Normal flow**

1. A server sends `REGISTER` (name, IP, port, TTL) to the registry and gets `REGISTERED` back.
2. It repeats every 2 s (TTL / 3), so a single lost packet does not cause a false expiry.
3. A client sends `DISCOVER`; the registry replies with only the non-expired entries and their remaining TTL.
4. The client sends `REQUEST` straight to the server's IP and port and receives `RESPONSE`.

**Failure and recovery**

1. A server crashes (`kill -9`) and heartbeats stop.
2. After the TTL, the registry logs `EXPIRED` and appends a `BLOCK_HOST` event to `/tmp/udp_sdn_events.jsonl`.
3. Ryu polls that file every 0.5 s and installs `priority=100, ip, nw_dst=<host> -> drop` on the switch.
4. When the server restarts and re-registers, the registry appends `UNBLOCK_HOST` and Ryu deletes the rule.

**Flow table**

| Priority | Match | Action | Purpose |
|---|---|---|---|
| 100 | `ip, nw_dst=<dead host>` | drop | SDN block (dynamic) |
| 1 | `in_port, dl_src, dl_dst` | output | Learned MAC forwarding |
| 0 | any | send to controller | Table-miss |

> **Why a shared file between registry and Ryu?** Mininet hosts run in separate network namespaces and cannot reach the controller's `127.0.0.1`. Since both run on the same machine, a file is the simplest bridge for a lab. A real deployment should use an authenticated controller API.

## Repository layout

```text
.
├── protocol.py            # Shared UDP message encode/decode
├── registry.py            # UDP registry, TTL expiry, SDN event writer
├── service_server.py      # Example service: advertise, heartbeat, respond
├── client.py              # Discover (and optionally invoke) services
├── benchmark.py           # Discovery latency / success-rate benchmark
├── ryu_controller.py      # Ryu learning switch + block/unblock from events
├── topology.py            # Mininet topology (remote controller 127.0.0.1:6653)
├── requirements.txt
├── scripts/
│   └── reset_sdn_events.sh
├── tests/
│   └── test_protocol.py
└── docs/
    ├── DEMO_STEPS.md      # Full demo runbook
    ├── EXPLAIN_CHEATSHEET.md
    └── TOPIC_MAPPING.md   # Requirement-to-implementation mapping
```

## UDP protocol

Each message is one UDP datagram containing a JSON object with `"version": 1`.

| Message | Direction | Purpose |
|---|---|---|
| `REGISTER` | server → registry | Advertise name, IP, port, TTL, optional metadata |
| `REGISTERED` | registry → server | Acknowledge registration or refresh |
| `DISCOVER` | client → registry | Look up one service, or all if `service` is empty |
| `DISCOVERY_RESULT` | registry → client | Active endpoints and `ttl_remaining` |
| `REQUEST` / `RESPONSE` | client ↔ server | Example service call |
| `UNREGISTER` | server → registry | Graceful removal |
| `PING` / `PONG` | any | Connectivity check |
| `ERROR` | receiver → sender | Malformed or unsupported message |

## Requirements

- Linux (Ubuntu 20.04/22.04 VM recommended) with sudo access
- Mininet, Open vSwitch, Python 3
- Ryu (installed in a virtualenv is recommended on newer Python versions)

```bash
sudo apt update
sudo apt install -y mininet openvswitch-switch python3 python3-pip python3-venv git

python3 -m venv ~/ryuenv
~/ryuenv/bin/pip install -r requirements.txt
```

The registry, server, client and protocol modules use only the Python standard library.

## Quick start

Use three terminals. Replace `/path/to/udp-service-discovery` with your clone's absolute path.

> **Never run `sudo mn -c` after Ryu has started.** It kills Ryu. Clean up first, then start Ryu.

**Terminal 1: Ryu controller**

```bash
cd /path/to/udp-service-discovery
sudo mn -c
sudo ./scripts/reset_sdn_events.sh
sudo env SDN_EVENTS_FILE=/tmp/udp_sdn_events.jsonl ~/ryuenv/bin/ryu-manager ryu_controller.py
```

**Terminal 2: Mininet**

```bash
cd /path/to/udp-service-discovery
sudo python3 topology.py
```

At the `mininet>` prompt:

```text
pingall
registry python3 /path/to/udp-service-discovery/registry.py > /tmp/registry.log 2>&1 &
server1 python3 /path/to/udp-service-discovery/service_server.py --name weather --registry-ip 10.0.0.1 --port 10000 > /tmp/server1.log 2>&1 &
server2 python3 /path/to/udp-service-discovery/service_server.py --name maps --registry-ip 10.0.0.1 --port 10000 > /tmp/server2.log 2>&1 &
registry cat /tmp/registry.log
```

**Discover and invoke**

```text
client1 python3 /path/to/udp-service-discovery/client.py --registry-ip 10.0.0.1
client1 python3 /path/to/udp-service-discovery/client.py --registry-ip 10.0.0.1 --service weather --invoke --payload hello
client2 python3 /path/to/udp-service-discovery/client.py --registry-ip 10.0.0.1 --service maps --invoke
client1 python3 /path/to/udp-service-discovery/client.py --registry-ip 10.0.0.1 --service nonexistent
```

## Demo: crash, block and recovery

1. **Before:** in a plain terminal, confirm there are no drop rules.
   ```bash
   sudo ovs-ofctl -O OpenFlow13 dump-flows s1 | grep priority=100
   ```
2. **Crash only `weather`:**
   ```text
   mininet> server1 pkill -9 -f "name weather"
   ```
3. **Wait ~8 s.** Ryu prints `SDN BLOCK: installed drop rule for IPv4 destination 10.0.0.2`.
4. **During:** the rule appears.
   ```text
   priority=100,ip,nw_dst=10.0.0.2 actions=drop
   ```
   ```text
   mininet> client1 ping -c 2 10.0.0.2     # 100% loss
   mininet> client1 ping -c 2 10.0.0.3     # 0% loss (maps untouched)
   ```
5. **Recover:** restart the `weather` server. Ryu prints `SDN UNBLOCK`, the rule disappears, and pings to `10.0.0.2` work again.

> Do not use `pkill -9 -f service_server.py`; it kills both servers. Use `-f "name weather"` to target one.

The full event history is in `/tmp/udp_sdn_events.jsonl`. See [`docs/DEMO_STEPS.md`](docs/DEMO_STEPS.md) for the complete runbook, including the registry-down test and shutdown order.

## Tests and benchmark

```bash
python3 -m unittest discover -s tests -v
```

Inside Mininet, with the registry running:

```text
client1 python3 /path/to/udp-service-discovery/benchmark.py --registry-ip 10.0.0.1 --service weather --count 100
```


**Static vs dynamic:** with the plain `ryu.app.simple_switch_13` controller, no drop rule is installed after a server dies and the network keeps forwarding traffic toward the dead host. With this project's controller, the switch blocks it within seconds of failure detection. The benefit is policy following availability, not faster normal discovery.

## Configuration

| Item | Default | How to change |
|---|---|---|
| Registry port | 9999 | `registry.py --port` |
| Service port | 10000 | `service_server.py --port` |
| TTL | 6 s | `--ttl` on registry / server |
| Heartbeat | TTL / 3 (2 s) | derived from TTL |
| Event file | `/tmp/udp_sdn_events.jsonl` | `SDN_EVENTS_FILE` env var |
| Controller address | `127.0.0.1:6653` | `topology.py` |

## Troubleshooting

- **Ryu never shows "Switch connected":** Ryu was not running, or `mn -c` ran after it started. Type `exit` in Mininet, run `sudo mn -c`, and restart from the top.
- **`ryu-manager: No such file`:** use the full path, e.g. `~/ryuenv/bin/ryu-manager`.
- **Port 6653 in use:** `sudo pkill -f ryu-manager`.
- **No "registered" lines:** wait a few seconds, or check `server1 cat /tmp/server1.log`.
- **One `advertisement failed: timed out` after recovery:** expected. The registry replies before Ryu removes the rule (up to ~0.5 s), so that one reply is dropped.

## Limitations

- Registry is in-memory; a restart clears it (servers re-register on their next heartbeat).
- UDP is lossy; several consecutive lost heartbeats can cause a false expiry and a temporary block.
- The shared event file assumes registry and Ryu run on the same machine.
- The drop rule blocks all IPv4 traffic to the host, not just the service port. Matching on UDP port would be narrower.
- No authentication or encryption, single controller, no replicated registry.
- Clients take the first matching endpoint; there is no load balancing or health scoring.

## Possible improvements

- Match drop rules on IP + UDP port instead of the whole host
- Replace the event file with an authenticated REST/gRPC controller API
- Retry with backoff on advertisements
- Round-robin or health-scored endpoint selection
- Registry replication


