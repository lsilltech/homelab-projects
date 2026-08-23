# Isolated Attack Range VLAN

A fully isolated network segment in my homelab, built for offensive security practice. It is running intentionally vulnerable targets (Metasploitable 2, DVWA) and an attacker box (Kali Linux), completely walled off from the rest of the network.

## Goal

Practice real exploitation and privilege escalation techniques without any risk to production services or other network segments. All while still keeping the range usable (attacker and victim need to talk to each other) rather than locked down so tight it's non-functional.

## Architecture

- Dedicated VLAN, isolated at the network layer, not just via host firewall rules
- Dedicated firewall zone, so the default posture is deny-all in every direction except explicitly allowed paths
- Three hosts:
  - Metasploitable 2 — intentionally vulnerable Linux target
  - DVWA (Damn Vulnerable Web Application) — web app vulnerability target, containerized
  - Kali Linux — attacker workstation, with a remote desktop layer for a GUI experience

## Design decisions worth noting

**Same-zone traffic isn't automatic on a custom zone.** Unlike built-in firewall zones which typically allow same-zone traffic by default, a newly created custom zone starts fully closed, including to itself. Had to add an explicit "same zone → same zone" allow rule before the attacker box could even reach the victim boxes.

**A "network isolation" toggle and a "firewall zone" are not the same protection, and stacking them can backfire.** The router's built-in per-network isolation toggle blocks *peer-to-peer* traffic within a single subnet — which is the opposite of what's needed here, since Kali needs to reach Metasploitable/DVWA on the same VLAN. Zone-based firewall rules already handled cross-VLAN containment; the isolation toggle was redundant for that job and actively broke intra-VLAN attacker-to-victim traffic.

**Old guest OSes and modern hypervisor defaults don't always mix.** Metasploitable's decade-plus-old kernel has no VirtIO drivers, so the hypervisor's default high-performance disk controller left it unable to find its own boot disk. Switching to a legacy-compatible IDE controller fixed it immediately — a good reminder that "modern default" isn't always "correct default" when dealing with legacy guest systems.

**Narrow, purpose-scoped admin access rather than broad trust.** Rather than allowing full access from the admin workstation into the range, each management protocol (SSH, web management, remote desktop) got its own explicitly scoped, individually named firewall rule — easy to audit, easy to revoke independently.

## What's in this range

| Component | Purpose |
|---|---|
| Vulnerable Linux target | Network/service-level exploitation practice (backdoored services, weak configs) |
| Vulnerable web app | OWASP Top 10 practice — SQLi, XSS, CSRF, command injection, etc. |
| Attacker workstation | Kali Linux with full remote desktop access for a proper working environment |

## Lessons learned

- Zone-based firewalls are powerful but require understanding exactly what's default-allowed vs default-denied within your specific implementation. Never assume, always verify with an actual connectivity test rather than trusting a UI's summary view.
- Legacy vulnerable-by-design VMs often need hardware compatibility accommodations (older disk controllers, BIOS vs UEFI) that modern VM creation wizards don't default to.
- Isolation should be validated directly (actual ping/connection tests between hosts) rather than assumed from configuration alone.