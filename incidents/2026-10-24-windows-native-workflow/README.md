# Returning Windows 11 to a native workflow

| Field | Value |
| --- | --- |
| Date of investigation | 24 October 2026 |
| Host environment | Windows 11 Pro, build 26200 (later reported as 26200.9711) |
| Other operating system | Fedora Linux installed separately and used as the primary workstation environment |
| Scope | Keyboard shortcuts, window management, optional customization utilities, WSL, and a leftover Explorer integration |
| Outcome | Native Windows behavior restored; unused AutoHotkey workflow retired; PowerToys, Fedora WSL distribution, and WSL app removed |
| Root-cause confidence | AutoHotkey shortcut conflict confirmed; original Zen window-position cause not isolated |

> This is a historical account of one workstation cleanup, not a general-purpose Windows debloating guide. Some commands below are destructive. Read the observations, prerequisites, and risks before adapting any procedure to another system.

## Executive summary

I primarily work in Fedora Linux and use Windows for selected Windows-specific applications and tasks. Earlier, I configured Windows to resemble my GNOME workflow using AutoHotkey and PowerToys. Later, I discovered that some modifications duplicated functionality that Windows already provides and, in one case, directly replaced native shortcuts.

The most reproducible symptom was that `Win + Up` opened Task View instead of maximizing the focused window. Inspection identified an AutoHotkey v2 startup script, `WindowsWorkflow.ahk`, with an explicit mapping for that behavior. After I exited the script, `Win + Up` immediately maximized windows normally. The startup script was archived rather than deleted.

I also verified that Windows Snap Layouts and virtual desktop navigation worked natively. Zen Browser, which had repeatedly opened on the left side of the display, subsequently reopened maximized. That behavior remained correct with FancyZones re-enabled, so the original cause of Zen's placement issue was **not established**.

After reviewing PowerToys' enabled modules, I uninstalled it and verified its removal. I then inspected the Fedora WSL distribution, archived selected shell and editor configuration, unregistered the distribution, uninstalled the separate WSL application, and verified that the Windows WSL optional feature was disabled. Finally, a leftover **Linux** entry disappeared from File Explorer following a user-specific shell-integration cleanup.

The guiding decision was to use native Windows features by default, introducing extra tools only when they provide a capability that justifies their configuration and maintenance cost.

## Initial environment and symptoms

The Windows host was running **Windows 11 Pro, build 26200**. Fedora was installed separately and was my everyday operating system. Windows served specific applications and workflows; the objective was not to make the two installations identical.

Observed issues included:

- `Win + Up` displayed all windows and virtual desktops instead of maximizing the active window.
- Custom `Ctrl + Alt + Left/Right` desktop switching behaved inconsistently around elevated terminal windows, particularly while the mouse pointer was over them.
- Zen Browser repeatedly reopened in a left-side window position rather than maximized.
- Windows Snap Layouts and the built-in virtual desktop shortcuts were initially unfamiliar, so I had added third-party replacements.
- AutoHotkey, PowerToys, and a Fedora WSL environment were present even though their continued value was uncertain.

The issue was approached as a **configuration review and targeted repair**, not as a justification to reinstall Windows.

## 1. Identifying the shortcut conflict

### Read-only observations

A process check found `AutoHotkey64.exe` alongside PowerToys and Zen processes. Process command-line inspection identified the script launched from the current user's Startup folder:

```powershell
Get-CimInstance Win32_Process |
    Where-Object { $_.Name -match '^AutoHotkey' } |
    Format-List Name, ProcessId, ExecutablePath, CommandLine

Get-ChildItem -LiteralPath ([Environment]::GetFolderPath('Startup')) -Force |
    Select-Object Name, FullName, Extension, LinkType, Target
```

The startup script contained these AutoHotkey v2 mappings (selected excerpts):

```autohotkey
; Open terminal and applications
^!t::Run("wt.exe")
#b::Run("zen.exe")
#f::Run("explorer.exe")

; Replace native Windows window actions
#Up::Send("#{Tab}")
#Down::Send("#{Tab}")

; GNOME-like virtual desktop navigation
^!Left::Send("^#{Left}")
^!Right::Send("^#{Right}")
```

AutoHotkey uses `#` for the Windows key, `^` for Ctrl, and `!` for Alt. The `#Up` mapping therefore explicitly translated `Win + Up` into `Win + Tab` (Task View).

Other mappings included `Win + Q` to close the active window, `Win + M` to toggle maximization, `Win + Shift + M` to minimize the active window, and `Win + Esc` to lock the workstation. Several collided with established Windows shortcuts.

### Controlled test and conclusion

I exited `WindowsWorkflow.ahk` from the notification area and immediately retested `Win + Up` in a normal window. It maximized correctly.

**Confirmed cause:** The custom AutoHotkey script overrode the native maximize shortcut. This conclusion is supported by both the script contents and the immediate behavior change when the script stopped.

I decided to use native shortcuts instead:

| Capability | Native Windows shortcut |
| --- | --- |
| Maximize a window | `Win + Up` |
| Open Snap Layouts | `Win + Z` |
| Open Task View | `Win + Tab` |
| Switch virtual desktops | `Win + Ctrl + Left/Right` |
| Create a virtual desktop | `Win + Ctrl + D` |
| Close the current virtual desktop | `Win + Ctrl + F4` (may also require `Fn` depending on keyboard settings) |
| Close the active window | `Alt + F4` |
| Open File Explorer | `Win + E` |

The native `Win + Ctrl + Left/Right` shortcuts worked even with an administrator terminal active, eliminating the need for the custom desktop-switching mapping in this workflow.

### Retiring the startup script safely

An initial attempt to archive the Startup script stopped because a same-named backup already existed. SHA-256 comparison showed that the existing backup was an **empty file** (the known SHA-256 of zero bytes), so it was not a usable copy of the script.

Rather than overwrite anything, I moved the Startup script to a distinctly named, timestamped archive under my private Windows configuration-backup directory. Follow-up checks returned:

```text
Still in Startup: False
Retired copy exists: True
```

AutoHotkey remained installed at that stage. Removing one startup workflow did not require deleting the application or its archived configuration. This also avoided treating a previous empty backup as valid recovery material.

## 2. Testing native window management and Zen

Windows Snap Layouts were verified with `Win + Z` and by hovering over an application's maximize button. Normal window maximization and native virtual desktop switching also worked.

PowerToys' FancyZones settings were inspected. Notably:

- **Move newly created windows to their last known zone:** Off.
- **Override Windows Snap:** Off.
- **FancyZones:** Enabled during the original inspection.

After stopping AutoHotkey and temporarily disabling FancyZones, I maximized Zen, closed it normally, and reopened it. Zen now reopened maximized. I then re-enabled FancyZones without changing its other settings and repeated the test; Zen still reopened maximized.

**Established:** Zen's intended startup behavior was restored, and FancyZones being enabled did not reproduce the issue during the follow-up test.

**Not established:** Whether the original left-side startup behavior resulted from a previously saved Zen window position, a custom launcher, FancyZones, or another interaction. Multiple conditions changed before the first successful test, so assigning a single root cause would overstate the evidence.

## 3. Removing PowerToys

The PowerToys dashboard showed version **0.100.2**. Only **FancyZones** and **Always On Top** were enabled among the visible utilities; Keyboard Manager, PowerToys Run, Command Palette, and Workspaces were disabled.

Native Windows Snap met my actual window-layout needs, and I did not identify a current need for Always On Top. After deciding those additional capabilities were unnecessary for this Windows profile, I removed PowerToys with WinGet:

```powershell
winget uninstall --id Microsoft.PowerToys --exact
```

A prior attempt from an elevated PowerShell session failed with a message that the package had been installed in **user scope** and could not be uninstalled from that administrator context. Repeating the uninstall from a normal PowerShell session succeeded.

Verification:

```powershell
winget list --id Microsoft.PowerToys --exact
Get-Process -Name 'PowerToys*' -ErrorAction SilentlyContinue
```

WinGet reported **no matching installed package**, and PowerShell returned no matching running processes.

This reinforced an administration lesson: elevation is not always helpful; the correct installation scope and user context matter.

## 4. Retiring the redundant Fedora WSL environment

I had installed Fedora under Windows Subsystem for Linux even though Fedora was already my primary, separately installed operating system. This made WSL optional for my current needs, not inherently unnecessary for other Windows users.

### Inventory before deletion

The registered WSL distribution was **`FedoraLinux-44`**, using **WSL 2**. Its home directory contained approximately **356 MB**:

| Location | Approximate size | Treatment |
| --- | ---: | --- |
| `.local/lib/python3.14` | 143 MB | Installed Python application/library state; not migrated |
| `.local/share/nvim` | 126 MB | Neovim application/plugin state; not migrated |
| `.local/bin` | 32 MB | Preserved temporarily; contents included user-level executable entry points |
| `.cache` | 57 MB | Cache; not preserved |
| `.config` | 20 KB | Preserved selected Neovim configuration |
| Shell and tmux dotfiles | Small | Preserved as intentional configuration |

The directory also contained a `winhome` symbolic link pointing to the Windows home directory. That link was not a second copy of Windows files.

### Selective preservation

Instead of cloning the WSL distribution, I used `tar` from within WSL to save selected configuration files to a private Windows directory. The archive included:

```text
.bashrc
.bash_profile
.bash_logout
.zshrc
.custom_aliases
.tmux.conf
.config/nvim/init.lua
.config/nvim/nvim-pack-lock.json
.local/bin/ (selected user-level executables and entry points)
```

The archive was **12,412,140 bytes**. The `tar` command completed without a reported error, and `tar -tzf` successfully listed its contents. Shell command history, caches, the bulk of installed Python libraries, and Neovim plugin data were intentionally excluded.

**Verification limit:** Listing an archive demonstrates readability, not a tested restore. The archive is a selective configuration snapshot, not a complete WSL backup. Its contents may include personal configuration, so it remains outside public Git repositories. User-level executables may be environment-specific and are not automatically suitable for migration to native Fedora.

### Destructive removal and verification

After reviewing the files and preserving the selected configuration, I ran:

```powershell
# Destructive: permanently removes this WSL distribution's filesystem.
wsl --unregister FedoraLinux-44
```

The command reported that unregistration completed successfully. Subsequent `wsl --list --verbose` output reported **no installed distributions**.

The separately serviced WSL application was still installed at **2.7.11.0**, so it was removed independently:

```powershell
winget uninstall --id Microsoft.WSL --exact
```

WinGet reported successful uninstallation. Follow-up checks found no matching WinGet package and no `MicrosoftCorporationII.WindowsSubsystemForLinux` AppX package in the current user account.

The Windows optional feature `Microsoft-Windows-Subsystem-Linux` was separately checked from an elevated PowerShell session and reported **Disabled**. No changes were made to Hyper-V, Virtual Machine Platform, firmware virtualization, or the separate native Fedora installation.

**Important distinction:** Unregistering a distro, uninstalling the WSL app, and disabling a Windows optional feature are different operations. `wsl --unregister` is the operation here that destroyed the registered Linux filesystem.

## 5. Removing the leftover File Explorer entry

After the Fedora distribution and WSL application were removed, File Explorer still displayed **Linux** in the navigation pane, even after a Windows restart. WSL distribution and package checks were already clean, and the WSL optional feature was disabled.

A user-specific Explorer shell namespace pinning override was investigated as a non-destructive way to hide the residual navigation entry. I subsequently confirmed that the **Linux** entry had disappeared.

**Evidence limit:** The exact registry value changed and the corresponding command/output were not captured in the final terminal record. Therefore, this report records the outcome but does not present a specific registry edit as a fully verified procedure. A disappearing navigation entry is a user-interface observation, not additional proof about the native Fedora disk.

## Decision record

| Decision | Rationale |
| --- | --- |
| Use Windows-native shortcuts and Snap by default | The built-in functionality meets my actual needs and avoids avoidable conflicts |
| Retire the AutoHotkey workflow, not blindly delete all scripts | The specific conflict was confirmed, and archival preserved useful history |
| Uninstall PowerToys | The enabled modules provided no currently justified benefit for this Windows profile |
| Remove Fedora WSL after selective preservation | I already use native Fedora; valuable shell/editor configuration is more portable than the installed WSL environment |
| Keep Windows virtualization features unchanged | WSL removal alone did not justify changing capabilities other software may need |
| Address Explorer integration separately | A stale shell entry is different from an installed Linux environment |
| Avoid reinstalling Windows | Targeted, verified changes resolved the reported problems with less disruption |

## Outcomes and remaining limits

- [x] `Win + Up` restored to native maximize behavior.
- [x] Native Snap Layouts, maximize-button layouts, and virtual desktops verified.
- [x] Zen reopened maximized, including with FancyZones re-enabled during testing.
- [x] AutoHotkey workflow removed from Startup and archived.
- [x] PowerToys uninstalled, with package and process checks showing no matches.
- [x] Selected Fedora WSL files archived and archive listing verified.
- [x] Fedora WSL distribution unregistered; no distributions remained.
- [x] WSL application uninstalled; optional WSL feature reported disabled.
- [x] Residual Linux navigation entry no longer appeared in File Explorer.
- [ ] No full archive restoration test was performed.
- [ ] The original cause of Zen's left-side reopening was not conclusively determined.
- [ ] This was a targeted cleanup, not a full audit of all Windows customizations.

## Lessons learned

1. **Audit the native capability first.** A desired shortcut or window behavior may already exist in the operating system.
2. **Separate requirements from implementations.** I wanted fast desktop switching, not a permanent dependency on a particular remapping script.
3. **Use controlled tests to establish causation.** Disabling AutoHotkey and seeing `Win + Up` work immediately was stronger evidence than guessing based on background process names.
4. **Preserve intentional configuration, not accumulated state.** Shell aliases and editor settings were worth keeping; caches and installed packages were not automatically worth migrating.
5. **Recognize distinct removal layers.** Application packages, WSL distributions, Windows optional features, and Explorer shell entries require different checks.
6. **Do not equate minimalism with quality.** The goal is a maintainable system whose installed tools serve a known purpose—not an arbitrary minimum process count.
7. **Give each operating system its own role.** Fedora can remain my primary environment while Windows stays useful for applications and tasks better supported there.

## References

- [Microsoft: Windows keyboard shortcuts](https://support.microsoft.com/windows/keyboard-shortcuts-in-windows)
- [Microsoft: FancyZones](https://learn.microsoft.com/windows/powertoys/fancyzones)
- [Microsoft: WSL basic commands and unregistration warnings](https://learn.microsoft.com/windows/wsl/basic-commands)
- [Microsoft: PowerToys administrator-mode considerations](https://learn.microsoft.com/windows/powertoys/administrator)

## Privacy and reproducibility note

The evidence for this report came from terminal output, application settings screenshots, and behavior tests conducted on my Windows workstation. Exact account paths, private archive locations, raw configurations, and personal file contents are intentionally excluded. Commands are illustrative of the investigation and must be adapted to the actual machine and current installed state; in particular, **do not run `wsl --unregister` without first reviewing and preserving any required data**.
