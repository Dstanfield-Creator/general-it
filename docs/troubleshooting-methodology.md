# Troubleshooting Methodology

> A systematic method for diagnosing and resolving IT issues: define, reproduce, gather facts, split the problem, test one hypothesis at a time, verify, document, escalate.

**Status:** Active · **Updated:** 2026-10-08

## Why a method matters

Guesswork fixes problems some of the time and hides root causes most of the time. A repeatable method is faster on average, produces a record that helps the next person, and stops "I rebooted it and it went away" from becoming the standard outcome.

## The method at a glance

| Step | Goal | Output |
|------|------|--------|
| 1. Define | A precise problem statement | What, who, since when, what changed |
| 2. Reproduce | See the failure yourself | Exact steps, exact error text, timestamps |
| 3. Gather facts | Evidence before theories | Recent changes, live state, logs |
| 4. Split | Narrow to one layer | The layer where good becomes bad |
| 5. Hypothesise | One testable idea | A prediction that could be wrong |
| 6. Change one thing | Isolate cause and effect | A single change, noted with time |
| 7. Verify | Confirm from the user's side | User-visible success, not just a green check |
| 8. Document | Root cause and fix | A note the next person can act on |

## 1. Define the problem precisely

"The network is down" is a feeling, not a problem statement. Pin down four things before touching anything:

- **What** exactly fails: the exact error text, screenshot or HTTP status. "Slow" needs a number (seconds, Mbps).
- **Who** is affected: one user, one site, one VLAN, one role, everyone? Scope is the strongest early clue.
- **Since when**: first occurrence to the minute if possible. Intermittent or constant?
- **What changed**: patches, config pushes, new hardware, password change, moved desks, new ISP. Ask the user and check the change log.

A good statement reads like: "Since 09:10 today, three users on the second floor get `ERR_CONNECTION_TIMED_OUT` opening `https://intranet.example.com`. Other sites load. Nothing known changed on their PCs; a switch firmware update ran at 08:50."

## 2. Reproduce

Try to see the failure yourself, ideally from the user's machine or an identical path. If you cannot reproduce it, you cannot know when it is fixed. Record the exact steps and timestamps so log entries can be correlated later.

## 3. Gather facts before theories

Collect evidence first, then interpret. Three sources, in this order:

1. **Recent changes**: the change log, patch history, config management commits. Most incidents follow a change.
2. **Live state**: current IP config, routes, service status, open ports, certificate validity, disk space.
3. **Logs**: client, intermediate device and server logs around the reproduction timestamp.

Resist the urge to act on the first plausible idea. Write facts down as you find them.

## 4. Split the problem

Walk the path from the user to the data and find where "good" becomes "bad". A typical path has five layers:

```text
client -> network -> auth -> service -> data
```

Half-split: test the middle first. If the service responds correctly when queried from the server itself, the problem is on the client or network side; if not, it is on the service or data side. Each test halves the search space.

## 5. One hypothesis at a time

State the hypothesis so it can be wrong: "DNS on the client resolves `intranet.example.com` to a stale address." Then run the test that would disprove it. Discard it cleanly if it fails and move on. Running several theories at once makes the result uninterpretable.

## 6. Change one thing at a time

Each change gets a timestamp and a note. If you change three things and the problem disappears, you do not know the cause and you have two unnecessary changes to unwind later. Prefer a reversible test (a hosts-file entry, a temporary rule) over a permanent change.

## 7. Verify from the user's side

A service that passes a health check on the server is not fixed until the user can do the thing they were trying to do. Ask them to repeat the original steps, from their device, over their normal path. Check that the fix did not break something adjacent.

## 8. Document root cause and fix

A short record in the ticket or changelog:

- Problem statement (from step 1)
- Root cause (the actual defect, not the symptom)
- Fix applied, with time
- Verification performed
- Follow-up actions (permanent fix, monitoring, documentation update)

## When to escalate

Escalate when any of these is true:

- You have exhausted the facts you can gather with your access level.
- The fix requires a change outside your authority (core network, identity, production database).
- The impact is wide and the clock matters more than finishing your own diagnosis.
- You have spent the agreed time box (for example 30 minutes on a high-priority ticket) without narrowing the layer.

Hand over a written summary: problem statement, what has been ruled out, what you suspect, what you changed. A good handover saves the next person from repeating your first hour.

## Worked example: user cannot reach an internal web app

Problem statement: one user on a laptop gets a browser timeout opening `https://portal.example.com`. Other users on the same subnet are fine. No known change on the laptop.

| Layer | Question | Commands | Result |
|-------|----------|----------|--------|
| Client | Does the laptop have a valid address and DNS? | `ipconfig /all`, `ip -br addr` | Address 192.0.2.57/24, gateway 192.0.2.1, DNS 192.0.2.10. OK |
| Network | Can it reach the gateway and the server address? | `ping 192.0.2.1`, `Test-NetConnection 192.0.2.80 -Port 443`, `nc -zv 192.0.2.80 443` | Gateway replies; TCP 443 to the server succeeds |
| Network (DNS) | Does the name resolve to the right address? | `Resolve-DnsName portal.example.com`, `dig portal.example.com` | Resolves to 192.0.2.99. Wrong; the server is 192.0.2.80 |
| Client | Is there a local override? | `type C:\Windows\System32\drivers\etc\hosts`, `cat /etc/hosts` | A hosts entry points `portal.example.com` at 192.0.2.99 |
| Auth / Service | Not reached | `curl -vk https://portal.example.com/`, `journalctl -u nginx` | Not needed once DNS was isolated |

Hypothesis: a stale hosts-file entry from an earlier migration sends this laptop to a decommissioned address. Test: `curl -vk --resolve portal.example.com:443:192.0.2.80 https://portal.example.com/` succeeds. Fix: remove the hosts entry, then `ipconfig /flushdns`. Verify: the user opens the portal in their normal browser. Document: root cause was a manual hosts entry added during a migration; the follow-up is to search the fleet for the same entry.

## Symptom to first check

| Symptom | First thing to check |
|---------|----------------------|
| Address 169.254.x.x on the client | DHCP: link state, then the DHCP server and relay |
| Works by IP, fails by name | DNS: `nslookup` or `dig` against the configured resolver |
| Works on LAN, fails on VPN | Split DNS and VPN routes; which resolver the VPN pushes |
| One user affected, same site works | Client: profile, local config, hosts file, proxy, certificate store |
| Everyone at one site affected | Site uplink, gateway, site DHCP/DNS, recent change at the site |
| Login fails for everyone | Identity provider, domain controller health, time skew |
| Login fails for one user | Account locked, expired password, MFA state, group membership |
| Service up but slow | Resource saturation (CPU, memory, disk I/O), dependency latency |
| Failure started at a fixed time | Change log and scheduled tasks around that time |
| Certificate or TLS error | Clock on client and server, certificate expiry and chain |
| Intermittent timeouts | Duplicate IP, flapping link, DHCP lease churn, overloaded resolver |
| Access denied after a role change | Group membership and token refresh (sign out and back in) |

## Related

- [DNS, DHCP and connectivity](https://github.com/Dstanfield-Creator/network/blob/main/troubleshooting/dns-dhcp-and-connectivity.md)
- [Windows and Linux command equivalents](../reference/windows-linux-command-equivalents.md)
- [Change management for small teams](./change-management-for-small-teams.md)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
