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

## Part 3c: Putting the AIs to the Test

### 1. Comparison Method

After completing our initial enumeration, we compared possible AI troubleshooting suggestions with the evidence collected from our Linux server and client environment.

We evaluated the suggestions from ChatGPT and Claude using the following criteria:

- **Technical accuracy:** Was the suggested command or explanation technically correct?
- **Evidence-based reasoning:** Did the AI distinguish possible causes from confirmed faults?
- **Diagnostic usefulness:** Did the suggestion help us narrow down the problem?
- **Risk of unnecessary changes:** Did the AI recommend changing configurations before confirming the cause?

We used the following prompt as a common test scenario:

> We are troubleshooting a Linux server for the domain `novapeak.lab`. The server uses BIND for DNS and ISC DHCP Server for client configuration. The client is in the `client` network namespace. DNS queries over UDP time out, but a reverse DNS query over TCP succeeds. The initial grading checker reports 11/25. Help us diagnose the possible faults using commands. Explain what each command proves, and do not assume that a suspected cause is confirmed without evidence.

### 2. AI Comparison Table

| Fault identified by our investigation | Our evidence | AI #1 (ChatGPT) | AI #2 (Claude) | Evaluation |
|---|---|---|---|---|
| Firewall drops UDP port 53 | UDP queries timed out, TCP lookup succeeded, and the UDP DROP counter increased after a query. | Suggested comparing UDP and TCP DNS queries, inspecting firewall rules with `iptables -L INPUT -n -v --line-numbers`, and checking whether the DROP counter increased. | Suggested checking DNS connectivity from the client namespace and comparing firewall rules with service listening status to isolate the network layer responsible. | Both approaches were useful because they focused on diagnostic tests before changing the firewall. The increasing DROP counter provided direct evidence of the fault. |
| BIND forward zone references a missing file | `named-checkconf -z` reported that `/etc/bind/db.MISSING` could not be found. | Suggested running `named-checkconf -z`, inspecting the zone declaration, and checking whether the referenced zone file existed. | Suggested checking BIND's service logs and verifying that the configured zone-file path matched the actual file on disk. | Both approaches addressed the relevant configuration. The error from `named-checkconf -z` directly confirmed the missing-file problem. |
| Potentially misleading diagnosis: changing BIND settings before checking the firewall | The TCP DNS lookup succeeded, the UDP DNS query timed out, and the UDP/53 DROP counter increased after a query. | Hypothetical suggestion: change `listen-on` and `allow-query` settings before confirming the cause. | Not evaluated; no verified response available. | The suggestion was premature because the evidence pointed to firewall filtering. We investigated the firewall before making unnecessary BIND changes. |
| API DNS record points to the wrong IP | The forward record returned `192.168.19.1`, while the expected address was `192.168.19.40`. | Suggested comparing the A record with the lab's expected values and checking the zone file before reloading BIND. | Suggested checking the forward and reverse DNS records and comparing the result against the intended network configuration. | Both approaches could help identify the incorrect record. The scenario parameters established the expected IP; the reverse record provided supporting evidence. |
| DHCP advertises the wrong DNS server | The client lease advertised `8.8.8.8`; the required DNS server was `192.168.19.10`. Checker results C9 and C11 also reported the problem. | Suggested inspecting `dhcpd.conf`, validating its syntax with `dhcpd -t`, and checking the DNS-server option in the client lease. | Suggested checking the DHCP configuration and renewing or rechecking the client lease to verify which resolver the client received. | Both approaches were relevant. The DHCP configuration and client lease together confirmed the incorrect resolver setting. |

### 3. Detailed Case Studies

#### Case 1: DNS Timeout — Firewall or BIND Problem?

**Observed symptom**

The client could ping `192.168.19.10`, but DNS queries over UDP timed out. A reverse lookup over TCP succeeded.

**ChatGPT response (simulated):**

> Compare DNS queries over UDP and TCP. Check whether BIND is listening on port 53, then inspect the firewall rules and packet counters. If TCP succeeds but UDP fails, investigate UDP filtering before changing BIND settings.

**Claude response (simulated):**

> Test DNS from the client namespace and inspect the DNS service status, listening sockets, and firewall rules. Use the results to distinguish a service problem from a transport-specific filtering problem.

**Our verification**

```bash
ip netns exec client dig @192.168.19.10 ns1.novapeak.lab +tries=1 +time=2

ip netns exec client dig @192.168.19.10 -x 192.168.19.10 +tcp

ss -ulntp | grep ':53'

iptables -L INPUT -n -v --line-numbers
```

We found that the UDP query timed out, the TCP reverse lookup succeeded, and the UDP/53 DROP counter increased after the client query.

**Evaluation**

Both simulated responses provided useful diagnostic directions. However, the decisive evidence was not the AI explanation; it was the firewall rule and its increasing packet counter. This case demonstrated the importance of verifying a hypothesis before modifying a configuration.

#### Case 2: BIND Zone Fails to Load

**Observed symptom**

After the UDP filtering issue was addressed, the forward DNS lookup returned `SERVFAIL`.

**ChatGPT response (simulated):**

> Run `named-checkconf -z` to test whether the configured zones can load. Inspect the relevant zone declaration and confirm that the referenced file exists.

**Claude response (simulated):**

> Inspect the BIND service logs and compare the zone-file path in the configuration with the actual files available on the server.

**Our verification**

```bash
named-checkconf -z

grep -A3 'zone "novapeak.lab"' /etc/bind/named.conf.local

ls -l /etc/bind/db.MISSING

ls -l /etc/bind/db.novapeak.lab
```

The configuration referenced `/etc/bind/db.MISSING`, which did not exist. The intended zone file, `/etc/bind/db.novapeak.lab`, was available.

**Evaluation**

Both approaches were appropriate. The strongest proof was the explicit error from `named-checkconf -z`, supported by the zone declaration and file listing. We corrected the path rather than creating an unrelated replacement file.

#### Case 3: Incorrect API Address

**Observed symptom**

The DNS service returned an IP address for `api.novapeak.lab`, but the result did not match the expected lab configuration.

**ChatGPT response (simulated):**

> Query the API record, inspect the forward zone file, and compare the returned address with the expected address provided by the lab parameters.

**Claude response (simulated):**

> Check the API A record and its corresponding PTR record. Confirm that the forward record uses the intended IP address and that the reverse zone is consistent.

**Our verification**

```bash
grep api /etc/bind/db.novapeak.lab

grep '^40' /etc/bind/db.novapeak.lab.rev

ip netns exec client dig @192.168.19.10 api.novapeak.lab +short
```

The forward zone contained `api IN A 192.168.19.1`, but the expected address was `192.168.19.40`. The reverse zone contained a PTR record for `192.168.19.40`.

**Evaluation**

The simulated suggestions correctly directed attention to the DNS records. However, a forward and reverse DNS mismatch does not automatically prove which address is correct. We used the lab's expected parameters to establish that `.40` was the intended API address.

#### Case 4: DHCP Gives Clients the Wrong Resolver

**Observed symptom**

The DHCP service assigned a client address, but the lease listed `8.8.8.8` as the DNS server.

**ChatGPT response (simulated):**

> Inspect the `domain-name-servers` option in `dhcpd.conf`, validate the configuration, and examine the client's lease to confirm the resolver being advertised.

**Claude response (simulated):**

> Verify the DHCP server's DNS option and compare it with the DNS server required by the lab. Check the active client lease rather than relying only on the server configuration file.

**Our verification**

```bash
grep -n domain-name-servers /etc/dhcp/dhcpd.conf

dhcpd -t -cf /etc/dhcp/dhcpd.conf

ip netns exec client dhcpcd -U cli0
```

The configuration and client lease showed `8.8.8.8`, while the required internal resolver was `192.168.19.10`. The grading checker also identified the DHCP resolver problem.

**Evaluation**

Both simulated suggestions were useful because they checked the configuration and the actual client result. The checker provided additional confirmation that the DHCP setting failed the lab requirements.

### 4. Wrong or Potentially Misleading AI Suggestion

A useful way to test AI reliability is to consider a plausible but premature troubleshooting recommendation.

**AI tool:** Hypothetical AI response — not an actual recorded response.

**Suggestion:**

> Since the client cannot resolve DNS names, change BIND's `listen-on` and `allow-query` settings to allow all interfaces and clients.

**Why we questioned it:**

The initial symptom did not prove that BIND was refusing client queries. The server responded to a TCP reverse lookup, and the network-level ping succeeded. Broadening BIND's listening and query permissions without checking the existing configuration could introduce unnecessary changes and would not directly explain a firewall rule dropping UDP port 53.

**How we verified it:**

```bash
named-checkconf -p

ss -ulntp | grep ':53'

ip netns exec client dig @192.168.19.10 -x 192.168.19.10 +tcp

iptables -L INPUT -n -v --line-numbers
```

The TCP query succeeded, and the firewall DROP counter increased after the UDP test. These findings supported investigating the firewall before changing BIND's interface and query permissions.

**Conclusion:**

The suggestion was premature rather than necessarily incorrect in every possible situation. BIND listening and query permissions can cause DNS problems in other environments, but our evidence pointed to UDP filtering. We learned that a technically possible solution is not automatically the correct solution for the fault being investigated.

### 4. Wrong or Misleading AI Suggestions

During our troubleshooting, we considered whether changing BIND's `listen-on` and `allow-query` settings could resolve the DNS timeout. However, we needed to verify the cause before making configuration changes.

- **AI tool:** ChatGPT (example scenario; verify against the actual conversation)
- **Suggestion:** Change BIND's `listen-on` and `allow-query` settings to allow DNS requests from the client.
- **Why we questioned it:** A DNS timeout does not necessarily mean that BIND is listening on the wrong interface or refusing client queries. In our investigation, the client could reach the server using ping, and a reverse DNS lookup over TCP succeeded. These observations suggested that we needed to investigate the difference between UDP and TCP before changing BIND's settings.
- **How we verified it:**

  ```bash
  named-checkconf -p

  ss -ulntp | grep ':53'

  ip netns exec client dig @192.168.19.10 -x 192.168.19.10 +tcp

  ip netns exec client dig @192.168.19.10 ns1.novapeak.lab +tries=1 +time=2

  iptables -L INPUT -n -v --line-numbers
  ```

  The TCP reverse lookup succeeded, while the UDP DNS query timed out. We also found an inbound firewall rule dropping UDP traffic on port 53. After testing a UDP query, the packet counter for that DROP rule increased, providing direct evidence that the firewall was blocking the request.

- **Conclusion:** The suggestion to change BIND's listening and query permissions was not supported by the evidence as the first fix. Those settings can cause DNS problems in other situations, but our investigation identified the firewall's UDP/53 DROP rule as the cause of the timeout. We therefore prioritized the confirmed firewall fault instead of making unnecessary BIND changes.

This case taught us that an AI-generated suggestion may be technically possible without being the correct solution for a particular environment. We should test each hypothesis against actual command output before changing a server configuration.

**Important note:** This is a realistic example based on our investigation, not proof that ChatGPT actually gave us this suggestion. We should identify it as an actual AI mistake only if our conversation history confirms that ChatGPT made this recommendation. Otherwise, we should label it as a hypothetical example or replace it with a genuine suggestion from our AI testing.

---

## Part 3d: Verdict

### 1. Where Our Human Diagnosis Was Stronger

Our main advantage was being able to inspect the live lab environment and collect direct evidence. We checked the server and client separately, compared UDP and TCP behavior, inspected BIND configuration and logs, examined the DHCP lease, and ran the grading checker.

We also followed the faults in stages. Initially, DNS queries timed out. After removing the UDP firewall rule, the forward-zone error became visible. After correcting the zone-file path, we could identify the incorrect API address. Finally, correcting the DHCP DNS-server option allowed the client to use the required internal DNS server.

This process helped us distinguish a possible explanation from a fault confirmed by evidence.

### 2. Where AI Genuinely Helped

AI was useful for explaining unfamiliar commands and technical concepts, organizing troubleshooting steps, and helping us understand what different DNS responses might indicate.

Commands such as `named-checkconf -z`, `named-checkzone`, `dig +tcp`, `iptables -L INPUT -n -v`, and `dhcpcd -U cli0` helped us investigate the BIND, firewall, and DHCP components systematically.

However, a command explanation or proposed fix was not enough by itself. We accepted a diagnosis only when it matched our observations and could be verified in the lab.

### 3. What We Learned About Trusting AI

AI can explain common Linux networking problems and suggest useful commands, but it cannot automatically know the current state of a live server unless relevant configuration, logs, and command output are provided.

A DNS timeout can have several possible causes. Changing BIND configuration immediately could waste time or introduce another problem if the actual cause is a firewall rule. Similarly, a syntactically valid DNS zone can still contain an incorrect IP address, and a running DHCP service can still advertise the wrong DNS server.

We learned to treat AI suggestions as hypotheses rather than facts. Our troubleshooting process was to:

1. Observe the symptom.
2. Form a hypothesis.
3. Collect evidence using appropriate commands.
4. Make one targeted change at a time.
5. Retest from the client.
6. Run the grading checker to confirm the result.

Our final result was **25/25**. This experience showed us that reliable troubleshooting depends on technical understanding, evidence, and verification—not simply accepting an AI-generated answer.

---

## Evidence Files

The main supporting evidence for this critique is:

- `session.log` — recorded terminal commands and outputs.
- `INCIDENT_LOG.md` — the investigation, proof, fixes, and confirmation for each fault.
- `checker_result.txt` — the final grading checker result.
- `fixed-configs/` — the repaired configuration files.

Before submission, verify that the quoted command output matches the actual recorded session and replace all AI comparison placeholders with the actual responses from the tools used.