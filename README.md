# General IT

Core IT operations, system administration, troubleshooting, and best practices for Windows, Linux, and macOS environments.

## Structure

```
├── docs/                # Guides, walkthroughs, and documentation
├── troubleshooting/     # Common issues and resolutions
├── administration/      # Admin tasks, automation, maintenance
├── reference/           # Quick reference sheets and checklists
├── CONTRIBUTING.md      # Contribution guidelines
└── LICENSE
```

## Topics

- **Operating Systems** — Windows, Linux, macOS administration and configuration
- **User & Access Management** — Active Directory, user provisioning, permissions
- **System Performance** — Optimization, monitoring, bottleneck diagnosis
- **Backup & Disaster Recovery** — Strategies, tools, failover procedures
- **Hardware & Peripherals** — Installation, drivers, troubleshooting
- **Networking Basics** — DNS, DHCP, IP configuration, connectivity
- **Software Deployment** — Packaging, distribution, updates
- **Troubleshooting Methodology** — Systematic diagnosis and resolution

## Contents

### docs/

- [Troubleshooting Methodology](./docs/troubleshooting-methodology.md) - A systematic method for diagnosing IT issues, with a worked example and a symptom-to-first-check table.
- [Change Management for Small Teams](./docs/change-management-for-small-teams.md) - Lightweight change control: record template, change types, maintenance windows, armed rollbacks and post-incident notes.

### troubleshooting/

- [DNS, DHCP and Connectivity](./troubleshooting/dns-dhcp-and-connectivity.md) - Layered diagnosis of "no network" on Windows and Linux, failure signatures and server-side DHCP checks.

### administration/

- [User Onboarding and Offboarding Checklist](./administration/user-onboarding-offboarding-checklist.md) - Tickable joiner and leaver checklists for Active Directory and Microsoft 365, with PowerShell for each step.

### reference/

- [Windows and Linux Command Equivalents](./reference/windows-linux-command-equivalents.md) - Side-by-side command table grouped by task, plus PowerShell-to-bash idiom notes.

## Getting Started

Start with the [docs/](./docs/) folder for foundational guides, or jump to [troubleshooting/](./troubleshooting/) if you're solving a specific problem.

---

**Author:** Danny Stanfield · Perth, WA
**License:** MIT
