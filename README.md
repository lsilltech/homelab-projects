# Homelab Projects

Documentation and write-ups from my self-hosted homelab. A mix of networking, virtualization, monitoring, and security projects I build and break in my own time.

## Why this exists

I run a small homelab on standalone Proxmox nodes with a segmented UniFi network, and I like documenting the interesting parts, not just what I built, but what actually went wrong along the way and how I figured it out. Most of the real learning happens in the troubleshooting so that's what these write-ups focus on.

## Projects

- **[Isolated Attack Range VLAN](./attack-range)** — a fully isolated network segment for offensive security practice, running intentionally vulnerable targets and an attacker workstation, walled off from the rest of the network at both the VLAN and firewall-zone level.
- **[Proxmox Monitoring Stack](./monitoring-stack)** — Prometheus and Grafana across two standalone (non-clustered) Proxmox hosts, including debugging a community dashboard built against an outdated metric schema.
- **[Wazuh SIEM Deployment](./wazuh-siem)** — full-fleet SIEM coverage across every homelab service and both hypervisor hosts, with tuned file integrity monitoring and working email alerting.

More to come as I build them out
