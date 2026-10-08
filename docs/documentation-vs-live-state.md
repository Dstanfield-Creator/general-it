# Documentation vs Live State

> Documentation drifts; the live system does not. Verify with live commands before acting, date and status every document, and retire docs the day the hardware goes.

**Status:** Active · **Updated:** 2026-10-08

## The rule

A document records what someone believed on the day they wrote it. The running system is what is true now. When the two disagree, the system is right and the document has a bug. Read the document for intent, then confirm every fact you are about to act on with a live command.

## Verify before acting

| Claim in the doc | Live check |
|---|---|
| "One NIC, on vmbr0" | `ip -br link` and `ip -br addr` |
| "`local-lvm` has 200 GB free" | `pvesm status` |
| "VM 101 is the database server" | `qm list` and `qm config 101` |
| "Three containers run on the Docker host" | `docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'` |
| "Only SSH and 9100 are listening" | `ss -tlnp` |
| "Backups run from a systemd timer" | `systemctl list-units --type=timer` and `systemctl list-units --state=failed` |
| "The firewall allows 443 from the LAN" | `sudo ufw status numbered` or `sudo nft list ruleset` |

If a check contradicts the document, stop. Fix or flag the document, then proceed from the live state.

## Date and status every document

Every page carries a status and a date in its header so a reader can judge freshness at a glance:

```markdown
**Status:** Active · **Updated:** 2026-10-08
```

| Status | Meaning |
|---|---|
| Draft | Not yet verified against a live system |
| Active | Verified; procedure in use |
| Deprecated | Still accurate, but a replacement exists; link to it |
| Retired | The system no longer exists; kept for history only |

Refresh the date whenever you re-verify the content, not only when you edit it. "Verified 2026-10-08" is worth more than "written 2024-03-02" even when the text is identical.

## Retire docs with the hardware

When a host, VM or service is decommissioned, retire its documentation the same day:

1. Set **Status:** Retired and add one line: what replaced it and when.
2. Remove it from indexes, dashboards, SSH inventories and backup jobs.
3. Add a row to the decommission log.

## Keep a decommission log

One append-only table in a predictable place:

```markdown
| Date | Asset | Replaced by | Data disposition | Docs retired | By |
|---|---|---|---|---|---|
| 2026-10-01 | vm-app-old (VM 150) | vm-app (VM 160) | Backups kept 90 days, then purged | yes | ops |
```

## Never put passwords in a wiki page

Credentials in documentation leak through exports, search indexes, screenshots and backups of the wiki itself. Write a reference to the vault item instead:

```markdown
Admin credential: 1Password item `Infra / pve1 root` (vault: Infrastructure)
API token: `op://Infra/pve-svc-power/credential`
```

The reference tells the reader where to look without exposing anything, and a rotated secret does not make the page wrong.

## Prefer generated inventories

A hand-written inventory is stale the first time someone forgets to edit it. Generate it from the source of truth on a schedule and commit the output:

```bash
pvesh get /cluster/resources --type vm --output-format json \
  | jq -r '.[] | [.vmid, .name, .node, .status] | @tsv' | sort -n
docker ps -a --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
tailscale status --json | jq -r '.Peer[] | [.HostName, .DNSName, .Online] | @tsv'
```

Keep the human-only fields (purpose, owner, decommission plan) in a small file keyed by VM ID and join the two in a script. The generated half is always right; the human half is small enough to keep right.

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
