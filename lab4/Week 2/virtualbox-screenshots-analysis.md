# VirtualBox Lab: Screenshot Analysis & Differences

Two Ubuntu VMs (`ubuntu` and `ubuntu Clone`) running side by side in Oracle VirtualBox, attached to a NAT Network called **Dummy**. This document describes what each screenshot shows and how they differ from one another.

---

## 1. Summary Table

| # | Screenshot | Time | What it shows |
|---|-----------|------|---------------|
| 1 | VirtualBox Manager – Resource Use | n/a | Both VMs running; CPU, RAM, Network, Disk I/O graphs |
| 2 | Both VM consoles | 17:02 | Ubuntu login screen (user `vboxuser`) |
| 3 | Both VM terminals | 17:04 | `ip a` output on both VMs |
| 4 | VirtualBox Network Manager | n/a | NAT Network **Dummy** configuration |
| 5 | `ubuntu - Settings` dialog | 17:11 | Adapter 1 attached to NAT Network `Dummy` |
| 6 | Both VM terminals | 17:19 | `ping 10.0.2.15` on both VMs |

---

## 2. Screenshot-by-Screenshot Details

### Screenshot 1: VirtualBox Manager, Resource Use tab

- **VM list:** `ubuntu` (selected) and `ubuntu Clone`, both **Running**.
- **CPU Load:** Guest Load 0%, VMM Load 95% (tooltip shows `0% / 99%`). The guest is idle, but the hypervisor is using a lot of host CPU.
- **RAM Usage:** shows *"This metric requires guest additions to work"*. Total, Free and Used are all `--`, so Guest Additions are **not installed** in the VM.
- **Network Rate:** Download 0 B, Upload 0 B (current). Total Downloaded 17.24 KB, Total Uploaded 22.67 KB.
- **Disk IO:** Write 0 B, Read 0 B (current). Total Written 27.35 MB, Total Read 1.14 GB.

### Screenshot 2: Login screen (17:02)

- Both VM windows show the Ubuntu GDM login screen with a single user, **vboxuser**.
- Both clocks read **Aug 22, 17:02**.
- The two windows are identical in state. Only the window size and position differ slightly.

### Screenshot 3: `ip a` on both VMs (17:04)

Both users are logged in and have run `ip a`. The prompt on both is `vboxuser@ubuntu:~$`, so the **hostname is also identical**.

| Field | `ubuntu` (left) | `ubuntu Clone` (right) |
|-------|-----------------|------------------------|
| Interface | `enp0s3` | `enp0s3` |
| MAC (`link/ether`) | `08:00:27:8c:ff:51` | `08:00:27:8c:ff:51` |
| IPv4 | `10.0.2.15/24` | `10.0.2.15/24` |
| Broadcast | `10.0.2.255` | `10.0.2.255` |
| `valid_lft` | 430 sec | 425 sec |
| IPv6 link-local (`fe80::…`) | **not shown** | `fe80::a00:27ff:fe8c:ff51/64` |
| Loopback | `127.0.0.1`, `::1` | `127.0.0.1`, `::1` |

**Key differences / findings**

- The **MAC address and IPv4 address are identical on both VMs**. The clone was made without generating a new MAC address, so the two VMs are effectively duplicates on the network.
- The clone shows an extra `inet6 fe80::…` line. The original does not (probably a timing difference in when IPv6 autoconfiguration finished).
- The `valid_lft` difference (430 vs 425 sec) is just the few seconds between the two `ip a` commands.

### Screenshot 4: VirtualBox Network Manager, NAT Networks tab

| Setting | Value |
|---------|-------|
| Network name | `Dummy` |
| IPv4 prefix | `10.0.2.0/24` |
| IPv6 prefix | `fd17:625c:f037:2::/64` (IPv6 **disabled**, checkbox unchecked) |
| DHCP server | **Enabled** |
| Port Forwarding | Tab available (no rules visible) |

Note that `10.0.2.0/24` is also the subnet used by VirtualBox's plain **NAT** mode, where every VM gets `10.0.2.15`. This makes the two setups easy to confuse.

### Screenshot 5: `ubuntu - Settings` → Network (17:11)

| Setting | Value |
|---------|-------|
| Adapter 1 | Enabled |
| Attached to | **NAT Network** |
| Name | **Dummy** |
| Adapter Type | Intel PRO/1000 MT Desktop (82540EM) |
| Promiscuous Mode | Deny |
| MAC Address | `0800278CFF51` (greyed out because the VM is running) |
| Virtual Cable Connected | Checked |

- The MAC here matches `08:00:27:8c:ff:51` from Screenshot 3, which confirms the duplicate MAC.
- In the background, the terminal on the original VM still shows the old `ip a` output.
- The clone window (behind the dialog) now shows the desktop top bar with network, sound and battery icons.

### Screenshot 6: `ping 10.0.2.15` on both VMs (17:19)

| Field | `ubuntu` (left) | `ubuntu Clone` (right) |
|-------|-----------------|------------------------|
| Target | `10.0.2.15` | `10.0.2.15` |
| Packets sent / received | 7 / 7 | 6 / 6 |
| Packet loss | 0% | 0% |
| Reply time range | 0.054 – 0.090 ms | 0.062 – 0.089 ms |
| rtt min/avg/max/mdev | 0.054 / 0.065 / 0.090 / 0.010 ms | 0.062 / 0.069 / 0.089 / 0.009 ms |
| Total time | 14384 ms | 5192 ms |
| TTL | 64 | 64 |

**Key finding:** Each VM pinged `10.0.2.15`, which is **its own IP address**. The sub-millisecond times and TTL of 64 show the replies come from the local machine itself, so this ping does **not** prove the two VMs can talk to each other. Because both VMs share the same IP, neither can reach the other one by this address.

Also, the `ip a` output visible at the top of each terminal is the old output from 17:04 that has scrolled up, so the `valid_lft 430sec` values are stale.

---

## 3. Differences Across Screenshots (Timeline)

| Aspect | 1 | 2 (17:02) | 3 (17:04) | 4 | 5 (17:11) | 6 (17:19) |
|--------|---|-----------|-----------|---|-----------|-----------|
| Focus | Host manager | Guest login | Guest terminal | Host network config | Guest settings | Guest terminal |
| Guest state | Running | Login screen | Logged in | n/a | Logged in | Logged in |
| Activity | Monitoring | None | `ip a` | Config view | Settings dialog | `ping` |
| Shows IP info | No | No | Yes | Subnet only | MAC only | Yes (own IP) |
| Evidence of problem | Guest Additions missing | n/a | Same MAC & IP | Same subnet as NAT | MAC matches | Self-ping only |

---

## 4. Differences Between the Two VMs (Overall)

**Identical**
- Hostname (`ubuntu`), username (`vboxuser`), interface name (`enp0s3`)
- MAC address `08:00:27:8c:ff:51`
- IPv4 address `10.0.2.15/24`
- Adapter type and network (NAT Network `Dummy`)

**Different**
- IPv6 link-local address shown only on the clone (Screenshot 3)
- Lease timers (430 vs 425 sec), which is just timing
- Ping counts and durations (7 packets / 14.4 s vs 6 packets / 5.2 s)
- Window size and position on screen

---

## 5. Conclusions & Suggested Fixes

1. **Duplicate MAC and IP.** The clone was created without a new MAC address. On a NAT Network with DHCP, two identical MACs will receive the same lease, so the VMs conflict.
   - Fix: power off the clone → **Settings → Network → Advanced** → click the refresh icon next to **MAC Address**. (Or re-clone using *"Generate new MAC addresses for all network adapters"*.)
   - Then inside the guest, renew DHCP (`sudo dhclient -r && sudo dhclient`, or reboot) so it gets a new IP.
2. **Test connectivity properly.** After fixing, ping the **other** VM's IP (for example from `ubuntu` ping the clone's new address), not `10.0.2.15`.
3. **Install Guest Additions** to get RAM metrics in Resource Use and better integration.
4. **Optional:** change the `Dummy` NAT Network to a different subnet (for example `10.0.3.0/24`) to avoid confusion with VirtualBox's default NAT range.
