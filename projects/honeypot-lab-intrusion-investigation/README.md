# MySQL Ransom Compromise & RDP Brute-Force Investigation — Cyber Range Capstone

**Host:** `datafiles-corp-s01` (intentionally exposed honeypot lab, not a production system)
**Analyst:** Grace Agosto, SOC Analyst (Capstone Exercise)
**Tools:** Microsoft Defender for Endpoint, Microsoft Sentinel (KQL / Advanced Hunting), MySQL Audit Log

## Summary

Between September 1–2, 2026, an internet-exposed MySQL 8.0 server was compromised by an automated ransom/wiper script within about 20 hours of being exposed to the internet. The attacker authenticated as `root` using blank credentials, enumerated four databases (including one named `credentials`), dropped all of them, replaced them with a ransom note demanding 0.0109 BTC, purged the binary logs, and shut the service down — the entire sequence took under a minute. The same pattern repeated from at least eleven other source IPs over the following day, consistent with opportunistic internet-wide scanning rather than a targeted actor.

Separately, the Windows host's RDP service was hit with a large-scale brute-force campaign (22,906 failed logons from 11+ external IPs) that locked out the built-in Guest account 110 times, plus three anonymous SMB logons. No evidence of a successful, attacker-driven RDP session or lasting compromise was found on the Windows side.

This repo documents both investigations end to end: detection, timeline reconstruction, indicators of compromise, MITRE ATT&CK mapping, and remediation recommendations, using Microsoft Defender for Endpoint's Investigation Package and the MySQL audit log as primary evidence.

## Reports

- **[Cyber Defense Final Report](reports/Cyber_Defense_Final_Report.pdf)** — the primary incident report: executive summary, timeline, root cause, impact assessment, indicators of compromise, response actions, and lessons learned for the MySQL ransom event and the Windows administrator logon.
- **[Pre/Post-Breach Package Comparison](reports/DFIR_PrePost_Comparison.pdf)** — a diff-style forensic comparison of the MDE Investigation Packages collected before and after the incident, covering the RDP brute-force campaign, with MITRE ATT&CK mapping and an explicit list of evidence gaps.

## Key findings

| | |
|---|---|
| Attack vector (1) | MySQL exposed on TCP/3306 with a blank `root` password |
| Attack vector (2) | RDP exposed on TCP/3389, targeted by distributed brute force |
| Time to compromise | ~20 hours from exposure to first successful attacker logon |
| MySQL wipe duration | ~49 seconds, start to shutdown — scripted, not manual |
| Failed RDP logons | 22,906, from 11+ external IP addresses |
| Databases destroyed | `world`, `sakila`, `cr_corp_01` (contained a `credentials` table), plus the attacker's own `recover_your_data` |
| Ransom demand | 0.0109 BTC to `bc1q0l7hr5v220f5qlqhjhg4p3jjqfkgudazzjnlkw` |
| Confirmed RDP compromise | None — brute force only, no successful authenticated RDP session found |

## Evidence

- [`evidence/logs/`](evidence/logs/) — exported MySQL audit logs (logons and queries) and VM telemetry (logons, network connections, network analytics, processes, registry) covering the incident window.
- [`evidence/pre-breach/`](evidence/pre-breach/) — key files from the MDE Investigation Package collected shortly after the host was exposed, before the attack (baseline).
- [`evidence/post-breach/`](evidence/post-breach/) — the same file categories from the Investigation Package collected at containment/isolation, for direct comparison against the baseline.

These are a curated subset of the full Defender for Endpoint Investigation Packages — selected for what's relevant to the write-up (process lists, services, scheduled tasks, network state, user/group info, system info). The full packages also include Prefetch files and raw `Security.evtx` event logs, which are summarized and quoted in the reports rather than included here in full.

## Methodology

1. **Detect** — reviewed MySQL audit logs and Defender/Sentinel tables (`DeviceLogonEvents`, `DeviceNetworkEvents`, `DeviceProcessEvents`) for the exposure window.
2. **Collect** — captured a Defender for Endpoint Investigation Package at isolation, and compared it against a baseline package collected shortly after the host was first exposed.
3. **Analyze** — reconstructed a UTC timeline of attacker activity from the MySQL query log and Windows Security event log, and mapped observed behavior to MITRE ATT&CK techniques.
4. **Report** — documented impact, indicators of compromise, root cause, and remediation recommendations in a formal incident report and a pre/post comparison appendix.

## Recommendations (summary)

1. Never expose a database service directly to the internet with default or blank credentials.
2. Ship database audit logs off-box in near real time so an attacker can't erase their own trail with `PURGE BINARY LOGS`.
3. Capture a complete, correctly-scoped `DeviceLogonEvents`/`DeviceProcessEvents` export before closing an incident.
4. Restrict RDP exposure, disable or rename default local admin accounts, and require MFA or a jump host.
5. Build a proper Sentinel analytics rule for this class of MySQL destructive-command pattern instead of relying on manual hunting.

Full detail on each of these, including supporting KQL queries, is in the [final report](reports/Cyber_Defense_Final_Report.pdf).
