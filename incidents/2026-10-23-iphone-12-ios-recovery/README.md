# iPhone 12 recovery after failed iOS 27.0.1 update

| Field | Value |
| --- | --- |
| Incident period | 23–24 October 2026 |
| Device | iPhone 12 |
| Recovery host | Windows computer |
| Outcome | iOS 27.0.1 installed; encrypted backup restored; external backups archived |
| Root-cause status | Not conclusively determined |

> This is an incident record, **not** a recommendation to erase a device. Restoration to factory settings can permanently remove data that is not included in an accessible backup. Do not share backup encryption passwords or raw backup files.

## Executive summary

An attempted update from iOS 26.5.2 to iOS 27.0.1 using Apple Devices on Windows failed, leaving the phone in Recovery Mode. Repeated attempts to use the non-erasing **Update** option ended with error **1109** and checkpoint **06B4.0000**. The Apple Devices updater log identified a failed WKMS key request and reported that no authorization token was available.

Basic TCP connectivity to tested Apple endpoints worked; it did not establish that the WKMS authorization exchange succeeded. Switching to iTunes initially encountered two Windows-side barriers: iTunes did not recognize the recovery-mode phone until Apple Mobile Device Support was installed, and the standalone iTunes distribution subsequently displayed an "install the latest version" warning despite reporting version 12.13.11.1. Installing iTunes through Microsoft Store removed that version warning, allowing the software installation to proceed.

Although **Update** was selected, iTunes later reported that the phone had been restored to factory settings. The device then recovered personal information from a previously created **encrypted local backup**, using its saved backup passphrase. I confirmed a working Home Screen, familiar wallpaper, and functioning applications; some apps required sign-in again.

**Important distinction:** The successful attempt occurred after software-environment changes and at a later time. There is insufficient evidence to conclude that the Microsoft Store installer, the device-support component, or a transient Apple service change specifically resolved the earlier WKMS failure.

## Environment and safeguards

| Item | Observed information |
| --- | --- |
| Device | iPhone 12 |
| Requested iOS version | 27.0.1 |
| Initial free device storage | Approximately 2.3 GB |
| Initial Windows device manager | Apple Devices |
| iTunes tested | Standalone and Microsoft Store distributions; application version shown as 12.13.11.1 |
| Standalone Apple Mobile Device Support | 19.4.0.10 reported by WinGet |
| Connection | Direct USB connection; known-good Anker cable; alternate USB port also tested |
| Pre-update safety measure | Encrypted local computer backup created before software recovery |
| Credentials | Apple Account and backup encryption passphrase retrieved from password manager; **not included in this report** |

The pre-recovery backup was the recovery lifeline. Its successful use to restore the phone is stronger evidence of usefulness than file existence or hashes alone. No assertion is made that every possible data category was recovered.

## Timeline of significant events

Dates are local to the recovery environment. Precise timestamps are included only where recorded in the available evidence.

| Date | Event |
| --- | --- |
| 23 Oct | Created a local encrypted iPhone backup (iTunes subsequently displayed a backup from around **12:02 PM**). |
| 23 Oct | Apple Devices update attempts failed repeatedly with **1109 / 06B4.0000**. Force restarting returned the device to Recovery Mode. |
| 23 Oct, approx. 13:00 | The updater log recorded missing WKMS authorization token and key retrieval failure. |
| 23 Oct | Confirmed TCP port 443 connectivity to `gs.apple.com`, `wkms-public.apple.com`, and `fcs-keys-pub-prod.cdn-apple.com`. |
| 23 Oct | Installed standalone iTunes; device was initially absent. Windows PnP detected Apple recovery USB devices, while the expected Apple Mobile Device Service was absent. |
| 23 Oct | Installed Apple Mobile Device Support; iTunes detected the phone, but displayed a newer-iTunes warning. The available standalone iTunes version was 12.13.11.1. |
| 23 Oct | Replaced standalone iTunes with the Microsoft Store distribution. iTunes reported version **12.13.11.1** and no longer displayed the version warning. |
| 23 Oct | Initiated Update. iTunes downloaded firmware and installed iOS. At completion it explicitly reported **factory settings**, rather than a guaranteed data-preserving update. |
| 23 Oct | Restored the encrypted computer backup; phone booted and showed the familiar Home Screen and apps. |
| 24 Oct, 05:25 | Created a fresh encrypted post-recovery computer backup. |
| 23–24 Oct | Archived pre- and post-recovery backups to separate dated directories on an exFAT Ventoy utility drive; performed follow-up comparisons. |

## Diagnostic evidence

### 1. Apple Devices error 1109

Sanitized excerpts from the Apple Devices updater log:

```text
WKMS: No authorization token available for wkms.sd.apple.com; failing
CFError domain:AMRestoreErrorDomain code:1109 description:Error retrieving WKMS decryption key
Restore Checkpoint Fingerprint: 06B4.0000
Finished RestoreOS Restore Phase: Failed
```

**Observed:** The restore/update engine failed during key retrieval; the log explicitly reported an unavailable authorization token.

**Not established:** Why the token was unavailable. Potential server, authorization, client-environment, and network-related causes were not distinguished conclusively. An internal `RestoreOS` phase label is **not** in itself proof of a user-data erase.

### 2. Network reachability was not authentication success

The Windows host successfully completed TCP port 443 checks for the named Apple endpoints. This establishes DNS resolution and a TCP connection at that moment—not successful HTTPS application traffic, authorization, or key retrieval.

Example diagnostic command (read-only):

```powershell
Test-NetConnection -ComputerName wkms-public.apple.com -Port 443
```

### 3. Windows USB detection was separate from iTunes recognition

Windows PnP listed an Apple Recovery (iBoot) USB Composite Device while standalone iTunes showed no connected iPhone. The Apple Mobile Device Service lookup returned no matching service. After a separately installed Apple Mobile Device Support package, iTunes detected the phone.

Example read-only checks:

```powershell
Get-Service | Where-Object DisplayName -Match 'Apple Mobile Device'
Get-PnpDevice -PresentOnly |
    Where-Object FriendlyName -Match 'Apple|iPhone|Recovery|iBoot'
```

**Lesson:** USB enumeration, device drivers/services, and application-level support are different layers. Detection by Windows alone does not guarantee a particular app can manage the device.

### 4. iTunes distribution and version warning

The standalone iTunes installation reported version 12.13.11.1 and no newer WinGet upgrade. Nonetheless, the recovery interface required the "latest version" to continue. Reinstall/repair did not resolve that warning. Replacing the standalone distribution with the Microsoft Store version (which also displayed 12.13.11.1 in About iTunes) cleared it.

**Observed workaround:** Microsoft Store iTunes proceeded where the standalone installation did not. **Unproven cause:** The exact incompatibility in the original installation.

## Recovery and restoration outcome

The final iTunes attempt passed the version warning and installed iOS 27.0.1. iTunes explicitly displayed a completion message stating the phone was restored to factory settings, despite the earlier selection of **Update**. Therefore, the operation was not considered a successful non-erasing update.

The encrypted local backup was then restored using its existing encryption password. The phone restarted normally; I confirmed the previous wallpaper, Home Screen layout, downloaded applications, and successful operation of a sample of apps. Some app sessions had to be reauthenticated. A stalled, optional iTunes media sync was cancelled separately after backup restoration.

This establishes operational recovery and meaningful data restoration, not a byte-for-byte validation of all personal information.

## Backup preservation and verification

Backup copies were stored in distinct dated directories on a Ventoy utility drive (exFAT). The raw backups and their device-specific identifiers are **not** part of this Git repository.

| Recovery point | Copy result | Follow-up verification |
| --- | --- | --- |
| **23 Oct, pre-recovery** | 151,235 files; 40.712 GB reported; Robocopy exit code 1; 0 reported failures | SHA-256 of `Info.plist`, `Manifest.db`, `Status.plist` matched source |
| **24 Oct, post-recovery** | 151,108 files; 39.575 GB reported; Robocopy exit code 1; 0 reported failures | Read-only Robocopy listing found nothing needing copying (exit code 0); all three metadata SHA-256 hashes matched source |

The copy used `robocopy` options `/E /Z /R:2 /W:5 /XJ /FFT /COPY:DAT /DCOPY:DAT /NP` to preserve directory contents without mirroring or deleting destination files. The follow-up comparison used `/L` (list-only).

**Verification limits:** Robocopy's listing comparison checks file attributes, sizes, and timestamps, not all bytes. Hashing three metadata files does not hash all backup files or guarantee successful future restoration. The backups remain sensitive even though encrypted, and the external utility drive is not a substitute for an additional independent backup device.

## Decisions, tradeoffs, and lessons

1. **Preserve recoverability before experimentation.** The encrypted pre-update backup and its passphrase made recovery possible after the factory reset.
2. **Read exact diagnostic errors.** `1109` alone was broad; the updater log revealed the specific missing-WKMS-token message.
3. **Test layers separately.** Network TCP reachability, USB enumeration, Apple device services, and iTunes compatibility were separate concerns.
4. **Treat hypotheses as hypotheses.** The original WKMS root cause remained unknown even after a successful subsequent attempt.
5. **Treat "Update" and "Restore" carefully.** The selected action and actual outcome diverged. Verify what the device and iTunes report before claiming data was preserved.
6. **Keep independent recovery points.** Preserve the known-good pre-recovery snapshot separately before making a new post-recovery backup.
7. **Verify archival transfers.** Copy success codes, file inventories, and targeted hashes each provide useful but different levels of assurance.
8. **Do not publish raw recovery artifacts.** Log files can expose device identifiers; backup data and encryption credentials must remain outside ordinary version control.

## Follow-up

- [x] Confirm normal iPhone boot and basic app functionality
- [x] Recover data from the encrypted local backup
- [x] Create a fresh post-recovery encrypted backup
- [x] Archive pre- and post-recovery backup snapshots externally
- [x] Compare archived backup files and selected metadata hashes
- [ ] Periodically test a full restore on a suitable device or follow Apple's supported backup-validation procedure where feasible
- [ ] Maintain an additional backup on another independent physical device
- [ ] Review this report for public release before changing repository visibility

## Privacy note

The actual backup folders, raw logs, precise local account paths, device identifiers, Apple Account information, and encryption passphrase were deliberately excluded. These records should be kept in separately protected storage, not committed to GitHub.
