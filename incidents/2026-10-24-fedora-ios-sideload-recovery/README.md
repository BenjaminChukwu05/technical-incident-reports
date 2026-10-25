# Restoring SideStore and Aidoku from Fedora after iPhone recovery

| Field | Value |
| --- | --- |
| Incident date | 24 October 2026 |
| Host | Fedora Workstation 44, x86-64 |
| Device | iPhone |
| Software | iloader 2.3.4 AppImage / 2.4.0 RPM, usbmuxd, libimobiledevice, SideStore, LocalDevVPN, Aidoku, curl, aria2 |
| Outcome | SideStore installed and refreshed; Aidoku reinstalled; previous Aidoku library and reading progress preserved |
| Root-cause status | Slow GitHub release-asset transfers reproduced; underlying network cause not proven; authentication failure's exact cause unresolved |

> This is a sanitized incident report, not an official iloader, SideStore, or Aidoku installation guide. Never commit an Apple Account password, verification code, pairing file, signing certificate, device identifier, raw debug log, IPA file, or personal backup.

## Executive summary

Following an iPhone software recovery, SideStore and Aidoku needed to be installed again. The initial iloader workflow produced a USB communication error and later an Apple Account GrandSlam proof-login error. Separate tests showed that Fedora could detect the iPhone and communicate with the `usbmuxd` service, while the same Apple Account could sign in at Apple's website. Those tests narrowed the problems but did **not** prove an exact cause for the iloader authentication failure.

After iloader progressed to SideStore installation, the interface stayed at **Download SideStore**. A headers-only request to the GitHub release endpoint completed successfully, but a full-file download was independently measured at approximately **21 KB/s** and timed out after two minutes. Downloading the same official asset with eight `aria2c` connections completed successfully. An equivalent direct-download workaround was used for Aidoku. Both downloaded IPA archives passed basic ZIP structure checks.

SideStore was installed on the iPhone, Developer Mode and developer trust were enabled as needed, LocalDevVPN was connected, and SideStore successfully refreshed. Aidoku was then installed and opened with its previous library and reading progress intact. A separate Aidoku `.aib` backup was exported to Google Drive; the file's presence was checked, but a test restore was not performed.

## Context and evidence boundaries

This case follows the [iPhone iOS recovery incident](../2026-10-23-iphone-12-ios-recovery/README.md) but covers **app restoration**, not operating-system recovery. A previous [Fedora iloader TLS certificate case](https://github.com/BenjaminChukwu05/fedora-iloader-tls-fix) addressed a **different** September 2026 failure and must not be assumed to explain this October incident.

The evidence combines (1) a locally retained terminal recording from 24 October, (2) user-observed application states, and (3) subsequent confirmation that the applications worked on the iPhone. The recording captured terminal activity, **not** an authoritative trace of every action inside the iPhone or iloader GUI.

## Investigation

### 1. Separate USB detection from application recognition

iloader initially displayed a `usbmuxd` device-socket error. Rather than reinstall the daemon, the host was inspected with read-only commands:

```bash
rpm -q usbmuxd
lsusb -d 05ac:
systemctl status usbmuxd --no-pager
idevice_id -l
```

- `rpm -q` queries whether the RPM package is installed.
- `lsusb -d 05ac:` filters USB enumeration for Apple's vendor ID; it does not verify application pairing.
- `systemctl status` inspects the service without starting or restarting it.
- `idevice_id -l` checks whether libimobiledevice can enumerate a connected device. Its output contains a unique device identifier and must be **redacted** before sharing.

**Observed:** The package was installed, the daemon was running, the phone appeared on USB, and `idevice_id` returned a device entry.

**Interpretation:** Basic USB enumeration and the libimobiledevice path worked at that moment. This did not prove that iloader's own socket handling was functioning, nor that all pairing operations would succeed. Reinstalling or restarting system packages without further evidence was not justified.

The workflow moved from the older iloader AppImage toward the Fedora x86-64 RPM for version 2.4.0. Installing an RPM with `dnf` is a system change, unlike the preceding diagnostic commands; it should be reviewed in the package transaction before confirmation.

### 2. Distinguish Apple credentials from iloader's login protocol

iloader reported:

```text
GrandSlam error during proof login request
Auth error -22406: Enter the correct password for this Apple Account.
```

Signing in to `https://account.apple.com/` with that account succeeded.

**Observed:** Browser authentication worked, while iloader had reported a proof-login failure.

**Not established:** The precise reason for the iloader error, whether it depended on a particular Anisette server, or which later change allowed progress. A working web login does not guarantee that a third-party client's different authentication flow will work. Resetting the Apple Account password was not supported by the evidence.

The error was **not** identified as a recurrence of the earlier TLS certificate-chain problem.

### 3. Reproduce the download bottleneck outside iloader

The SideStore interface remained on **Download SideStore** without completing.

First, a headers-only request tested endpoint availability:

```bash
curl -L -I --max-time 20 \
  -o /dev/null \
  -w 'HTTP: %{http_code} | Time: %{time_total}s\n' \
  'https://github.com/SideStore/SideStore/releases/latest/download/SideStore.ipa'
```

Recorded result:

```text
HTTP: 200 | Time: 4.172105s
```

This checked HTTP response headers, **not** full-file throughput. A successful `200` response did not mean that the 27.77 MB asset would download promptly.

A separate full-file test was limited to 120 seconds and used HTTP/1.1:

```bash
curl --fail --location --http1.1 \
  --connect-timeout 15 \
  --max-time 120 \
  --remove-on-error \
  --progress-bar \
  --output "$HOME/Downloads/SideStore-direct-test.ipa" \
  --write-out '\nHTTP: %{http_code} | Downloaded: %{size_download} bytes | Time: %{time_total}s | Speed: %{speed_download} B/s\n' \
  'https://github.com/SideStore/SideStore/releases/latest/download/SideStore.ipa'
```

Recorded result:

```text
curl: (28) Operation timed out after 120165 milliseconds
HTTP: 200 | Downloaded: 2538885 bytes | Time: 120.164563s | Speed: 21128 B/s
```

The server supplied data, but transfer was **very slow**. The test used `--remove-on-error` to remove an incomplete output file. This behavior differs from a resumable download.

**Established:** The slow transfer was reproducible outside iloader and affected the path to a GitHub release asset.

**Not established:** Whether the cause was ISP routing, GitHub's asset hosting/CDN, congestion, rate limiting, or another network factor. The plan's advertised bandwidth by itself cannot identify that cause.

### 4. Download official IPA assets through aria2

The host already had `aria2c`. Using several simultaneous connections was a limited workaround for poor per-connection throughput:

```bash
aria2c \
  --split=8 \
  --max-connection-per-server=8 \
  --min-split-size=1M \
  --continue=true \
  --file-allocation=none \
  --dir="$HOME/Downloads" \
  --out="SideStore.ipa" \
  'https://github.com/SideStore/SideStore/releases/latest/download/SideStore.ipa'
```

The options allow up to eight connections, retain resumable partial state, avoid eager file preallocation, and save the download in the user's Downloads directory. This command **writes files locally** and accesses a public HTTPS endpoint; it does not modify Fedora packages or the iPhone.

The terminal recording showed:

```text
10/24 19:11:05 [NOTICE] Download complete
Status: OK
Average speed: 106 KiB/s
```

At that time GitHub's `releases/latest` endpoint redirected to SideStore `0.7.0-alpha`. The `latest` URL should not be equated with a stable-channel guarantee, and future users should check the intended release before downloading.

Aidoku was downloaded from the official project's release asset using the same method:

```bash
aria2c \
  --split=8 \
  --max-connection-per-server=8 \
  --min-split-size=1M \
  --continue=true \
  --file-allocation=none \
  --dir="$HOME/Downloads" \
  --out="Aidoku.ipa" \
  'https://github.com/Aidoku/Aidoku/releases/download/v0.9-4/Aidoku.ipa'
```

The terminal recording showed completion at approximately **19:34 local time**. The explicitly versioned Aidoku URL records the asset used during the incident and is **not** a recommendation to always install that version.

Both IPA files were checked with:

```bash
unzip -tq "$HOME/Downloads/SideStore.ipa"
unzip -tq "$HOME/Downloads/Aidoku.ipa"
```

The ZIP checks returned no archive errors. **Limit:** ZIP integrity testing does not establish publisher authenticity, code safety, or successful iOS signing.

### 5. Complete on-device installation and verify data

The recovered SideStore IPA was installed via the sideloading workflow, after which the user:

1. Enabled iOS Developer Mode and trusted the development-signed app as required.
2. Connected LocalDevVPN.
3. Opened and successfully refreshed SideStore.
4. Installed the separately downloaded Aidoku IPA via SideStore.
5. Confirmed that Aidoku's **previous library and reading progress had survived**.

The exact SideStore import and pairing-file GUI operations were not fully captured in the terminal recording. A successful refresh confirms a working user-visible result, but it does not justify inventing unrecorded GUI steps.

An Aidoku `.aib` backup was later exported and its presence in Google Drive verified. **Not tested:** Restoring that backup into Aidoku. An uploaded backup file is not equivalent to a successful recovery test.

## Results and lessons

| Area | Outcome | Important limit |
| --- | --- | --- |
| USB / usbmuxd | Package, running service, and device enumeration confirmed | iloader-specific pairing/socket cause not proven |
| Apple login | Browser login succeeded; later installation progressed | Cause of original `-22406` not established |
| SideStore download | Single connection timed out; segmented download completed | Underlying network bottleneck not identified |
| SideStore operation | App installed and manually refreshed | Future seven-day refresh reliability not yet established |
| Aidoku | Installed; original library and history visible | Not proof all user data was exhaustively compared |
| Aidoku backup | Exported to Google Drive | Restore was not tested |

The general diagnostic sequence was **observe → isolate the layer → reproduce the failure independently → apply a narrowly scoped workaround → verify the user-visible outcome**.

This incident is primarily a reproducible **troubleshooting method**, not a configuration module for `workstation-automation`. Any future automated downloader should use explicit versions, verify asset provenance/integrity when a trusted digest is available, handle interrupted downloads carefully, and require conscious user action before signing or installing onto an iPhone.

## Privacy and retained evidence

The local terminal recording included `commands.log`, `terminal.raw`, and `timing.log`. It and the iloader debug logs are **not included** in this repository. Raw logs may expose Apple Account information, unique device IDs, personal paths, signed GitHub asset URLs, authentication/session information, or pairing material. No private IPA, signing certificate, pairing record, or `.aib` backup is stored here.

The reusable documentation contains only sanitized commands, outcomes, limits, and links to official projects.

## Related projects

- [Previous iPhone software recovery](../2026-10-23-iphone-12-ios-recovery/README.md)
- [Separate Fedora iloader TLS troubleshooting](https://github.com/BenjaminChukwu05/fedora-iloader-tls-fix)
- [iloader](https://github.com/nab138/iloader)
- [SideStore](https://sidestore.io/)
- [Aidoku](https://aidoku.app/)
