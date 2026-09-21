# Mininet Topology Experiment

## 1. Aim

To create and test different network topologies in **Mininet 2.3.0** (`minimal`, `single`, `linear`, `tree`, `reversed`), and to observe how they differ in number of hosts, switches, links, interface/port mapping, and connectivity (`pingall`).

## 2. Environment

| Item | Value |
|---|---|
| Emulator | Mininet |
| Version | 2.3.0 (`sudo mn --version`) |
| OS / user | Ubuntu Linux, terminal user `souvik@linux` |
| Switch type | Open vSwitch (OVS Bridge) |
| Controller | None. Mininet printed *"No default OpenFlow controller found for default switch! Falling back to OVS Bridge"*, so the switch works as a normal learning switch |

## 3. Commands used

| # | Command | Purpose |
|---|---|---|
| 1 | `sudo mn --version` | Check installed Mininet version |
| 2 | `sudo mn --test pingall` | Run the default topology, ping test, exit automatically |
| 3 | `sudo mn --topo minimal` | Default topology in interactive CLI |
| 4 | `sudo mn --topo single,3` / `single,4` | One switch with 3 / 4 hosts |
| 5 | `sudo mn --topo linear,3` / `linear,4` | Chain of 3 / 4 switches, one host each |
| 6 | `sudo mn --topo tree,2` / `tree,3` | Tree of depth 2 / 3 (fanout 2) |
| 7 | `sudo mn --topo reversed,3` / `reversed,4` | Single switch, ports assigned in reverse order |
| 8 | `sudo mn -c` | Clean up leftover Mininet processes and interfaces |

Inside the Mininet CLI, two commands were used for every topology:

- `links` shows which interfaces are connected and their status
- `pingall` tests reachability between every pair of hosts

Then `exit` was used to stop the network.

---

## 4. Experiments and observations

### 4.1 Default test: `sudo mn --test pingall` (Screenshot 1)

- Topology created: **minimal** (default), i.e. `h1 h2` connected to `s1`
- Links: `(h1, s1) (h2, s1)`
- Ping result: **0% dropped (2/2 received)**
- Ran non-interactively and finished by itself in **0.877 seconds**

### 4.2 Minimal: `--topo minimal` (Screenshot 2)

```
h1 ── s1 ── h2
```

- Hosts: 2 (`h1 h2`), Switches: 1 (`s1`), Links: 2
- `links` output:
  - `h1-eth0<->s1-eth1`
  - `h2-eth0<->s1-eth2`
- `pingall`: **0% dropped (2/2 received)**
- Total time: 16.985 s (includes the time spent in the CLI)
- After exiting, `sudo mn -c` was run to clean up old controllers, datapaths and processes

### 4.3 Single: `--topo single,3` and `single,4` (Screenshots 3, 4)

```
single,3                 single,4
h1 ─┐                    h1 ─┐
h2 ─┼─ s1                h2 ─┼─ s1
h3 ─┘                    h3 ─┤
                         h4 ─┘
```

| | `single,3` | `single,4` |
|---|---|---|
| Hosts | h1 h2 h3 | h1 h2 h3 h4 |
| Switches | s1 | s1 |
| Links | 3 | 4 |
| Port mapping | h1→s1-eth1, h2→s1-eth2, h3→s1-eth3 | h1→s1-eth1 … h4→s1-eth4 |
| `pingall` | 0% dropped (6/6) | 0% dropped (12/12) |

Every host connects directly to the same switch, so all hosts are one hop apart. Increasing the number only adds hosts and links; the switch count stays 1.

### 4.4 Linear: `--topo linear,3` and `linear,4` (Screenshots 5, 6)

```
linear,3:  h1─s1─s2─s3─h3
                 │
                 h2

linear,4:  h1─s1─s2─s3─s4─h4
                 │   │
                 h2  h3
```

Each switch has exactly one host, and the switches are chained together.

| | `linear,3` | `linear,4` |
|---|---|---|
| Hosts | 3 | 4 |
| Switches | 3 (s1 s2 s3) | 4 (s1 s2 s3 s4) |
| Links | 5 | 7 |
| Host links | h1-s1, h2-s2, h3-s3 | h1-s1, h2-s2, h3-s3, h4-s4 |
| Switch-switch links | s2-s1, s3-s2 | s2-s1, s3-s2, s4-s3 |
| `pingall` | 0% dropped (6/6) | 0% dropped (12/12) |

Interface mapping:

- Every host uses `eth0` connected to `eth1` of its switch (e.g. `h2-eth0<->s2-eth1`)
- Inter-switch links use the following ports: `s2-eth2<->s1-eth2`, `s3-eth2<->s2-eth3` (and `s4-eth2<->s3-eth3` in `linear,4`)
- Middle switches (s2, s3) have 3 ports, end switches have 2

Unlike `single`, traffic between distant hosts (e.g. h1 to h4) passes through several switches, so the path length grows with the topology size.

### 4.5 Tree: `--topo tree,2` and `tree,3` (Screenshots 7, 8)

The number after `tree` is the **depth**; the default fanout is 2.

**`tree,2`** (depth 2)

```
s1
├── s2
│   ├── h1
│   └── h2
└── s3
    ├── h3
    └── h4
```

**`tree,3`** (depth 3)

```
s1
├── s2
│   ├── s3
│   │   ├── h1
│   │   └── h2
│   └── s4
│       ├── h3
│       └── h4
└── s5
    ├── s6
    │   ├── h5
    │   └── h6
    └── s7
        ├── h7
        └── h8
```

| | `tree,2` | `tree,3` |
|---|---|---|
| Hosts | 4 | 8 |
| Switches | 3 (s1 s2 s3) | 7 (s1 to s7) |
| Links | 6 | 14 |
| Hosts attached to | s2, s3 | s3, s4, s6, s7 (the leaf switches) |
| `pingall` | 0% dropped (12/12) | Output cut off in screenshot, but every host was listed reaching all 7 others |

Key points:

- Hosts attach only to the **leaf** switches; the upper switches (s1, s2, s5) only connect other switches
- Example mapping in `tree,2`: `s1-eth1<->s2-eth3`, `s1-eth2<->s3-eth3`, `s2-eth1<->h1-eth0`, `s2-eth2<->h2-eth0`
- In a tree, the parent switch uses the last port (`eth3`) of the child to connect upward, while the child's first ports face hosts/children

### 4.6 Reversed: `--topo reversed,3` and `reversed,4` (Screenshots 9, 10)

`reversed` has the same shape as `single` (one switch, n hosts, n links) but the **switch ports are numbered in the opposite order**.

| Host | `single,3` port | `reversed,3` port | `single,4` port | `reversed,4` port |
|---|---|---|---|---|
| h1 | s1-eth1 | **s1-eth3** | s1-eth1 | **s1-eth4** |
| h2 | s1-eth2 | s1-eth2 | s1-eth2 | **s1-eth3** |
| h3 | s1-eth3 | **s1-eth1** | s1-eth3 | **s1-eth2** |
| h4 | none | none | s1-eth4 | **s1-eth1** |

- `reversed,3`: `pingall` **0% dropped (6/6)**
- `reversed,4`: `pingall` **0% dropped (12/12)**
- Connectivity is identical to `single`; only the port numbers differ. This matters when writing OpenFlow rules that refer to specific port numbers.

---

## 5. Comparison of all topologies

| Command | Hosts | Switches | Links | pingall result | Shape |
|---|---|---|---|---|---|
| `minimal` | 2 | 1 | 2 | 0% drop (2/2) | h1 and h2 on one switch |
| `single,3` | 3 | 1 | 3 | 0% drop (6/6) | Star with one switch |
| `single,4` | 4 | 1 | 4 | 0% drop (12/12) | Star with one switch |
| `linear,3` | 3 | 3 | 5 | 0% drop (6/6) | Chain, one host per switch |
| `linear,4` | 4 | 4 | 7 | 0% drop (12/12) | Chain, one host per switch |
| `tree,2` | 4 | 3 | 6 | 0% drop (12/12) | Binary tree, depth 2 |
| `tree,3` | 8 | 7 | 14 | Not visible in screenshot | Binary tree, depth 3 |
| `reversed,3` | 3 | 1 | 3 | 0% drop (6/6) | Star, ports reversed |
| `reversed,4` | 4 | 1 | 4 | 0% drop (12/12) | Star, ports reversed |

### Formulas that match the observations

For `n` hosts, `pingall` sends **n × (n − 1)** pings: 2 hosts gives 2, 3 gives 6, 4 gives 12, and 8 would give 56.

| Topology | Hosts | Switches | Links |
|---|---|---|---|
| `single,n` / `reversed,n` | n | 1 | n |
| `linear,n` | n | n | 2n − 1 |
| `tree,d` (fanout 2) | 2^d | 2^d − 1 | 2^(d+1) − 2 |

## 6. Key differences at a glance

- **single vs reversed**: same structure; only the switch port numbering is flipped.
- **single vs linear**: `single` uses one switch for all hosts; `linear` uses one switch per host, so it needs more switches and links, and packets can cross several switches.
- **linear vs tree**: `linear` is a chain (long paths, only two ends), while `tree` is hierarchical (hosts on leaf switches only, shorter paths, more hosts for the same depth).
- **Scaling**: `tree` grows fastest (doubling hosts and switches with each level), `linear` grows by one switch and host at a time, and `single` only adds hosts.

## 7. Notes

- Every topology showed **0% packet loss**, so all hosts could reach each other in every case. This works without an external controller because Mininet fell back to the OVS bridge, which acts as a normal L2 switch.
- The "completed in … seconds" values (0.877 s, 16.985 s, 65.386 s) mostly reflect how long the CLI stayed open before `exit`, so they should not be used to compare topology performance. Only `--test pingall` (0.877 s) is a real measure, since it runs without waiting for user input.
- Screenshots 4, 6 and 7 are cut off at the bottom, so the final timing lines and the full `tree,3` ping results are not visible.
- `sudo mn -c` was used to clear leftover state after the runs.

## 8. Conclusion

Different Mininet topologies were built and verified using `links` and `pingall`. Each one connected all hosts with 0% loss, but they differ in the number of switches and links, how ports are numbered, and how many hops packets take. `single` and `reversed` are the simplest, `linear` shows a chained multi-switch layout, and `tree` shows a hierarchical, data-centre-style layout that scales quickly with depth.
