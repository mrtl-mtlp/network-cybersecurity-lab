# Enterprise Network Security Lab

A small enterprise network built in GNS3 and used as a hands-on lab for network design, firewalling, traffic analysis, reconnaissance and security monitoring.

The lab is built around a VyOS router/firewall, an Ubuntu Server and ten virtual clients. Each stage of the project is documented in its own folder: what was built, how it was verified, and what is still missing.

## Current topology

```
                 Internet / WAN
                       |
                  eth0 (DHCP)
                 ┌───────────┐
                 │   VyOS    │  NAT, routing, stateful firewall, SSH
                 └─────┬─────┘
                  eth1 (192.168.10.1/24)
                 ┌─────┴─────┐
                 │  Switch   │  unmanaged (no VLANs)
                 └─────┬─────┘
        ┌──────────────┼──────────────┐
   PC1 – PC10      Ubuntu Server
 192.168.10.11-20  192.168.10.100
```

The target design uses five VLANs (ADMIN, USERS, SERVERS, DMZ, SOC) with inter-VLAN routing on the router. It could not be deployed because the switch image available in GNS3 does not support VLANs. The design is documented in [01 – Network Architecture](01-network-architecture/README.md) and all hosts currently share `192.168.10.0/24`.

## Project stages

| # | Stage | Summary | Status |
|---|-------|---------|--------|
| 01 | [Network Architecture](01-network-architecture/README.md) | Topology, addressing, routing, NAT, DNS, planned VLAN design | Done (VLANs planned) |
| 02 | [Firewall](02-firewall/README.md) | Stateful, default-deny firewall on VyOS; SSH restricted to an admin group | Done |
| 03 | [Traffic Analysis](03-traffic-analysis/README.md) | Wireshark analysis of ICMP, DNS, SSH, HTTP/HTTPS and firewall behaviour | Done |
| 04 | [Nmap Reconnaissance](04-nmap-reconnaissance/README.md) | Host discovery, port scanning and service detection from inside the LAN | Done |
| 05 | Wazuh / SIEM | Centralised monitoring and attack detection | In progress |
| 06 | Security testing | Attack simulation and detection validation | In progress |

## Main results

- Internal hosts reach the Internet through source NAT; no unsolicited inbound traffic is accepted from the WAN.
- SSH to the router is limited to three hosts in an `ADMIN` address group, and all other traffic to the router is dropped by default.
- Packet captures confirm DNS resolution, TLS 1.3 browsing and an encrypted SSH session, with no unexpected protocols.
- Nmap shows a minimal attack surface on the server: a single open port (22/tcp).
- Two weaknesses were identified: the flat network means the router firewall does not filter traffic between internal hosts, and SSH and DNS expose information to passive observers (version banners, cleartext DNS).

## Skills demonstrated

- **Network design:** segmentation planning, IP addressing, router-on-a-stick design, NAT.
- **Firewalling:** stateful rule sets, address groups, chain/jump logic on VyOS, rule testing.
- **Traffic analysis:** Wireshark display filters, protocol dissection, TLS/SSH handshake analysis.
- **Reconnaissance:** Nmap host discovery, port scanning and service fingerprinting, and interpreting results against the intended configuration.
- **Documentation:** stated limitations, planned improvements and verification steps for each stage.

## Tools

GNS3, VyOS, Ubuntu Server, VPCS, Wireshark (Dumpcap 4.2.2), Nmap. Wazuh is being added.

## Limitations and roadmap

- Deploy the five VLANs once a switch image with VLAN support is available.
- Add inter-VLAN firewall policies and a dedicated DMZ.
- Move the SOC onto its own network and finish the Wazuh deployment.
- Run attack simulations and check that they are detected and logged.
- Harden the server (host firewall, SSH configuration) and re-scan to measure the difference.

## Repository structure

```
.
├── README.md
├── 01-network-architecture/
├── 02-firewall/
├── 03-traffic-analysis/
└── 04-nmap-reconnaissance/
```
