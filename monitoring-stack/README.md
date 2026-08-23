# Proxmox Monitoring Stack

A Prometheus + Grafana monitoring stack for a two-node standalone Proxmox environment, giving visibility into host-level hardware metrics and guest (VM/LXC) resource usage across both nodes from a single dashboard.

## Goal

Operational visibility across two independent Proxmox hosts and the container/VM fleet running on them, without clustering the nodes (a deliberate choice made elsewhere in this homelab for stability reasons). Scoped initially to core infrastructure services and both physical hosts, with room to expand.

## Architecture

- **Prometheus, Grafana, and prometheus-pve-exporter** running as a Docker Compose stack inside a single unprivileged LXC container
- **node_exporter** installed directly on both Proxmox hosts as a systemd service (not containerized) needed for host-level hardware metrics (CPU, memory, disk, network, temperature) that an API-based exporter can't see
- **prometheus-pve-exporter** queries the Proxmox API on each node separately, using scoped read-only API tokens created independently on each host

## Design decisions worth noting

**Standalone nodes mean everything gets built twice.** Since the two Proxmox hosts aren't clustered, they don't share a user or permission database. An API token created on one node simply doesn't exist on the other. Every user, token, and ACL grant had to be created independently on each host and the exporter's config split into two named blocks (one per node) with Prometheus routing to each via a distinct `module` parameter.

**API tokens need permission granted twice.** Even with privilege separation enabled on a Proxmox API token, the permission check requires the ACL grant to exist on *both* the token identity and the parent user account. Granting only the token (which is what the UI flow implies you should do) results in a `403 Forbidden` despite the ACL appearing correctly configured in the interface.

**Imported dashboards need to be validated against your actual exporter output, not assumed correct.** A popular community Grafana dashboard for this exporter was built against an older metric schema. Different metric names, different label conventions. Every panel showed "No data" on import. Fixing it meant systematically checking each panel's query against the real metrics the exporter was actually producing, one section at a time.

**Not every panel in an imported dashboard has a real data source.** Some panels (fan speed, power draw, voltage, SMART disk health/temperature) simply have no corresponding metric available without dedicated hardware (IPMI) or additional exporters (a SMART exporter) that weren't part of this build. Rather than leaving them showing "No data" indefinitely, they were deleted.

## Problems hit along the way

- **Invalid YAML silently crash-looped Prometheus** — a duplicate key under one scrape job's config wasn't caught until checking the container logs directly.
- **Exporter config path assumptions were outdated** — a commonly referenced config file path in older documentation didn't match what the actual container image expected, causing a crash loop until the correct path was found and the volume mount corrected.
- **A shell history-expansion character broke API token strings** — a special character used in Proxmox's token ID format was being interpreted by the shell itself when typed in double quotes, silently corrupting the value. Needed to be disabled for the session to type the token correctly.
- **Docker Compose doesn't always notice a bind-mounted config file changed** — editing a config file that's mounted into a container doesn't trigger a reload on its own; an explicit service restart is needed to force the container to re-read it.
- **Inconsistent metric naming caused repeated "no data" panels** — the exporter uses "written" in some metric names where a dashboard query expected "write," a small mismatch that silently broke several panels until caught.
- **Double-counted percentages** — one panel had both a `* 100` multiplier in its query *and* the panel's unit already set to display as a percentage, resulting in numbers that were 100x too large until one of the two was removed.
- **Not every metric lives where you'd expect** — several host-level stats (load average, I/O wait, hardware temperatures, disk IOPS) simply aren't tracked by the Proxmox-API-based exporter at all, and had to be sourced from the host-level exporter instead.

## Lessons learned

- When two servers are intentionally kept unclustered for stability, expect to duplicate *all* configuration across them. There's no shared state to lean on, which is the tradeoff for the added resilience.
- Debug crash-looping containers by reading their actual logs before assuming the config is correct. Several issues here were invisible until checked directly against container output.
- A dashboard "not working" after import is almost always a schema mismatch, not a broken dashboard. Check what metrics your actual exporter version produces before assuming the dashboard itself is at fault.