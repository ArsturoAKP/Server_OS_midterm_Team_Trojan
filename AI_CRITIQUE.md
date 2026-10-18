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


### 5. Enumerate Firewall Rules and Prove Fault 1

**Commands used:**

```bash
iptables -S

iptables -L INPUT -n -v --line-numbers

iptables -Z INPUT

ip netns exec client dig @192.168.19.10 www.novapeak.lab +tries=1 +time=2

iptables -L INPUT -n -v --line-numbers
```

**Evidence collected:**

```text
-A INPUT -p udp -m udp --dport 53 -j DROP
```

After resetting the firewall counters and sending one DNS query, the matching rule showed:

```text
num  pkts  bytes  target  prot  opt  in  out  source       destination
1    1     85     DROP    17    --   *   *    0.0.0.0/0   0.0.0.0/0  udp dpt:53
```

**Finding:** The DROP rule's packet counter increased to one after the test query. This directly supported the conclusion that the firewall was dropping DNS traffic over UDP.

**Conclusion:** Fault 1 was an inbound firewall rule dropping UDP port 53.

### 6. Enumerate BIND Configuration and Prove Fault 2

**Commands used:**

```bash
named-checkconf -p

named-checkconf -z

grep -A3 'zone "novapeak.lab"' /etc/bind/named.conf.local

ls -l /etc/bind/db.MISSING

ls -l /etc/bind/db.novapeak.lab

journalctl -u named --no-pager | grep -i missing | tail -5
```

**Evidence collected:**

```text
zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
zone novapeak.lab/IN: not loaded due to errors.
```

The configuration contained:

```text
zone "novapeak.lab" {
    type master;
    file "/etc/bind/db.MISSING";
};
```

The intended file, `/etc/bind/db.novapeak.lab`, existed, but `/etc/bind/db.MISSING` did not.

**Finding:** The forward DNS zone referenced a nonexistent file. After the firewall fault was resolved, the forward lookup returned `SERVFAIL`, exposing the second fault.

### 7. Enumerate DNS Records and Prove Fault 3

**Commands used:**

```bash
python3 /root/grading/params.py lab01 | grep api

grep api /etc/bind/db.novapeak.lab

grep '^40' /etc/bind/db.novapeak.lab.rev

for h in ns1 www mail api; do
    echo -n "$h: "
    ip netns exec client dig @192.168.19.10 "$h.novapeak.lab" +short
done
```

**Evidence collected:**

The scenario parameters specified:

```text
api.novapeak.lab -> 192.168.19.40
```

However, the forward zone contained:

```text
api IN A 192.168.19.1
```

The reverse zone contained:

```text
40 IN PTR api.novapeak.lab.
```

The client lookup results included:

```text
ns1: 192.168.19.10
www: 192.168.19.20
mail: 192.168.19.30
api: 192.168.19.1
```

**Finding:** The `api` record pointed to the gateway address, `192.168.19.1`, instead of the expected `192.168.19.40`. The scenario parameters and reverse-zone record supported the expected address.

### 8. Enumerate DHCP Settings and Prove Fault 4

**Commands used:**

```bash
ip netns exec client dhcpcd -U cli0 \
    | grep -E 'ip_address|routers|domain_name'

grep -n domain-name-servers /etc/dhcp/dhcpd.conf

dhcpd -t -cf /etc/dhcp/dhcpd.conf

python3 /root/grading/check_lab.py --group lab01 --phase build
```

**Evidence collected:**

The client lease contained:

```text
ip_address=192.168.19.104
routers=192.168.19.1
domain_name_servers=8.8.8.8
domain_name=novapeak.lab
```

The DHCP configuration contained:

```text
option domain-name-servers 8.8.8.8;
```

The checker reported:

```text
C9: DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10
C11: using the leased resolver 8.8.8.8, www did not resolve to 192.168.19.20
```

**Finding:** DHCP was advertising a public DNS resolver instead of the company's internal DNS server. Consequently, clients were not automatically configured to use the DNS service required by the lab.

### 9. Summary of Initial Enumeration

| Area investigated | Evidence | Finding |
|---|---|---|
| Network connectivity | Client ping succeeded with 0% packet loss | Network path to the server was working |
| DNS transport | UDP timed out while TCP reverse lookup succeeded | UDP/53 firewall filtering suspected |
| Firewall | UDP/53 DROP rule; counter increased after a query | Fault 1 |
| BIND zone configuration | `named-checkconf -z` reported `db.MISSING` not found | Fault 2 |
| DNS records | `api` resolved to `.1`, but expected `.40` | Fault 3 |
| DHCP configuration | Client lease advertised `8.8.8.8` | Fault 4 |

### 10. Verification After Fixing the Faults

We retested the system after applying the repairs instead of assuming that editing the configuration files was sufficient.

| Stage | Checker result | Interpretation |
|---|---:|---|
| Initial baseline | 11/25 | Multiple faults remained |
| After the first three faults were addressed | 19/25 | DNS reachability and records improved, but DHCP integration still failed |
| Final verification | 25/25 | All checker requirements passed |

The final result was recorded in `checker_result.txt`.

Our investigation identified four separate faults. The sequence of results was **timeout → SERVFAIL → incorrect API address → incorrect DHCP resolver**. Fixing each fault revealed or resolved the next issue, and the final checker result confirmed that the lab requirements were met.

---
