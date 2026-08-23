# Wazuh SIEM Deployment

A centralized SIEM (Wazuh 4.14, all-in-one mode) covering every service and both hypervisor hosts in the homelab, with tuned file integrity monitoring and email alerting on suspicious activity.

## Goal

Real security visibility across the environment. Not just "is it running," but "did anything change that shouldn't have, and did anyone try to log in who shouldn't have." All in one mode (indexer + server + dashboard on a single VM) made sense at this scale. A distributed Wazuh cluster would be overkill for my homelab.

## Architecture

- Wazuh 4.14, all-in-one install, on a dedicated VM in its own isolated network zone
- Reverse proxied dashboard access through the existing internal proxy, backend over HTTPS since Wazuh's dashboard terminates its own self-signed TLS
- Agents deployed to every running service (container-level) *and* both Proxmox hypervisor hosts directly — host-level visibility (SSH logins, sudo usage, hypervisor config changes) is invisible from inside the containers alone
- Alerting piped through a local mail relay to an external email account, since Wazuh's built-in mailer doesn't support the authenticated SMTP most providers now require

## Design decisions worth noting

**A dedicated firewall zone forced explicit, minimal access rules.** Rather than dropping the SIEM into an existing trusted zone, it got its own zone with deny-by-default in every direction. That meant deliberately building out each access path it actually needed: DNS resolution out, the reverse proxy reaching in to manage/display the dashboard, and the biggest one, every other zone that runs an agent being allowed to reach the manager on the two ports agents actually need. Nothing broader than that.

**Reusable port groups over repeated raw ports.** Rather than typing the same two agent-communication ports into every rule that needed them, they got grouped into a single named object once and referenced everywhere. One place to update if that ever changes.

**Full-fleet agent rollout, containers and hosts alike.** Every service container got an agent, but so did both underlying hypervisors. A compromise or misconfiguration at the hypervisor layer is arguably more consequential than one at the container layer and it's a blind I did not want to skip.

**Narrow, deliberate FIM scope beats broad default watching.** Wazuh's default file integrity monitoring config watches broad system directories and immediately drowns in noise from routine package updates and log rotation. Instead, each agent's watch list was built individually around what that specific service actually cares about. Credential files, the service's real config/database, and (for the hypervisors) the directory holding VM/container definitions while explicitly excluding known high-churn, low-value paths (icon caches, rotating logs, auto-renewing certs) identified by physically inspecting each service's directory layout rather than guessing.

**Alert thresholds tuned to the actual threat model, not a generic default.** In an environment with only known, authorized users, any failed login attempt is inherently worth immediate visibility. There's no legitimate reason for one to happen. The alert threshold was set low enough to catch single failed logins rather than waiting for a sustained brute-force pattern to trip a higher-severity correlation rule.

## Problems hit along the way

- **A firewall rule silently never matched** — a port restriction was accidentally placed on the *source* port field instead of the destination, which meant real traffic (using ephemeral source ports, as all client connections do) never matched the rule and fell through to the zone's default deny. Diagnosed and fixed by confirming port restrictions belong on the destination side only.
- **IPS flagged legitimate admin traffic as a port scan** — an early SSH connection attempt to the SIEM host got blocked by an intrusion-prevention signature designed to catch scanning behavior. Resolved by suppressing that specific signature for the known admin source IP, narrowly, rather than weakening IPS coverage more broadly.
- **The original firewall rule plan had a gap** — it accounted for the reverse proxy reaching the dashboard, but not for direct admin access from a personal device for testing/troubleshooting. Caught during setup and patched in as an additional narrowly-scoped rule.
- **`apt-key` is gone on modern Debian** — the SIEM vendor's documented install method for adding their package repo assumed a deprecated key-management approach; fixed by using the modern keyring + `signed-by=` method instead.
- **Gmail's authenticated-SMTP requirement broke the SIEM's built-in mailer** — the SIEM's native email alerting doesn't support authenticated SMTP, which most mail providers now require. Fixed with a local mail relay that handles the authenticated hop to the mail provider on the SIEM's behalf — the SIEM talks to `localhost`, the relay does the real, authenticated delivery.
- **App-password generation was initially unavailable** on the mail account being used — required adjusting account security settings before an app-specific password could even be generated.
- **Alert threshold tuning took two passes** — an initial, more conservative threshold didn't trigger on isolated failed login attempts, since those land at a lower severity than the correlation rule that watches for sustained brute-force patterns. Lowered the threshold instead of chasing the correlation rule's higher bar, matching the actual "any failed login is notable" threat model for this environment.

## Result

Full-fleet coverage: every running service plus both hypervisor hosts reporting into the SIEM, tuned FIM watching the things that actually matter per service, and working email alerts on suspicious activity — end-to-end tested with a real triggered alert that arrived by email within seconds.

## Lessons learned

- Firewall port restrictions belong on the destination side source-port restrictions on client traffic will silently never match.
- Intrusion-prevention systems can and will flag your own legitimate admin traffic, know how to narrowly suppress a specific false positive without disabling protection broadly.
- Default security-tool configs are usually tuned for "catch everything, generate lots of noise" deliberately narrowing scope to what actually matters for your environment makes the tool more useful, not less secure.
- When a vendor's built-in alerting can't handle modern authenticated SMTP, a local relay is often the officially supported workaround.
- Physically inspect a service's directory layout before deciding what to monitor, guessing at meaningful paths leads to either missing what matters or drowning in noise.