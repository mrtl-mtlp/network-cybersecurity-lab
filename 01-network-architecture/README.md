# 01 — Network Architecture

## Overview

The objective of this step was to design a small enterprise network in a virtualized environment using **GNS3**.

The architecture was designed with network segmentation, centralized routing, server infrastructure and security monitoring in mind.

The initial design includes several logical network zones:

* **ADMIN** — administrative workstations
* **USERS** — employee workstations
* **SERVERS** — internal servers
* **DMZ** — services that may need to be exposed
* **SOC** — security monitoring infrastructure

The network is built around a **VyOS router**, which provides routing, Internet access, NAT and network services.

---

## Network Architecture

### Planned Architecture

The initial architecture was designed around VLAN-based segmentation:

```text
                         Internet / WAN
                               |
                               |
                         ┌───────────┐
                         │   VyOS    │
                         │  Router   │
                         └─────┬─────┘
                               |
                         802.1Q Trunk
                               |
                         ┌─────┴─────┐
                         │  Switch   │
                         └─────┬─────┘
                               |
        ┌──────────┬───────────┼───────────┬───────────┐
        │          │           │           │           │
      VLAN 10    VLAN 20     VLAN 30     VLAN 40     VLAN 50
      ADMIN      USERS       SERVERS      DMZ          SOC
        │          │           │           │           │
      PCs        PCs         Servers     Services    Wazuh
```

The router would use **802.1Q subinterfaces** to route traffic between the different VLANs.

### Current Implementation

Due to the limitations of the switch image available in my GNS3 environment, I was not able to deploy and configure the VLAN segmentation shown above.

The switch is therefore currently operating without VLAN configuration, and the connected hosts are placed in the same network:

```text
                    Internet / WAN
                          |
                          |
                    ┌───────────┐
                    │   VyOS    │
                    │  Router   │
                    └─────┬─────┘
                          |
                    ┌─────┴─────┐
                    │  Switch   │
                    │ Unmanaged │
                    └─────┬─────┘
                          |
          ┌───────────────┼───────────────┐
          │               │               │
       Client 1        Client 2       Ubuntu Server
     192.168.10.10    192.168.10.x     192.168.10.100
```

All hosts currently use the **192.168.10.0/24** network.

This limitation does not prevent the implementation and testing of the routing, NAT, firewall, network analysis and reconnaissance components of the project.

---

# Network Addressing

The current implemented network uses the following addressing scheme:

| Device        | Interface | IP Address          | Role            |
| ------------- | --------- | ------------------- | --------------- |
| VyOS          | LAN       | `192.168.10.1/24`   | Default gateway |
| Ubuntu Server | `ens3`    | `192.168.10.100/24` | Internal server |
| PC 1          | —         | `192.168.10.10/24`  | Client          |
| PC 2          | —         | `192.168.10.x/24`   | Client          |
| VyOS          | WAN       | DHCP                | Internet access |

The planned VLAN addressing scheme is:

| VLAN | Name    | Network           | Gateway        |
| ---: | ------- | ----------------- | -------------- |
|   10 | ADMIN   | `192.168.10.0/24` | `192.168.10.1` |
|   20 | USERS   | `192.168.20.0/24` | `192.168.20.1` |
|   30 | SERVERS | `192.168.30.0/24` | `192.168.30.1` |
|   40 | DMZ     | `192.168.40.0/24` | `192.168.40.1` |
|   50 | SOC     | `192.168.50.0/24` | `192.168.50.1` |

> **Note:** The VLAN configuration above represents the intended architecture. It is not currently active in the GNS3 topology because of the switch image limitation.

---

# Switch Configuration

## Planned VLAN Configuration

The switch was intended to provide Layer 2 segmentation using five VLANs.

```bash
enable
configure terminal

vlan 10
 name ADMIN

vlan 20
 name USERS

vlan 30
 name SERVERS

vlan 40
 name DMZ

vlan 50
 name SOC

show vlan brief
```

Access ports would then be assigned to their corresponding VLAN.

For example:

```bash
interface ethernet0
 switchport mode access
 switchport access vlan 10
```

The connection between the switch and the VyOS router would be configured as a trunk:

```bash
interface gigabitEthernet0/1
 switchport mode trunk

show interfaces trunk
```

## Why is this configuration not currently deployed?

The available switch image in the GNS3 environment did not provide the required official/compatible switch functionality for the planned VLAN configuration.

Rather than pretending that the VLANs were operational, the current implementation uses the switch as a basic Layer 2 device without VLAN segmentation.

The VLAN configuration is therefore documented here as the **target architecture** and can be implemented later if a suitable switch image is available.

---

# VyOS Router Configuration

The VyOS router is responsible for Layer 3 connectivity and, in the planned architecture, inter-VLAN routing.

## Planned VLAN Interfaces

The router was configured to use VLAN subinterfaces on `eth1`:

```bash
set interfaces ethernet eth1 vif 10 address 192.168.10.1/24
set interfaces ethernet eth1 vif 20 address 192.168.20.1/24
set interfaces ethernet eth1 vif 30 address 192.168.30.1/24
set interfaces ethernet eth1 vif 40 address 192.168.40.1/24
set interfaces ethernet eth1 vif 50 address 192.168.50.1/24

commit
save
```

These interfaces correspond to the five planned network zones.

The configuration follows the **router-on-a-stick** architecture, where a single physical interface carries multiple VLANs using 802.1Q tagging.

---

# WAN Configuration

The WAN interface obtains its IP address through DHCP:

```bash
set interfaces ethernet eth0 address dhcp
```

This interface provides external network connectivity.

---

# NAT Configuration

Source NAT is used to allow internal hosts to access external networks through the WAN interface.

```bash
set nat source rule 10 outbound-interface name eth0
set nat source rule 10 source address 192.168.10.0/24
set nat source rule 10 translation address masquerade

commit
save
```

The `masquerade` translation allows private addresses from the internal network to be translated to the WAN address.

---

# DNS Configuration

The VyOS router is configured to use external DNS servers:

```bash
set system name-server 8.8.8.8
set system name-server 1.1.1.1
```

The configured DNS servers are:

* `8.8.8.8` — Google Public DNS
* `1.1.1.1` — Cloudflare DNS

DNS resolution allows hosts to translate domain names such as `google.com` into IP addresses.

---

# SSH Administration

SSH is enabled on the VyOS router for remote administration:

```bash
set service ssh port 22
```

SSH will later be restricted by the firewall so that administrative access is not unnecessarily exposed.

---

# Client Configuration


The current implementation uses ten virtual PCs connected to the same Layer 2 network.

All clients are currently part of the `192.168.10.0/24` network because VLAN segmentation could not be deployed with the available switch image.

| Device | IP Address      | Subnet Mask | Default Gateway | DNS       |
| ------ | --------------- | ----------- | --------------- | --------- |
| PC1    | `192.168.10.11` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC2    | `192.168.10.12` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC3    | `192.168.10.13` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC4    | `192.168.10.14` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC5    | `192.168.10.15` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC6    | `192.168.10.16` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC7    | `192.168.10.17` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC8    | `192.168.10.18` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC9    | `192.168.10.19` | `/24`       | `192.168.10.1`  | `8.8.8.8` |
| PC10   | `192.168.10.20` | `/24`       | `192.168.10.1`  | `8.8.8.8` |

### Example configuration

For example, PC1 is configured as:

```text
IP address:       192.168.10.11/24
Default gateway:  192.168.10.1
DNS server:       8.8.8.8
```

PC10 is configured as:

```text
IP address:       192.168.10.20/24
Default gateway:  192.168.10.1
DNS server:       8.8.8.8
```

Connectivity between clients can be tested using:

```bash
ping 192.168.10.12
```

Gateway connectivity can be tested using:

```bash
ping 192.168.10.1
```

External connectivity can be tested using:

```bash
ping 8.8.8.8
```

DNS resolution can be tested using:

```bash
ping google.com
```

---

# Ubuntu Server

An Ubuntu Server virtual machine was added to the topology to provide a server component for the security lab.

The server uses a static IPv4 address:

```text
IP address:      192.168.10.100/24
Default gateway: 192.168.10.1
DNS:             8.8.8.8
                 1.1.1.1
```

The network configuration is managed using Netplan.

Example configuration:

```yaml
network:
  version: 2

  ethernets:
    ens3:
      addresses:
        - 192.168.10.100/24

      routes:
        - to: default
          via: 192.168.10.1

      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

The configuration can be applied with:

```bash
sudo netplan apply
```

---

# Network Testing

After configuring the network, connectivity was tested between the different components.

### Gateway connectivity

```bash
ping 192.168.10.1
```

### Server connectivity

```bash
ping 192.168.10.100
```

### Internet connectivity

```bash
ping 8.8.8.8
```

### DNS resolution

```bash
ping google.com
```

These tests verify basic Layer 2/Layer 3 connectivity, gateway configuration, routing, NAT and DNS resolution.

---

# Tools Used

| Tool              | Purpose                                           |
| ----------------- | ------------------------------------------------- |
| **GNS3**          | Network topology virtualization and simulation    |
| **VyOS**          | Router, NAT, routing and network services         |
| **Ubuntu Server** | Server infrastructure and security testing target |
| **VPCS**          | Lightweight virtual client machines               |


Additional security monitoring components will be added in later stages of the project.

---

# Limitations and Future Improvements

The main limitation of the current implementation is the lack of a suitable configurable switch image in the GNS3 environment.

The target architecture includes five VLANs:

```text
VLAN 10 → ADMIN
VLAN 20 → USERS
VLAN 30 → SERVERS
VLAN 40 → DMZ
VLAN 50 → SOC
```

The next version of the architecture should implement these VLANs to provide proper network segmentation.

Additional improvements include:

* Inter-VLAN firewall policies
* Dedicated DMZ
* Dedicated SOC network
* Wazuh deployment
* Centralized security monitoring
* Network attack detection
* Additional security testing
* Improved access control between network zones

---

# Project Progress

| Component            | Status                           |
| -------------------- | -------------------------------- |
| Network architecture | ✅ Completed                      |
| Basic routing        | ✅ Completed                      |
| WAN connectivity     | ✅ Completed                      |
| NAT                  | ✅ Completed                      |
| VLAN design          | 🟡 Planned                       |
| VLAN deployment      | 🟡 Pending suitable switch image |
| Firewall             | ✅ Completed                      |
| Wireshark analysis   | ✅ Completed                      |
| Nmap reconnaissance  | ✅ Completed                      |
| Wazuh / SIEM         | 🔴 In progress                   |
| Security testing     | 🔴 In progress                   |

The project is progressively evolving from a basic enterprise network into a complete **network security laboratory**.

