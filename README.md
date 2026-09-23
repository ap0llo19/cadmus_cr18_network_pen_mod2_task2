# CR-18 / Module 2 / N2d — External Network Enumeration

CADMUS Cyber Range scenario. A perimeter assessment against a client's public
`/28` from an outside vantage point. The trainee works out how traffic reaches
the range, finds the hosts a default sweep misses, fingerprints the edge
services, and pulls a hostname out of a TLS certificate that was never meant
to be public.

## Topology

Two routed segments. The trainee sits on `198.51.100.0/28` and the targets are
on `203.0.113.0/28`; the router forwards between them. Both are RFC 5737
documentation ranges, so nothing here can collide with real address space.

| Node      | Image               | IP             | Role |
| --------- | ------------------- | -------------- | ---- |
| router    | debian-12-x86_64    | 198.51.100.1 / 203.0.113.1 | Forwards between the attacker segment and the DMZ. |
| vma       | kali-2026.1-x86_64  | 198.51.100.10  | Trainee workstation. |
| web-gw01  | ubuntu-noble-x86_64 | 203.0.113.5    | nginx on 80/443/18443. Virtual hosts for `www`, `portal`, `partners` and an administration gateway. Two certificates: `*.cadmus-corp.lab` on 443 and `admin.cadmus-corp.lab` (SAN `edge-admin.cadmus-corp.lab`) on 18443 — deliberately outside nmap's default port set. |
| mail01    | ubuntu-noble-x86_64 | 203.0.113.9    | Postfix and Dovecot on 25/465/587/993. Everything else is dropped, so it does not answer default host discovery. |
| vpn01     | ubuntu-noble-x86_64 | 203.0.113.13   | nginx on 443 plus strongSwan IKEv2 on UDP 500/4500. Its certificate is `CN=vpn.cadmus-corp.lab` with `remote.cadmus-corp.lab` in the SAN list. |

`vma` runs on `c2_r4_d30` — Kali's image sets a 25 GiB `min_disk`, so `standard.small` is not an option. The four other nodes are `standard.small`.

## Provisioning

`provisioning/playbook.yml` runs a common Linux baseline and then one play per
host:

- **baseline** — disables unattended upgrades, installs common packages, sets
  hostnames and an SSH banner.
- **router** — enables IPv4 forwarding and installs it persistently.
- **vma** — installs the external scanning toolset, adds a persistent
  route to the DMZ, stages a virtual-host wordlist, and provisions the trainee
  login `user` / `Password123` (sudo) via the `user-access` role.
- **targets** — add a return route to the attacker segment, then deploy their
  services and an iptables ruleset that exposes only the intended ports.

The per-host filter scripts run as oneshot systemd units and derive the DMZ
interface at boot from the route table.

## Trainee workflow

Eleven levels, 200 points, roughly 50 minutes. Five graded hands-on levels,
then a short knowledge check.

1. Open the Kali desktop on **vma** (`Open GUI`) and log in as
   `user` / `Password123`.
2. *Background* — routing to the range, and host discovery.
3. **Reach what you cannot see** — route to the range, sweep it, notice three
   replies in a sixteen-address block, and find the host discovery misses
   (`203.0.113.9`, which permits only its own mail ports).
4. *Background* — full-range and UDP scanning, certificates, IKE, banners.
5. **The port nobody scans** — tcp/18443 on web-gw01, deliberately outside
   nmap's top 1000; its certificate discloses `edge-admin.cadmus-corp.lab`.
6. **What the certificate gives away** — vpn01's SAN discloses
   `remote.cadmus-corp.lab`.
7. **Beyond TCP** — a UDP scan of vpn01; nmap names 500/udp `isakmp`. None of
   this appears in any TCP scan.
8. **Mail, in its own words** — the SMTP greeting on the host from step 3
   announces `mail.cadmus-corp.lab`.
9. **Knowledge check** — five questions on the gateway, web-gw01, mail01 and
   vpn01, answerable from output already produced.

The middle levels are one theme: the perimeter answers what you ask it, and
the default questions are not very searching.

## Flags

`variables.yml` is empty and every answer is static. Unlike N2c, nothing here
is a planted secret — each answer is a fact read off a live service: a
hostname in a certificate, a service name, an IP address.

APG cannot help with these. The platform accepts either a fixed `answer` or an
`answer_variable_name`, never a composition of the two (see `97ba6d9` in
`cr-18-n3b-intermediate-arp`), so an APG-bound answer must be the entire
submitted string. APG can generate a label; it cannot generate
`<label>.cadmus-corp.lab`. Randomising the hostnames would cost realism in the
certificates without making any answer per-sandbox.

If per-sandbox grading becomes a requirement, the route is to plant a genuine
secret — a flag served by the admin gateway on 18443 — and demote the
hostname to a knowledge-check question.

## Tools used

`nmap` (TCP, UDP and NSE), `openssl s_client`, `ike-scan`, `swaks`, `netcat`,
`ffuf`. `gobuster`, `dig`, `whois` and `curl` are installed but not required by
any level.

## MITRE mapping

- `T1590` Gather Victim Network Information
- `T1592` Gather Victim Host Information
- `T1595` Active Scanning

`T1596` (Search Open Technical Databases) was previously mapped but is not
exercised: this lab reads certificates off live services rather than querying
any public database.
