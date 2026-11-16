# 02 — Architecture

How the lab is built: layers, data models, fault semantics, orchestration and lifecycle.

Decisions locked for v1:

| Decision | Choice |
|---|---|
| VMM | Firecracker |
| Guest OS | Alpine Linux |
| Guest kernel | Firecracker published CI `vmlinux` (ELF) |
| Privileges | orchestrator run interactively under `sudo` |
| Reset | Firecracker full snapshot + restore |
| Containers | none (no Docker, no containerlab, no Kata) |
| Scope of v1 | pure network: 3 routers + 3 hosts |

---

## 1. Overview

```
                      HOST (Linux, root)
+---------------------------------------------------------------+
|  orchestrator (Python)                                        |
|   topology -> plan -> ip/tc  |  firecracker API  |  vsock RPC |
|                                                               |
|   br-mgmt ---- tap-r1-0  tap-r2-0  tap-r3-0  tap-h1-0 ...     |
|   br-l1   ---- tap-h1-1  tap-r1-1                             |
|   br-l2   ---- tap-r1-2  tap-r2-1                             |
|   br-l3   ---- tap-r2-2  tap-r3-1                             |
|   br-l4   ---- tap-r3-2  tap-h3-1                             |
|   br-l5   ---- tap-r2-3  tap-h2-1                             |
|                                                               |
|   fc-r1  fc-r2  fc-r3  fc-h1  fc-h2  fc-h3   (VMM processes)  |
+---------------------------------------------------------------+
        |        |        |        |        |        |
   +----v--------v--------v--------v--------v--------v----+
   |  guest r1     r2       r3       h1       h2      h3  |
   |  Alpine+FRR  Alpine+FRR ...      Alpine hosts        |
   |  kernel       kernel     kernel   kernel   ...        |
   +-------------------------------------------------------+
```

Router topology (logical):

```
   h1 --- r1 --- r2 --- r3 --- h3
                  |
                  h2
```

Every edge is a bridge with two TAPs. Every node is one microVM. The orchestrator is the
only component that touches the host network, the Firecracker API, and the guests.

---

## 2. Design principles

1. **One source of truth.** `topology.yaml` generates the bridges, TAPs, MACs, addresses,
   FRR configs, the graph, and the ground-truth identifiers. Nothing is hand-edited twice.
2. **The link is the fault boundary.** If you cannot point at the exact host object a fault
   touches, the fault is not designed yet.
3. **Verify, do not assume.** Every fault has an `apply`, a `revert`, and a `verify`. An
   unverified fault is a failed experiment.
4. **Determinism through reset.** Snapshots are taken after convergence, never mid-run.
5. **Host-side truth.** Timestamps and measurement cadence come from the host, so guest
   clock drift cannot corrupt a timeline.
6. **Real software only.** Real kernel, real FRR, real `tc`. No mock control plane.
7. **Fail loudly.** Preflight checks abort the run rather than producing plausible garbage.

### Why not containerlab / Kata

Recorded here so the decision is not relitigated:

- containerlab starts nodes with `runtime: docker|podman` and wires links by moving veth
  ends into a node's network namespace.
- a Kata guest only sees NICs that Kata creates as virtio devices (from CNI/Multus or
  annotations). Extra veths in the sandbox namespace do not appear in the guest.
- a router needs 3+ guest-visible NICs; containerlab+Kata has no supported path for that.

A microVM lab must own the TAP-to-virtio mapping itself. That is the orchestrator.

---

## 3. Layer model

| Layer | Owned by | Objects | Fault surface |
|---|---|---|---|
| Guest | Alpine + FRR + agent | `eth0..n`, `lo`, routes, BGP |
| Link | orchestrator | bridge, 2 TAPs, qdiscs | `ip link`, `tc` |
| VMM | orchestrator | Firecracker process, API socket, snapshots | kill, restore |
| Host | orchestrator (sudo) | mgmt bridge, state dirs, telemetry |

Rule: a **link fault** never enters the guest; a **service/process fault** does (via the
agent). Keeping this separation makes faults reproducible.

---

## 4. Topology model

### 4.1 Schema

```yaml
name: pure-3r3h
seed: 42

defaults:
  vcpu: 1
  mem_mb: 256
  kernel: kernel/vmlinux
  base_image: images/alpine-frr.ext4
  mgmt_subnet: 172.31.0.0/24
  mgmt_gateway: 172.31.0.1

nodes:
  r1: { role: router, asn: 65001, router_id: 10.0.0.1, loopback: 10.0.0.1/32, mgmt_ip: 172.31.0.11 }
  r2: { role: router, asn: 65002, router_id: 10.0.0.2, loopback: 10.0.0.2/32, mgmt_ip: 172.31.0.12 }
  r3: { role: router, asn: 65003, router_id: 10.0.0.3, loopback: 10.0.0.3/32, mgmt_ip: 172.31.0.13 }
  h1: { role: host, mgmt_ip: 172.31.0.21 }
  h2: { role: host, mgmt_ip: 172.31.0.22 }
  h3: { role: host, mgmt_ip: 172.31.0.23 }

links:
  - { id: l1, a: h1, b: r1, subnet: 10.0.10.0/30, a_ip: 10.0.10.2, b_ip: 10.0.10.1 }
  - { id: l2, a: r1, b: r2, subnet: 10.0.12.0/30, a_ip: 10.0.12.1, b_ip: 10.0.12.2 }
  - { id: l3, a: r2, b: r3, subnet: 10.0.23.0/30, a_ip: 10.0.23.1, b_ip: 10.0.23.2 }
  - { id: l4, a: r3, b: h3, subnet: 10.0.30.0/30, a_ip: 10.0.30.1, b_ip: 10.0.30.2 }
  - { id: l5, a: r2, b: h2, subnet: 10.0.20.0/30, a_ip: 10.0.20.1, b_ip: 10.0.20.2 }

hosts:
  h1: { default_via: 10.0.10.1 }
  h2: { default_via: 10.0.20.1 }
  h3: { default_via: 10.0.30.1 }

probes:
  ping:
    - { from: h1, to: 10.0.30.2, interval_ms: 1000 }
    - { from: h1, to: 10.0.0.2,  interval_ms: 1000 }
  iperf:
    - { from: h1, to: 10.0.30.2, rate: 10m, duration_s: 10 }
```

### 4.2 Interface assignment rule

Deterministic, enforced by the loader:

- `eth0` is **always** the management NIC (added first to the VMM).
- data NICs are `eth1`, `eth2`, ... assigned in the order the node's links appear in
  `links:`.
- guest NIC names follow VMM device-add order; this is verified in phase 1 and the run
  aborts if it does not hold.

For `r2`, links `l2`, `l3`, `l5` appear in that order, so:

| guest if | link | address |
|---|---|---|
| `eth0` | mgmt | `172.31.0.12/24` |
| `eth1` | `l2` | `10.0.12.2/30` |
| `eth2` | `l3` | `10.0.23.1/30` |
| `eth3` | `l5` | `10.0.20.1/30` |

### 4.3 Address and AS plan

| Item | Value |
|---|---|
| loopbacks | `10.0.0.1/32`, `10.0.0.2/32`, `10.0.0.3/32` |
| router ASNs | `65001`, `65002`, `65003` |
| inter-router links | `10.0.12.0/30`, `10.0.23.0/30` |
| host links | `10.0.10.0/30`, `10.0.20.0/30`, `10.0.30.0/30` |
| management | `172.31.0.0/24`, gateway `172.31.0.1` |
| guest MAC | `02:fc:00:00:<node_idx>:<if_idx>` |

---

## 5. Node model

### 5.1 Image strategy (v1)

Per-node image **copy** with configuration baked in:

```
images/alpine-frr.ext4          base, built once
runtime/<lab>/<node>/rootfs.ext4   copy per node, config baked in
```

Why copy instead of a shared read-only base: it removes all tmpfs/overlay plumbing from
v1, and snapshots make rebuilds rare. Trade-off: ~200 MB per node on disk (6 nodes ≈ 1.2 GB).
A read-only base plus a small config drive is the v2 optimisation, not a v1 requirement.

### 5.2 Guest layout

| Path | Purpose |
|---|---|
| `/etc/frr/frr.conf` | rendered per node (routers only) |
| `/etc/lab/node.json` | identity, addresses, MACs, role |
| `/etc/lab/net.sh` | host-style address setup (hosts only) |
| `/etc/init.d/lab-net` | applies `net.sh`, openrc `default` runlevel |
| `/etc/init.d/lab-agent` | starts the vsock agent |
| `/var/log/lab/` | local logs (also captured by host) |

Routers have **no** `net.sh`: zebra programs interface addresses from `frr.conf`. Hosts use
`ip` commands from `net.sh`. This keeps the router configuration single-sourced.

Rendered `frr.conf` for `r2`:

```
frr version 10.7
frr defaults traditional
hostname r2
log syslog informational
service integrated-vtysh-config
!
interface eth1
 ip address 10.0.12.2/30
!
interface eth2
 ip address 10.0.23.1/30
!
interface eth3
 ip address 10.0.20.1/30
!
interface lo
 ip address 10.0.0.2/32
!
router bgp 65002
 bgp router-id 10.0.0.2
 no bgp ebgp-requires-policy
 neighbor 10.0.12.1 remote-as 65001
 neighbor 10.0.23.2 remote-as 65003
 !
 address-family ipv4 unicast
  network 10.0.0.2/32
  network 10.0.12.0/30
  network 10.0.20.0/30
  network 10.0.23.0/30
 exit-address-family
!
```

### 5.3 Firecracker desired state (per node)

The orchestrator renders this and translates it into API calls on a per-node Unix socket
(`runtime/<lab>/<node>/fc.sock`).

```json
{
  "boot-source": {
    "kernel_image_path": "kernel/vmlinux",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off root=/dev/vda rw"
  },
  "drives": [
    {"drive_id": "root", "path_on_host": "runtime/pure-3r3h/r2/rootfs.ext4",
     "is_root_device": true, "is_read_only": false}
  ],
  "machine-config": {"vcpu_count": 1, "mem_size_mib": 256, "smt": false},
  "network-interfaces": [
    {"iface_id": "eth0", "host_dev_name": "tap-r2-0", "guest_mac": "02:fc:00:00:02:00"},
    {"iface_id": "eth1", "host_dev_name": "tap-r2-1", "guest_mac": "02:fc:00:00:02:01"},
    {"iface_id": "eth2", "host_dev_name": "tap-r2-2", "guest_mac": "02:fc:00:00:02:02"},
    {"iface_id": "eth3", "host_dev_name": "tap-r2-3", "guest_mac": "02:fc:00:00:02:03"}
  ],
  "vsock": {"guest_cid": 3, "uds_path": "runtime/pure-3r3h/r2/vsock.sock"}
}
```

Notes:

- **All devices are declared before `InstanceStart`.** Firecracker does not hot-add NICs.
- `guest_cid` must be unique per running VM (3, 4, 5, ... here).
- The kernel must be an **ELF `vmlinux`**; Alpine's `bzImage` will not work.
- Swapping containers for a different VMM (Cloud Hypervisor) changes this block only; the
  rest of the architecture is VMM-agnostic.

### 5.4 Boot sequence

1. `preflight`: `sudo`, `/dev/kvm`, kernel file, base image, `ip`/`tc`, free memory.
2. `plan`: load YAML, assign interfaces/MACs/CIDs, render `frr.conf` / `node.json`,
   compute the link/bridge plan.
3. `build`: for each node, copy base image, inject config, set up `/etc/init.d` services.
4. `wire`: create bridges and TAPs, attach, bring up.
5. `launch`: start Firecracker per node, apply config over the API, `InstanceStart`.
6. `converge`: poll `show bgp summary json` until expected sessions are `Established` and
   expected routes exist; then poll probes for a stability window.
7. `snapshot`: one full snapshot per node, recorded in the lab manifest.

---

## 6. Link model

### 6.1 Composition

A link `l2` between `r1` and `r2` becomes:

```
guest r1 eth2  --tap-r1-2--+
                           +-- br-l2 (no IP, no STP) --+
guest r2 eth1  --tap-r2-1--+
```

- the bridge carries no IP address and runs no routing;
- exactly two ports, so it behaves as a two-port switch;
- `br-l2` is the **fault handle** for the link.

### 6.2 Direction semantics (the subtle part)

- A root qdisc on `tap-r1-2` shapes packets the **host sends into r1**, i.e. the
  `r2 -> r1` direction of that link.
- To delay `r1 -> r2`, apply netem on `tap-r2-1` (the receiving TAP).
- Symmetric delay: apply on both TAPs.
- `ip link set tap-x down` removes the bridge port; the link stops carrying traffic in
  both directions. That is `link_down`.

Every fault record carries an explicit `direction` so this can never be ambiguous.

### 6.3 Naming

| Object | Pattern | Example |
|---|---|---|
| bridge | `br-<link_id>` | `br-l2` |
| data TAP | `tap-<node>-<if_idx>` | `tap-r2-1` |
| mgmt TAP | `tap-<node>-0` | `tap-r2-0` |

---

## 7. Control plane

### 7.1 Transport

vsock, host to guest, plus configuration baked into the image. MMDS is not required in v1.

- Each VM declares a `vsock` device at boot.
- The host connects to `runtime/<lab>/<node>/vsock.sock` and speaks **line-delimited JSON**.
- The guest agent listens on the same vsock port, executes an allowlisted command, and
  returns a structured reply.
- All timestamps in replies are host-side; guest clocks are not trusted.

### 7.2 Protocol

Request:

```json
{"id": 7, "cmd": "exec", "argv": ["vtysh", "-c", "show bgp summary json"], "timeout_s": 5}
```

Response:

```json
{"id": 7, "ok": true, "rc": 0, "stdout": "{...}", "stderr": "", "host_ts": "2026-11-17T10:04:59.812Z"}
```

Command set:

| cmd | Purpose | Allowlist |
|---|---|---|
| `ping` | liveness | — |
| `exec` | run a command | `vtysh`, `ip`, `ping`, `iperf3`, `tc`, `cat`, `stress-ng` |
| `write_file` | push a config | paths under `/etc/lab/`, `/etc/frr/` |
| `stress_start` | CPU/memory load | params validated |
| `stress_stop` | stop load | — |
| `shutdown` | clean guest stop | — |

The allowlist lives in the agent, not only in the orchestrator.

### 7.3 Why not telemetry over the network

Pulling metrics over the data plane would corrupt the thing being measured. vsock is
outside the emulated network, so it does not perturb any link. The management NIC exists
for interactive debugging only.

---

## 8. Fault model

### 8.1 Record shape

Every fault is a first-class object with the NIKA-style tuple plus enough to reproduce it:

```json
{
  "fault_id": "f-0007",
  "experiment_id": "exp-2026-11-17T10-00-00",
  "device": "r1",
  "component": "tap-r1-1",
  "link": "l2",
  "type": "link_delay",
  "params": {"delay_ms": 200, "jitter_ms": 10, "direction": "r2->r1"},
  "t_start": "2026-11-17T10:05:00.123Z",
  "t_end": "2026-11-17T10:06:00.123Z",
  "applied": true,
  "verified": true,
  "verify_detail": "qdisc present on tap-r1-1: netem delay 200ms 10ms"
}
```

### 8.2 Catalog (pure lab)

| id | layer | apply | revert | verify |
|---|---|---|---|---|
| `link_down` | link | `ip link set <tap> down` | `ip link set <tap> up` | operstate down; probes fail |
| `link_delay` | link | `tc qdisc add dev <tap> root netem delay Xms` | `tc qdisc del dev <tap> root` | `tc -s qdisc` shows netem |
| `link_jitter` | link | `... netem delay Xms Yms distribution normal` | as above | qdisc params |
| `link_loss` | link | `... netem loss P%` | as above | qdisc params + counter |
| `link_corrupt` | link | `... netem corrupt P%` | as above | qdisc params |
| `link_reorder` | link | `... netem reorder P% 50%` | as above | qdisc params |
| `link_bandwidth` | link | `... root tbf rate R burst B latency L` | as above | qdisc params |
| `iface_flap` | link | loop down/up on interval | stop loop; leave up | N transitions in log |
| `bgp_reset` | guest | `vtysh -c 'clear bgp *'` | — | FSM leaves/returns to Established |
| `bgp_withdraw` | guest | `no network P` under `router bgp` | re-add `network P` | prefix gone from peer RIB |
| `bgp_asn_mismatch` | guest | change `neighbor ... remote-as` | restore value | session drops, mismatch logged |
| `router_crash` | vmm | `kill -9 <fc pid>` | snapshot restore | VMM gone; guest unreachable |
| `router_reboot` | vmm | kill + cold boot | snapshot restore | reboot observed in serial log |
| `cpu_stress` | guest | `stress-ng --cpu 0 --timeout Ts` | `stress_stop` | load average / steal time |
| `mem_stress` | guest | `stress-ng --vm ...` | `stress_stop` | memory usage delta |

Every entry implements the same interface:

```python
class Fault:
    id: str
    def precheck(self, lab) -> None: ...
    def apply(self, lab) -> None: ...
    def verify(self, lab) -> bool: ...
    def revert(self, lab) -> None: ...
```

`apply` without a passing `verify` fails the run.

### 8.3 What is deliberately absent in v1

Retries, timeouts, backoff, circuit breakers and application-level overload. These need
real services and belong to the hybrid phase; simulating them in the pure lab would be
exactly the "pretty animation" failure mode.

---

## 9. Telemetry model

### 9.1 Records

Long format, one JSON object per line, host-timestamped:

```json
{"ts":"2026-11-17T10:04:59.500Z","node":"r2","scope":"iface","iface":"eth1","metric":"rx_bytes","value":812311}
{"ts":"2026-11-17T10:04:59.500Z","node":"r2","scope":"qdisc","iface":"eth1","metric":"backlog_bytes","value":0}
{"ts":"2026-11-17T10:04:59.500Z","node":"r2","scope":"bgp","peer":"10.0.12.1","metric":"state","value":"Established"}
{"ts":"2026-11-17T10:04:59.500Z","node":"r1","scope":"rib","metric":"ipv4_routes","value":6}
{"ts":"2026-11-17T10:04:59.500Z","node":"h1","scope":"probe","target":"10.0.30.2","metric":"rtt_ms","value":0.41}
{"ts":"2026-11-17T10:04:59.500Z","node":"r2","scope":"sys","metric":"cpu_user_pct","value":3.1}
```

### 9.2 Sources

| Scope | Source | Cadence |
|---|---|---|
| `iface` | `/proc/net/dev` in guest | 1 s |
| `qdisc` | `tc -s qdisc show dev <if>` | 1 s |
| `bgp` | `vtysh -c 'show bgp summary json'` / `neighbors json` | 1 s |
| `rib` | `vtysh -c 'show ip route json'` | 1 s |
| `probe` | `iputils-ping` (and `iperf3` runs) from `probes:` | 1 s |
| `sys` | `/proc/stat`, `/proc/meminfo` in guest | 1 s |
| `events` | orchestrator state transitions | on change |

### 9.3 Baseline and windows

Each experiment declares windows relative to `t_start`:

| Window | Meaning |
|---|---|
| `pre` | 30 s before injection, used for the baseline envelope |
| `fault` | injection to revert |
| `post` | 60 s after revert, used for recovery |

A run is only exported if the `pre` window has no gaps and passes a stability check.

---

## 10. Experiment lifecycle

```
init -> planned -> built -> wired -> launched -> converged
     -> snapshotted -> baseline -> faulted -> observing
     -> reverted -> post -> restored -> exported
```

Any failure moves the experiment to `failed` and leaves the state directory intact for
post-mortem.

### 10.1 Directory layout

```
experiments/<experiment_id>/
  manifest.yaml            # topology path, seed, fault schedule, windows, versions
  topology.resolved.yaml   # fully expanded: ifaces, MACs, CIDs, rendered configs
  ground_truth.jsonl       # one line per fault
  metrics.jsonl            # telemetry
  events.jsonl             # BGP/iface state transitions
  summary.md               # human-readable: what was injected, what changed
```

### 10.2 Reading a run

`summary.md` answers, without opening the raw data:

- what was injected, where, for how long;
- which sessions dropped and when;
- which probes changed (RTT, loss) and by how much;
- which routes disappeared and from which RIBs;
- when recovery completed;
- whether verification passed.

---

## 11. Snapshot and reset

- Snapshot **after** convergence, never during a fault.
- Full snapshot per node: memory file plus device state.
- Bridges and TAPs are **not** destroyed between runs. The VMM reopens the same TAP names
  on restore, so keeping them alive is a requirement, not an optimisation.
- After restore: wait for BGP reconvergence and probe stability, then compare a metric
  fingerprint against the pre-fault snapshot. A mismatch aborts the repetition.

Open item to verify in phase 1: **vsock behaviour across snapshot restore.** If the
session does not survive, the agent reconnects on the next request; the orchestrator must
not assume a persistent connection.

---

## 12. Orchestrator

### 12.1 Modules

| Module | Responsibility | Needs root |
|---|---|---|
| `topology.py` | parse/validate YAML, assign ifaces/MACs/CIDs, render configs | no |
| `net.py` | bridges, TAPs, attach/detach, qdisc apply/clear, teardown | yes |
| `vm.py` | Firecracker API, boot, states, snapshot/restore, serial logs | yes |
| `guest.py` | vsock client, exec, readiness, file push | no |
| `inject.py` | fault classes, registry, apply/verify/revert | yes (link), no (guest) |
| `telemetry.py` | pollers, clock, writers, gap detection | no |
| `experiment.py` | state machine, windows, reset, export | yes |
| `cli.py` | command surface | — |
| `agent/agent.py` | in-guest vsock server, allowlist | — (runs in guest) |

### 12.2 CLI

```
lab build    -f lab/topologies/pure-3r3h.yaml
lab up
lab status
lab snapshot --name converged
lab run      --fault link_delay --target l2 --params delay_ms=200 --observe 60
lab restore  --name converged
lab export   --experiment <id>
lab down     --clean
```

`lab down` removes TAPs, bridges and Firecracker processes, but never the images.

### 12.3 State

```
lab/runtime/<lab_name>/
  lab.json                 # resolved plan + status
  <node>/rootfs.ext4
  <node>/fc.sock
  <node>/vsock.sock
  <node>/serial.log
  snaps/<name>/<node>.{snap,mem}
```

`lab/runtime/` is generated and git-ignored.

---

## 13. Repository layout

```
sandbox-lab/
  docs/
    00-PREREQUISITES.md
    01-PROJECT.md
    02-ARCHITECTURE.md
  lab/
    images/                  # build scripts (no Docker)
      build-alpine.sh
      build-image.sh
    kernel/                  # vmlinux from Firecracker CI artifacts
    topologies/
      pure-3r3h.yaml
    runtime/                 # generated, git-ignored
  orchestrator/
    topology.py net.py vm.py guest.py inject.py telemetry.py experiment.py cli.py
  agent/
    agent.py
  configs/
    frr/frr.conf.j2
    guest/net.sh.j2
    guest/node.json.j2
  experiments/               # generated, git-ignored
  tests/
```

---

## 14. Security and isolation

Guests are isolated from each other by the hypervisor, which is why this architecture
satisfies "no shared kernel". Within the host:

- the orchestrator runs as root because bridges, TAPs and `tc` require `CAP_NET_ADMIN`;
- the scope is a dedicated state directory plus a naming prefix (`br-l*`, `tap-*`);
- teardown is prefix-scoped so it cannot delete unrelated host interfaces;
- the agent executes an allowlist only; no shell interpolation;
- no secrets in topology files; no guest has host filesystem access beyond its own image.

The `jailer` is out of scope for v1 (single-user lab) and noted as a hardening step.

---

## 15. Fidelity boundaries

See `01-PROJECT.md` §8 for the full contract. Summary:

- real: guest kernel, FRR/BGP, forwarding, `tc` semantics, process/kernel behaviour;
- not real: ASICs, optics, line rate, buffers, hardware faults, scale, application retries.

Therefore: convergence *shapes* and affected sets are meaningful; exact convergence
*seconds* are host-load dependent and must be reported as distributions, not single values.

---

## 16. Sharp edges to verify (not assume)

1. Firecracker x86_64 requires an ELF `vmlinux`; the CI kernel satisfies this.
2. Guest NIC enumeration order must follow device-add order; abort if not.
3. `root=/dev/vda` and the rest of the kernel cmdline must match the root drive.
4. netem direction on TAPs (§6.2); symmetric faults need both ends.
5. TAPs and bridges must persist across snapshot restore.
6. vsock connectivity across snapshot restore.
7. FRR needs netlink/namespace support in the chosen kernel; test `vtysh` in phase 1.
8. `stress-ng` must be present in the Alpine image for CPU/memory faults.
9. Concurrent Firecracker boots should be staggered to keep convergence timing sane.

---

## 17. Validation plan

The project's credibility rests on these checks, not on the dashboard.

| Check | Method | Pass condition |
|---|---|---|
| Fault actually applied | `verify()` per fault | 100% pass |
| Fault has observable effect | targeted probe changes | effect size above baseline noise |
| No false effect | fault on a link not in a probe's path | probe unchanged beyond noise |
| Baseline stability | 3 runs, compare variance | variance documented, no drift |
| Reset determinism | snapshot → restore → fingerprint | identical fingerprint |
| BGP correctness | expected RIB entries per router | exact match to plan |
| Repeatability | same fault, N repetitions | same affected set every time |

Later, cross-validation: reproduce one link-down and one delay scenario in a real lab
(e.g. the EVE-NG environment from the internship) and compare the **direction and shape**
of the effect, not the exact timings.

---

## 18. Phase plan

### Phase 0 — preflight
- [ ] confirm `sudo`, `/dev/kvm`, Firecracker version
- [ ] fetch Firecracker CI `vmlinux` into `lab/kernel/`
- [ ] confirm Alpine `main` + `community` reachable
- [ ] confirm host has `ip`, `tc`, `mkfs.ext4`

### Phase 1 — image and one VM
- [ ] build Alpine rootfs with `apk --root` (frr, iproute2, python3, stress-ng, iperf3)
- [ ] `mkfs.ext4 -d` into a base image
- [ ] boot one Firecracker VM, serial shell, `vtysh -c 'show version'`
- [ ] confirm guest NIC order and vsock exec
- [ ] create and restore a snapshot; record whether vsock survives

### Phase 2 — links
- [ ] `net.py`: bridge + two TAPs, idempotent create/destroy
- [ ] two VMs, ping across one link
- [ ] apply `netem delay`, confirm RTT shift in the correct direction

### Phase 3 — full topology
- [ ] render `frr.conf` for all routers from YAML
- [ ] 3 routers + 3 hosts, 4 eBGP sessions Established
- [ ] h1 ↔ h3 reachable through r1-r2-r3; h2 reachable

### Phase 4 — control plane
- [ ] guest agent with allowlist
- [ ] orchestrator drives `vtysh`, readiness checks, reboot detection

### Phase 5 — telemetry
- [ ] all pollers, 1 s cadence, host timestamps
- [ ] gap-free 60 s baseline
- [ ] events stream for BGP/iface transitions

### Phase 6 — faults and experiments
- [ ] all 15 faults implement `apply/verify/revert`
- [ ] experiment state machine, windows, export, `summary.md`
- [ ] one end-to-end labeled experiment per fault family

### Phase 7 — determinism
- [ ] snapshot all nodes post-convergence
- [ ] restore + reconverge + fingerprint equality
- [ ] N repetitions of the full catalog

### Phase 8 — hybrid (later)
- [ ] services as hosts, placement across routers
- [ ] service telemetry, load generator
- [ ] retry/timeout/overload faults
- [ ] dataset comparability with RCAEval schemas
