# Change Management for Small Teams

> Lightweight change control for a small IT team or homelab: enough record-keeping to know what changed and how to undo it, without a change advisory board.

**Status:** Active · **Updated:** 2026-10-08

## What counts as a change

Anything that alters the behaviour, configuration or availability of a system that someone else depends on. Examples: firewall rule edits, DNS record changes, package upgrades on a server, firmware updates, new VLANs, storage layout changes, certificate renewals, permission changes on shared resources, config pushes from automation.

Not a change: reading logs, running a report, restarting your own workstation, anything fully inside a test environment nobody else uses. If in doubt, write the record; it takes two minutes.

## Change types

| Type | Definition | Approval | Record |
|------|------------|----------|--------|
| Standard | Pre-approved, repeatable, low risk, with a known rollback (patch a non-critical server from a tested baseline, add a DNS A record, renew a certificate with automation) | None per instance; the procedure itself was approved once | One-line changelog entry |
| Normal | Anything not on the standard list; needs thought about risk and rollback | One other person reads the record and agrees before the window | Full change record |
| Emergency | Fixing an active outage or closing an actively exploited vulnerability; speed matters more than review | Do it, tell someone immediately, write the record within 24 hours | Full record, filled in afterwards, plus a post-incident note |

Keep a short written list of what is standard. Review it whenever a standard change goes wrong; it probably should not be standard any more.

## The one-page change record

```markdown
# CHG-0042: Replace edge firewall WAN rule set

- **What:** Replace the inbound WAN rule set on fw01.example.com with the version in commit abc1234.
- **Why:** Remove three legacy port forwards and add a rate limit on 443.
- **When:** 2026-10-11 21:00-22:00 local. Window announced 2026-10-09.
- **Risk:** Medium. A mistake could block remote access (including mine) and the VPN.
- **Rollback:** Automatic revert timer (10 min). Manual: restore `fw01-2026-10-11-pre.conf` via console.
- **Test:** From an external host: `nc -zv 192.0.2.5 443` succeeds; `nc -zv 192.0.2.5 3389` fails; VPN client connects; one internal web app loads end to end.
- **Approver:** Second team member, agreed in chat 2026-10-10.
- **Outcome:** (filled in after) Completed 21:25. All tests passed. Timer cancelled 21:27.
```

Every field has a job. If you cannot write the rollback, you are not ready. If you cannot write the test, you will not know whether it worked.

## Maintenance windows

Pick a fixed recurring window (for example Tuesday 21:00-23:00) and announce exceptions. Users know when to expect disruption, you stop doing risky work at 16:55 on a Friday, and a problem the next morning has an obvious first suspect. Emergency changes ignore the window by definition; everything else waits for it.

## Arm the rollback before the change

Make the system undo your change on its own unless you confirm it worked. For anything that could cut off your own access (firewall, routing, SSH config, VPN), this is the difference between a two-minute blip and a drive to the site.

Firewall example: schedule a revert, make the change, run the tests from outside, and only then cancel the revert.

```bash
# Before touching anything
cp /etc/nftables.conf /root/nftables.pre-chg0042.conf
systemd-run --on-active=10m --unit=chg0042-revert \
  sh -c 'nft -f /root/nftables.pre-chg0042.conf'

# ... apply the new rule set and run the tests from an external host ...

# Tests passed: disarm
systemctl stop chg0042-revert.timer
```

The same idea in other places:

- Network devices: `reload in 10` (Cisco IOS) or `commit confirmed 10` (Junos) before a config change.
- SSH daemon changes: keep the existing session open and test a new connection before closing it.
- DNS: lower the TTL a day before the change so a revert propagates quickly.
- Virtual machines: snapshot immediately before, delete the snapshot after verification.

## Post-change verification

Run the tests written in the record, from the user's side where possible, and write the result into the record. Then wait: some problems only appear at the next business day, backup run or certificate check. Keep the change open until the next working morning for anything that touches shared infrastructure.

## A changelog in git

Keep `CHANGELOG.md` (or a `changes/` folder with one file per record) in the same repository as your configuration or documentation. Each change is one commit with the record ID in the message, so `git log --grep CHG-0042` finds everything about it. Newest first; one line per standard change, a link to the full record for the rest.

```markdown
## 2026-10

- 2026-10-11 CHG-0042 fw01: replaced WAN rule set, removed legacy forwards. [record](changes/CHG-0042.md)
- 2026-10-08 CHG-0041 dns01: added A record app.example.com -> 192.0.2.80. Standard.
- 2026-10-07 CHG-0040 all: monthly patching, 6 hosts, 1 reboot each. Standard.
```

A changelog that is three weeks behind is worse than none, because people trust it. Write the entry as part of the change, not afterwards.

## Blameless post-incident note

Write one after any emergency change, any failed change and any outage noticed by users. The goal is a better system, not a culprit. Keep it to one page: facts and timestamps, no names attached to mistakes, every action item with an owner and a date.

```markdown
# PIR: VPN outage after firewall change, 2026-10-11

- **Summary:** VPN down for 12 minutes after CHG-0042 applied a rule set that dropped UDP/1194.
- **Timeline:** 21:05 change applied. 21:08 first test fails. 21:10 revert timer fires. 21:17 VPN confirmed up.
- **Impact:** Two remote users unable to connect. No data loss.
- **Root cause:** The new rule set came from a template without the VPN rule; the test plan covered 443 and 3389 but not 1194.
- **What went well:** The revert timer restored service without manual action.
- **What to change:** Add VPN connectivity to the standard firewall test list; add the VPN rule to the template. Tracked as CHG-0043.
- **Follow-up owner and date:** Second team member, by 2026-10-18.
```

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
