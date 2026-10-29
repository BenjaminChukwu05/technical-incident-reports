# Restoring Secure Boot on a Fedora/Windows dual-boot workstation and testing Ventoy

| Field | Value |
| --- | --- |
| Investigation | 29 October 2026 |
| Host | ThinkPad P15 Gen 2; Fedora 44 and Windows 11 on separate NVMe drives |
| Scope | UEFI Secure Boot, encrypted Fedora /boot, NVIDIA signing, BitLocker/TPM, Ventoy USB |
| Outcome | Both operating systems booted with Secure Boot enabled; NVIDIA worked; Ventoy updated to 1.1.18; Omarchy remained unsupported under the tested Secure Boot path |
| Confidence | Missing NVIDIA MOK enrollment confirmed; precise historical reason Secure Boot had been disabled not independently established |

> Sanitized historical case study, **not a general-purpose Secure Boot runbook**. The raw terminal archive, device identifiers, disk/LUKS UUIDs, MOK fingerprints, recovery keys, user paths and sensitive screenshots are intentionally excluded.

## Summary

Secure Boot had been left disabled during an earlier Fedora configuration and bootloader repair. Before switching it back on, the host was audited for a trustworthy shim/GRUB path, encrypted boot compatibility, NVIDIA module signing and Windows BitLocker recovery risk. A locally signed NVIDIA module was present, but its existing akmods public certificate was not enrolled. That certificate was enrolled through MOK Manager; no new key was generated.

A controlled firmware change enabled Secure Boot without clearing or replacing firmware keys. Fedora then booted successfully despite having `/boot` inside LUKS2/Btrfs, NVIDIA remained operational, and kernel lockdown entered integrity mode. Windows also started without BitLocker recovery and reported Secure Boot enabled, BitLocker protection on, and TPM ready.

A separate Ventoy USB was upgraded from 1.0.99 to 1.1.18 without reinstalling partitions. The inventory of 12 ISO/image files remained present. Other tested Linux live images worked, but Omarchy reported a Ventoy security-boot violation. Secure Boot was **not** disabled to accommodate this one ISO.

## Baseline and risk assessment

- Fedora 44, kernel `7.2.11-200.fc44.x86_64`; Fedora EFI entry pointed to `shimx64.efi`.
- Customized GRUB EFI configuration used `cryptomount` and `GRUB_ENABLE_CRYPTODISK=y` to read an encrypted Btrfs `/boot`.
- NVIDIA RTX A2000 Laptop GPU used an RPM Fusion akmods driver.
- Windows 11 used BitLocker on a separate NVMe disk and had TPM available.
- Ventoy USB had GPT, exFAT data and FAT EFI partitions; initial installed version was **1.0.99** with Secure Boot support enabled.

Possible independent failure points included signed-EFI loading, GRUB module availability under lockdown, LUKS2 PBKDF compatibility, third-party module trust, Windows TPM measurement changes, and USB-specific bootloader trust. They were assessed separately instead of changing everything at once.

## 1. Read-only audit

Fedora checks (some require administrator permissions to read protected files):

```bash
mokutil --sb-state
sudo efibootmgr -v
lsblk -o NAME,TYPE,FSTYPE,LABEL,MOUNTPOINTS
findmnt -T /boot
findmnt -T /boot/efi
rpm -q shim-x64 grub2-efi-x64 grub2-efi-x64-modules akmods akmod-nvidia
modinfo -F signer nvidia
modinfo -F sig_hashalgo nvidia
sudo mokutil --test-key /etc/pki/akmods/certs/public_key.der
sudo mokutil --list-new
sudo rpm -V shim-x64 grub2-efi-x64 grub2-efi-x64-modules
```

**Confirmed:** Secure Boot was off. NVIDIA was already signed using a local akmods identity, but the corresponding certificate was **not enrolled**, and no MOK request was pending. The package verification run with `sudo` produced no mismatches. An earlier unprivileged `rpm -V` produced permission errors; those were **not evidence of corruption**.

A targeted `cryptsetup luksDump` inspection identified three LUKS2 keyslots: one Argon2id and two PBKDF2. No key material or passphrases were collected for publication.

A string scan of the installed Fedora GRUB EFI executable found `cryptomount`, `cryptodisk`, `luks2`, `btrfs`, `gcry_sha256`, `gcry_rijndael` and `pbkdf2`. The first attempt without elevated file-read privileges falsely reported that all were absent. Repeating it with `sudo` corrected that mistake. **String matches were capability indicators, not proof of embedded and usable Secure Boot modules.** Neither `sbverify` nor `pesign` was available for a separate signature listing.

## 2. Targeted repair: enroll the existing NVIDIA public certificate

After checking the subject against the NVIDIA module signer, the already existing akmods certificate was queued:

```bash
sudo mokutil --import /etc/pki/akmods/certs/public_key.der
sudo mokutil --list-new
```

The enrollment was confirmed via the shim/MOK Manager interface at reboot. The next check returned `public_key.der is already enrolled`; Secure Boot was still disabled at this intermediate checkpoint.

**Risk:** MOK enrollment changes the computer's trusted-signing-key set. This is a case-specific operation, **not** a command to blindly copy to client computers. No replacement keys, GRUB reinstall, NVIDIA rebuild, or LUKS changes were needed.

## 3. Controlled Secure Boot test and verification

After preparing access to recovery credentials, Secure Boot was enabled in firmware without clearing trusted keys. The normal Fedora boot completed, demonstrating that this machine's shim/GRUB/encrypted-`/boot` path worked under its active Secure Boot policy.

```bash
mokutil --sb-state
modinfo -F signer nvidia
nvidia-smi -L
cat /sys/kernel/security/lockdown
```

Observed:

```text
SecureBoot enabled
NVIDIA RTX A2000 Laptop GPU detected
none [integrity] confidentiality
```

The brackets indicate that kernel **integrity** lockdown mode was active. This demonstrates the observed boot path worked, not that every future kernel or recovery entry has been tested.

Windows was booted separately. It reached the desktop without requesting BitLocker recovery.

```powershell
Confirm-SecureBootUEFI
Get-BitLockerVolume -MountPoint 'C:' |
    Select-Object MountPoint, VolumeStatus, ProtectionStatus, EncryptionPercentage
Get-Tpm | Select-Object TpmPresent, TpmReady, TpmEnabled
```

Observed: `Confirm-SecureBootUEFI` returned `True`; BitLocker showed `FullyEncrypted`, protection `On`, 100%; TPM showed present/ready/enabled. BitLocker protection did not need to be resumed.

## 4. Separate Ventoy update and live-media validation

The Ventoy Linux release archive for **1.1.18** passed SHA-256 verification. Before updating, `Ventoy2Disk.sh -l` confirmed USB version **1.0.99**, GPT and Secure Boot support `YES`.

An initial update script invoked the interactive updater inside a Bash heredoc. The updater's yes/no prompt did not receive an answer, and a subsequent version check still read **1.0.99**. This was caught before treating the update as a success.

A subsequent invocation was run directly in the terminal against the independently identified USB device:

```bash
# Replace the placeholder only after verifying the intended USB disk.
sudo sh ./Ventoy2Disk.sh -u -s /dev/<USB-DISK>
# Answer the displayed confirmation prompt interactively.
sudo sh ./Ventoy2Disk.sh -l /dev/<USB-DISK>
```

The installer reported success. The readback then showed **Ventoy Version in Disk: 1.1.18**, GPT and `Secure Boot Support: YES`. Post-update inspection found all **12** ISO/image files in the `/_ISOs` tree and retained `VTOY_DEFAULT_SEARCH_ROOT="/_ISOs"`. Approximately 425 GB remained used on the exFAT volume. File presence is not equivalent to a full hash validation.

Ventoy documents that its `VTOY_SECURE_BOOT_POLICY` value `"0"` bypasses subsequent EFI verification, while `"1"` follows the UEFI Secure Boot policy. The terminal command history contains an attempted backed-up change to `"1"`, but an independent **post-write JSON readback was not available** in the supplied recording. Therefore the effective setting is not claimed as conclusively verified.

An initial Ventoy boot attempt displayed `Verification failed: (0x1A) Security Violation`. Subsequent live sessions reached the Fedora KDE desktop; the operator reported that the other Linux distributions tested also booted. The exact action between that initial error and the successful sessions was not fully recorded, so the initial verification error's resolution is not attributed to a specific certificate or step.

**Omarchy exception:** its live EFI boot path displayed repeated:

```text
Ventoy Security Boot Violation
.../CDROM(0x1)/EFI/BOOT/BOOTX64.EFI
```

The [official Omarchy installation guide](https://omarchy.org/manual/getting-started/) instructs users to turn off Secure Boot and/or TPM. This is consistent with the observed incompatibility; it does not prove that the ISO is corrupt. The decision was to preserve the now-working host Secure Boot configuration and consider testing Omarchy separately in a VM. No physical Omarchy installation was attempted.

Ventoy references: [Secure Boot documentation](https://www.ventoy.net/en/doc_secure.html), [global control plugin and policy](https://www.ventoy.net/en/plugin_control.html), [release notes](https://www.ventoy.net/en/doc_news.html).

## Result, limitations and lessons

| Check | Outcome |
| --- | --- |
| Fedora Secure Boot and encrypted boot | **Passed** |
| NVIDIA with enrolled local signing key | **Passed** |
| Windows Secure Boot | **Passed** |
| BitLocker and TPM | **Passed** |
| Ventoy version and ISO filename inventory | **Passed** |
| Other tested Linux live images | **Reported successful** |
| Omarchy live boot under Secure Boot | **Failed; unsupported workflow retained as an exception** |
| Strict Ventoy policy readback | **Not independently verified** |
| Every ISO's hash and all recovery/Windows boot paths | **Not tested** |

**Reusable lessons:** audit before changing firmware; distinguish module signatures from trust enrollment; avoid confusing permission errors with file corruption; do not treat `strings` as attestation; confirm actual installer version after interactive updates; check file contents separately from filename inventory; keep BitLocker recovery preparations available; do not weaken an otherwise verified Secure Boot configuration for a single unsupported image.

## Evidence boundaries

The private `fix_secure_boot` terminal recording covers Fedora diagnostics, MOK preparation, Ventoy inspection and update commands. Windows PowerShell results and live-boot screens were supplied separately by the operator during the case. No raw logs, screenshots with identifiers, machine IDs, usernames, disk UUIDs, key fingerprints, passwords or BitLocker recovery materials are published here.
