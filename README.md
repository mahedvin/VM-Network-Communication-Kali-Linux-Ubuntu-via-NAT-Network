# 🖧 VM Network Communication — Kali Linux & Ubuntu via NAT Network

> Setting up and troubleshooting inter-VM communication using a NAT Network in VirtualBox.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Environment & Tools](#environment--tools)
- [Network Setup](#network-setup)
- [Configuration Steps](#configuration-steps)
- [Troubleshooting](#troubleshooting)
- [Verification](#verification)
- [Lessons Learned](#lessons-learned)

---

## Project Overview

The goal of this project was to establish network communication between two virtual machines — **Kali Linux** and **Ubuntu** — running inside VirtualBox. This was achieved by placing both VMs on the same **NAT Network**, allowing them to communicate with each other while sharing the host's internet connection.

---

## Environment & Tools

| Component        | Details                        |
|------------------|-------------------------------|
| Hypervisor       | Oracle VirtualBox              |
| VM 1             | Kali Linux                     |
| VM 2             | Ubuntu                         |
| Network Type     | NAT Network                    |
| Tools Used       | `ifconfig`, `ip`, `dhcpcd`, `netplan` |

---

## Network Setup

### What is a NAT Network?

A **NAT Network** in VirtualBox allows multiple VMs to:
- Communicate with **each other**
- Share the **host machine's internet connection**
- Remain **isolated** from the external network

This is different from a standard **NAT** adapter, which gives each VM its own isolated network and prevents VM-to-VM communication.

### Creating the NAT Network in VirtualBox

1. Open VirtualBox and go to **File → Tools → Network Manager**
2. Click the **NAT Networks** tab
3. Click **Create** to add a new NAT Network
4. Make sure **Enable DHCP** is checked
5. Note the network name (e.g., `NatNetwork`)

### Assigning Both VMs to the NAT Network

For **each VM**:
1. Go to **VM Settings → Network**
2. Set **Attached to:** `NAT Network`
3. Set **Name:** to the same NAT Network name on both VMs
4. Click **OK** and start the VMs

---

## Configuration Steps

### Verifying the Netplan Config (Ubuntu)

After booting Ubuntu, verify that DHCP is enabled for the network interface:

```bash
sudo cat /etc/netplan/*.yaml
```

Expected output should include:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:        # your interface name may vary
      dhcp4: true
```

If the interface name is wrong or `dhcp4` is not set to `true`, edit the file:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Then apply the changes:

```bash
sudo netplan apply
```

---

## Troubleshooting

### Problem: One VM Not Showing an IP Address

After setting up the NAT Network and booting both VMs, running `ifconfig` on the Ubuntu VM showed no `inet` (IP address) section for the network interface.

**Symptoms:**
- `ifconfig` showed the interface but no IP address
- No `inet` entry visible under the interface

---

### Diagnosis Steps

**Step 1 — Check all interfaces and their status:**
```bash
ip link show
```
Look for interfaces listed as `DOWN` or missing an IP.

**Step 2 — Verify DHCP is configured in Netplan:**
```bash
sudo cat /etc/netplan/*.yaml
```
Confirmed `dhcp4: true` was set — so the config was correct.

**Step 3 — Try to manually request an IP using `dhclient`:**
```bash
sudo dhclient <interface-name>
```
**Result:** `sudo: dhclient: command not found`

This is expected on newer versions of Ubuntu, which no longer include `dhclient` by default.

---

### Root Cause

Even though DHCP was enabled in the Netplan config, the DHCP client service was **not automatically triggered** when the VM connected to the NAT Network. The interface was up, but it had never actually sent a DHCP request to the NAT Network's DHCP server to obtain an IP address.

---

### Fix — Manually Trigger DHCP with `dhcpcd`

Since `dhclient` was not available, `dhcpcd` was used instead:

```bash
sudo dhcpcd <interface-name>
```

> Replace `<interface-name>` with your actual interface (e.g., `enp0s3`). Find it by running `ip link show`.

**Why this worked:** `dhcpcd` is a DHCP client daemon that manually sends a DHCP request to the network's DHCP server. The server responded by assigning an IP address to the interface.

---

## Verification

After running `dhcpcd`, confirm the IP was assigned:

```bash
ip a
```
or
```bash
ifconfig
```

You should now see an `inet` entry with an IP address under your interface.

**To test communication between the two VMs**, get the IP of each VM and ping from one to the other:

```bash
ping <ip-address-of-other-vm>
```

---

## Lessons Learned

- **NAT Network ≠ NAT** — Standard NAT isolates VMs from each other. NAT Network allows VM-to-VM communication.
- **DHCP must be enabled** on the NAT Network in VirtualBox's Network Manager.
- **Newer Ubuntu versions** do not include `dhclient`. Use `dhcpcd` or `systemctl restart systemd-networkd` instead.
- **Interface names matter** — The interface name in your Netplan config must exactly match the real interface name shown by `ip link show`.
- **DHCP isn't always automatic** — Sometimes the DHCP client needs to be manually triggered after a VM boots or connects to a new network.

---

## 📎 References

- [VirtualBox NAT Network Documentation](https://www.virtualbox.org/manual/ch06.html)
- [Ubuntu Netplan Documentation](https://netplan.io/)
- [dhcpcd man page](https://man.archlinux.org/man/dhcpcd.8)
