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
