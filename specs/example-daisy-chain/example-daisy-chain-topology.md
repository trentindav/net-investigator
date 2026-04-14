# Topology to test 6Wind use case

## Operational Notes for Audits

This lab is regularly re-deployed from scratch with no persistent network configuration.
Agents running connectivity audits should treat the following as **expected, not findings**:

- **Ethernet data interfaces admin-DOWN**: In a fresh deployment, all data interfaces
  (`cx7-eth-p0/p1`, `bf3-p0/p1` on all hosts) will typically be administratively DOWN
  (`qdisc noop`, no `UP` flag). This is normal. Always bring them up per the spec and
  assign the static IPs from the direct-connection table before testing. Do not flag this.
- **No static IPs pre-configured**: Test addresses (10.10–10.60.x.x/24) must be assigned
  fresh at every audit session. They are ephemeral by design.
- **No IB Subnet Manager running**: `opensm` will not be running at audit start. Start it on a
  designated host for the duration of testing, then stop it at cleanup. Do not flag this.
- **doca-ofed / MLNX OFED must be pre-installed by the user**: The audit agent must not install
  OFED. If IB tools are missing on a host, skip that host's IB tests and note the gap as a
  prerequisite note in the report, not a blocking finding. The user is responsible for choosing
  and installing the correct OFED version before requesting an audit.

## Machine inventory

The lab is composed of 2 servers with one management interface (other 1G not connected),
2 ConnectX-7 (CX7) NICs and one BlueField-3 (BF3) DPU each.
Hostnames you can use to `ssh` are: `h1`, `h2`

Every server has one CX7 in Ethernet mode, the other in Infiniband mode, 
and the BlueField-3 in Infiniband mode.

| Host     | CX7 #1 | CX7 #2 | BF3 |
|----------|--------|--------|-----|
| h1       | Eth    | IB     | IB  |
| h2       | Eth    | IB     | IB  |

The interface names are the following, but for this testing use the aliases marked below for clarity.
In the report you can prepend the interface with the hostname, e.g. `h1-cx7-1-1`.

| Interface Name | Alias      |
|----------------|------------|
| ens27f3        | mgt        |
| ens16f0np0     | cx7-eth-p0 |
| ens16f1np1     | cx7-eth-p1 |
| ens2f0np0      | bf3-p0     |
| ens2f1np1      | bf3-p1     |

When BF3 is in Infiniband mode, the corresponding interfaces don't show up like this.
All Infiniband interfaces show up because of doca-ofed as `ibsXfY`.

## Infiniband Network

All IB interfaces are connected to the same switch. Starting `opensm` on a single machine
makes the entire network usable. Every interface from every host can reach every other
interface (full mesh).

## Direct host-to-host connections

All Ethernet links are direct connections between pairs of hosts, to create a daisy chain.
You can set a static IP on all of them (except `mgt`) to test each connection.

| Interface A         | IP A         | Interface B         | IP B         | 
|---------------------|--------------|---------------------|--------------|
| h1-cx7-eth-p0       | 10.10.0.1/24 | h2-cx7-eth-p0       | 10.10.0.2/24 | 
| h1-cx7-eth-p1       | 10.20.0.1/24 | h2-cx7-eth-p1       | 10.20.0.2/24 |