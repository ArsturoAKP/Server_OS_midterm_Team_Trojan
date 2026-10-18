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
| Claude (original session, during the lab) | Explained commands and concepts; corrected our notes (`dhcpcd` instead of `dhclient`, broken `dig` syntax); helped us read outputs. Every command was run by us on the live lab; all output comes from our `session.log`. |
| ChatGPT | Part 3c test subject only (AI #1). |
| Claude (brand-new session, no prior context) | Part 3c test subject only (AI #2). It is the same product we used during the lab, but the new session had no access to that conversation. |

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

### 1. Test design

After completing our own diagnosis (Part 3b), we gave the same input to two AI tools: **AI #1 ChatGPT** and **AI #2 Claude** (a brand-new session with no access to our earlier conversation). We ran two rounds:

- **Round 1 (configs only):** the six broken configuration files and the goal. By mistake, our baseline checker output was left out (a placeholder line was sent instead), and both AIs pointed this out.
- **Round 2 (one live symptom):** a follow-up in the same chat giving the UDP-vs-TCP evidence from our own test.

Round 1 tests what an AI can deduce from files alone. Round 2 tests what it can do once a human supplies evidence from the running system.

**Round 1 prompt** (followed by the six config files exactly as they were before our fixes):

> Our company network is broken: a client that joins via DHCP cannot resolve the company's own host names (domain novapeak.lab). The DNS server is 192.168.19.10 on interface lan0. Here are the server's config files and the output of our test checker. Find every fault and tell us how to fix it.

**Round 2 prompt:**

> From the client, ping to 192.168.19.10 works, but "dig @192.168.19.10 -x 192.168.19.10" times out over UDP, while the same query with +tcp returns NOERROR with the aa flag. named-checkconf shows listen-on any and allow-query any. What does this tell you?

We judged every AI claim against our own evidence (`session.log`, `INCIDENT_LOG.md`, and the final 25/25 checker result), asking: was it correct, was it supported by evidence, and would following it have caused unnecessary changes?

### 2. Comparison table

| Fault (our words) | We found it? | AI #1 ChatGPT | AI #2 Claude | Was each AI actually right? (how we verified) |
| --- | --- | --- | --- | --- |
| 1. iptables DROP on inbound UDP/53 | Yes | R1: **missed**. R2: correctly concluded UDP/53 filtering; suggested `ufw`, `nft list ruleset`, `ss`, `tcpdump`, but did not name `iptables`. | R1: **missed**, and said "items 1 and 2 alone explain the symptom". R2: correctly concluded a firewall DROP; explained why a timeout (not REFUSED or port-unreachable) means packets are silently dropped; suggested `iptables -L -n -v --line-numbers`, the exact command we used. | Neither could find it from files, because no config file contains the rule. Both found the right direction only after we supplied the UDP-vs-TCP evidence. Verified by our counter test (`iptables -Z`, one query, DROP counter 0 → 1). |
| 2. Forward zone points to `db.MISSING` | Yes | R1: found and fixed correctly. | R1: found, fixed correctly, and correctly predicted SERVFAIL. | Both correct. Matches `named-checkconf -z` ("file not found") and the SERVFAIL we saw over TCP. |
| 3. `api` A record is .1 (gateway) instead of .40 | Yes | R1: flagged as "likely" wrong; refused to change it blindly and said to confirm the intended IP first. | R1: said "almost certainly .40", reasoning from the `40 PTR api` record and .1 being the router. | Both right. ChatGPT was appropriately cautious: from files alone the intended IP cannot be proven. We confirmed .40 from the scenario card (`params.py lab01`), which neither AI had. |
| 4. DHCP advertises 8.8.8.8 | Yes | R1: found and fixed correctly (also duplicated the option globally, which is harmless). | R1: found and fixed correctly; explained that a public resolver cannot know a private zone. | Both correct. Verified by `dhcpcd -U cli0` (8.8.8.8 → 192.168.19.10) and C9/C11 passing. |
| Extra "faults" that were not faults | n/a | Listed `recursion no` as a fault (then said "not necessarily"); rewrote `listen-on` and `allow-query` to restrict them. | Claimed a broken PTR line (`www`); enabled recursion with forwarders; added `authoritative;`; said "some checkers test" serials. | None of these were needed: our checker scored 25/25 with the original `named.conf.options` unchanged and no `authoritative;`. See section 3. |

### 3. Wrong or misleading AI suggestions

**1. Claude: "Items 1 and 2 alone explain the symptom."** This is wrong and stated with confidence. Fixing the zone path and the DHCP resolver would still have left every UDP query timing out. Our baseline C1 showed a timeout (not SERVFAIL), and the DROP counter rose from 0 to 1 on a single client query. Only removing the rule fixed UDP. (ChatGPT made a softer version of the same claim: "Faults 1 and 2 directly explain why internal name resolution can fail.")

**2. ChatGPT: restrict `listen-on` and `allow-query`.** This was presented as a fault, but these settings were not causing any problem. `named-checkconf -p` and `ss -ulntp` showed named listening on 192.168.19.10:53 and accepting queries, and the final 25/25 used the original `any`/`any`. Changing reachability settings while a reachability problem is still undiagnosed risks adding a second fault. Notably, in Round 2 ChatGPT itself said `listen-on` and `allow-query` "do not explain the UDP-only failure".

**3. Claude: enable `recursion yes` with forwarders to 8.8.8.8 / 1.1.1.1.** This is unnecessary. The goal is to resolve the company's own names, which needs only authoritative answers. We passed 25/25 with `recursion no`. The lab also has no internet route, so forwarders could never work (our `dig @8.8.8.8` from the client timed out).

**4. Claude: "broken PTR line" for `www` in the reverse zone.** This is a false positive caused by formatting from copy-pasting (the chat turned `www.novapeak.lab` into a link). Claude did hedge ("if that text is really in the file"). Our real file was valid: `named-checkzone 19.168.192.in-addr.arpa` returned OK, and the reverse lookup over TCP returned NOERROR with `aa`. In Round 2, Claude correctly reasoned that the TCP success meant the reverse zone had loaded.

**5. Both: client-side steps that do not fit this environment.** Claude said to renew with `dhclient`, and ChatGPT pointed to NetworkManager and `/etc/resolv.conf`. Our client is a network namespace using `dhcpcd` (we used `dhcpcd -k cli0` / `dhcpcd -b cli0` and `dhcpcd -U cli0`). This is the same `dhclient` mistake we hit during the lab.

**6. Both: testing "from the server".** Both suggested `dig @192.168.19.10` from the server itself. The handout warns this "proves almost nothing", and in our case the server would have answered while the client timed out because of the firewall. We tested every fix from the client namespace.

***7. Claude Round 2 fix: `sudo iptables -I INPUT -i lan0 -p udp --dport 53 -j ACCEPT`.** This would work, because `-I` inserts the ACCEPT rule at the top of the INPUT chain, above the DROP rule, so UDP queries on lan0 would be accepted before reaching the DROP. But it leaves the faulty rule in place: the chain would then contain both an ACCEPT and a DROP for the same traffic, which hides the real cause from the next administrator. Claude also told us to make the change persistent; `netfilter-persistent save` would have saved the DROP rule along with the ACCEPT. We deleted the DROP rule itself with `iptables -D INPUT -p udp --dport 53 -j DROP`, which removes the cause instead of working around it. Verified with `iptables -S INPUT`, which then showed only `-P INPUT ACCEPT`.

---

## Part 3d: Verdict

### 1. Where our human diagnosis was better

The biggest difference was that we could see the live system and the AIs could not. In Round 1, both AIs only had the config files, and both missed the firewall rule. This makes sense, because the rule was not in any config file. It only existed in the running firewall. Claude even said that the zone file and DHCP faults "alone explain the symptom". That was wrong. Even with those two faults fixed, every UDP DNS query would still time out.

We found the firewall fault with our own tests. Ping worked, UDP DNS timed out, and TCP DNS worked. Then we reset the iptables counters, sent one query, and saw the DROP counter go from 0 to 1. This proved the cause. The AIs could not do this, because they could not run commands on our server.

We also knew the correct address for `api` (192.168.19.40) from the scenario card. The AIs did not have this information. Claude guessed .40 from the reverse zone, and the guess was right, but it was still a guess. ChatGPT was more careful and said we should confirm the correct IP first. We think ChatGPT was right to be careful here.

### 2. Where AI really helped

In Round 2, we gave both AIs our UDP and TCP test result. After that, both AIs said the problem was UDP port 53 being blocked, which was correct. Claude also explained clearly why a timeout means packets are being dropped, while a fast REFUSED answer would point to a BIND setting. This matches the hint in the midterm handout. Claude also suggested `iptables -L -n -v --line-numbers`, which is the same command we used.

Both AIs found the zone file fault (`db.MISSING`) and the DHCP fault (8.8.8.8) correctly from the config files alone. During the lab, Claude also helped us by correcting our notes: the client uses `dhcpcd`, not `dhclient`, and some of our `dig` commands had mistakes.

So AI was helpful for reading config files and for explaining what an error means. But it only found the hardest fault after a human gave it real evidence from the live system.

### 3. What we learned about trusting AI

**AI adds problems that are not real.** Both AIs listed extra "faults" that were not faults. ChatGPT wanted to change `listen-on` and `allow-query`. Claude wanted to turn on recursion, add forwarders, and add `authoritative;`. Claude also reported a broken PTR line that was only a copy-paste formatting problem. Our checker gave 25/25 without any of these changes. If we had followed every AI suggestion, we would have changed many settings for no reason, and we might have created new problems.

**AI advice does not always fit the system.** Claude said to renew the lease with `dhclient`, and ChatGPT talked about NetworkManager. Our lab client uses `dhcpcd` inside a network namespace, so neither instruction would work. Both AIs also suggested testing DNS from the server. The handout says this proves almost nothing, and in our lab the server would have answered while the client still failed.

**AI can fix the symptom instead of the cause.** In Round 2, Claude suggested adding an ACCEPT rule above the DROP rule. This would work, but the bad rule would still be there. We deleted the DROP rule instead, which removes the real cause.

**A confident answer is not always a correct answer.** Claude sounded very sure in Round 1, but it missed the firewall fault. ChatGPT was less sure about the `api` address, and that was the better answer.

Because of this, we now treat AI answers as ideas to test, not as facts. Our process is:

1. Observe the symptom.
2. Form a hypothesis.
3. Collect evidence with commands.
4. Make one change at a time.
5. Test again from the client.
6. Run the checker to confirm the result.

We reached 25/25 because we proved each fault on the live system before fixing it, not because an AI told us the answer.

---

## Evidence Files

The main supporting evidence for this critique is:

- `session.log` — recorded terminal commands and outputs.
- `INCIDENT_LOG.md` — the investigation, proof, fixes, and confirmation for each fault.
- `checker_result.txt` — the final grading checker result.
- `fixed-configs/` — the repaired configuration files.
