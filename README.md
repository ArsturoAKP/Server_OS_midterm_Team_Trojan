# Server OS Midterm: Lab 01 (novapeak)

**Class Name:** Server OS (Tuesday Morning)

**Class Code:** 192-442(Group 02)

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

## Result

Checker (`python3 /root/grading/check_lab.py --group lab01 --phase build`): **25 / 25**.
See `checker_result.txt`.

## Repository contents

| File / folder | Contents |
|---|---|
| `fixed-configs/` | Repaired `named.conf.options`, `named.conf.local`, `db.novapeak.lab`, `db.novapeak.lab.rev`, `dhcpd.conf`, `isc-dhcp-server` |
| `INCIDENT_LOG.md` | How each of the 4 faults was found, proved, fixed and confirmed (Part 2) |
| `AI_CRITIQUE.md` | AI usage declaration, comparison and verdict (Part 3) |
| `session.log` | Recorded shell session (`script -a /root/session.log`) |
| `checker_result.txt` | Final checker output |

## Faults found and fixed

| # | Fault | Fix |
|---|---|---|
| 1 | iptables rule dropping inbound UDP port 53 (not in any config file) | `iptables -D INPUT -p udp --dport 53 -j DROP` |
| 2 | Forward zone pointed to the missing file `/etc/bind/db.MISSING` | `file "/etc/bind/db.novapeak.lab";` in `named.conf.local` |
| 3 | `api` A record was 192.168.19.1 (the gateway) | `api IN A 192.168.19.40`, serial bumped to 2024010102 |
| 4 | DHCP advertised DNS server 8.8.8.8 | `option domain-name-servers 192.168.19.10;` in `dhcpd.conf` |