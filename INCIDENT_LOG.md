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

## Baseline (before any fix): 11 / 25

```
# python3 /root/grading/check_lab.py --group lab01 --phase build
  [   1/2 ] C4   named-checkconf -z FAILED; named-checkzone clean
  [   2/2 ] C5   dhcpd -t clean; service running
  [   0/2 ] C1   client cannot reach named at 192.168.19.10 (timeout/no status). Check listen-on, allow-query, firewall, and that it serves on lan0.
  [ 0.0/3 ] C2   ns1 expected 192.168.19.10 got nothing; www expected 192.168.19.20 got nothing; mail expected 192.168.19.30 got nothing; api expected 192.168.19.40 got nothing
  [   0/2 ] C3   reverse lookup of 192.168.19.10 did not return ns1 (got nothing). Check the reverse zone and its PTR.
  [   3/3 ] C6   client obtained a lease: 192.168.19.104
  [ 2.0/2 ] C7   pool+netmask correct
  [   2/2 ] C8   gateway advertised correctly
  [   0/3 ] C9   DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10 -- this is the integration the whole project is about
  [   1/1 ] C10  company domain advertised
  [   0/3 ] C11  using the leased resolver 8.8.8.8, www did not resolve to 192.168.19.20 (got nothing). If the leased resolver is a public one, that is the integration bug.
  --------------------------------------------------
  TOTAL: 11.0 / 25
```

## Summary of findings

| No. | Area | Finding | Evidence (command) | Status |
| --- | --- | --- | --- | --- |
| 1 | Server firewall | `-A INPUT -p udp -m udp --dport 53 -j DROP` | `iptables -S`; counter 0 → 1 after one client query | Fault (fixed) |
| 2 | BIND config | forward zone file is `/etc/bind/db.MISSING` | `named-checkconf -z`, journal | Fault (fixed) |
| 3 | Forward zone | `api IN A 192.168.19.1` (gateway), should be .40 | client `dig` vs `params.py` | Fault (fixed) |
| 4 | DHCP | `option domain-name-servers 8.8.8.8;` | `dhcpcd -U cli0`, `dhcpd.conf`, C9/C11 | Fault (fixed) |
| 5 | Reverse zone | PTRs correct with trailing dots | TCP `dig -x 192.168.19.10` → NOERROR, aa | OK |
| 6 | Client firewall | no rules, empty nft ruleset | `ip netns exec client iptables -S` / `nft list ruleset` | OK |
| 7 | named startup | runs as `bind` (`OPTIONS="-u bind"`) | `ps -o user,pid,cmd -C named`, `/etc/default/named` | OK |
| 8 | listen-on / allow-query | `any` / `any`; listening on 192.168.19.10:53 UDP+TCP | `named-checkconf -p`, `ss -ulntp` | OK |
| 9 | Zone file ownership | `bind:bind -rw-r--r--`, dir `root:bind`; AppArmor not blocking | `namei -l`, `runuser -u bind -- cat` → "bind CAN read" | OK |
| 10 | `/etc/default/isc-dhcp-server` | `INTERFACESv4="lan0"` | `cat`; journal "Listening on LPF/lan0" | OK |
| 11 | DHCP range / router / domain | .104–.144, router .1, `novapeak.lab` | `dhcpd.conf` vs `params.py`; C7, C8, C10 pass | OK |
| 12 | Client → server layer 3 | ping 0% loss | `ip netns exec client ping -c 3 192.168.19.10` | OK |
| 13 | Saved firewall rules | none in `/etc/iptables/`; `nftables.conf` has no DROP | `ls /etc/iptables`, `cat /etc/nftables.conf` | OK |

Order of discovery: each fix exposed the next fault. The score went from
**timeout → SERVFAIL → wrong api IP → wrong resolver**, and from
11 → 19 → 25 / 25.

---

## Fault 1: Firewall drops DNS queries (UDP port 53)

- **Symptom (what we observed):**
  ```
  # ip netns exec client ping -c 3 192.168.19.10
  3 packets transmitted, 3 received, 0% packet loss, time 2060ms

  # ip netns exec client dig @192.168.19.10 ns1.novapeak.lab +tries=1 +time=2
  ;; communications error to 192.168.19.10#53: timed out
  ;; no servers could be reached
  ```
- **Hypothesis (what we suspected and why):**
  Ping works, so the client reaches the server at layer 3. DNS gets a
  **timeout**, not REFUSED or SERVFAIL, which means no reply came back at all.
  The checker's C1 points at listen-on, allow-query or a firewall.
  `named-checkconf -p` shows `listen-on { any; }` and `allow-query { any; }`,
  and `ss` shows named on `192.168.19.10:53` (UDP and TCP). That ruled out the
  first two and left a firewall. If only UDP is blocked, TCP should get through.
- **Proof (the command that confirmed the cause):**
  ```
  # ss -ulntp | grep ':53'
  udp   UNCONN 0  0   192.168.19.10:53   0.0.0.0:*   users:(("named",pid=2452,fd=25))
  tcp   LISTEN 0  10  192.168.19.10:53   0.0.0.0:*   users:(("named",pid=2452,fd=26))

  # ip netns exec client dig @192.168.19.10 -x 192.168.19.10 +tries=1 +time=2
  ;; communications error to 192.168.19.10#53: timed out
  ;; no servers could be reached

  # ip netns exec client dig @192.168.19.10 -x 192.168.19.10 +tcp
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16833
  ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
  ;; ANSWER SECTION:
  10.19.168.192.in-addr.arpa. 3600 IN	PTR	ns1.novapeak.lab.
  ;; SERVER: 192.168.19.10#53(192.168.19.10) (TCP)

  # iptables -S
  -A INPUT -p udp -m udp --dport 53 -j DROP
  ```
  The same query times out over UDP but succeeds over TCP, so named works and
  only UDP/53 is blocked. To prove this exact rule drops the client's
  packets, we reset the counters and sent one query:
  ```
  # iptables -Z INPUT
  # ip netns exec client dig @192.168.19.10 www.novapeak.lab +tries=1 +time=2
  ;; communications error to 192.168.19.10#53: timed out
  # iptables -L INPUT -n -v --line-numbers
  num   pkts bytes target     prot opt in     out     source               destination
  1        1    85 DROP       17   --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
  ```
  The counter went from 0 to exactly 1 packet: our one query. No
  configuration file contains this rule; only the running firewall shows it.
  We also checked it is not saved for boot: `/etc/iptables/` does not exist,
  and `/etc/nftables.conf` contains no DROP rule.
- **Fix (what we changed):**
  `iptables -D INPUT -p udp --dport 53 -j DROP`. This is a runtime change, and
  rebuilding the lab adds the rule back.
- **Confirmation (output showing it now works):**
  ```
  # iptables -S INPUT
  -P INPUT ACCEPT

  # ip netns exec client dig @192.168.19.10 -x 192.168.19.10
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15675
  ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
  10.19.168.192.in-addr.arpa. 3600 IN	PTR	ns1.novapeak.lab.
  ;; SERVER: 192.168.19.10#53(192.168.19.10) (UDP)
  ```
  UDP queries now get answers. The forward lookup now returns SERVFAIL
  instead of timing out, which exposed Fault 2.
- **Found/proved by:** Aung Kyaw Phyo

---

## Fault 2: Forward zone points to a file that does not exist

- **Symptom (what we observed):**
  With the firewall fixed, the server replies, but with an error for its own
  domain, while the reverse zone works (output above):
  ```
  # ip netns exec client dig @192.168.19.10 www.novapeak.lab
  ;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 20885
  ;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
  ;; SERVER: 192.168.19.10#53(192.168.19.10) (UDP)
  ```
  The same SERVFAIL was already visible over TCP before the firewall fix
  (`dig ... ns1.novapeak.lab +tcp` → `status: SERVFAIL, id: 35582`). The
  checker reported `C4 named-checkconf -z FAILED`.
- **Hypothesis (what we suspected and why):**
  The reverse zone answers authoritatively (`aa`), so named itself works. The
  forward zone gives SERVFAIL with no `aa` flag and no answer, which means
  named is configured for `novapeak.lab` but the zone did not load.
- **Proof (the command that confirmed the cause):**
  ```
  # named-checkconf -z
  zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
  zone novapeak.lab/IN: not loaded due to errors.
  _default/novapeak.lab/IN: file not found
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101

  # grep -A3 'zone "novapeak.lab"' /etc/bind/named.conf.local
  zone "novapeak.lab" {
  	type master;
  	file "/etc/bind/db.MISSING";
  };

  # ls -l /etc/bind/db.MISSING
  ls: cannot access '/etc/bind/db.MISSING': No such file or directory
  # ls -l /etc/bind/db.novapeak.lab
  -rw-r--r-- 1 bind bind 212 Oct 17 07:06 /etc/bind/db.novapeak.lab

  # journalctl -u named --no-pager | grep -i missing | tail -5
  Oct 17 07:06:25 ubuntu named[2452]: zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
  ```
  The correct zone file exists, but the config points to `db.MISSING`, which
  doesn't.
- **Fix (what we changed):**
  Backed up `named.conf.local`, then changed `file "/etc/bind/db.MISSING";` to
  `file "/etc/bind/db.novapeak.lab";` (nano). Ran `named-checkconf -z`, then
  `systemctl restart named`.
- **Confirmation (output showing it now works):**
  ```
  # named-checkconf -z
  zone novapeak.lab/IN: loaded serial 2024010101
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101

  # ip netns exec client dig @192.168.19.10 novapeak.lab SOA
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 22316
  ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
  novapeak.lab.		3600	IN	SOA	ns1.novapeak.lab. admin.novapeak.lab. 2024010101 3600 1800 604800 3600

  # ip netns exec client dig @192.168.19.10 www.novapeak.lab +short
  192.168.19.20
  ```
- **Found/proved by:** L Peter San Awng

---

## Fault 3: `api.novapeak.lab` points to the gateway (192.168.19.1) instead of 192.168.19.40

- **Symptom (what we observed):**
  Once the zone loaded, we checked all four hosts from the client:
  ```
  # for h in ns1 www mail api; do echo -n "$h: "; ip netns exec client dig @192.168.19.10 $h.novapeak.lab +short; done
  ns1: 192.168.19.10
  www: 192.168.19.20
  mail: 192.168.19.30
  api: 192.168.19.1
  ```
- **Hypothesis (what we suspected and why):**
  `named-checkzone` reported OK, so the record's syntax is valid but its value
  can still be wrong. 192.168.19.1 is the network's gateway (the client's
  `default via 192.168.19.1`), not a service host. We suspected a wrong value.
- **Proof (the command that confirmed the cause):**
  ```
  # python3 /root/grading/params.py lab01 | grep api
      api.novapeak.lab        -> 192.168.19.40

  # grep api /etc/bind/db.novapeak.lab
  api IN A 192.168.19.1

  # grep '^40' /etc/bind/db.novapeak.lab.rev
  40 IN PTR api.novapeak.lab.

  # ip netns exec client dig @192.168.19.10 -x 192.168.19.1 +short
  (no answer)
  ```
  The scenario card and the reverse zone both say api is .40, and .1 has no
  PTR. The forward record is wrong. `named-checkzone` cannot catch this kind
  of data error.
- **Fix (what we changed):**
  Backed up the zone file, then in `/etc/bind/db.novapeak.lab` changed
  `api IN A 192.168.19.1` to `api IN A 192.168.19.40` and increased the SOA
  serial from 2024010101 to 2024010102. Kept the file owned by `bind:bind`,
  then restarted named.
- **Confirmation (output showing it now works):**
  ```
  # named-checkzone novapeak.lab /etc/bind/db.novapeak.lab
  zone novapeak.lab/IN: loaded serial 2024010102
  OK

  # for h in ns1 www mail api; do echo -n "$h: "; ip netns exec client dig @192.168.19.10 $h.novapeak.lab +short; done
  ns1: 192.168.19.10
  www: 192.168.19.20
  mail: 192.168.19.30
  api: 192.168.19.40
  ```
  The checker then scored **19 / 25**: C1, C2, C3 and C4 now pass, and only
  C9/C11 still fail.
- **Found/proved by:** Aung Kyaw Phyo

---

## Fault 4: DHCP gives clients a public DNS server (8.8.8.8)

- **Symptom (what we observed):**
  DNS now works when queried directly, but a new client is still told to use
  8.8.8.8:
  ```
  # ip netns exec client dhcpcd -U cli0 | grep -E 'ip_address|routers|domain_name'
  ip_address=192.168.19.104
  routers=192.168.19.1
  domain_name_servers=8.8.8.8
  domain_name=novapeak.lab

  # python3 /root/grading/check_lab.py --group lab01 --phase build
    [   0/3 ] C9   DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10 -- this is the integration the whole project is about
    [   0/3 ] C11  using the leased resolver 8.8.8.8, www did not resolve to 192.168.19.20 (got nothing). If the leased resolver is a public one, that is the integration bug.
    TOTAL: 19.0 / 25
  ```
- **Hypothesis (what we suspected and why):**
  The lease itself is fine (C6, C7, C8, C10 pass). A machine joining the
  network only knows the DNS server that DHCP gives it. 8.8.8.8 is a public
  resolver and has never heard of the private `novapeak.lab` zone, so a new
  client still cannot resolve company names.
- **Proof (the command that confirmed the cause):**
  ```
  # ip netns exec client dig @8.8.8.8 www.novapeak.lab +tries=1 +time=2
  ;; communications error to 8.8.8.8#53: timed out
  ;; no servers could be reached

  # grep -n domain-name-servers /etc/dhcp/dhcpd.conf
  8:	option domain-name-servers 8.8.8.8;
  ```
  The resolver the client was given cannot answer for the company's names,
  and the DHCP config is what hands it out.
- **Fix (what we changed):**
  Backed up `dhcpd.conf`, then changed `option domain-name-servers 8.8.8.8;`
  to `option domain-name-servers 192.168.19.10;`. Ran `dhcpd -t` (clean) and
  `systemctl restart isc-dhcp-server`. The client keeps its old lease until it
  renews, so we released and renewed it with `dhcpcd -k cli0` and
  `dhcpcd -b cli0`.
- **Confirmation (output showing it now works):**
  ```
  # ip netns exec client dhcpcd -U cli0 | grep -E 'ip_address|routers|domain_name'
  ip_address=192.168.19.104
  routers=192.168.19.1
  domain_name_servers=192.168.19.10
  domain_name=novapeak.lab

  # journalctl -u isc-dhcp-server -n 5 --no-pager
  Oct 17 07:29:23 ubuntu dhcpd[3449]: DHCPRELEASE of 192.168.19.104 from 06:05:1c:96:89:7d via lan0 (found)
  Oct 17 07:29:34 ubuntu dhcpd[3449]: DHCPDISCOVER from 06:05:1c:96:89:7d via lan0
  Oct 17 07:29:35 ubuntu dhcpd[3449]: DHCPOFFER on 192.168.19.104 to 06:05:1c:96:89:7d via lan0
  Oct 17 07:29:35 ubuntu dhcpd[3449]: DHCPREQUEST for 192.168.19.104 (192.168.19.10) from 06:05:1c:96:89:7d via lan0
  Oct 17 07:29:35 ubuntu dhcpd[3449]: DHCPACK on 192.168.19.104 to 06:05:1c:96:89:7d via lan0

  # ip netns exec client dig @192.168.19.10 www.novapeak.lab +short
  192.168.19.20
  ```
- **Found/proved by:** L Peter San Awng

---
