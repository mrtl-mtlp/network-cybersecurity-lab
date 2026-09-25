# Nmap — Network Reconnaissance

## Objective

After configuring the network and firewall with VyOS, Nmap was used to assess the **exposure of the lab infrastructure**.

The objective was to verify:

* which hosts are reachable on the LAN
* which ports are exposed on the VyOS router
* which services are running
* whether the firewall behaves as expected regarding SSH access

The tests were performed from the Ubuntu Server (192.168.10.100) inside the GNS3 laboratory.

---

## Lab



The VyOS firewall currently allows SSH access to the router only from the `ADMIN` address group:

```text
192.168.10.11
192.168.10.12
192.168.10.100
```

---

## 1. Network Discovery

First, the LAN was scanned to identify reachable hosts:

```bash
nmap -sn 192.168.10.0/24
```

This provided an overview of the active devices in the laboratory before performing more targeted scans.
Only one device is active : the Ubuntu Server. It is what we expected because all the others are VPCS.

---

## 2. VyOS Service Enumeration

The VyOS LAN interface was then scanned to identify exposed services:

```bash
nmap -sV 192.168.10.1
```

### Result

The scan identified SSH as the only open TCP service:

```text
22/tcp open ssh
```

This result is consistent with the current VyOS configuration, where SSH is the only intentionally exposed management service.

---

## 3. Full TCP Scan

To ensure that no additional TCP service was exposed, the complete TCP port range was scanned:

```bash
nmap -p- 192.168.10.1
```

### Result

```text
22/tcp open ssh
```

No other open TCP port was identified.

This confirms that the current VyOS configuration exposes a limited number of TCP services on the LAN interface.

---

## 4. Firewall Validation

Nmap was also used to verify the SSH policy implemented in the VyOS firewall.

The objective was to check whether the source IP of a machine affects its ability to reach TCP port 22.

From an authorized `ADMIN` machine:

```bash
nmap -p 22 192.168.10.100
```

Expected:

```text
22/tcp open ssh
```

From a non-authorized machine:

```bash
nmap -p 22 192.168.10.11
```

Expected:

```text
22/tcp filtered ssh
```

This provides a practical verification of the firewall rule restricting SSH access to the `ADMIN` group.

---

## 5. LAN Scan

The entire LAN was also enumerated:

```bash
nmap -sV 192.168.10.0/24
```

The purpose was to identify services exposed by the different devices in the laboratory and detect potential unexpected exposure.

---

## Results

| Target            | Scan                | Main observation                  |
| ----------------- | ------------------- | --------------------------------- |
| `192.168.10.0/24` | Host discovery      | Identification of reachable hosts |
| `192.168.10.1`    | Service detection   | SSH detected                      |
| `192.168.10.1`    | Full TCP scan       | `22/tcp` only                     |
| `192.168.10.1`    | SSH test            | Access controlled by firewall     |
| `192.168.10.0/24` | Service enumeration | LAN service discovery             |

---

## Conclusion

The Nmap assessment confirmed the current network exposure after implementing the VyOS firewall.

The main finding was that **SSH is the only open TCP service detected on the VyOS router**. Nmap was also used to validate the firewall's source-based SSH access policy.

This phase complements the previous **Wireshark traffic analysis**:

* **Wireshark** → analysis of network traffic;
* **Nmap** → identification of exposed hosts, ports and services;
* **Wazuh** → monitoring and security event detection.

The next phase of the project is the deployment of **Wazuh** on a dedicated monitoring setup.

