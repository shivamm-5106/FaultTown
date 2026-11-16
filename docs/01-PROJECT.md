# 01 — The Project

A deterministic microVM network lab where you build a real routed topology, break it on
purpose, and record exactly what happened.

> One line: **a real network, small enough to run on a laptop, that you can break and
> reset byte-for-byte.**

---

## 1. The problem

Distributed systems and networks fail in ways that are hard to see, hard to reproduce, and
hard to learn from:

- Failures are rare and random in production. You cannot wait for a link to flap.
- Post-incident analysis is contaminated by missing data and clock skew.
- Existing network simulators are too abstract; existing monitoring tools only observe
  systems that are already running.
- Benchmarks for troubleshooting (NIKA, RCAEval) either run on containers (shared kernel)
  or supply data without a topology you control.

We want the opposite: a **fixed, real topology** where a fault is an action you take, and
the resulting telemetry is a labeled artifact you keep.

---

## 2. The idea

Build a small network as a set of **microVMs**, each with its own kernel, running a real
routing stack (FRR). Connect them with real virtual links. Then:

1. **Build** a topology from one declarative file.
2. **Converge** the routing control plane until it is stable.
3. **Snapshot** every node, so the starting state is reproducible.
4. **Inject** one fault at a known time.
5. **Observe** the telemetry until the system settles.
6. **Revert** and **restore** the snapshot.
7. **Export** the fault label together with the measurements.

The result is a dataset of `(topology, fault, telemetry, ground truth)` tuples that you
generated yourself, on a topology you fully control.

### The analogy: a city's road network

| Network concept | City analogy |
|---|---|
| Router | a major intersection |
| Link | the road between two intersections |
| Host | a building |
| Interface | a driveway entrance |
| Routing table / FIB | the signs telling drivers which way to go |
| BGP session | the agreement between two cities about which roads are open |
| Link down | a bridge closed |
| Delay / loss | a traffic jam / a road with potholes |
| Route withdrawal | a city stops advertising a road |
| Convergence | traffic finding new routes |
| Snapshot reset | rewinding the whole city to 8:00 a.m. |

A road simulator lets you close a bridge and watch the chaos. This project differs in one
important way: the cars are real packets, the intersections run real routing software, and
the traffic jams emerge — they are not drawn.

---

## 3. Where this sits

Three related things are easy to confuse. They are complementary, not competitors.

| | **NetAI** (monitoring) | **FaultTown** (simulator) | **This lab** (terrain) |
|---|---|---|---|
| Direction | observe, then react | inject, then observe | inject, then observe |
| Environment | real devices (EVE-NG, Arista, BGP, gNMI) | abstract simulated services | real microVMs, real FRR |
| Layer | network (routing, interfaces) | application / service | network (phase 1), service (phase 2) |
| Output | alerts, health score, RCA | simulated cascade | labeled telemetry + ground truth |
| Strength | grounded in reality | cheap, repeatable, many services | real code paths, repeatable via snapshots |
| Weakness | cannot replay a failure | unvalidated model is "a pretty dashboard" | no hardware, small scale |

The short version:

- **NetAI** watches a real network and tells you when it breaks.
- **FaultTown** breaks a simulated network on purpose to study how damage spreads.
- **This lab** is the ground truth both of them lack: real packets, real control plane,
  deliberately broken, recorded, and resettable.

The key claim this project can make and the others cannot: **every fault here is applied to
real software (FRR, the Linux kernel) and verified to have taken effect, then measured.**

---

## 4. Goals

- **G1. Declarative topology.** One YAML file defines nodes, links, addresses and ASNs;
  everything else is generated from it, so the graph and the lab cannot drift apart.
- **G2. Real guest kernels.** Every node is a microVM. No shared host kernel between
  nodes. This is a hard requirement.
- **G3. Real control plane.** FRR (`zebra` + `bgpd`) with eBGP between routers.
- **G4. Verified faults.** Every injection has an `apply`, a `revert`, and a `verify` step.
  A fault that did not take effect is a failed experiment, not a data point.
- **G5. Labeled telemetry.** Each run yields machine-readable metrics plus a ground-truth
  record with a stable schema.
- **G6. Deterministic reset.** Snapshot/restore returns the lab to a byte-identical
  starting state, so repetitions are comparable.
- **G7. No Docker.** The lab is built and run with Firecracker, `apk`, `mkfs`, `ip` and
  `tc`. Container tooling is not part of the runtime.

## 5. Non-goals

- **Not a hardware fidelity claim.** We do not model ASICs, optics, line-rate forwarding,
  or hardware faults. See the fidelity contract below.
- **Not a production network.** This is a functional replica, not a digital twin of any
  real deployment.
- **Not a general-purpose emulator.** It runs one curated topology family well rather than
  every topology badly. Extensibility is via the YAML schema and the fault registry.
- **Not a UI project.** Data first. A dashboard can come later.
- **Not a replacement for a simulator.** It validates and generates data; it does not
  cheaply explore thousands of abstract scenarios.

---

## 6. Core concepts

| Term | Meaning in this project |
|---|---|
| **Node** | One microVM. Role is `router` or `host`. |
| **Link** | One Linux bridge with two TAP devices, one per endpoint. The unit of network fault injection. |
| **Topology** | The graph of nodes and links, declared in YAML. |
| **Convergence** | The state where all BGP sessions are `Established` and expected routes are present. |
| **Fault** | One deliberate change applied to a node or link, with parameters, a start and an end. |
| **Ground truth** | The record of what was injected: device, component, type, parameters, timestamps, and whether verification passed. |
| **Snapshot** | A Firecracker full snapshot of a node's memory and device state, used to reset. |
| **Experiment** | One run: topology + fault schedule + observation windows + seed. |
| **Blast radius** | The set of nodes/links/prefixes whose telemetry changed measurably because of the fault. |
| **Recovery** | The time from fault end until the system returns to its baseline envelope. |

Ground truth follows the `(device, component, root_cause)` shape borrowed from NIKA,
because it is the natural shape of a network fault. Example: `(r1, eth2, link_down)`.

---

## 7. Success criteria

The project is working when all of these are demonstrably true:

1. `lab up` brings up 3 routers and 3 hosts from one YAML file, with 4 BGP sessions
   `Established` and host-to-host reachability.
2. `lab snapshot` captures all nodes; `lab restore` returns to that state and the lab still
   converges identically.
3. One fault of each type can be applied, verified, reverted, and appears in telemetry.
4. A 60-second baseline has no gaps at the chosen cadence.
5. Running the same experiment twice from a restored snapshot produces the same
   *qualitative* result (same fault, same affected set) and comparable numbers.
6. Every experiment directory contains: manifest, ground truth, metrics, and a short
   human-readable summary.
7. No Docker is required at any point.

Quantitative targets (to be filled in after phase 6, not invented now):

| Metric | Target | Measured |
|---|---|---|
| Time to convergence (3 routers) | TBD | — |
| Baseline telemetry gap rate | 0 | — |
| Fault verify pass rate | 100% | — |
| Snapshot restore time | TBD | — |
| Run-to-run variance (RTT, loss) | TBD | — |

---

## 8. The fidelity contract

Stated plainly, because overclaiming is the failure mode of projects like this.

**Faithful (these are real):**

- the guest kernel, its network stack and its routing table;
- FRR's BGP implementation, timers, FSM and best-path selection;
- packet forwarding between guests, including MTU, ARP, TTL, fragmentation behaviour;
- link delay, jitter, loss, corruption, reordering and rate limiting at the link;
- interface down/up, session reset, route withdrawal, configuration mismatch;
- process crash and CPU/memory starvation inside a real kernel.

**Not faithful (these are synthetic or absent):**

- forwarding hardware: no ASIC, TCAM, per-port buffers, or line rate;
- physical failures: optics, PSU, backplane, bad cables (only emulated symptoms);
- scale: six nodes is not a data centre; retry storms and blast radius depend on fan-out;
- timing: several VMs on one host contend for CPU, so convergence times jitter;
- application behaviour: no retries, timeouts or overload in phase 1.

The honest claim: **functionally real, physically synthetic.** The shape of a failure is
real; the exact seconds are not.

This is precisely why the lab is valuable to FaultTown: it supplies the validation the
simulator lacks. If FaultTown predicts "r3 loses prefix X after the r1–r2 link dies", this
lab can be asked whether that actually happens.

---

## 9. Relationship to datasets

Datasets are **recordings**, not injectors. You cannot replay RCAEval to cause a fault.

What this project takes from prior work:

| Source | Taken | Not taken |
|---|---|---|
| NIKA | fault taxonomy, `(device, component, root_cause)` labels, `verify_fault` discipline, failure domains | container runtime, Kathará |
| RCAEval | `delay`/`loss` fault semantics and parameters, eval methodology, baseline ideas | pod-level faults in phase 1, its telemetry |
| Google SRE | vocabulary for cascading failure and overload | — |

What the project gives back: a **network-layer, microVM, snapshot-resettable** source of
labeled telemetry that none of the existing datasets provide.

---

## 10. Roadmap

| Phase | Scope | Exit condition |
|---|---|---|
| 0 | Preflight | Firecracker, kernel, Alpine repos, `/dev/kvm`, sudo all verified |
| 1 | Image + one VM | guest boots, serial shell, vsock exec |
| 2 | Links + two VMs | ping across one bridge/TAP link |
| 3 | 3 routers + BGP | 4 sessions Established, h1↔h3 reachable |
| 4 | Control plane | orchestrator drives guests over vsock |
| 5 | Telemetry | gap-free 60 s baseline |
| 6 | Injector + experiments | one verified, labeled experiment end-to-end |
| 7 | Snapshots + full catalog | N repetitions from identical state |
| 8 | Hybrid | services on hosts; retry/overload behaviour emerges |

Phase 8 is where the project starts to overlap with FaultTown and RCAEval. Everything
before it is about making the terrain trustworthy.

---

## 11. Risks and mitigations

| Risk | Why it matters | Mitigation |
|---|---|---|
| Kernel/format mismatch (`bzImage` vs ELF `vmlinux`) | VM will not boot | use the Firecracker CI kernel; verify in phase 0 |
| Guest NIC naming order differs from assumptions | FRR configs land on wrong interfaces | assign mgmt to `eth0`, verify enumeration in phase 1, fail loudly otherwise |
| netem direction confusion | delay applied to the wrong direction | explicit direction in the fault spec and in verification |
| vsock behaviour across snapshot restore | control plane may break after reset | spike in phase 1/2; fallback is to re-establish the session after restore |
| FRR needs kernel features absent from a minimal kernel | daemons fail to start | test FRR in phase 1, before building the full lab |
| Clock skew between host and guests | timelines lie | single time source; timestamps from the host side |
| Telemetry volume | run directories become unmanageable | fixed cadence, columnar export, retention policy |
| Scope creep into a dashboard | data work stops | UI is explicitly a non-goal until phase 7 |
| Overclaiming fidelity | the project becomes "a pretty animation" | the fidelity contract is part of the docs and the report |

---

## 12. What "done" looks like in one paragraph

A single command builds a six-node microVM network from a YAML file. BGP converges. A
second command snapshots it. A third applies a fault at a recorded time, verifies the
fault is really in effect, and records what every node did. A fourth restores the snapshot
and the lab is identical to before. Run it fifty times and you have a labeled dataset of a
real network being broken in controlled ways — the thing that neither a monitoring tool, a
simulator, nor a recording of someone else's cluster can give you.
