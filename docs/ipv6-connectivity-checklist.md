# IPv6 / Connectivity Diagnostic Checklist

A practical, command-oriented checklist for diagnosing IPv6 and general connectivity problems on enterprise and SaaS-connected systems. The recurring theme: modern clients are **dual-stack**, and IPv4 and IPv6 are often treated differently by DNS, firewalls, and security whitelists — so a service can be broken on one address family while looking fine on the other.

Commands are shown in their common Linux/macOS form. On Windows, use the noted equivalents.

---

## 1. Understand dual-stack and Happy Eyeballs

- A dual-stack client has both IPv4 and IPv6 connectivity and will try both.
- **Happy Eyeballs** (RFC 8305) makes the client race IPv6 and IPv4 connection attempts and prefer whichever completes first, falling back if one stalls. This is why a broken IPv6 path can show up as *intermittent slowness* rather than an outright failure — the client wastes time on IPv6 before falling back to IPv4, or gets stuck when the IPv6 attempt is silently dropped rather than cleanly refused.
- Symptom pattern to watch for: slow initial connections, timeouts that eventually succeed, or one client/site working while another on the same LAN does not.

## 2. Check what addresses the host has

- Linux: `ip -6 addr` (IPv6 addresses per interface); `ip addr` for both families.
- Linux: `ip -6 route` to see the IPv6 default route and whether one exists.
- Windows: `ipconfig /all` (shows IPv4 and IPv6); `netsh interface ipv6 show address`.
- macOS: `ifconfig` and `netstat -rn` for routes.
- Confirm the host actually has a global IPv6 address and a default route before blaming IPv6 — a link-local-only address (`fe80::`) means no real IPv6 reachability.

## 3. Compare AAAA vs. A resolution

DNS returns `A` records for IPv4 and `AAAA` records for IPv6. A service that publishes AAAA but is unreachable over IPv6 is a classic dual-stack trap.

- `dig AAAA <host>` — IPv6 addresses the client will try.
- `dig A <host>` — IPv4 addresses.
- `dig <host> +short` for a quick answer; add `@<resolver>` to test a specific resolver.
- Windows: `nslookup -type=AAAA <host>` and `nslookup -type=A <host>`.
- Question to answer: does the host publish AAAA records at all, and does the IPv6 address it hands back actually work?

## 4. Test each address family directly

Force the tool to use one family so you can compare them side by side.

- `ping -6 <host>` vs. `ping -4 <host>` (Windows: `ping -6` / `ping -4`).
- `curl -6 -v https://<host>/` vs. `curl -4 -v https://<host>/` — the `-v` output shows which address was tried and where it stalled (DNS, TCP connect, TLS).
- `curl --connect-timeout 5 -6 ...` to bound how long a stalled IPv6 attempt hangs.
- `traceroute -6 <host>` vs. `traceroute <host>` (Windows: `tracert -6` / `tracert -4`) to see where the IPv6 path dies versus the IPv4 path.
- If IPv4 succeeds cleanly and IPv6 stalls or is dropped, you have isolated the problem to the IPv6 path — often a firewall or whitelist that never got IPv6 rules.

## 5. Check firewall / whitelist rules that differ between IPv4 and IPv6

This is the most common enterprise failure mode: allow-lists and security rules were written for IPv4 and never mirrored for IPv6.

- Confirm whether outbound/inbound rules, allow-lists, and security policies exist for **both** families or only IPv4.
- On the host: `ip6tables -L -n` (Linux, IPv6 firewall) vs. `iptables -L -n` (IPv4); `nft list ruleset` for nftables; Windows Defender Firewall rules are per-profile and can differ by protocol.
- A dropped (silently discarded) IPv6 connection behaves very differently from a rejected one — dropped connections cause the long stalls Happy Eyeballs then has to time out.
- Ask specifically: is the destination service (or its address range) whitelisted for IPv6, not just IPv4?

## 6. Check MTU / PMTUD

IPv6 relies on Path MTU Discovery and does not fragment in transit; if ICMPv6 "Packet Too Big" messages are filtered, large packets vanish and connections hang after the handshake.

- Test with sized, do-not-fragment pings: Linux `ping -6 -M do -s 1400 <host>`; Windows `ping -6 -f -l 1400 <host>`.
- Reduce the size until it succeeds to find the working MTU; a handshake that completes but transfers then stall is a classic PMTUD/MTU black-hole signature.
- Ensure ICMPv6 is not being blocked wholesale by a firewall — IPv6 needs more of ICMP than IPv4 does to function correctly.

## 7. Isolate application vs. network

- If a browser/app is slow but `curl -4`/`curl -6` and `ping` behave, suspect the application or its config.
- If the raw connectivity tools reproduce the stall or failure, the problem is in the network/path, not the application.
- Test from a second host on the same segment: if it fails too, the cause is shared infrastructure, not the one workstation.

## 8. What to check with security / IT

Bring these questions (and the evidence above) to the team that owns the controls:

- Are firewall, proxy, and allow-list / whitelist rules defined for **both** IPv4 and IPv6, or IPv4 only?
- Is the destination SaaS/cloud service (its hostnames or address ranges) permitted over IPv6, not just IPv4?
- Are IPv6 connections being **dropped silently** rather than cleanly refused (causing timeouts and Happy Eyeballs stalls)?
- Is ICMPv6 (including "Packet Too Big") allowed so PMTUD works?
- Did a recent security or network change add IPv4 coverage without mirroring it for IPv6?

## Quick reference

| Goal | Linux / macOS | Windows |
| --- | --- | --- |
| Show IPv6 addresses | `ip -6 addr` / `ifconfig` | `ipconfig /all` |
| Show IPv6 routes | `ip -6 route` | `netsh interface ipv6 show route` |
| Resolve AAAA / A | `dig AAAA <host>` / `dig A <host>` | `nslookup -type=AAAA <host>` |
| Ping over IPv6 / IPv4 | `ping -6` / `ping -4` | `ping -6` / `ping -4` |
| HTTP over IPv6 / IPv4 | `curl -6 -v` / `curl -4 -v` | `curl -6 -v` / `curl -4 -v` |
| Trace path over IPv6 | `traceroute -6 <host>` | `tracert -6 <host>` |
| MTU test (no-frag) | `ping -6 -M do -s 1400 <host>` | `ping -6 -f -l 1400 <host>` |
| IPv6 firewall rules | `ip6tables -L -n` / `nft list ruleset` | Windows Defender Firewall (per profile) |
