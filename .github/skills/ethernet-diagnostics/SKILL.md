---
name: ethernet-diagnostics
description: Validates and troubleshoots Ethernet connectivity on Linux lab machines. Covers per-machine interface enumeration (L1 link state, L2 VLAN/bond/bridge, L3 IP/routing), ARP and ICMP reachability testing, iperf3 bandwidth measurement, and structured report generation. Use when auditing Ethernet interfaces, diagnosing host-to-host connectivity failures, validating cabling against a topology spec, or measuring Ethernet throughput in a lab environment.
compatibility: bash
allowed-tools: bash ssh
---

# Ethernet Diagnostics

## Overview

Provides a complete, repeatable workflow for diagnosing Ethernet connectivity on Linux lab nodes.
Covers the full OSI stack from L1 (link state, speed) through L3 (IP, routing, reachability) and
L4 performance (iperf3). For detailed command flags and troubleshooting patterns, see
`references/ethernet-checklist.md`.

## Workflow

Follow these steps in order.

### Step 1 — Scope & Inventory

Identify all machines in scope (from user input, cabling docs, or host files in the repo).

```bash
# Confirm SSH access to each machine; note any unreachable hosts immediately
ssh <host> "hostname && uptime"
```

- Read any available topology documents (cabling plans, subnet tables) from the repo before
  starting — cross-reference every finding against the expected design.
- If a machine is unreachable, report it immediately; do not skip silently.

### Step 2 — Per-Machine Interface Audit

For each machine, enumerate all interfaces and check L1 through L2 state:

```bash
ssh <host> "ip link show"
ssh <host> "ip -s link show"
ssh <host> "ip addr show"
ssh <host> "ethtool <iface>"          # speed, duplex, link detected
ssh <host> "bridge link show 2>/dev/null"
ssh <host> "ip link show type vlan"
ssh <host> "ip link show type bond"
ssh <host> "ip netns list"            # check for non-default namespaces
```

Only if the spec defines specific interfaces to be present and used for testing,
if they are found in `Down` state, bring them up.
```bash
ssh <host> "sudo ip link set <iface> up"
```

Flag any other interface that is DOWN, has no IP, has RX/TX errors, or deviates from the
expected plan. See `references/ethernet-checklist.md` for per-layer checks and severity
criteria.

### Step 3 — IP Configuration Validation

Verify addressing and routing match the topology spec:

```bash
ssh <host> "ip addr show"
ssh <host> "ip route show"
ssh <host> "ip -6 route show"
ssh <host> "ip rule show"
ssh <host> "ip neigh show"
```

- Confirm each interface has an IP in the expected subnet.
- Verify the default gateway exists and is reachable.
- For directly connected host pairs, confirm they are on the same subnet.
- If a direct host-to-host link is expected but no IPs are configured, request explicit
  user permission before assigning temporary static IPs for testing:
  ```bash
  ssh <host> "sudo ip addr add <ip>/<prefix> dev <iface>"
  ```
  Document any temporary address assignments in the report; clean them up after testing.

### Step 4 — Inter-Host Connectivity Testing

Build a full connectivity matrix for every expected pair:

**L2 check (ARP):**
```bash
ssh <source> "arping -c 3 -I <iface> <target-ip>"
```

**L3 check (ICMP):**
```bash
ssh <source> "ping -c 5 <target-ip>"
```

**Path tracing (on failure):**
```bash
ssh <source> "traceroute <target-ip>"
ssh <source> "tracepath <target-ip>"
```

Use `traceroute`/`tracepath` for any pair that is unreachable or takes unexpected hops.
Record L2 result, L3 result, and RTT in the Ethernet Connectivity Matrix.

### Step 5 — Performance Testing (iperf3)

For each expected high-throughput pair, measure bandwidth:

1. **Start server** on the target:
   ```bash
   ssh <target> "iperf3 -s -D --logfile /tmp/iperf3.log"
   # Capture PID for cleanup: ssh <target> "pgrep -n iperf3"
   ```

2. **Run client** from source:
   ```bash
   ssh <source> "iperf3 -c <target-ip> -t 10 --bidir"
   ```

3. **Stop server** by PID after the test:
   ```bash
   ssh <target> "kill <pid> && rm -f /tmp/iperf3.log"
   ```

4. Record peak and average bandwidth in the report. Flag results below 80% of the
   expected line rate as Medium severity.

### Step 6 — Cleanup

After all tests, confirm no lingering processes or temp files remain on any host:

```bash
for host in <all-hosts>; do
  ssh $host "pgrep -a 'iperf3' 2>/dev/null || echo clean"
done
```

Remove any temporary static IP assignments added in Step 3.

## Output Format

Produce these sections in the connectivity report:

```markdown
## Interface Inventory

| Host | Interface | State | IP Address | MTU | Speed | Notes |
|------|-----------|-------|------------|-----|-------|-------|
| node-01 | eth0 | UP | 192.168.1.10/24 | 1500 | 10Gb/s | Management |
| node-01 | eth1 | DOWN | — | 1500 | Unknown | ⚠️ No carrier |

## Ethernet Connectivity Matrix

| Source | Destination | L2 (arping) | L3 (ping) | RTT | Notes |
|--------|-------------|-------------|-----------|-----|-------|
| node-01 | node-02 | ✅ | ✅ | 0.4 ms | |
| node-01 | node-03 | ✅ | ❌ | — | No route to host |

## Ethernet Performance (iperf3)

| Source | Destination | TX (Gb/s) | RX (Gb/s) | Line-rate % | Notes |
|--------|-------------|-----------|-----------|-------------|-------|
| node-01 | node-02 | 9.4 | 9.3 | 93% | |
```

## Examples

### Example 1: Enumerate interfaces on a single machine

**Input:**
```
Audit the network interfaces on node-01 (192.168.1.10)
```

**Approach:**
```
Scope: single machine, node-01 at 192.168.1.10.
Start at L1: enumerate all interfaces and their link state.
Then L2: check for VLANs, bridges, bonds.
Then L3: IPs, routes.
Commands to run (read-only via SSH):
  ip link show
  ip addr show
  ip -s link show
  ip route show
  ethtool <each_iface>
  bridge link show 2>/dev/null
  ip neigh show
No changes. Output an interface inventory table.
```

**Output:**
```markdown
## Interface Inventory — node-01 (192.168.1.10)

| Interface | State | IP Address       | MTU  | Speed     | Notes                        |
|-----------|-------|------------------|------|-----------|------------------------------|
| lo        | UP    | 127.0.0.1/8      | 65536| —         | Loopback, normal             |
| eth0      | UP    | 192.168.1.10/24  | 1500 | 1000Mb/s  | Primary management interface |
| eth1      | DOWN  | —                | 1500 | Unknown   | ⚠️ No carrier detected       |
| eth2      | UP    | 10.0.0.1/30      | 9000 | 10000Mb/s | Jumbo frames enabled         |

### 🟠 High
- **node-01 eth1**: Link is DOWN, no carrier. `ip link show eth1` → `state DOWN` /
  `ethtool eth1` → `Link detected: no`

### Recommendations
1. **node-01 eth1**: Verify physical cable connection and switch port status.
```

---

### Example 2: Full inter-host connectivity matrix

**Input:**
```
Verify connectivity between node-01 (10.0.0.1), node-02 (10.0.0.2), and node-03 (10.0.0.3)
```

**Approach:**
```
Three machines. Build a full N×N connectivity matrix (excluding self-loops).
For each pair (A→B):
  1. arping from A to B's MAC (L2 check)
  2. ping -c 5 from A to B (L3 ICMP check)
  3. If unreachable: traceroute to find where it breaks
  4. Check ip route on A to confirm a route to B's subnet exists
Run in structured order: node-01→node-02, node-01→node-03, node-02→node-03.
No modifications. Document all results.
```

**Output:**
```markdown
## Connectivity Matrix

| Source  | Destination | L2 (arping) | L3 (ping) | RTT     | Notes                    |
|---------|-------------|-------------|-----------|---------|--------------------------|
| node-01 | node-02     | ✅           | ✅         | 0.4 ms  |                          |
| node-01 | node-03     | ✅           | ❌         | —       | ICMP unreachable         |
| node-02 | node-03     | ✅           | ✅         | 0.3 ms  |                          |

### 🔴 Critical
- **node-01 → node-03**: L2 reachable (arping OK) but L3 ping fails.
  `ip route show` on node-01 shows no route to 10.0.0.3; default gateway not set.

### Recommendations
1. **node-01**: Add route to 10.0.0.0/24 via 10.0.0.2, or configure a default gateway.
   Example: `ip route add 10.0.0.0/24 via 10.0.0.2 dev eth0`
```

3. Customize or delete example files in scripts/, references/, and assets/
4. Run validation: `python3 scripts/validate_skill.py --path .github/skills/ethernet-diagnostics`
5. Test the skill with real use cases
6. Iterate based on feedback
