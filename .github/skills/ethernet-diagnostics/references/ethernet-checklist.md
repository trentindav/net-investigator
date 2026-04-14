# Ethernet Diagnostics Checklist

Detailed command reference and troubleshooting patterns for the `ethernet-diagnostics` skill.

---

## Table of Contents

1. [L1 — Physical / Link State](#l1--physical--link-state)
2. [L2 — Data Link / Ethernet](#l2--data-link--ethernet)
3. [L3 — Network / IP](#l3--network--ip)
4. [Reachability Testing](#reachability-testing)
5. [Performance Testing](#performance-testing)
6. [Severity Criteria](#severity-criteria)
7. [Common Failure Patterns](#common-failure-patterns)

---

## L1 — Physical / Link State

```bash
# Enumerate all interfaces and link state
ip link show

# Per-interface statistics (RX/TX bytes, errors, drops, carrier losses)
ip -s link show <iface>

# Speed, duplex, auto-negotiation, and physical link detection
ethtool <iface>
```

**Key fields from `ethtool`:**
- `Speed:` — negotiated speed (e.g. `10000Mb/s`); `Unknown!` means no link
- `Duplex:` — should be `Full`; `Half` indicates a mismatch
- `Link detected: yes/no` — definitive L1 state
- `Auto-negotiation: on/off` — mismatch between peers causes degraded speeds

**Flag conditions:**
- `state DOWN` in `ip link show` → no carrier or interface not brought up
- `Link detected: no` in `ethtool` → no physical connection
- Non-zero `errors` or `dropped` in `ip -s link show` → hardware or driver issue

---

## L2 — Data Link / Ethernet

```bash
# VLAN sub-interfaces
ip link show type vlan

# Bond interfaces and member state
ip link show type bond
cat /proc/net/bonding/<bond_iface>   # detailed bond state, active slave

# Bridge and ports
bridge link show
bridge vlan show

# ARP / neighbor table
ip neigh show

# Layer-2 reachability to a peer
arping -c 3 -I <iface> <target-ip>
```

**Flag conditions:**
- VLAN ID mismatch between hosts → frames silently dropped at switch
- Bond slave in `backup` state only → failover happened, investigate primary
- ARP entry in `FAILED` state → L2 unreachable despite link up
- `arping` 100% loss with `ip link` UP → L2 issue (wrong VLAN, wrong switch port, MAC filter)

---

## L3 — Network / IP

```bash
# All assigned addresses
ip addr show

# IPv4 routing table
ip route show

# IPv6 routing table
ip -6 route show

# Policy routing rules
ip rule show

# Neighbor / ARP cache
ip neigh show

# Non-default network namespaces
ip netns list
ip netns exec <ns> ip addr show
ip netns exec <ns> ip route show
```

**Flag conditions:**
- Interface UP but no `inet` address → not configured or DHCP failed
- Address in wrong subnet per topology spec → misconfiguration
- No default route → off-subnet destinations will be unreachable
- Routing table entry missing for an expected peer subnet → add route or investigate
- IP address collision (duplicate) → `ip neigh show` will show `FAILED` for the conflicting address

---

## Reachability Testing

### ICMP (L3)

```bash
# Basic reachability (5 packets)
ping -c 5 <target-ip>

# Specify source interface
ping -c 5 -I <iface> <target-ip>

# Large packet / MTU path check (IP MTU = 9000 → payload 8972)
ping -c 3 -s 8972 -M do <target-ip>
```

### Path Tracing (on failure)

```bash
traceroute <target-ip>
tracepath <target-ip>        # no root required, also reports MTU
```

**Interpreting results:**
- `ping` fails but `arping` succeeds → L3 issue (routing, firewalling)
- `ping` fails and `arping` fails → L2 or L1 issue (wrong subnet, no cable, wrong VLAN)
- `traceroute` shows unexpected hop → traffic routed via wrong path; check routing tables
- `traceroute` times out at first hop → default gateway unreachable

---

## Performance Testing

### iperf3

```bash
# Server (on target machine, daemon mode)
iperf3 -s -D --logfile /tmp/iperf3.log

# Client (bidirectional, 10 second test)
iperf3 -c <target-ip> -t 10 --bidir

# Client — specific source interface / bind address
iperf3 -c <target-ip> -t 10 -B <source-ip> --bidir

# Cleanup server
kill $(pgrep -n iperf3) && rm -f /tmp/iperf3.log
```

**Expected throughput by interface speed:**

| Link speed | Expected min (80% line rate) | Near-line-rate |
|------------|------------------------------|----------------|
| 1 GbE      | 800 Mb/s                     | ~940 Mb/s      |
| 10 GbE     | 8 Gb/s                       | ~9.4 Gb/s      |
| 25 GbE     | 20 Gb/s                      | ~23.5 Gb/s     |
| 100 GbE    | 80 Gb/s                      | ~94 Gb/s       |

Flag results below 80% of line rate as **Medium** severity.

---

## Severity Criteria

| Severity | Condition examples |
|----------|--------------------|
| 🔴 Critical | Interface DOWN, no IP address assigned, complete L3 unreachability between required peers |
| 🟠 High | Asymmetric routing, ARP FAILED for required peer, wrong subnet, missing default gateway |
| 🟡 Medium | MTU mismatch, suboptimal path (unexpected hops), throughput < 80% line rate, duplex mismatch |
| 🔵 Low / Info | Speed auto-neg negotiated below maximum, cosmetic naming deviation from spec |

---

## Common Failure Patterns

### "arping OK, ping fails"
- L2 path exists but L3 routing is broken
- Check `ip route show` on source for route to target subnet
- Check for firewall rules: `sudo iptables -L -n` (read-only)

### "ping OK, iperf3 fails to connect"
- iperf3 server not running or bound to wrong interface
- Check with: `ssh <target> "ss -tlnp | grep 5201"`

### "link UP, no IP, DHCP expected"
- DHCP client not running or lease expired
- Check: `ssh <host> "systemctl status systemd-networkd NetworkManager dhclient"`

### "jumbo frames not working (MTU 9000)"
- MTU mismatch anywhere along the path will silently fragment or drop
- Verify every hop: `ip link show <iface> | grep mtu`
- Test with: `ping -c 3 -s 8972 -M do <target-ip>` (expect success end-to-end)

### "bond stuck in backup-only mode"
- Primary interface link down or LACP negotiation failed
- Check: `cat /proc/net/bonding/<bond>` → look for `Active Slave` and `MII Status`

## When Reference Docs Are Useful

Reference docs are ideal for:
- **Comprehensive guides** - Multi-page documentation that would bloat SKILL.md
- **API documentation** - Endpoint specifications, parameters, examples
- **Schemas** - Database schemas, data models, type definitions
- **Domain knowledge** - Company policies, industry standards, legal templates
- **Detailed workflows** - Step-by-step procedures with screenshots/diagrams

## Structure Suggestions

Choose a structure that fits your content:

### API Reference Example
```markdown
## Authentication
- Method: Bearer token
- Header: `Authorization: Bearer <token>`

## Endpoints

### GET /api/users
**Description:** Retrieve user list
**Parameters:**
- `page` (int, optional): Page number (default: 1)
- `limit` (int, optional): Results per page (default: 20)

**Response:**
\`\`\`json
{
  "users": [...],
  "total": 150,
  "page": 1
}
\`\`\`

**Error codes:**
- 401: Unauthorized
- 429: Rate limit exceeded
```

### Workflow Guide Example
```markdown
## Prerequisites
- Python 3.8+
- API credentials configured
- Required packages: `pip install -r requirements.txt`

## Step 1: Initialize Connection
1. Import the client: `from myapi import Client`
2. Create instance: `client = Client(api_key='...')`
3. Test connection: `client.ping()`

## Step 2: Fetch Data
[Detailed instructions...]

## Troubleshooting
**Problem:** Connection timeout
**Solution:** Check firewall settings...
```

### Schema Documentation Example
```markdown
## Database Schema

### Table: users
| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | INTEGER | PRIMARY KEY | Unique user ID |
| email | VARCHAR(255) | UNIQUE, NOT NULL | User email |
| created_at | TIMESTAMP | DEFAULT NOW() | Registration time |

### Relationships
- users.id → orders.user_id (one-to-many)
- users.id → profiles.user_id (one-to-one)
```

## Tips for Large Reference Files

If this file will be >100 lines:
- Include a **Table of Contents** at the top
- Use clear section headers (##, ###)
- Add grep hints in main SKILL.md for easier navigation

## Best Practices

✅ **DO:**
- Keep SKILL.md lean, move details here
- Use tables for structured data
- Include concrete examples
- Link back to SKILL.md where relevant

❌ **DON'T:**
- Duplicate content from SKILL.md
- Create reference docs for <50 lines of content (put in SKILL.md)
- Use vague descriptions (be specific)
- Forget to mention this file exists in SKILL.md
