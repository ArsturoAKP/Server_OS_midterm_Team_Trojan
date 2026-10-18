# Server OS Midterm: Lab 01 (novapeak)

**Group Name:** Trojan
**Company / domain:** novapeak / novapeak.lab
**Checker group id:** lab01
**Date:** 18 October 2026

## Members

| No | Student Name              | Student ID | GitHub Username                |
|----|---------------------------|------------|--------------------------------|
| 1  | L Peter San Awng (Leader) | 6705142021 | 6705142021-peter               |
| 2  | Paing Oo Thant            | 6705142006 | potpot2626                     |
| 3  | Aung Kyaw Phyo            | 6705142012 | cybotuid0 and ArsturoAKP       |
| 4  | Kaung Myat Tun            | 6705142016 | KenzoChann                     |
| 5  | Thin Thiri Zaw            | 6705142020 | 6705142020-sketch              |

---

## Part 3a: Declaration (required)

| AI tool | What we used it for |
| --- | --- |
| Claude  | Gave us a step-by-step explanation and repair guide. Reviewed our working notes and corrected them (`dhcpcd` instead of `dhclient`, broken `dig` commands). Helped us read our outputs. We ran every command ourselves on the live lab, and every output in our log comes from our own `session.log`. |
| ChatGPT (OpenAI) | **In Part 3c only:** AI #2. Given the broken configuration and our baseline checker output, and asked to diagnose and fix (prompt at the end of this file). |

We used AI tools as supporting resources for understanding technical concepts and comparing possible troubleshooting approaches. We did not treat AI-generated suggestions as proof that a fault existed. We verified possible explanations against the actual server configuration, terminal output, service logs, client-side DNS queries, and grading checker results.

## Part 3b: Initial Enumeration and Our Own Diagnosis

### 1. Objective and Approach

Before comparing our findings with AI suggestions, we performed an initial enumeration of the server, client network, DNS service, BIND configuration, firewall rules, and DHCP settings.

Our objective was to identify the faults systematically without changing the configuration too early. We followed a **symptom → hypothesis → proof → fix → confirmation** approach.

We tested DNS from the client network namespace because a successful query from the server itself would not prove that clients could reach and use the DNS service correctly.

### 2. Check the Initial Lab Status and Baseline

**Commands used:**

```bash
cat /root/.lab_ready

python3 /root/grading/params.py lab01

python3 /root/grading/check_lab.py --group lab01 --phase build
```

**Evidence and findings:**

- The scenario parameters provided the expected network, DNS server, domain, host records, and DHCP settings.
- The baseline checker reported **11/25**.
- Check C1 reported that the client could not reach the DNS service at `192.168.19.10`.
- Check C4 reported that `named-checkconf -z` failed.
- Checks C9 and C11 identified a DHCP DNS-server configuration problem: the client was being given `8.8.8.8` instead of `192.168.19.10`.

These results showed that the problem involved multiple configuration areas.

### 3. Enumerate Network Connectivity

**Commands used:**

```bash
ip -br addr
ip route
ip netns list
ip netns exec client ip -br addr
ip netns exec client ip route
ip netns exec client ping -c 3 192.168.19.10
```

**Evidence collected:**

```text
3 packets transmitted, 3 received, 0% packet loss
```

**Finding:** The client could reach the DNS server at the network layer. A complete network-connectivity failure was therefore less likely, so we continued investigating DNS service reachability and configuration.

### 4. Enumerate the DNS Service and Compare UDP with TCP

**Commands used:**

```bash
systemctl status named --no-pager

ss -ulntp | grep ':53'

ip netns exec client dig @192.168.19.10 ns1.novapeak.lab +tries=1 +time=2

ip netns exec client dig @192.168.19.10 -x 192.168.19.10 +tries=1 +time=2

ip netns exec client dig @192.168.19.10 -x 192.168.19.10 +tcp
```

**Evidence collected:**

The UDP query timed out:

```text
;; communications error to 192.168.19.10#53: timed out
;; no servers could be reached
```

The reverse lookup over TCP returned an authoritative answer:

```text
status: NOERROR
flags: qr aa rd
ANSWER: 1

10.19.168.192.in-addr.arpa. 3600 IN PTR ns1.novapeak.lab.
SERVER: 192.168.19.10#53 (TCP)
```

**Finding:** UDP DNS requests timed out while the TCP reverse lookup succeeded. This indicated that the DNS service was responding over TCP and made UDP/53 filtering a strong possibility.
