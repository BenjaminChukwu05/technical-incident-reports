# Technical Incident Reports

A collection of evidence-based troubleshooting and recovery case studies. Each report records the observed problem, relevant environment, diagnostic evidence, interventions, outcome, and remaining uncertainties.

## Incidents

| Date | Incident | Outcome |
| --- | --- | --- |
| 2026-10-23–24 | [iPhone 12: iOS 27.0.1 recovery and encrypted backup restoration](incidents/2026-10-23-iphone-12-ios-recovery/README.md) | Device operational; two encrypted backups archived externally, with copy results and targeted integrity checks verified |
| 2026-10-24 | [Fedora: restoring SideStore and Aidoku after iPhone recovery](incidents/2026-10-24-fedora-ios-sideload-recovery/README.md) | SideStore installed/refreshed; Aidoku reinstalled with previous reading library and progress preserved; GitHub release download bottleneck isolated |
| 2026-10-24 | [Windows 11: returning to a native workflow](incidents/2026-10-24-windows-native-workflow/README.md) | Native shortcuts restored; unnecessary AutoHotkey workflow, PowerToys and Fedora WSL environment retired |

## Documentation principles

- Distinguish **observations** from **hypotheses**, and **successful workarounds** from proven root causes.
- Explain what commands inspect or change and record meaningful side effects.
- Preserve enough context for someone else to reproduce the *diagnostic method*, without presenting machine-specific recovery commands as universal fixes.
- Keep raw logs, screenshots containing identifiers, device backups, encryption passwords, credentials, and secrets **out of Git**.
- Review every file before staging; `.gitignore` is a precaution, not a security boundary.

## Status and scope

Incident reports are historical records, not guaranteed repair procedures. The outcome of a restore may differ by device, OS version, available backup, and circumstances.
