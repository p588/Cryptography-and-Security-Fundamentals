# GNS3 Lab 1: Two Routers, Two PCs (Beginner Guide)

A step-by-step guide to building your first network in **GNS3**: two PCs, two Cisco routers, and a **static route** so the PCs can talk to each other. It also covers capturing traffic with **Wireshark**.

> Goal: make `PC1` (192.168.1.10) successfully ping `PC2` (192.168.2.10) through `R1` and `R2`.

---

## Table of Contents

1. [Concepts you need first](#1-concepts-you-need-first)
2. [Part A: Add the router image](#2-part-a-add-the-router-image-to-gns3)
3. [Part B: The lab topology and addressing](#3-part-b-the-lab-topology-and-addressing)
4. [Step 1: Build the topology](#step-1-build-the-topology)
5. [Step 2: Configure the routers](#step-2-configure-the-routers-cisco)
6. [Step 3: Configure the PCs](#step-3-configure-the-pcs-vpcs)
7. [Step 4: Test](#step-4-test-the-network)
8. [Step 5: Capture with Wireshark](#step-5-capture-traffic-with-wireshark)
9. [Step 6: Save and submit](#step-6-save-your-project)
10. [Troubleshooting](#troubleshooting)
11. [What I learned](#what-i-learned)

---

## 1. Concepts you need first

### What is GNS3?
GNS3 is a **network simulator**. It lets you build and test networks on your computer using real router software, with no physical equipment.

### What is a router?
A router connects **different networks** together and decides where to send data. Think of it as a **post office**: it reads the destination address on each packet and forwards it in the right direction.

### What is an IP address and a subnet mask?
- **IP address**: the address of a device, like `192.168.1.10`.
- **Subnet mask**: tells which part of the address is the *network* and which part is the *device*.

| Notation | Mask | Meaning | Usable devices |
|---|---|---|---|
| `/24` | 255.255.255.0 | First 3 numbers = network, last number = device | 254 |
| `/30` | 255.255.255.252 | Tiny network, used for links between two routers | 2 |

Example: in `192.168.1.0/24`, every address starting with `192.168.1.` is in the same network.

> Why `/30` for the router-to-router link? That link only needs **2 addresses** (one per router). `/30` wastes none.

### What is a default gateway?
The gateway is the **router's address on your LAN**. When a PC wants to talk to a device *outside* its own network, it sends the packet to the gateway. PC1's gateway is `192.168.1.1` (R1).

### What is a static route?
A router only automatically knows about networks **directly attached** to it. To reach other networks, you must tell it the way. A static route says:

```
ip route <destination network> <mask> <next-hop router>
```

Example on R1: `ip route 192.168.2.0 255.255.255.0 10.0.0.2`
Meaning: *"To reach 192.168.2.0/24, send the packets to 10.0.0.2 (R2)."*

**Both routers need a route**, otherwise the ping goes there but the reply cannot come back.

### What are ping, trace and ICMP?
- **ping**: sends an *Echo request*; the target answers with an *Echo reply*. If you see replies, the path works.
- **trace**: shows each router (hop) the packet passes through.
- Both use the protocol **ICMP**.

### What is Idle-PC?
A simulated Cisco router would otherwise use 100% of a CPU core doing nothing. **Idle-PC** tells GNS3 which moment in the router's code is "idle", so it can rest the CPU. Always set it.

### What is VPCS?
A very lightweight **virtual PC** inside GNS3. It only has basic network commands (`ip`, `ping`, `trace`), which is perfect for testing.

---

## 2. Part A: Add the router image to GNS3

You need a Cisco IOS image file (for example c7200). Add it once; it is reused in every lab.

1. In GNS3 click **Edit > Preferences** (Mac: **GNS3 > Preferences**).
2. In the left menu open **Dynamips > IOS routers** and click **New**.
3. Choose **Run this IOS router on the GNS3 VM** and click **Next**.
4. Click **Browse** and select the Cisco image file.
   - "Would you like to decompress this IOS image?" -> **Yes** (it starts faster).
   - "Copy the image to the GNS3 VM?" -> **Yes**.
5. Keep the suggested name and platform (for example `c7200`) -> **Next**.
6. Keep the default RAM (for example 512 MB) -> **Next**.
7. **Network adapters**: keep slot 0 as default and set **slot 1 to `PA-2FE-TX`** (two FastEthernet ports) -> **Next**.
8. **WIC modules**: click **Next** (not used on the 7200).
9. **Idle-PC**: click **Idle-PC finder**, wait for a value to appear, then click **Finish**.
   > Very important: without Idle-PC every router can use 100% of a CPU core.
10. Click **Apply** and **OK**. The router now appears under **Routers**.

---

## 3. Part B: The lab topology and addressing

### Topology

```
  PC1 ---- R1 ---------------- R2 ---- PC2

     LAN 1       WAN link          LAN 2

 192.168.1.0/24  10.0.0.0/30   192.168.2.0/24
```

There are **three separate networks**:
- **LAN 1** (192.168.1.0/24): PC1 and R1
- **WAN link** (10.0.0.0/30): R1 and R2
- **LAN 2** (192.168.2.0/24): R2 and PC2

### Addressing table

| Device | Interface | IP address / mask | Gateway |
|---|---|---|---|
| PC1 | Ethernet0 | 192.168.1.10 /24 | 192.168.1.1 |
| R1 | FastEthernet0/0 (LAN) | 192.168.1.1 /24 | - |
| R1 | FastEthernet1/0 (WAN) | 10.0.0.1 /30 | - |
| R2 | FastEthernet1/0 (WAN) | 10.0.0.2 /30 | - |
| R2 | FastEthernet0/0 (LAN) | 192.168.2.1 /24 | - |
| PC2 | Ethernet0 | 192.168.2.10 /24 | 192.168.2.1 |

> Every router interface gets its own IP, and each interface belongs to a **different network**. That is what makes it a router.

---

## Step 1: Build the topology

1. **File > New blank project**, name it `Lab1_YourName`, click **OK**.
2. Drag **two routers** (Cisco c7200) onto the canvas. Right-click each one -> **Change hostname** -> name them `R1` and `R2`.
3. Click the **End devices** icon and drag **two VPCS**. Name them `PC1` and `PC2`.
4. Click the **Add a link** tool (cable icon) and connect:

| From | To |
|---|---|
| PC1 `Ethernet0` | R1 `FastEthernet0/0` |
| R1 `FastEthernet1/0` | R2 `FastEthernet1/0` |
| R2 `FastEthernet0/0` | PC2 `Ethernet0` |

   Click the cable icon again to stop adding links.
5. Click the green **Start/Resume all nodes** (play) button. Wait about **one minute** for the routers to boot.
6. **Double-click** any device to open its console.

---

## Step 2: Configure the routers (Cisco)

Open the console of each router and paste the commands.

### R1

```
enable
configure terminal
hostname R1
interface FastEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
interface FastEthernet1/0
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit
ip route 192.168.2.0 255.255.255.0 10.0.0.2
end
write memory
```

### R2

```
enable
configure terminal
hostname R2
interface FastEthernet0/0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
interface FastEthernet1/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
ip route 192.168.1.0 255.255.255.0 10.0.0.1
end
write memory
```

### What each command does

| Command | Meaning |
|---|---|
| `enable` | Enter privileged mode (admin level) |
| `configure terminal` | Enter configuration mode |
| `hostname R1` | Set the router's name |
| `interface FastEthernet0/0` | Select the port you want to configure |
| `ip address ... ...` | Give the port an IP and subnet mask |
| `no shutdown` | Turn the port **on** (router ports are off by default) |
| `exit` | Go back one level |
| `ip route ...` | Add a static route (tells the router how to reach a remote network) |
| `end` | Leave configuration mode |
| `write memory` | **Save** the configuration so it survives a restart |

### The logic of the static routes
- **R1** knows 192.168.1.0/24 and 10.0.0.0/30 (directly connected). It does **not** know 192.168.2.0/24, so we tell it: *"go via 10.0.0.2"*.
- **R2** knows 192.168.2.0/24 and 10.0.0.0/30. It does **not** know 192.168.1.0/24, so we tell it: *"go via 10.0.0.1"*.

---

## Step 3: Configure the PCs (VPCS)

Open each PC console.

### PC1
```
ip 192.168.1.10/24 192.168.1.1
save
```

### PC2
```
ip 192.168.2.10/24 192.168.2.1
save
```

Format: `ip <PC address>/<prefix> <gateway>`. `save` stores the settings.

---

## Step 4: Test the network

Test from **near to far**. If a step fails, you know where the problem is.

| # | On PC1 type | What it tests | Expected |
|---|---|---|---|
| 1 | `ping 192.168.1.1` | PC1 -> its gateway (R1) | Replies |
| 2 | `ping 10.0.0.2` | PC1 -> far router (R2) via R1 | Replies |
| 3 | `ping 192.168.2.10` | PC1 -> PC2 (full path) | Replies |
| 4 | `trace 192.168.2.10` | Shows the hops | `192.168.1.1`, `10.0.0.2`, `192.168.2.10` |

> The first ping may show one timeout while the devices learn each other's MAC addresses (ARP). This is normal; ping again.

If step 3 works, **your network is working**.

---

## Step 5: Capture traffic with Wireshark

1. Right-click the **link between R1 and R2** -> **Start capture**. Wireshark opens.
2. On PC1 run `ping 192.168.2.10` again.
3. In Wireshark type `icmp` in the filter bar and press **Enter**. You will see **Echo request** and **Echo reply** packets.
4. Right-click the link -> **Stop capture**.

**What to notice:** the source and destination IPs stay `192.168.1.10` and `192.168.2.10` the whole way. The routers just forward the packet. Click a packet to see its layers (Ethernet, IP, ICMP).

---

## Step 6: Save your project

1. **File > Save project**. On Cisco routers make sure you ran `write memory`.
2. To submit: **File > Export portable project**, tick **Include base images** only if your lecturer asks, and save the `.gns3project` file.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Ping to the gateway fails | Port is off or wrong IP | `show ip interface brief` on the router; make sure status is `up/up`; use `no shutdown` |
| Can reach R2 but not PC2 | Missing or wrong route | `show ip route` on both routers |
| Ping goes out but no reply | Return route missing on one router | Check **both** routers have a static route |
| Interface shows `administratively down` | Forgot `no shutdown` | Enter the interface and run `no shutdown` |
| Whole computer is slow | Idle-PC not set | Re-run Idle-PC finder |
| Config lost after restart | Forgot to save | Run `write memory` (routers) and `save` (PCs) |

Useful commands:

```
show ip interface brief
show ip route
show running-config
```

---

## What I learned

- A router connects different networks, and each of its interfaces has an IP in a different network.
- PCs send traffic for other networks to their **default gateway**.
- A **static route** tells a router how to reach a network that is not directly connected, and both directions are needed.
- `ping` and `trace` (ICMP) are the first tools for checking connectivity.
- Wireshark lets you see the actual packets crossing a link.

---

*Tools: GNS3, Cisco IOS c7200 (Dynamips), VPCS, Wireshark.*
