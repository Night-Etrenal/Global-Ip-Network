# Secure Network Roadmap

## Objective

Build a secure, maintainable network environment that separates:

1. personal interactive access;
2. cluster management traffic;
3. overseas business egress;
4. commercial VPN disaster recovery.

The design must avoid making any single airport/VPN provider, protocol, cloud vendor, or server a universal dependency.

## Target topology

```text
Android / Windows / WSL
        |
        +-- WireGuard --> edge-us-1
        +-- TCP 443 fallback --> edge-us-1
        +-- backup commercial VPN
        +-- iKuuu as last-resort backup

S0-S7 domestic core
        |
        +-- Tailscale management network
                |
                +-- edge-us-1
                +-- residential-us-1 (optional)

S7 Alertmanager
        |
        +-- internal webhook --> S0
        +-- Tailscale --> edge-us-1 Telegram gateway --> Telegram API
```

## Node roles

### edge-us-1

Required first-stage node.

Responsibilities:

- overseas Telegram gateway;
- application-level overseas API gateway;
- restricted HTTP CONNECT service for S7 during transition;
- personal WireGuard egress;
- personal backup egress after a residential node is introduced;
- overseas route and service health checks.

Must not host:

- core databases;
- S0 control-plane services;
- S7 monitoring databases;
- public unauthenticated proxies;
- production credentials unrelated to its gateway role.

### residential-us-1

Optional second-stage node.

Responsibilities:

- personal ChatGPT/Codex/browser egress;
- long-lived personal ISP/residential exit identity;
- backup personal access path independent from commercial VPN providers.

Must not host:

- S7 Telegram traffic;
- business bots;
- high-frequency RPC jobs;
- crawlers;
- CI/CD workers;
- production databases.

## Protocol responsibilities

| Technology | Purpose |
|---|---|
| Tailscale | management plane and server-to-server access |
| WireGuard | primary personal encrypted egress |
| TCP 443 fallback | compatibility path when UDP/WireGuard is unavailable |
| HTTPS gateway | application-level egress for S6/S7 |
| Commercial VPN | disaster recovery only |

Do not deploy every protocol at once. The initial production set is Tailscale, WireGuard, nftables, and an HTTPS/Telegram gateway. Add a TCP 443 fallback only after the primary path is accepted.

## Security boundaries

- Debian amd64 only.
- SSH keys only.
- Disable root password login and password authentication after key access is verified.
- nftables default-deny inbound policy.
- Separate WireGuard keys per device.
- No public SOCKS5 or HTTP proxy ports.
- SSH, gateway administration, metrics, and proxy services listen on Tailscale addresses only where possible.
- No S0-S7 global proxy settings.
- No Docker daemon global proxy on S6/S7.
- No forced Tailscale exit-node routing on production nodes.
- No secrets, real server IPs, domains, SNI values, subscription links, or tokens in this public repository.

## Phase 1: edge-us-1 acceptance

1. Purchase one month only.
2. Select Debian 12 or Debian 13 amd64.
3. Confirm key-based SSH access before disabling password login.
4. Verify:
   - fixed public IPv4;
   - GitHub connectivity;
   - Telegram API connectivity;
   - general HTTPS connectivity;
   - DNS behavior;
   - packet loss and jitter during evening peak hours;
   - service recovery after reboot.
5. Install Tailscale and assign a dedicated edge tag.
6. Install WireGuard for one test device only.
7. Test Android lock/unlock, Wi-Fi/5G switching, long sessions, uploads, and GitHub access for 7-14 days.

Acceptance targets:

- sustained packet loss <= 1%;
- stable reconnect after network switching;
- no public access to management/proxy ports;
- services restart automatically after reboot;
- exit IP remains correct and stable.

## Phase 2: S7 Telegram integration

1. Deploy a Telegram gateway or restricted HTTP CONNECT service on `edge-us-1`.
2. Bind the service to the Tailscale address only.
3. Permit only the S7 Tailscale identity/address.
4. Restrict the target to Telegram HTTPS where the proxy model is used.
5. Preserve the S7 -> S0 internal webhook path.
6. Add receiver-level proxy/gateway configuration through a reviewed feature branch and Draft PR.
7. After explicit production authorization:
   - back up the current Alertmanager configuration;
   - validate the rendered configuration;
   - recreate only the Alertmanager container;
   - verify the other S7 containers are unchanged;
   - send one test alert;
   - roll back on failure.

## Phase 3: residential node decision

Purchase `residential-us-1` only when at least one of these conditions remains after edge testing:

- a stable fixed residential/ISP exit is materially valuable for daily interactive use;
- commercial VPN failures remain frequent;
- the datacenter egress is technically stable but unsuitable for the desired personal environment;
- separating interactive and machine traffic provides enough operational value to justify the cost.

When introduced, use it only for personal interactive traffic. Production services continue to use `edge-us-1`.

## Failover order

### Personal access

```text
Primary: edge-us-1 initially, or residential-us-1 after purchase
Secondary: edge-us-1
Disaster recovery: backup commercial VPN
Last resort: iKuuu
```

Use manual switching during the initial phase. Avoid rapid automatic IP changes during active sessions.

### Production Telegram

```text
Primary: edge-us-1 Telegram gateway
Internal fallback: S7 -> S0 webhook remains active
Future high availability: add edge-us-2 from a different provider/ASN
```

Do not automatically fail production Telegram traffic over to the residential node.

## Repository separation

### Public repository

`Global-Ip-Network` stores:

- public source metadata;
- protocol and provider research;
- sanitized test methodology;
- anonymized measurements;
- architecture and incident lessons.

### Private infrastructure repository

Create a private repository for:

- Ansible inventory;
- hardening roles;
- nftables templates;
- Tailscale policy;
- WireGuard templates;
- gateway deployment;
- monitoring;
- encrypted variables.

No production deployment should run automatically from the public repository.

## Immediate next action

Provision and accept `edge-us-1` before purchasing a residential node or changing S7. The first deployment task is host acceptance and hardening, not proxy installation.
