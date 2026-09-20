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
| pentestvm | ubuntu-noble-x86_64 | 198.51.100.10  | Trainee workstation. |
| web-gw01  | ubuntu-noble-x86_64 | 203.0.113.5    | nginx on 80/443/18443. Virtual hosts for `www`, `portal`, `partners` and an administration gateway. Two certificates: `*.cadmus-corp.lab` on 443 and `admin.cadmus-corp.lab` (SAN `edge-admin.cadmus-corp.lab`) on 18443 — deliberately outside nmap's default port set. |
| mail01    | ubuntu-noble-x86_64 | 203.0.113.9    | Postfix and Dovecot on 25/465/587/993. Everything else is dropped, so it does not answer default host discovery. |
| vpn01     | ubuntu-noble-x86_64 | 203.0.113.13   | nginx on 443 plus strongSwan IKEv2 on UDP 500/4500. Its certificate is `CN=vpn.cadmus-corp.lab` with `remote.cadmus-corp.lab` in the SAN list. |

All five nodes run on `standard.small`.

## Provisioning

`provisioning/playbook.yml` runs a common Linux baseline and then one play per
host:

- **baseline** — disables unattended upgrades, installs common packages, sets
  hostnames and an SSH banner.
- **router** — enables IPv4 forwarding and installs it persistently.
- **pentestvm** — installs the external scanning toolset, adds a persistent
  route to the DMZ, stages a virtual-host wordlist, and provisions the trainee
  login `user` / `Password123` (sudo) via the `user-access` role.
- **targets** — add a return route to the attacker segment, then deploy their
  services and an iptables ruleset that exposes only the intended ports.

The per-host filter scripts run as oneshot systemd units and derive the DMZ
interface at boot from the route table.

## Trainee workflow

1. Console into **pentestvm** as `user` / `Password123`.
2. Work out which gateway carries traffic to `203.0.113.0/28`.
3. Sweep the range. It will not show every host — one target permits only its
   own service ports and drops discovery probes, so it has to be found by
   scanning past host discovery entirely.
4. Fingerprint each host, including UDP where it matters.
5. Inspect the certificates. A Common Name is rarely the whole story.
6. Submit the alternate hostname disclosed in the VPN host's certificate.
7. A ten-question knowledge check follows.

## Flags

`variables.yml` is empty: this lab uses static answers rather than APG
variables, because every answer is information read off a live service rather
than a planted secret.

## Tools used

`nmap` (TCP, UDP and NSE), `openssl s_client`, `ike-scan`, `ffuf`, `gobuster`,
`swaks`, `dig`, `whois`, `curl`, `netcat`.

## MITRE mapping

- `T1590` Gather Victim Network Information
- `T1595` Active Scanning
- `T1596` Search Open Technical Databases
