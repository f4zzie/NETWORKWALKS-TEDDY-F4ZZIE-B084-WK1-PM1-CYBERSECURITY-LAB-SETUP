# Cybersecurity Lab Environment Setup

**An isolated virtual lab for ethical hacking and penetration-testing practice, built with VirtualBox and Kali Linux on an Arch Linux host.**

![Domain](https://img.shields.io/badge/Domain-Cybersecurity-404040?style=flat-square)
![Host](https://img.shields.io/badge/Host-Arch%20Linux-1793D1?style=flat-square&logo=archlinux&logoColor=white)
![Hypervisor](https://img.shields.io/badge/VirtualBox-7.2.22-0070C0?style=flat-square)
![Guest](https://img.shields.io/badge/Kali%20Linux-2026.3-E87500?style=flat-square&logo=kalilinux&logoColor=white)
![Network](https://img.shields.io/badge/Network-NAT%20Network-238F89?style=flat-square)
![Program](https://img.shields.io/badge/NetworkWalks-B083-404040?style=flat-square)

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Objectives](#objectives)
3. [Scope and Ethical Use](#scope-and-ethical-use)
4. [Lab Architecture](#lab-architecture)
5. [Lab Configuration](#lab-configuration)
6. [Setup Procedure](#setup-procedure)
7. [Lab Verification](#lab-verification)
8. [Security Considerations](#security-considerations)
9. [Problems Encountered and Solutions](#problems-encountered-and-solutions)
10. [Common Issues on an Arch Linux Host](#common-issues-on-an-arch-linux-host)
11. [What I Learned](#what-i-learned)
12. [Next Steps](#next-steps)
13. [Repository Contents](#repository-contents)
14. [Tools and Resources](#tools-and-resources)
15. [Author](#author)

---

## Project Overview

This project documents the construction of a virtual cybersecurity laboratory on my own computer. The lab uses VirtualBox as the hypervisor and Kali Linux as the attacker machine, connected through a dedicated private virtual network.

The aim is to have a controlled, repeatable environment where security tools can be run safely. Reconnaissance, scanning, vulnerability assessment and later exploitation exercises all happen inside the lab, and no real or third-party systems are involved. Because the lab sits on its own virtual network, additional machines (for example deliberately vulnerable targets) can be added in later projects without rebuilding anything.

The host machine runs Arch Linux rather than Windows, which means the hypervisor installation involves kernel modules and group membership that a Windows installer would normally handle automatically. Those host-specific steps are documented below.

---

## Objectives

- Prepare an Arch Linux host for virtualization and install VirtualBox.
- Download, verify and extract the official Kali Linux virtual machine image.
- Create a dedicated NAT Network for the laboratory.
- Import Kali Linux and attach it to the NAT Network.
- Assign Kali a consistent static IPv4 address.
- Verify connectivity, DNS resolution and tool availability.
- Create a clean snapshot to use as a recovery baseline.
- Document the full process with commands, screenshots and lessons learned.

---

## Scope and Ethical Use

This laboratory is built for education and authorized security testing only.

- Only systems that I own or have explicit written permission to test may be targeted.
- All testing stays inside the isolated virtual network.
- Tools installed in this lab must never be pointed at third-party or public systems without authorization.

---

## Lab Architecture

<!-- >>> SCREENSHOT 1 (architecture diagram) GOES HERE <<<
     Draw a simple diagram in draw.io or Excalidraw and export as PNG.
     Show: Arch Linux host -> VirtualBox -> NAT Network "NatNetwork" 10.0.0.0/24 -> Kali Linux VM 10.0.0.2.
     Add a dashed box labelled "Future target VMs" on the same network, and an arrow from the NAT Network to the Internet.
     Save as: 1-screenshot-title-image.png -->
![Lab architecture diagram](1-screenshot-title-image.png)

The diagram above shows the layout of the lab. The same relationships in text form:

```
                    Internet
                        |
                  (NAT translation by VirtualBox)
                        |
        +-------------------------------+
        |  NAT Network: NatNetwork      |
        |  10.0.0.0/24, gateway 10.0.0.1|
        +---------------+---------------+
                        |
              +---------+---------+
              |  Kali Linux VM    |      Future target VMs
              |  10.0.0.2/24      |      (added in later projects)
              +-------------------+

   Host: Arch Linux + VirtualBox 7.2.22
```

Machines attached to the same NAT Network can reach each other and also reach the internet through address translation. They are not directly reachable from the physical LAN unless port forwarding is configured, which keeps the lab contained.

---

## Lab Configuration

### Host and software

| Component | Configuration |
|---|---|
| Host operating system | Arch Linux |
| Processor | Intel Core i7-6600U @ 2.60 GHz |
| Host RAM | 16 GB |
| Free disk space at start | approximately 100 GB |
| Hypervisor | VirtualBox 7.2.22 |
| Security operating system | Kali Linux 2026.3 (rolling) |
| Kali RAM | 2048 MB |
| Kali CPUs | 2 |

### Network

| Setting | Value |
|---|---|
| Virtual network type | NAT Network |
| Network name | NatNetwork |
| Network address | 10.0.0.0/24 |
| DHCP | Enabled |
| IPv6 | Disabled |
| Default gateway | 10.0.0.1 |
| Kali IP address | 10.0.0.2/24 (static) |
| DNS server | 8.8.8.8 |
| Kali network adapter | Intel PRO/1000 MT Desktop (82540EM), attached to NatNetwork |

---

## Setup Procedure

### Step 1. Prepare the host and install VirtualBox

Hardware virtualization must be available and enabled in the firmware. I checked this, along with the running kernel, before installing anything:

```bash
lscpu | grep -i -E "model name|virtualization"
uname -r
free -h
```

On Arch Linux, VirtualBox needs kernel modules that match the running kernel. For the default `linux` kernel:

```bash
sudo pacman -S virtualbox virtualbox-host-modules-arch
```

For other kernels (LTS, Zen, Hardened), the DKMS package and the matching headers are required instead:

```bash
sudo pacman -S virtualbox virtualbox-host-dkms linux-lts-headers
```

The kernel module was then loaded and my user was added to the `vboxusers` group, followed by a logout and login so the group change took effect:

```bash
sudo modprobe vboxdrv
sudo usermod -aG vboxusers $USER
```

Verification of the installation:

```bash
vboxmanage --version
lsmod | grep vbox
groups
```

Result: VirtualBox 7.2.22 reported its version, the `vboxdrv`, `vboxnetflt` and `vboxnetadp` modules were loaded, and `vboxusers` appeared in my group list.

<!-- >>> OPTIONAL SCREENSHOT 5 GOES HERE <<<
     Screenshot of the Arch terminal showing the output of: vboxmanage --version, lsmod | grep vbox, groups
     Save as: 5-screenshot-virtualbox-install.png
     (If you skip it, delete the next line.) -->
![VirtualBox installed on the Arch Linux host](5-screenshot-virtualbox-install.png)

### Step 2. Download, verify and extract the Kali Linux image

The pre-built VirtualBox image was downloaded from the official Kali Linux website (Virtual Machines section, VirtualBox 64-bit). Because this is a security distribution, I verified the integrity of the download against the SHA256 checksum published on the same page:

```bash
cd ~/Downloads
sha256sum kali-linux-*-virtualbox-amd64.7z
```

The computed hash matched the published value. The archive uses the 7z format, so an extractor was installed and the image unpacked:

```bash
sudo pacman -S 7zip
7z x kali-linux-*-virtualbox-amd64.7z
```

The extracted folder contains the `.vbox` machine definition and the `.vdi` virtual disk.

### Step 3. Create the NAT Network

A dedicated NAT Network was created before importing the VM so that the VM could be attached to it straight away:

```bash
VBoxManage natnetwork add --netname NatNetwork --network "10.0.0.0/24" --enable --dhcp on
VBoxManage list natnetworks
```

The same settings can be reviewed in the VirtualBox GUI under the Network manager, on the NAT Networks tab.

<!-- >>> SCREENSHOT 2 GOES HERE <<<
     VirtualBox GUI -> Tools -> Network -> "NAT Networks" tab -> click NatNetwork.
     The name, the prefix 10.0.0.0/24 and the DHCP checkbox must all be visible.
     Save as: 2-screenshot-network-settings-1.png -->
![VirtualBox NAT Network settings](2-screenshot-network-settings-1.png)

A NAT Network was chosen instead of plain NAT for an important reason. Virtual machines on plain NAT are isolated from each other and can only reach the internet. Virtual machines attached to the same NAT Network can communicate with one another and still reach the internet through address translation. This is what makes the lab extensible: attacker and target machines can be placed on the same network later.

### Step 4. Import Kali Linux and attach it to the NAT Network

Using the VirtualBox GUI:

1. Machine, then Add, and select the extracted `.vbox` file.
2. Open the VM Settings and configure:
   - System, Motherboard: Base Memory set to 2048 MB.
   - System, Processor: 2 CPUs.
   - Network, Adapter 1: Enable Network Adapter, Attached to NAT Network, Name NatNetwork, and under Advanced the adapter type Intel PRO/1000 MT Desktop (82540EM).
3. Start the VM and sign in with the default credentials published on the Kali download page.

<!-- >>> SCREENSHOT 3 GOES HERE <<<
     The Kali desktop running, with the VirtualBox window visible
     (ideally the window title showing the VM name and "Running").
     Take it from the Arch host so the VirtualBox frame is included.
     Save as: 3-screenshot-kali-linux.png -->
![Kali Linux running in VirtualBox](3-screenshot-kali-linux.png)

### Step 5. Configure the Kali network

The address obtained by DHCP was checked first to confirm the NAT Network was working, then a fixed address was configured with NetworkManager. A consistent address makes the lab easier to document and to reference in later exercises.

```bash
ip a
ip route
nmcli connection show
```

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns 8.8.8.8
sudo nmcli connection up "Wired connection 1"
```

The connection name (`Wired connection 1`) can differ between systems, so it should be confirmed with `nmcli connection show` before running the commands above.

<!-- >>> SCREENSHOT 4 GOES HERE <<<
     Terminal inside Kali showing the output of both: ip a   and   ip route
     The address 10.0.0.2/24 and the default route via 10.0.0.1 must both be visible.
     Save as: 4-screenshot-kali-network-settings.png -->
![Kali network configuration](4-screenshot-kali-network-settings.png)

### Step 6. Create a clean snapshot

After the configuration was complete and verified, the VM was shut down cleanly (`sudo poweroff`) and a snapshot was taken in VirtualBox:

- Snapshot name: `Clean Kali - Network Setup`
- Description: Kali with static IP, NAT Network, internet and DNS verified.

The snapshot is the known-good baseline of the laboratory. If a later exercise breaks or contaminates the VM, it can be restored to this state in seconds.

<!-- >>> OPTIONAL SCREENSHOT 6 GOES HERE <<<
     VirtualBox -> Kali VM -> Snapshots tab showing "Clean Kali - Network Setup".
     Save as: 6-screenshot-snapshot.png
     (If you skip it, delete the next line.) -->
![Clean baseline snapshot](6-screenshot-snapshot.png)

---

## Lab Verification

Each part of the lab was tested after configuration.

| Test | Command | Expected result |
|---|---|---|
| Check IP address | `ip a` | 10.0.0.2/24 on the Kali interface |
| Check routing | `ip route` | Default route via 10.0.0.1 |
| Test gateway | `ping -c 3 10.0.0.1` | Replies received, 0% packet loss |
| Test internet access | `ping -c 3 8.8.8.8` | Replies received, 0% packet loss |
| Test DNS resolution | `nslookup networkwalks.com` | Domain resolves to an address |
| Verify security tooling | `nmap --version` | Nmap version displayed |
| Verify snapshot | Make a test change, restore the snapshot, run `ls` and `ip a` | Test change gone, baseline configuration intact |

### Actual results

<!-- >>> PASTE YOUR REAL OUTPUT BELOW <<<
     Replace each [REPLACE ...] with the actual text printed in your Kali terminal.
     Do not invent output. Copy exactly what your commands printed. -->

Gateway test:

```
[REPLACE: paste the output of  ping -c 3 10.0.0.1]
```

Internet test:

```
[REPLACE: paste the output of  ping -c 3 8.8.8.8]
```

DNS test:

```
[REPLACE: paste the output of  nslookup networkwalks.com]
```

Tool check:

```
[REPLACE: paste the output of  nmap --version]
```

Snapshot restore test: [REPLACE: one sentence on what you changed, that the restore removed it, and that the IP configuration was still correct afterwards.]

<!-- >>> OPTIONAL SCREENSHOT 7 GOES HERE <<<
     One terminal screenshot in Kali showing the ping and nslookup results together.
     Save as: 7-screenshot-verification.png
     (If you skip it, delete the next line.) -->
![Connectivity verification inside Kali](7-screenshot-verification.png)

---

## Security Considerations

- **Isolation:** the lab machines sit behind VirtualBox NAT on a private network. Nothing on the physical LAN can reach them unless port forwarding is added deliberately, and none has been configured.
- **Snapshots as a safety net:** a clean baseline allows risky experiments without the fear of permanently breaking the environment.
- **Integrity of downloads:** the Kali image was checked against the published SHA256 checksum before use.
- **Default credentials:** the image ships with well-known default credentials. They are acceptable only because the VM is on an isolated private network. They should be changed before the VM is exposed to any other network or used for anything beyond local practice.
- **Authorization:** all later testing must remain restricted to machines inside this lab or systems I have explicit permission to test.

---

## Problems Encountered and Solutions

<!-- >>> WRITE ONLY REAL PROBLEMS YOU ACTUALLY HIT <<<
     Copy this block once per problem. Use the exact error message you saw.
     If you genuinely had no problems, delete the blocks below and use the "no issues" sentence
     at the bottom of this section instead. Never describe a problem that did not happen to you.
     Whatever you write here, post the same text as a comment on the week's YouTube project video. -->

### Problem 1. [REPLACE: short title]

**Symptom:** [REPLACE: what happened, including the exact error message]

**Cause:** [REPLACE: what the actual cause turned out to be]

**Solution:** [REPLACE: the exact steps or commands that fixed it]

```bash
[REPLACE: commands used, if any]
```

### Problem 2. [REPLACE or delete this block]

**Symptom:** [REPLACE]

**Cause:** [REPLACE]

**Solution:** [REPLACE]

> If no problems occurred, replace this section's contents with: "No blocking issues were encountered during this project. The Arch Linux specific points that required care (kernel module package, `vboxusers` group membership and re-login) are listed in the next section."

---

## Common Issues on an Arch Linux Host

This section is a reference for others building the same lab on Arch Linux. It lists issues that commonly occur with this setup, and is separate from the problems I personally encountered above.

| Symptom | Likely cause | Fix |
|---|---|---|
| VM fails to start with a kernel driver (`vboxdrv`) error | Kernel module not loaded or does not match the running kernel | `sudo modprobe vboxdrv`; install the host modules package matching your kernel, or use the DKMS package with headers |
| VirtualBox cannot access USB devices or shared features | User not in the `vboxusers` group | `sudo usermod -aG vboxusers $USER`, then log out and back in |
| VM fails with a VT-x or hardware virtualization error | Virtualization disabled in BIOS/UEFI | Enable Intel VT-x in firmware settings, save and reboot |
| Network adapter dropdown shows no NAT Network | NAT Network not created or not enabled | Re-create it with `VBoxManage natnetwork add ...` and check `VBoxManage list natnetworks` |
| No internet or DNS after setting a static IP on Kali | NetworkManager settings or address conflict detection delay | Check `cat /etc/resolv.conf`; confirm gateway and DNS values; if needed `sudo nmcli connection modify "Wired connection 1" ipv4.dad-timeout 0` and bring the connection up again |

---

## What I Learned

**1. NAT versus NAT Network.** Plain NAT gives each VM its own isolated path to the internet, so VMs cannot see each other. A NAT Network places multiple VMs on one shared private subnet where they can communicate and still share outbound internet access. This is the right design for a multi-machine lab.

**2. How a hypervisor integrates with a Linux host.** On Arch Linux, VirtualBox depends on kernel modules that must match the running kernel, and a user must belong to the `vboxusers` group. Understanding these two requirements explains most host-side failures.

**3. Static addressing with NetworkManager.** I configured IPv4 address, prefix, gateway and DNS with `nmcli`, and learned to verify each layer separately: address with `ip a`, routing with `ip route`, reachability with `ping`, and name resolution with `nslookup`.

**4. Verifying downloads.** Checking the SHA256 checksum of a security tool before running it is a basic but important habit, because a tampered image would compromise the whole lab.

**5. Snapshots as a workflow tool.** A clean snapshot taken after verification gives a fast recovery point and encourages experimentation, since mistakes are cheap to undo.

**6. Documentation as part of the work.** Recording commands, results and problems makes the lab reproducible by others and by my future self.

---

## Next Steps

- Add deliberately vulnerable target machines to NatNetwork and record their addressing.
- Practise reconnaissance and scanning workflows from Kali against those targets.
- Take additional snapshots before and after each major exercise.
- Extend the lab with additional tooling and write-ups as the program progresses.

---

## Repository Contents

| File | Description |
|---|---|
| `README.md` | Full documentation of the lab build |
| `1-screenshot-title-image.png` | Lab architecture diagram |
| `2-screenshot-network-settings-1.png` | VirtualBox NAT Network settings |
| `3-screenshot-kali-linux.png` | Kali Linux running in VirtualBox |
| `4-screenshot-kali-network-settings.png` | Kali IP address and routing configuration |
| `5-screenshot-virtualbox-install.png` | VirtualBox installation check on the host (optional) |
| `6-screenshot-snapshot.png` | Clean baseline snapshot (optional) |
| `7-screenshot-verification.png` | Connectivity verification output (optional) |

---

## Tools and Resources

- VirtualBox: https://www.virtualbox.org/wiki/Downloads
- Kali Linux: https://www.kali.org/get-kali/
- 7-Zip: https://7-zip.org/download.html
- Arch Linux VirtualBox documentation: https://wiki.archlinux.org/title/VirtualBox

---

## Author

**Teddy (F4ZZIE)**
Cybersecurity learner, NetworkWalks Batch B083
GitHub: https://github.com/f4zzie

**Program:** Cybersecurity at NetworkWalks | **Batch:** B083 | **Week:** 01 | **Project:** PM1, Cybersecurity Lab Setup | **Completed:** October 27, 2026
