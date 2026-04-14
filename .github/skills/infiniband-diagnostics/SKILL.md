---
name: infiniband-diagnostics
description: Diagnoses InfiniBand fabric health on Linux lab machines equipped with Mellanox/NVIDIA ConnectX or BlueField (DPU) adapters. Covers doca-ofed/OFED prerequisite checks, Subnet Manager bring-up (opensm), topology discovery (ibnetdiscover), full-mesh reachability testing (ibping per-port per-LID), and bandwidth measurement (ib_send_bw). Use when auditing IB network state, troubleshooting IB connectivity, validating new IB fabric cabling, or measuring IB throughput as part of a lab readiness check.
compatibility: bash
allowed-tools: bash ssh
---

# InfiniBand Diagnostics

## Overview

Provides a complete, repeatable workflow for diagnosing InfiniBand fabrics on Linux lab nodes.
Works with any RDMA-capable adapters (ConnectX-7, BlueField-3 DPU IB mode) and a single
IB switch. Requires `doca-ofed` or `mlnx-ofed` installed on the target hosts.

For detailed command flags, state transition tables, and troubleshooting patterns, read
`references/ib-checklist.md`.

## Workflow

Follow these steps in order.

### Step 1 — Prerequisite Check

Verify OFED is installed on every host in scope before any IB command:

```bash
ssh <host> "dpkg -l | grep -E '(doca-ofed|mlnx-ofed|rdma-core)'"
```

- If **not installed**: note the missing package and skip all IB steps for that host.
  Do **not** install OFED — this is the user's responsibility to pre-install the version of
  their choice before running this skill. Report the gap as a prerequisite note, not a finding.
- If installed: note the version and continue.

### Step 2 — ib_umad Module Loaded

Verify the `ib_umad` kernel module is loaded on every host, and load it if missing:

```bash
ssh <host> "lsmod | grep ib_umad || sudo modprobe ib_umad"
```

### Step 3 — Enumerate IB Interfaces

On each host (and DPU if applicable — `ssh <hostname>-dpu`):

```bash
ssh <host> "ibstat 2>&1"
```

Collect for every CA (`mlx5_X`): State, Physical state, Rate (speed), Base LID, Port GUID,
Link layer. Record in the IB inventory table (see Output Format). For wach host with a
DPU, `ssh` into the DPU to run `ibstat` there as well. DPU IP/hostname must be known in advance.

Key states:
- `State: Down` + `Physical state: Polling` → no cable or switch port down
- `State: Down/Initializing` + `Physical state: LinkUp` → cable OK, no SM running → Step 4
- `State: Active` → healthy, proceed to Step 5

### Step 4 — Subnet Manager Bring-Up (if needed)

If all IB ports show `Down`/`Initializing` with `Physical state: LinkUp`, start opensm on
**ONE** designated host:

```bash
ssh <host> "sudo nohup opensm -B > /tmp/opensm.log 2>&1 &"
sleep 8
ssh <host> "ibstat 2>&1 | grep -E '(State:|Rate:)'"
```

Verify all ports transition to `State: Active`. Repeat the validation for all hosts.
If any  ports remain `Down`, report a potential cabling issue for this interface and
do not perform reachaility or bandwidth tests on it.

### Step 5 — Topology Discovery

Run `ibnetdiscover` (requires sudo) from one host with an Active port:

```bash
ssh <host> "sudo ibnetdiscover -p 2>&1"
```

Parse output to build the switch port→CA mapping. Cross-reference against the expected
cabling spec. See `references/ib-checklist.md §Topology Parsing` for output interpretation.
Note that corresponds to a DPU port shows up in the `ibnetdiscover` output with their
DPU side LID and GUID, and the host side remains hidden but it can be used for ping and
bandswidth tests. Add these hidden interfaces to the inventory table with a note to
indicate which are on DPU and which are the corresponding ones on the host.
Cross-reference the ibnetdiscover output with the ibstat output to confirm which
CA corresponds to which switch port and LID.

### Step 6 — Full-Mesh Reachability (ibping)

Test every CA→CA pair. For each target CA:

1. **Start server** on the target for the specific CA:
   ```bash
   ssh <target-host> "sudo ibping -S -C <target-mlx5> > /tmp/ibping_s.log 2>&1 &"
   # capture PID for cleanup: ssh <target-host> "pgrep -n ibping"
   ```

2. **Run client** from source, specifying source CA and target LID:
   ```bash
   ssh <source-host> "sudo ibping -C <source-mlx5> -c 5 -L <target-LID> 2>&1"
   ```

3. **Stop server** by PID after the test.

4. **Loop through all pairs** of CAs across all hosts (including DPU interfaces) and record loss and RTT in the connectivity matrix.
For each host, and each DPU, start a server on one CA at a time, and run clients from all other CAs to that server CA,
then stop the server before moving to the next CA.

**Critical rules** (from live lab experience — see `references/ib-checklist.md §ibping Pitfalls`):
- Always specify `-C mlx5_X` on both server and client — omitting it picks the wrong CA

### Step 7 — Bandwidth Measurement (ib_send_bw)

1. **Start server** on the target for the specific CA:
  ```bash
  ssh <target-host> "ib_send_bw -d <target-mlx5> --report_gbits > /tmp/ib_bw.log 2>&1 &"
  # capture PID for cleanup: ssh <target-host> "pgrep -n ib_send_bw"
  ```

2. **Run client** from source, specifying source CA and target management IP:
  ```bash
  ssh <source-host> "ib_send_bw -d <source-mlx5> <target-mgmt-ip> --report_gbits 2>&1"
  ```

3. **Stop server** by PID after the test.

4. **Loop through all pairs** of CAs across all hosts (including DPU interfaces) and record bandwidth results in the connectivity matrix.

For HDR (200G), near-line-rate is ~23,400 MiB/s (~91% of 25,600 MiB/s theoretical max).
Flag results below 80% of line-rate as Medium severity.

### Step 8 — Latency Measurement (ib_send_lat)

1. **Start server** on the target for the specific CA:
  ```bash
  ssh <target-host> "ib_send_lat -d <target-mlx5> --report_gbits > /tmp/ib_lat.log 2>&1 &"
  # capture PID for cleanup: ssh <target-host> "pgrep -n ib_send_lat"
  ```

2. **Run client** from source, specifying source CA and target management IP:
  ```bash
  ssh <source-host> "ib_send_lat -d <source-mlx5> <target-mgmt-ip> --report_gbits 2>&1"
  ```

3. **Stop server** by PID after the test.

4. **Loop through all pairs** of CAs across all hosts (including DPU interfaces) and record latency results in the connectivity matrix.

### Step 9 — Cleanup

After all IB tests, verify no lingering processes:

```bash
for host in <all-hosts-and-dpus>; do
  ssh $host "pgrep -a \"ibping|ib_send_bw|opensm\" 2>/dev/null || echo clean"
done
```

Stop survivors by PID, then remove temp files:
```bash
ssh <host> "sudo kill <pid>"
ssh <host> "rm -f /tmp/ibping_s.log /tmp/ib_bw.log /tmp/opensm.log"
```

## Output Format

Produce these sections in the connectivity report:

```markdown
## InfiniBand Inventory

| Host | CA (mlx5) | LID | Rate | State | Notes |
|------|-----------|-----|------|-------|-------|
| node-01 | mlx5_2 | 1 | 200G | Active | CX7 IB |
| node-01-dpu | mlx5_0 | 9 | 200G | Active | BF3 DPU ARM |

## InfiniBand Topology (ibnetdiscover)

Switch: <model> GUID <guid>
| Switch Port | Connected CA | CA GUID | Speed |
|-------------|-------------|---------|-------|
| 1 | node-01 mlx5_2 | 0x... | 4x HDR |

## InfiniBand Connectivity Matrix

| Source CA | Destination CA | Loss | RTT avg | Notes |
|-----------|---------------|------|---------|-------|
| luvbi mlx5_2 | acree mlx5_2 (LID 4) | 0% | 0.032 ms | |

## InfiniBand Bandwidth (ib_send_bw)

| Source CA | Destination CA | BW peak (MiB/s) | BW avg (MiB/s) | Line-rate % |
|-----------|---------------|-----------------|----------------|-------------|
| luvbi mlx5_2 | acree mlx5_2 | 23,432 | 23,431 | 91.5% |
```

## Constraints

- Never leave `opensm`, `ibping -S`, or `ib_send_bw` server processes running after the session.
- `ibnetdiscover` and `ibping` require `sudo` (UMAD port access).
- `ib_send_bw` client connects to the **management IP** of the target (TCP handshake first).
- Do not install packages on remote machines.
