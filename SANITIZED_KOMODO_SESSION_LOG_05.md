# SANITIZED SESSION LOG — KOMODO #05

**Sanitized for public publication.** Device identifiers, build fingerprints,
credential filenames and paths, account details, installed-app inventory and
current device security posture have been removed or generalized. The technical
findings, tool bugs and reusable lessons are preserved in full.

**Scope:** Linux workstation driving a rooted Android 17 Pixel over ADB.
Covers Android module auditing and remediation only. Does not describe server
infrastructure.

---

## 1. Root manager misidentification

A prior handoff document recorded the device's root manager as Magisk. It is
actually **APatch**. A stale `magisk.db` left in `/data/adb/` from an earlier
setup was the source of the error, and it had been carried forward unchallenged
across sessions.

📝 **A leftover database file is not evidence of the active root solution.**
Confirm against the running daemon and its binary path.

---

## 2. 🚨 Finding: a device-props spoofer silently reading a nonexistent config

A Zygisk props-spoofing module shipped a boot script that read its configuration
from a **flat file path**, while the module's own updater wrote that
configuration into a **directory of the same name**:

```
service.sh reads:   /data/adb/<MODULE>.json        <- did not exist
updater writes:     /data/adb/<MODULE>/<MODULE>.json
```

There was no existence guard. `cat` on the missing path failed silently, the
config variable came back empty, and two things followed:

1. **No identity profile was applied** — the WebUI showed "no profile set" and
   every field as "skip".
2. **Every resetprop group ran anyway** — the group-enable check looked for an
   explicit `"false"` in a config that was empty, so everything defaulted to on.

🚨 **The failure mode is indistinguishable from a deliberate configuration.**
A UI showing "nothing spoofed" plus "all masking groups active" is exactly what
an intentional setup looks like. Only reading the boot script and confirming
which path the *running* build opens revealed the truth.

### 2a. A same-version update that changed behaviour

A staged module update carried an **identical version string and versionCode**,
but its boot script had been rewritten to read the directory-based path with a
legacy fallback. On the next reboot it would have found a configuration the
updater had auto-fetched days earlier and applied it — with no user action and
no notification.

🚨 **Version equality is not behavioural equality.** Diff the scripts of a
staged module update before rebooting, especially when the version string has
not moved.

### 2b. Remediation and the settings-persistence trap

The profile was emptied and the boot auto-updater disabled. Three non-obvious
details:

- ⚠️ **Disabling the auto-updater was mandatory.** Left on, it rewrites the
  build fields — fingerprint among them — on a later boot, silently re-arming
  what was just disabled.
- ⚠️ **Settings had to live in the config JSON, not in flag files.** The newer
  build reads them from a settings object inside the config. A module update
  replaces the module directory wholesale, so flag files would not have survived
  the reboot.
- ⚡ An empty JSON object written as `{}` on a single line made the module's
  line-based parser report a hard error ("file cannot be read"). Written with
  the braces on separate lines it parsed clean. **A config that fails to parse
  makes such modules apply nothing, and often only the log says so.**

Verified stable across two reboots: config mtime unchanged, updater never fired,
module analyzer reporting clean.

---

## 3. 🚨 Tool bug: root backup utility reports zero-byte source sizes

A root backup module reported `orig_bytes: 0` for **every** item, in both its
dry-run plan and the final manifest of a completed backup. The real on-disk
sizes were multiple megabytes, confirmed with `du -sb`.

The archives themselves were correct and complete — `stored_bytes`, checksums
and contents were all accurate. Only the source-size accounting was broken.

🚨 **A dry run that reports zero bytes of app data is not evidence that there is
no data.** Had the plan been trusted, the reasonable conclusion would have been
"this app has nothing to back up."

### Verification performed

- Manifest self-checksum compared against computed value — matched.
- Every archive's checksum compared against its manifest entry — all matched.
- Archives decompressed and listed; the application's primary data file and
  database (plus WAL and SHM sidecars) confirmed present with expected sizes.
- One archive was near-empty; investigation confirmed that tree contained only
  cache directories, and cache exclusion was enabled by configuration. **Correct
  behaviour, not a gap** — worth checking rather than assuming either way.

📝 **Verify the artifact, not the exit code.** A success status plus a plausible
summary is not proof that the data is inside the archive.

---

## 4. 🚨 Secrets and the ADB transport

`adbd` writes **full command strings to logcat**, including everything passed to
`su -c`. This was observed directly: the session's own shell commands appeared
verbatim in the device log.

🚨 **Never pass a passphrase, key, or token through `adb shell`.** Any backup
encryption passphrase, repository password or credential must be entered on the
device itself. A secret used once over ADB should be treated as disclosed.

---

## 5. Credential hygiene on shared storage

A credential file was found in the device's shared `Download` directory. It was
relocated to root-only storage (`0600`, root-owned, restricted SELinux context,
inside a `0700` directory) and the original removed.

Procedure used:

1. Copy to the protected location and set ownership, mode and SELinux context.
2. Compare SHA-256 of source and destination.
3. **Re-verify the hash immediately before deleting the original.**
4. Confirm the source directory is clean afterwards.

📝 **On an encrypted-container question:** adding one here would have been
theatre. The relevant directory is *device-encrypted* storage — unlocked at boot,
before any user credential is entered — because the consuming module must read
its data at boot unattended. Encrypting a spare copy while the actively-used one
sits in plaintext in the same directory buys nothing. Spare credentials belong
off the device entirely, encrypted at rest.

---

## 6. Android storage permissions — what `media_rw` does and does not mean

Files under `/sdcard` display as owned by an application UID with group
`media_rw` and SELinux context `u:object_r:fuse:s0`. This is routinely
misread as evidence of broad access.

- The owning UID is the platform **MediaProvider** component, not whichever app
  created the file. **Every** file in shared storage looks this way.
- Those owner/group bits are **synthesized by the FUSE daemon**. They are not
  the access control mechanism.
- Actual enforcement is AppOps plus the storage daemon. Since scoped storage,
  reading an arbitrary non-media file in shared storage requires
  `MANAGE_EXTERNAL_STORAGE` ("All files access"), which must be granted
  explicitly and is restricted for Play-distributed apps.

### ⚡ Methodology error worth recording

An `awk` pass over `dumpsys appops` output appeared to show several applications
holding All-files access. The script was pairing package headers with the wrong
permission blocks. A per-package query —
`appops get <pkg> MANAGE_EXTERNAL_STORAGE` — disproved it.

🚨 **Do not derive per-package facts by parsing grouped `dumpsys` output.**
Query each package directly. The bulk-parse was wrong in the direction of
false alarm, which is the direction most likely to prompt unnecessary action.

---

## 7. SQLite state with a write-ahead log

Framework configuration stored in SQLite was copied out as a bare `.db` file and
queried. The result was a **stale pre-checkpoint snapshot** that omitted recent
changes, leading to an incorrect report that a component was not enabled when it
already was.

⚡ **Copying `foo.db` without `foo.db-wal` reads outdated state.** Either copy
the sidecars too, or use the owning application's own interface. In this case a
framework CLI existed and was authoritative.

---

## 8. Three ways to modify an Android app — and their very different risk

| Method | Package identity | APK on disk | Store-update risk |
|---|---|---|---|
| **Mount module** — patched APK bind-mounted over the original | unchanged | replaced at runtime | **High** — store still manages the package and will update the base out from under the mount |
| **Non-root clone** — repackaged under a new name | changed | new package | None — the store has no record of it |
| **Runtime hooking** — code-injection framework against a stock app | unchanged | **untouched** | None — and updates are desirable |

### Why this matters for integrity-sensitive apps

A patched APK is modified and **re-signed with a key the vendor does not
recognize**. Applications that verify their own package signature detect this.
For messaging platforms that enforce it, the penalty is an account-level ban,
not a crash — and it can take the registered phone number with it.

A runtime-hooking framework modifies nothing on disk. The application's own
integrity checks examine an unmodified, vendor-signed APK and pass. A long
history of using such a framework without incident is therefore **not** evidence
that patching the same app would be safe — the mechanisms are entirely
different, and only one of them is detectable by signature verification.

📝 **Where a modification method sits on this table matters more than how
aggressive the modification is.** Runtime hooking commonly offers a *superset*
of what a patch bundle provides, at none of the signature risk.

### Store-detachment decision rule

Detachment tooling hides an app from the store's update checks. Apply it to:

- **Mount-module patched apps** — required, and the failure is silent: the store
  updates the base, the mount goes stale, and the modification disappears with
  no error.
- **Sideloaded modified APKs reusing a published package name** — required.

Do **not** apply it to:

- **Clones under a new package name** — nothing to detach.
- **Runtime-hooked stock apps** — actively harmful; it freezes the app on an
  aging version and forfeits vendor security updates for no benefit.
- **Apps not distributed through the store at all** — a no-op.

---

## LESSONS LEARNED

1. A **missing** configuration file can be indistinguishable from a deliberate
   configuration. Confirm which path the running build actually opens.
2. **Version equality is not behavioural equality.** Diff staged module scripts
   before rebooting.
3. **Copying a `.db` without its `-wal` sidecar yields stale state.** Prefer the
   owning application's interface.
4. **Never derive per-package facts from grouped `dumpsys` output.** Query per
   package.
5. **A tool's own plan or dry-run output is not evidence.** Verify the artifact.
6. **`adbd` writes command strings to logcat.** Never pass a secret through
   `adb shell`.
7. **Re-verify a checksum immediately before deleting any original.**
8. Where a modification sits on the mount/clone/hook table determines its
   detection risk — not how extensive the modification is.

---

*Device identifiers, build fingerprints, credential names and paths, account
details, installed-application inventory and current device security posture
have been deliberately omitted. Operational detail is retained in the
corresponding unsanitized log, which is not published.*
