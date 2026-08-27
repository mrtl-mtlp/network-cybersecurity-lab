# 02 — Firewall Configuration

## Overview

A **firewall** is a security mechanism that controls network traffic according to predefined rules. It determines whether network connections should be **allowed or blocked** based on criteria such as source address, destination address, protocol, port and connection state.

In this project, the firewall is implemented on the **VyOS router**. Its purpose is to control traffic between the LAN, the WAN and the router itself.

The firewall follows a **default-deny approach**: traffic is explicitly allowed when required, while unauthorized traffic is blocked.

---

# Firewall Architecture

The firewall uses different rule sets depending on the direction and destination of the traffic.

| Rule Set    | Meaning                         |
| ----------- | ------------------------------- |
| `WAN-LOCAL` | Internet → VyOS                 |
| `LAN-LOCAL` | LAN → VyOS                      |
| `LAN-WAN`   | LAN → Internet / other networks |
| `WAN-LAN`   | Internet → LAN                  |

### LOCAL vs IN

The distinction between `LOCAL` and `IN` is important:

* **LOCAL**: traffic is destined for the VyOS router itself.
* **IN**: traffic passes through VyOS toward another network.

For example:

```text
PC → VyOS SSH
```

is `LAN-LOCAL`, because the destination is the router.

Whereas:

```text
PC → Internet
```

is `LAN-WAN`, because the traffic passes through VyOS.

---

# Firewall Rule Organization

Rules are processed from **top to bottom**. Therefore, more specific rules must be placed before the final blocking rule.

The rule numbering is organized as follows:

| Rule Range | Purpose             |
| ---------: | ------------------- |
|    `10–99` | General connections |
|  `100–199` | Administration      |
|  `200–299` | Published services  |
|      `300` | Final blocking rule |

This numbering scheme makes the firewall easier to maintain and extend.

---

# Security Policy

The main security policy implemented in this project is:

### WAN → LAN

* Allow `ESTABLISHED` connections
* Allow `RELATED` connections
* Drop other unsolicited traffic

### WAN → LOCAL

* Allow `ESTABLISHED` connections
* Allow `RELATED` connections
* Drop other unsolicited traffic

### LAN → WAN

* Allow legitimate outbound traffic

### LAN → LOCAL

* Allow SSH only from authorized administration hosts
* Allow ICMP for testing and diagnostics
* Drop other traffic

This provides a basic **stateful firewall** configuration.

---

# 1. WAN → LAN

Connections initiated from the LAN can receive their responses from the WAN.

The firewall therefore allows packets belonging to already established or related connections.

```bash
set firewall ipv4 name WAN-LAN rule 10 action accept
set firewall ipv4 name WAN-LAN rule 10 state established

set firewall ipv4 name WAN-LAN rule 20 action accept
set firewall ipv4 name WAN-LAN rule 20 state related

set firewall ipv4 name WAN-LAN default-action drop
```

The final `drop` prevents unsolicited inbound connections from reaching the LAN.

---

# 2. WAN → LOCAL

The same principle is applied to traffic coming from the WAN and destined for the VyOS router itself.

```bash
set firewall ipv4 name WAN-LOCAL rule 10 action accept
set firewall ipv4 name WAN-LOCAL rule 10 state established

set firewall ipv4 name WAN-LOCAL rule 20 action accept
set firewall ipv4 name WAN-LOCAL rule 20 state related

set firewall ipv4 name WAN-LOCAL default-action drop
```

This prevents unsolicited Internet traffic from directly accessing the router.

---

# 3. LAN → WAN

Traffic initiated from the LAN toward external networks is allowed.

```bash
set firewall ipv4 name LAN-WAN rule 10 action accept
```

The firewall can later be made more restrictive by explicitly defining the services that clients are allowed to use.

---

# 4. SSH Administration

SSH access to the VyOS router is restricted to authorized administration hosts.

The following hosts are included in the `ADMIN` address group:

```bash
set firewall group address-group ADMIN address 192.168.10.11
set firewall group address-group ADMIN address 192.168.10.12
set firewall group address-group ADMIN address 192.168.10.100
```

The Ubuntu server is also included as an authorized administration host.

SSH uses TCP port `22`.

```bash
set firewall ipv4 name LAN-LOCAL rule 100 action accept
set firewall ipv4 name LAN-LOCAL rule 100 destination port 22
set firewall ipv4 name LAN-LOCAL rule 100 protocol tcp
set firewall ipv4 name LAN-LOCAL rule 100 source group address-group ADMIN
```

The default action for traffic destined for the router is then set to `drop`:

```bash
set firewall ipv4 name LAN-LOCAL default-action drop
```

This means that only hosts belonging to the `ADMIN` group can access the VyOS SSH service.

---

# 5. ICMP / Ping

ICMP is allowed from the LAN to the router for network diagnostics and connectivity testing.

```bash
set firewall ipv4 name LAN-LOCAL rule 30 action accept
set firewall ipv4 name LAN-LOCAL rule 30 protocol icmp
```

This allows commands such as:

```bash
ping 192.168.10.1
```

to be used for testing connectivity with the VyOS router.

---

# 6. Applying the Firewall to Interfaces

VyOS uses the concepts of:

* `input`
* `output`
* `forward`

The custom firewall rule sets such as `LAN-LOCAL` and `WAN-LOCAL` are therefore connected to the appropriate traffic chains using **jump rules**.

## LAN → VyOS

Traffic entering through the LAN interface is sent to the `LAN-LOCAL` rule set:

```bash
set firewall ipv4 input filter rule 10 action jump
set firewall ipv4 input filter rule 10 jump-target LAN-LOCAL
set firewall ipv4 input filter rule 10 inbound-interface name eth1
```

## WAN → VyOS

Traffic entering through the WAN interface is sent to `WAN-LOCAL`:

```bash
set firewall ipv4 input filter rule 20 action jump
set firewall ipv4 input filter rule 20 jump-target WAN-LOCAL
set firewall ipv4 input filter rule 20 inbound-interface name eth0
```

## LAN → WAN

Forwarded traffic from the LAN to the WAN is sent to `LAN-WAN`:

```bash
set firewall ipv4 forward filter rule 10 action jump
set firewall ipv4 forward filter rule 10 jump-target LAN-WAN
set firewall ipv4 forward filter rule 10 inbound-interface name eth1
set firewall ipv4 forward filter rule 10 outbound-interface name eth0
```

## WAN → LAN

Forwarded traffic from the WAN to the LAN is sent to `WAN-LAN`:

```bash
set firewall ipv4 forward filter rule 20 action jump
set firewall ipv4 forward filter rule 20 jump-target WAN-LAN
set firewall ipv4 forward filter rule 20 inbound-interface name eth0
set firewall ipv4 forward filter rule 20 outbound-interface name eth1
```

---

# Applying the Configuration

After modifying the firewall configuration:

```bash
commit
save
```

`commit` applies the configuration to the running system, while `save` makes the configuration persistent across reboots.

---

# Firewall Testing

The firewall was tested to verify that legitimate traffic is allowed while unauthorized traffic is blocked.

## Test 1 — ICMP

A LAN host should be able to ping the VyOS router:

```bash
ping 192.168.10.1
```

### Expected result

```text
PING 192.168.10.1
64 bytes from 192.168.10.1
```

The test confirms that ICMP traffic is allowed by the `LAN-LOCAL` rule set.

---

## Test 2 — Authorized SSH

An authorized host should be able to connect to the VyOS router using SSH:

```bash
ssh vyos@192.168.10.1
```

The connection is allowed because the source address belongs to the `ADMIN` address group.

---

## Test 3 — Unauthorized SSH

A host that is not part of the `ADMIN` address group should not be able to establish an SSH connection to the router.

```bash
ssh vyos@192.168.10.1
```

### Expected result

```text
Connection refused / timeout
```

The connection is blocked by the default `drop` policy.

---

# Security Analysis

The firewall implements several important security principles:

### Default deny

Traffic is blocked unless it is explicitly authorized.

### Stateful filtering

`ESTABLISHED` and `RELATED` states allow legitimate response traffic without opening unrestricted inbound access.

### Restricted administration

SSH access is limited to a predefined group of trusted hosts.

### Network boundary protection

Traffic entering from the WAN is controlled before reaching either the LAN or the router itself.

### Separation of traffic types

Using separate rule sets for `LAN-LOCAL`, `WAN-LOCAL`, `LAN-WAN` and `WAN-LAN` makes the security policy easier to understand and maintain.

---

# Current Limitations

The current network does not yet implement the planned VLAN segmentation because of the switch limitation described in the previous architecture stage.

Consequently, the firewall currently operates primarily between the **LAN and WAN** and protects the VyOS router itself.

Once VLANs are deployed, the firewall policy can be extended to control traffic between:

```text
ADMIN ↔ USERS
ADMIN ↔ SERVERS
USERS ↔ SERVERS
WAN ↔ DMZ
SOC ↔ Network infrastructure
```

This will provide more granular **east-west traffic filtering** in addition to the current LAN/WAN protection.

---

# Status

| Component             | Status                     |
| --------------------- | -------------------------- |
| Stateful firewall     | ✅ Implemented              |
| WAN → LAN filtering   | ✅ Implemented              |
| WAN → LOCAL filtering | ✅ Implemented              |
| LAN → WAN filtering   | ✅ Implemented              |
| SSH restriction       | ✅ Implemented              |
| ICMP policy           | ✅ Implemented              |
| Firewall testing      | ✅ Completed                |
| Inter-VLAN filtering  | 🟡 Pending VLAN deployment |
| DMZ firewall policy   | 🟡 Planned                 |

