# InfiniBand Diagnostics — Reference Checklist

## Prerequisites

| Check | Command | Pass Condition |
|-------|---------|----------------|
| doca-ofed installed | `dpkg -l \| grep -E '(doca-ofed\|mlnx-ofed)'` | Package listed as `ii` |
| rdma-core installed | `dpkg -l \| grep rdma-core` | Package listed as `ii` |
| IB kernel interfaces visible | `ip link show type infiniband` | At least one `ibs*` or `ib*` interface |
| ibstat available | `which ibstat` | Non-empty path |
| ibnetdiscover available | `which ibnetdiscover` | Non-empty path |

If any prerequisite fails, report and skip IB testing for that host.

---

## ibstat Output Fields

```
CA 'mlx5_X'
    CA type: MT<model>        # MT41692=CX7, MT4129=BF3
    Firmware version: <ver>
    Port 1:
        State: <state>        # Down | Initializing | Active
        Physical state: <ps>  # Polling | LinkUp | ...
        Rate: <Gbps>          # 100=HDR100, 200=HDR, 400=NDR
        Base lid: <lid>       # 0/65535 = not assigned; >0 = SM assigned
        Link layer: <ll>      # InfiniBand | Ethernet
```

### State Decision Table

| State | Physical State | Meaning | Action |
|-------|---------------|---------|--------|
| Down | Polling | No cable or switch port down | Check cabling |
| Down | LinkUp | Cable OK, no SM running | Start opensm |
| Initializing | LinkUp | Cable OK, no SM running | Start opensm |
| Active | LinkUp | Fully operational | Proceed to ibping |
| Active | — | Link layer=Ethernet | This is an RoCE port, not IB |

---

## opensm

**Start (daemon mode, background, log to file):**
```bash
sudo nohup opensm -B > /tmp/opensm.log 2>&1 &
```

**Verify SM is managing the fabric:**
```bash
grep "SM state" /tmp/opensm.log | tail -3
ibstat | grep -E "(State:|SM lid)"
```

**Check SM is running:**
```bash
pgrep -a opensm
```

**Stop SM after audit:**
```bash
sudo kill <opensm_pid>
rm -f /tmp/opensm.log
```

Only one SM is needed per fabric. Run on any host with an Active IB port.

---

## Topology Parsing (ibnetdiscover -p)

Output format:
```
CA  <lid> <port> <GUID> <speed> - SW  <sw-lid> <sw-port> <sw-GUID> ( '<CA-name>' - '<switch-name>' )
SW  <sw-lid> <sw-port> <sw-GUID> <speed> - CA  <ca-lid> <ca-port> <ca-GUID> ( '<switch-name>' - '<CA-name>' )
```

**Building the switch port map:**
- Lines starting with `SW  5  <port>` → switch port N is connected to the CA on that line
- Lines starting with `CA  <lid>` → the CA is connected to switch port on that line
- `4x ???` means an empty switch port (no cable)
- `4x HDR` = 200 Gbps, `4x NDR` = 400 Gbps, `4x HDR100` = 100 Gbps

**Validating against a cabling spec:**
1. Count expected CAs in spec → verify same count in ibnetdiscover output
2. Check each expected CA name appears in output (hostnames embedded in GUID aliases)
3. Verify no unexpected CAs are present (security / wrong fabric)
4. Flag any `4x ???` port that should have a cable per the spec

---

## ibping Pitfalls

### Must specify `-C <mlx5_X>` on both sides

```bash
# WRONG — picks arbitrary CA, pings wrong LID
sudo ibping -S
sudo ibping -L 4

# CORRECT — explicit CA on server and client
sudo ibping -S -C mlx5_2
sudo ibping -C mlx5_2 -L 4
```

### LID source

- Get LIDs from `ibstat` → `Base lid` field (assigned by SM after opensm is running)
- LID 0 or 65535 means SM has not assigned a LID yet — wait or restart opensm
- ibnetdiscover shows LIDs but only after SM is active

### DPU (BlueField) IB ports

Host-side view vs DPU ARM view:
```
Host ibstat mlx5_0  → Shows PCIe-side CA GUID, may have LID assigned
DPU ibstat mlx5_0   → Shows fabric-facing CA GUID, different LID

For ibping target:  use DPU ARM LID (from ssh <host>-dpu ibstat)
For ibping server:  run on DPU ARM (ssh <host>-dpu sudo ibping -S -C mlx5_0)
```

### Process cleanup

Always capture PIDs when starting background servers:
```bash
ssh <host> "sudo ibping -S -C mlx5_2 > /tmp/ibping_s.log 2>&1 & echo \$!"
# store returned PID, kill it after the test
```

---

## ib_send_bw Reference

**Default test parameters (sufficient for qualification):**
- Message size: 65536 bytes (default)
- Iterations: 1000 (default)
- Transport: RC (reliable connected)

**Expected line-rate benchmarks:**

| Link Speed | Theoretical max (MiB/s) | Expected ~91% (MiB/s) | Flag below |
|------------|------------------------|-----------------------|------------|
| HDR (200G) | 25,600 | ~23,300 | 20,480 (80%) |
| HDR100 (100G) | 12,800 | ~11,600 | 10,240 (80%) |
| NDR (400G) | 51,200 | ~46,600 | 40,960 (80%) |

**Connection model:**
- Server listens on TCP port 18515 (default) on the management interface
- Client connects via `<mgmt-ip>`, negotiates QP parameters, then sends IB traffic
- Ensure management IP is reachable before running `ib_send_bw`

**Common failure: "Couldn't connect to <ip>:18515"**
- Server process did not start or crashed
- Wrong IP (use management IP, not IB IP)
- Firewall blocking port 18515
- Used `--one-sided` flag on server but not client (must match)

---

## Severity Classification

| Finding | Severity |
|---------|---------|
| IB port Down + Physical=Polling (no cable) | 🔴 Critical |
| SM not running → all ports Initializing | 🟠 High |
| ibping loss > 0% on any port pair | 🟠 High |
| ib_send_bw below 80% line-rate | 🟡 Medium |
| ib_send_bw 80-90% line-rate | 🔵 Low |
| SM not configured as persistent service | 🔵 Low |
| ibnetdiscover/ibping require sudo (no NOPASSWD) | 🔵 Low |
