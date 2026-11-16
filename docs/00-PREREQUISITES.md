# 00 — Prerequisites

What you need to learn to understand and build this project, in the order you should
learn it, with the minimum depth that is actually enough.

This document is a **learning path**, not a reference manual. For each topic it gives:

- **Why it matters here** — the concrete place in this project where you use it.
- **Your bar** — the level at which you can stop and move on.
- **Things to actually try** — commands and small exercises.
- **Where to read** — canonical sources.

If you only have a weekend, read sections 1, 2, 4 and 5. If you have two weeks, follow the
schedule at the end.

---

## 0. The mental model you are aiming for

This project builds a **mini network out of microVMs**:

```
+-----------+     +-----------+     +-----------+
|  guest r1 |     |  guest r2 |     |  guest r3 |
|  (Alpine) |     |  (Alpine) |     |  (Alpine) |
|  + FRR    |     |  + FRR    |     |  + FRR    |
|  kernel A |     |  kernel B |     |  kernel C |
+-----+-----+     +-----+-----+     +-----+-----+
      | virtio-net      |                 |
   +--+--+          +---+---+          +--+--+
   | TAP |          |  TAP  |          | TAP |
   +--+--+          +---+---+          +--+--+
      |                 |                 |
   +--+-----------------+-----------------+--+
   |        Linux bridge(s) on the HOST      |
   +-----------------------------------------+
```

The **guest** is a real kernel. The **link** is a Linux bridge with TAP devices. The
**VMM** is Firecracker. The **control plane** is vsock. The **fault** is `tc` or a
process kill on the host side.

If you can explain that diagram and where each component lives (host vs guest), you
understand about 80% of this project.

---

## 1. Linux networking fundamentals

**Why it matters here.** Every "link" in the topology is a bridge plus two TAP devices.
Every fault is a change to an interface or a qdisc. If you cannot reason about
interfaces, routes and neighbour tables, you cannot debug this project.

**Your bar.** You can answer, without looking it up:

- what a `veth` pair is, and what a `TAP` device is, and why we use a TAP here;
- what a Linux bridge does at layer 2;
- what happens to a packet from guest A to guest B, hop by hop, through bridges and TAPs;
- where the routing table lives for a router guest versus the host;
- why the guest kernel **cannot** see a veth that only exists on the host.

That last point is the reason container-based tools do not fit this project: a microVM
only sees the devices the VMM exposes to it.

**Things to actually try.**

```bash
# Inspect the host
ip -br link
ip -br addr
ip route show
ip neigh show

# Create a bridge and two TAPs by hand (needs root)
sudo ip link add name br-test type bridge
sudo ip link set br-test up
sudo ip tuntap add dev tap-a mode tap
sudo ip tuntap add dev tap-b mode tap
sudo ip link set tap-a master br-test
sudo ip link set tap-b master br-test
sudo ip link set tap-a up
sudo ip link set tap-b up
ip -br link show master br-test
sudo ip link del br-test   # deletes ports too
```

```bash
# A veth pair and a network namespace, to feel the difference from a TAP
sudo ip netns add n1
sudo ip link add v0 type veth peer name v1
sudo ip link set v1 netns n1
sudo ip netns exec n1 ip link set lo up
sudo ip netns exec n1 ip -br link
sudo ip netns del n1
```

**Where to read.** `man 8 ip-link`, `man 8 ip-route`, `man 8 ip-neighbour`,
`man 8 bridge`, the kernel `Documentation/networking/` tree, and any serious write-up
on "veth vs tap vs macvtap".

---

## 2. `tc`, qdiscs and netem

**Why it matters here.** Almost every network fault is a `tc` operation: delay, jitter,
loss, corruption, reordering, rate limiting. Getting the *direction* wrong is the classic
bug, so this section is not optional.

**Your bar.**

- Explain the difference between a root qdisc and an ingress qdisc.
- Know that a root qdisc on device `X` shapes **egress** from `X`.
- Know that on a **TAP**, egress means "packets the host sends into the guest" — i.e.
  the packets the **guest receives**.
- Know how to add and remove a `netem` and read `tc -s`.
- Know that ingress shaping needs an `ifb` device.

**Things to actually try.**

```bash
# Add 100 ms delay with 10 ms jitter, 1% loss, to everything sent out of tap-a
sudo tc qdisc add dev tap-a root netem delay 100ms 10ms distribution normal loss 1%
sudo tc -s qdisc show dev tap-a

# Change it
sudo tc qdisc change dev tap-a root netem delay 200ms

# Remove it (and everything under it)
sudo tc qdisc del dev tap-a root
```

```bash
# Rate limit instead
sudo tc qdisc add dev tap-a root tbf rate 10mbit burst 32kbit latency 400ms
sudo tc qdisc del dev tap-a root
```

Ingress shaping pattern (memorise the shape, not the numbers):

```bash
sudo ip link add ifb0 type ifb
sudo ip link set ifb0 up
sudo tc qdisc add dev tap-a handle ffff: ingress
sudo tc filter add dev tap-a parent ffff: \
    protocol all u32 match u32 0 0 action mirred egress redirect dev ifb0
sudo tc qdisc add dev ifb0 root netem delay 100ms
```

**Where to read.** `man 8 tc`, `man 8 tc-netem`, `man 8 tc-tbf`, `man 8 tc-htb`,
`man 8 tc-mirred`, and the LARTC / "tc netem" tutorials.

---

## 3. Virtualization, KVM and virtio

**Why it matters here.** You are running a hypervisor and debugging boot and device
problems. "It didn't boot" is 50% of the work in phase 1.

**Your bar.**

- Guest vs host, vCPU, guest RAM, KVM as the acceleration interface.
- What `virtio` is and why paravirtual devices exist.
- The four virtio devices this project uses: **virtio-net**, **virtio-blk**,
  **virtio-vsock**, and (optionally) **virtio-balloon**.
- Guest boot: kernel image + kernel command line + root filesystem + PID 1.
- Device enumeration: the order in which a VMM adds NICs determines guest names
  (`eth0`, `eth1`, ...). This project depends on that ordering, so verify it in phase 1.
- Why a guest kernel needs certain config options (netlink, namespaces, virtio drivers)
  for FRR to work.

**Things to actually try.**

- Boot a tiny VM with QEMU (`qemu-system-x86_64 -kernel ... -drive ...`) just to see the
  concepts, then do the same with Firecracker.
- Inside the guest, run `dmesg | grep -i virtio` and match each line to a device you
  declared.

**Where to read.** The KVM and virtio specifications, the Firecracker "design" document,
and the kernel `Documentation/virt/kvm/` tree.

---

## 4. Firecracker

**Why it matters here.** It is the VMM for every node in the lab.

**Your bar.** You can write a Firecracker configuration from memory and explain every
key:

- `machine-config` (vCPU count, memory, SMT)
- `boot-source` (kernel image path, kernel command line)
- `drives` (path, `is_root_device`, `is_read_only`, block device id)
- `network-interfaces` (host TAP name, guest MAC, device id)
- `vsock` (guest CID, host Unix socket path)
- `mmds-config` (optional metadata service)
- the REST API over a Unix socket, and the order of calls before `InstanceStart`
- serial console (`ttyS0`) as the only default output channel
- **snapshot create/load**, and the fact that devices cannot be hot-added after boot
- the `jailer` and why production deployments use it

Critical constraint: on x86_64 Firecracker wants an **ELF `vmlinux`**, not a compressed
`bzImage`. Alpine ships `vmlinuz-virt` (a `bzImage`), which is why this project uses the
**Firecracker CI kernel** instead of Alpine's kernel. This is a fact to verify in phase 0,
not to assume blindly.

**Things to actually try.**

- Start one microVM manually, from a config file, and get a shell on the serial console.
- Create a snapshot, kill the VMM, load the snapshot, and confirm the guest state resumed.
- Do it once with a TAP device, and confirm the TAP must still exist when you restore.

**Where to read.** The Firecracker documentation (getting started, API, snapshotting,
jailer) at `https://firecracker-microvm.github.io/`, and the `firecracker` API spec in the
repository's `src/firecracker/swagger/` directory.

---

## 5. FRR (Free Range Routing)

**Why it matters here.** FRR is the real routing stack inside every router guest. This is
what makes the lab "real" rather than simulated.

**Your bar.**

- Which daemons matter: `zebra` (interface/route manager), `bgpd` (BGP), and `mgmtd`
  (management frontend in FRR 9/10).
- `vtysh` as the single CLI over all daemons, and `-c` for non-interactive commands.
- The integrated config file (`frr.conf`) with `service integrated-vtysh-config`.
- That **zebra can program interface IP addresses** (`interface eth1` / `ip address ...`),
  so the router guests need no separate network configuration layer.
- JSON output, which is how the orchestrator reads state:
  `show bgp summary json`, `show bgp neighbors json`, `show ip route json`,
  `show interface json`.
- BGP in FRR 8+: `no bgp ebgp-requires-policy` unless you attach route-maps/policies.

**Things to actually try.**

Two VMs, one /30 link, AS 65001 and 65002, one `network` statement each, and ping the
far loopback. Then `clear bgp *` and watch the FSM go down and back up.

**Where to read.** FRR docs at `https://docs.frrouting.org/` (Basic Setup, BGP), and
`man 8 vtysh`.

---

## 6. BGP fundamentals

**Why it matters here.** The pure lab's control plane *is* eBGP. Faults like
"session reset", "prefix withdrawn" and "ASN mismatch" are only meaningful if you know
the state machine.

**Your bar.**

- AS numbers, eBGP vs iBGP, and why we use eBGP between all three routers here.
- The BGP finite state machine: `Idle → Connect → Active → OpenSent → OpenConfirm →
  Established`, and what each transition means.
- Hold time, keepalive, connect retry, and what a session reset looks like in telemetry.
- `network` statement vs redistribution, and next-hop behaviour on eBGP.
- Best-path basics (weight, local-pref, AS-path length, origin, MED).
- Loopback peering as a concept (used later for iBGP experiments).

**Things to actually try.** Draw the 3-router topology on paper. For each session, write
down: local AS, remote AS, local IP, remote IP. Predict what happens to routes in r3
when the r1–r2 link goes down, before you test it.

**Where to read.** FRR BGP docs, RFC 4271 (skim the FSM section), and any RIPE/APNIC
BGP tutorial.

---

## 7. Alpine Linux and building a rootfs without Docker

**Why it matters here.** Every guest runs Alpine. We build the rootfs with `apk --root`
and `mkfs.ext4 -d`, explicitly without Docker.

**Your bar.**

- `apk` basics: repositories (`main`, `community`), `apk add`, `apk --root`,
  `--initdb`, `--arch`, `--keys-dir`.
- openrc instead of systemd: `rc-update add <svc> default`, `rc-service <svc> start`,
  and init scripts in `/etc/init.d/`.
- `/etc/local.d/` and the `local` service, or a custom init script, as the place to run
  first-boot configuration.
- Building an ext4 image from a directory: `mkfs.ext4 -d <dir> -F <file> <size>`.
- Why Alpine is small: musl libc, BusyBox, no systemd.
- That FRR is packaged in `community` (`frr`, `frr-openrc`), version 10.x on recent
  releases.

**Things to actually try.**

```bash
# Build a minimal rootfs directory on the host (no Docker)
sudo apk --root /tmp/alpine-root --initdb --arch x86_64 \
    -X https://dl-cdn.alpinelinux.org/alpine/v3.23/main \
    -X https://dl-cdn.alpinelinux.org/alpine/v3.23/community \
    add alpine-base frr frr-openrc iproute2 bash python3

# Make an image out of it
sudo mkfs.ext4 -d /tmp/alpine-root -F rootfs.ext4 200M
```

Then chroot into it (or boot it) and confirm `vtysh -c 'show version'` works.

**Where to read.** Alpine wiki (`https://wiki.alpinelinux.org/`), `man 8 apk`,
`man 5 apk-repositories`, `man 8 mkfs.ext4`, and the Alpine `OpenRC` page.

---

## 8. Telemetry and time series

**Why it matters here.** The project's output is *data*: what happened, when, and to
whom. Bad telemetry makes the whole thing a demo.

**Your bar.**

- The Prometheus data model (counter, gauge, histogram) — even though phase 1 writes
  JSONL, the concepts keep the schema sane.
- Pull vs push, scrape interval, and why cadence affects what you can see.
- Clock discipline: if host and guests disagree, your timelines lie. Use a consistent
  time source (`chrony` or host-synchronised vsock timestamps).
- What to record per fault: state before, during, after; the fault window; the
  ground-truth label.

**Things to actually try.** Write a 60-second baseline of one link: `/proc/net/dev`
counters, `tc -s qdisc`, ping RTT, BGP session state. Confirm there are no gaps.

**Where to read.** Prometheus "data model" and "instrumentation" docs, `man 5 proc`
(`/proc/net/dev`), `man 8 tc`, and `man 1 ping`.

---

## 9. Distributed-systems failure concepts

**Why it matters here.** These concepts explain *why* this lab is interesting, and they
define what the hybrid phase adds.

**Your bar.** Be able to explain, with a network example and a service example:

- retry, timeout, backoff, jitter;
- cascading failure and retry amplification (retries making a struggling service worse);
- blast radius and dependency graphs;
- graceful degradation and failover;
- MTTR, SLO, and error budget at a conceptual level.

Key insight for this project: at the **network layer** you observe failures directly
(packets drop, sessions reset). At the **service layer** you observe them indirectly
(latency, retries, queueing). The pure lab gives you the first; the hybrid phase adds the
second.

**Where to read.** The Google SRE book chapters on handling overload and cascading
failures, and any good write-up on "retry storms".

---

## 10. Python tooling

**Why it matters here.** The orchestrator is Python.

**Your bar.**

- `subprocess` for `ip`/`tc`/`mkfs` with correct argument handling and error checking.
- `asyncio` for supervising several VMs and sockets at once.
- Unix domain sockets and a simple line-delimited JSON protocol.
- `pydantic` (or dataclasses plus JSON schema) for the topology model.
- `pathlib`, `json`, `logging`, `argparse` (or `click`/`typer`).
- `pytest` for the pieces that do not need root.

**Things to actually try.** Write a 40-line async client that connects to a Unix socket,
sends a JSON line, and prints the reply. That is the core of the control plane.

**Where to read.** The Python standard library docs for `asyncio`, `subprocess`,
`socket`, and `pathlib`; the `pydantic` docs.

---

## 11. Prior art: datasets, benchmarks and why they are not injectors

**Why it matters here.** You will be asked "why not just use RCAEval?" You need a precise
answer.

Three distinct things are often confused:

1. **A fault catalog** — a list of fault *types* with parameters. Portable. Reuse it.
2. **An injector** — the code that *performs* a fault against a running system. You must
   write this for your environment.
3. **A dataset** — recordings (metrics, logs, traces) taken while someone else injected
   faults. **You cannot inject a dataset.** You can replay its workload, copy its
   taxonomy, and use its labels to evaluate a model.

What to take from each:

- **NIKA** (network-incident benchmark on Kathará/Containerlab):
  - a fault registry with stable ids and `apply`/`verify` semantics;
  - a ground-truth tuple shape: `(device, component, root_cause)`;
  - a verification pattern where every injected fault is *checked* to have taken effect;
  - a taxonomy of failure domains (link/interface, routing/control plane,
    forwarding/policy, service networking, addressing, resource, security).
  - Versions differ: the paper describes 54 issues / 640 incidents; NIKA 0.2 lists
    75 fault types / 1,098 tasks. Cite the version you use.
  - It runs on containers, so you reuse the *catalog and schema*, not the runtime.

- **RCAEval** (microservice RCA benchmark):
  - 735 failure cases over three suites (RE1, RE2, RE3) and three systems
    (Online Boutique, Sock Shop, Train Ticket);
  - fault types: `cpu`, `mem`, `disk`, `delay`, `loss`, `socket`, and code-level
    `f1`–`f5`;
  - per case: `metrics.json`, `inject_time.txt`, and (RE2/RE3) `logs.csv`, `traces.csv`;
  - an evaluation framework with reproducible baselines.
  - Only `delay` and `loss` map cleanly onto a network-only lab. The rest are
    service-layer and belong to the hybrid phase.

Reading list, in order:

1. NIKA paper and its `failures.md` reference.
2. RCAEval paper (WWW 2025) section 3, especially 3.2 "Fault Types".
3. The Google SRE book, "Addressing Cascading Failures".
4. Anything on `netem` and on BGP convergence.

---

## 12. Two-week schedule

| Day | Morning | Afternoon | Deliverable |
|---|---|---|---|
| 1 | Linux networking (§1) | `tc`/netem (§2) hands-on | a bridge with 2 TAPs, netem applied |
| 2 | Virtualization (§3) | Firecracker basics (§4) | one microVM booted, serial shell |
| 3 | Firecracker API + snapshots (§4) | vsock + MMDS | snapshot create/restore |
| 4 | Alpine rootfs build (§7) | boot Alpine under Firecracker | Alpine guest with `apk` working |
| 5 | FRR basics (§5) | one router, static routes | `vtysh` in guest |
| 6 | BGP (§6) | two-router eBGP | session `Established`, ping works |
| 7 | Review + buffer | write down open questions | notes |
| 8 | Telemetry (§8) | metric list + pollers | 60 s baseline |
| 9 | Python asyncio + sockets (§10) | control-plane client | JSON line RPC |
| 10 | Failure concepts (§9) | map faults to `tc` ops | fault table v1 |
| 11 | Datasets (§11) | read NIKA + RCAEval | ported catalog |
| 12 | Integration | 3 routers, full topology | converged lab |
| 13 | Inject one fault end to end | measure + label | one labeled experiment |
| 14 | Reset via snapshot | repeatability check | same result twice |

---

## 13. Readiness checklist

You are ready to build when you can tick all of these:

- [ ] Explain why the lab uses TAPs plus bridges instead of `veth` pairs everywhere.
- [ ] Explain why a microVM guest cannot see an interface that only exists on the host.
- [ ] Add, change and remove a `netem` qdisc, and state which direction it shapes.
- [ ] Write a Firecracker config with two NICs and a vsock, and boot it.
- [ ] Create and restore a snapshot, and say which host resources must exist at restore.
- [ ] Build an Alpine rootfs with `apk --root` and turn it into an ext4 image.
- [ ] Write a two-router eBGP config that converges, and read its state as JSON.
- [ ] Name the BGP FSM states in order.
- [ ] Describe what a fault record and a metric record contain in this project.
- [ ] State the difference between a fault catalog, an injector, and a dataset.

When all of that is true, continue with `01-PROJECT.md` and `02-ARCHITECTURE.md`.
