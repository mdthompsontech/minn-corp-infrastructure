# MASTER BASELINE v17 — MINN-CORP INFRASTRUCTURE

> **PUBLIC / SANITIZED VERSION.** This is a real, actively-maintained infrastructure baseline with network identifiers replaced for publication: public IPs use RFC 5737/3849 documentation ranges (`198.51.100.x`, `192.0.2.x`, `203.0.113.x`, `2001:db8::/32`), the domain is rewritten to `.example`, and account emails, instance IDs, and policy UUIDs are replaced or redacted. Internal RFC 1918 addressing, hostnames, architecture, configuration, incident history, and verification status are unaltered. Nothing in this document was written for publication after the fact — it is the working document, sanitized.

**Date:** 2026-08-08
**Owner:** Matthew Thompson
**Supersedes:** all prior versions (v8–v16)
**Scope:** Complete technical documentation of the Contabo production VPS, the Claude Code deployments (VPS and Fedora), host SSH protection, the Cloudflare Zero Trust layer, and the Fedora workstation's relationship to them.

**What this document is:** a standalone infrastructure reference, readable cold by anyone competent with Linux, Docker, and DNS. Every claim is dated where verified; §14 states plainly what has and has not been confirmed.

**What this document is not:** a changelog, a tutorial, or a place where secrets live. Credentials are referenced only by on-disk or vault location.

**Changes in v17:**
- **§12** — v16 items #1, #2, #3, and the Fedora half of #12 close: the on-disk baseline copy was written (824 lines), `~/CLAUDE.md` was updated to 43 lines with four v16 facts, and all four drafted Fedora Skills are placed and registered (verified 2026-08-05). Items renumbered; new items added for the workstation changes and the portfolio sanitization pass.
- **§13** — two new incidents: the 2026-08-04 v16 document-divergence event (an agent edited a just-drafted versioned snapshot in place, producing two divergent copies labeled v16), and a 2026-08-08 ungated command chain that created a stray file when its preceding step failed silently.
- **§4.4** — new standing rule generalized from the divergence incident: **a versioned baseline is a snapshot, not a live tracker.** Items drafted open in vN stay open in vN and close in vN+1. Agents will attempt in-place closes; the rule exists because it happened twice in one day.
- **§4.6** — `~/CLAUDE.md` is now 43 lines; contents updated. §4.7 — the Fedora Skills directory now holds four registered Skills.
- **New §6.6 — NordVPN on the Fedora workstation.** Installed 2026-08-08 for manual, short-duration (~1h) sessions only. Documents the DNF5 packaging workaround (GPG verification never weakened), the split-tunnel allowlist covering both the VPS and the home LAN, and the systemd socket-activation trap.
- **New §6.7 — Open Notebook on the Fedora workstation.** Podman-managed, loopback-only, no boot auto-start. Runbook on-machine.
- **§6.1** — `IdentitiesOnly yes` added to the `contabo` SSH block; rationale recorded (agent key offers were generating `Failed publickey` entries on the VPS and counting toward the fail2ban threshold — a quiet self-ban vector, now closed).
- **§3.5** — new coupling note: the `ignoreip` exemption for the workstation's residential IP holds during VPN sessions only because of the `198.51.100.20/32` NordVPN allowlist.
- **§6** — workstation hostname (`minn-corp-workstation`) recorded explicitly.
- **Companion documents** — RULES is now **v1.3** (Hermes-era instruction stripped; six-context execution labeling; security rules collapsed to a conversational core pointing at §9 here).
- **§14** — verification status updated for everything touched 2026-08-05 through 2026-08-08.

**Section numbering note:** §9 (security policy), §12 (open items), and §14 (verification) numbers are preserved deliberately, because other documents and `CLAUDE.md` reference them.

---

## 1. SYSTEM OVERVIEW

A single production VPS (Contabo, Rocky Linux 10) hosts:

- **Open WebUI** (Docker) — chat frontend connected to several external model providers. The only container in the stack.
- **Claude Code** — Anthropic's interactive terminal agent, installed under the non-root user `minncore`. Runs on the host (not containerized), operated over SSH. This is the administrative AI tooling for the box.
- **nginx** (host) — reverse proxy with Cloudflare Access JWT validation for two service-auth backends.
- **cloudflared** (host) — Cloudflare Zero Trust tunnel. The VPS IP is never published in DNS.
- **fail2ban** (host) — automated SSH brute-force banning via firewalld/ipset.

**Network protection is layered four deep:** Contabo's cloud firewall (hypervisor) → firewalld (host) → fail2ban dynamic bans (host) → Docker loopback-only port publishes.

A **Fedora 44 workstation** (hostname `minn-corp-workstation`) is the daily driver and primary administrative client. It runs its own separate cloudflared tunnel for home-LAN devices — an independent trust boundary sharing only the `minn-corp.example` DNS zone. It also runs local LLM inference (llama.cpp, Ollama, Docker-managed), a Podman-managed local research app (Open Notebook, §6.7), a manually-invoked NordVPN client (§6.6), and **its own independent Claude Code install** (§4.7).

**Historical note:** from Jul 1–21 2026 this VPS ran the Nous Research Hermes Agent (autonomous agent runtime with Telegram bot, OpenAI-compatible API on 8642, dashboard on 9119, and GitHub memory backup). It was fully decommissioned 2026-07-21; nothing of it remains except `/root/salvage/` (§11) and the incident history (§13). If any hermes-named artifact (container, network, port 8642/9119 listener, DNS route, Cloudflare Access application) reappears, that is a finding, not a leftover.

---

## 2. CURRENT OPERATIONAL STATUS

| Component | State |
|---|---|
| Open WebUI (3000) | ✅ Running — loopback-only; HTTP 200 on localhost verified 2026-08-04 |
| Claude Code under minncore (VPS) | ✅ Operational; v2.1.220 as of 2026-07-28 (self-updates) |
| Claude Code on Fedora | ✅ Operational; v2.1.221, Pro subscription OAuth — verified 2026-08-02 (§4.7) |
| fail2ban | ✅ Active, enabled at boot, enforcing; survived reboot 2026-07-30 |
| nginx | ✅ `active (running)` since 2026-08-02 — verified 2026-08-04 |
| cloudflared (VPS) | ✅ v2026.7.3, `active (running)`, enabled — verified 2026-08-04 |
| Cloudflare Access apps/policies | ✅ 5 applications, 3 policies, rule contents confirmed 2026-08-04 |
| Contabo cloud firewall | ✅ Correctly restrictive (audited 2026-07-14) |
| firewalld | ✅ public zone, ssh + dhcpv6-client only — verified 2026-07-30 |
| Contabo panel 2FA | ✅ Active — verified 2026-07-30 |
| Auto Backup | ⚠️ Enabled, 10 backups, daily cadence — verified 2026-08-04. **Provider warns restores may be temporarily unavailable in US-central** |
| Snapshot | ✅ 1 exists, taken 2026-07-21 pre-decommission (the free slot is used) |
| VNC console | ✅ Reachable, credentials retrieved — verified 2026-08-02. Subject to the ~6h idle constraint (§3.4) |
| exim MTA | ✅ Present but `disabled` / `inactive (dead)` — retained deliberately (§3.5) |
| Disk | ✅ 38% used (37G/99G) — measured 2026-07-30 |
| Telegram firmware alerting | ✅ Verified end-to-end 2026-08-02 (§11) |

---

## 3. THE VPS — CONTABO

### 3.1 Hardware and identity

- **Provider:** Contabo Cloud VPS 20 NVMe · **Instance:** `vmi0000000` ("My Minn-VPS")
- **IPv4:** `198.51.100.20` · **IPv6:** `2001:db8:0:1::1` · **MAC:** `00:00:5e:00:53:01`
- **Hostname:** `contabo-vps` · **Region:** US-central · **Host system:** (redacted)
- **Specs:** 6 vCPU · 12 GB RAM · 100 GB NVMe
- **OS:** Rocky Linux 10.2 (Red Quartz), kernel 6.12.x el10_2
- **Cost:** ~EUR 11.10/mo including Auto Backup add-on · **Created:** Jul 1 2026

**Rocky 10 ships MINIMAL.** Verify a tool exists before relying on it: `rpm -qa | grep <tool>`

**Repositories:** baseos, appstream, extras, **and EPEL** (added 2026-07-30 via `epel-release-10-7.el10_1`). EPEL was required because `fail2ban` is not carried in Rocky's default repositories — a bare `dnf install fail2ban` fails with `No match for argument` until EPEL is enabled. The EPEL install scriptlet also recommends enabling the CodeReady Builder repo (`/usr/bin/crb enable`); this was **not** done and is not currently needed.

**Cockpit was enabled by default** by the installer on `0.0.0.0:9090`; found and permanently disabled 2026-07-11 (`systemctl disable --now cockpit.socket`). Its MOTD suggestion to re-enable it still prints at login — ignore it. If 9090 is ever listening again, investigate.

### 3.2 SSH and access

Host-wide sshd config (verified 2026-07-11): `PermitRootLogin prohibit-password` · `PasswordAuthentication no` → **all accounts are key-only; password authentication is globally disabled.**

**Two user accounts (plus service accounts):**

- **root** — `/root/.ssh/authorized_keys` contains exactly two keys: `Minn-Corp@fedora` (ed25519) and `pixel9-termux` (ed25519). Any third key is a security finding. Verify: `wc -l ~/.ssh/authorized_keys`
- **minncore** (created 2026-07-21, display name "Minn-Core") — the Claude Code operator account. Member of `wheel` (sudo) and `docker` (gid 992). Same two authorized keys copied from root (ownership minncore:minncore, dir 700, file 600). Password is set and confirmed working for interactive sudo (verified 2026-07-30).
- **exim** (UID 93, GID 93) — created automatically 2026-07-30 as a dependency of the fail2ban install. Service account, no login, MTA is disabled (§3.5).

**How to connect:**
- From Fedora: `ssh contabo` — **lands as root**, because Fedora's SSH config hardcodes `User root` for this host. This is by design, not a misconfiguration. As of 2026-08-08 the block also carries `IdentitiesOnly yes` (§6.1) — the client offers only the pinned key, never agent keys.
- From Fedora, as the operator account: `ssh minncore@198.51.100.20` — verified working 2026-08-04. Key-based, no password prompt, no SELinux context problem.
- From Pixel 9 Termux: `ssh contabo` (root). Termux is transit only — nothing executes locally on the device.
- To operate Claude Code: either connect directly as minncore (above), or `ssh contabo` then `su - minncore`, then `claude`.
- Last resort: Contabo VNC console (§3.4) — read the idle constraint before relying on it.

**SSH scanner noise is constant and heavy.** As of 2026-07-30, `lastb` showed **335,715 failed login attempts since 2026-07-02** — sustained, high-volume credential-stuffing against `root` and a rotation of generic usernames (`git`, `web1`, etc.), from a wide spread of source IPs. Top offenders clustered in `91.92.4x.x`, `45.148.10.x`, and `45.153.34.x` blocks, suggesting single actors rotating through subnets. **Key-only auth means none of this traffic can succeed**, but the volume is real resource and log churn — which is why fail2ban was installed (§3.5).

### 3.3 Firewall — three static layers

**Layer 0 — Contabo cloud firewall** ("My Contabo Firewall", panel-managed, hypervisor-level): ACCEPT TCP 22 from Any · DROP everything else. Last changed Jul 1 2026. ⚠️ Unconfirmed whether it covers the VNC console IP (`192.0.2.10`) — assume it does not.

**Layer 1 — firewalld (host):** public zone on `ens18`, services `ssh` and `dhcpv6-client` ONLY. No extra ports, no rich rules, no masquerade. SELinux Enforcing. Verified 2026-07-30. Verify: `firewall-cmd --list-all`

**Layer 2 — Docker publishes:** `open-webui` → `127.0.0.1:3000` only. The protection is the loopback bind — external packets never reach the DNAT path.

**No inbound 80/443 rule is needed and none exists.** The Cloudflare tunnel is outbound-only: cloudflared dials out to Cloudflare's edge and reaches nginx over localhost. By design, not a gap.

**Note on fail2ban:** its bans are dynamic firewalld/ipset entries, not part of the static zone config above. `firewall-cmd --list-all` will not show them; use `ipset list` or `fail2ban-client status sshd` (§3.5).

### 3.4 VNC console

**Endpoint:** `192.0.2.10:63138` — a Contabo console-proxy IP, not the VPS and not a guest-OS service. Managed in the panel (⋮ → VNC Control); credentials in panel Secret Management. Endpoint re-confirmed unchanged 2026-08-02.

✅ **Credentials have been retrieved and are stored operator-side** (2026-08-02). The storage requirement is that they be reachable *during* an SSH lockout — a copy that only exists on the Fedora workstation is insufficient if the lockout scenario involves being away from that desk.

✅ **Reachability proven 2026-08-02:** `nc -zv 192.0.2.10 63138` → `Connected`. Prior to this the console had never once been confirmed live.

🚨 **THE SIX-HOUR IDLE CONSTRAINT — read this before treating VNC as a recovery path.**

Owner-observed, first-party behavior on this instance: **the console connects reliably, but if no VNC login occurs within roughly six hours of activation or last use, subsequent connection attempts are refused until the VPS is rebooted.**

This was established by direct observation, not inference. Two other theories were considered and discarded:
- A 24-hour activation delay — the timeline does not support it (the toggle predated a reboot by two days, and the console was still refused afterward).
- The community-reported "stuck session after an abrupt disconnect" pattern — that mechanism requires a prior successful connection to have gone stale; here none had ever succeeded.

**Practical consequence, and it is significant:** this console cannot be treated as always-ready out-of-band recovery. If SSH access is lost and more than roughly six hours have passed since VNC was last used, the console will refuse, and a reboot will be required *before* the recovery path is usable — stacking a second problem onto the emergency that motivated using it.

**Two viable postures, neither yet chosen (§12):**
1. Accept reboot-on-demand as the cost of this recovery path.
2. Establish a "keep-warm" practice — periodically log into the console so it does not wedge before it is needed.

🚨 **Enable/disable requires a reboot to apply** (documented Contabo behavior). The panel toggle shows SAVED config, not LIVE listener state — they can disagree.

**Canonical diagnostic, from an external machine only:** `nc -zv 192.0.2.10 63138` — `Connected` = up; `Connection refused` = wedged or not provisioned. Never run this from the VPS; testing a rescue path from the box it is meant to rescue proves nothing.

**VPN caveat:** if the workstation is inside a NordVPN session (§6.6), this diagnostic runs from a shared VPN exit IP, a path never tested against the console proxy. Under the current manual/short-session VPN posture this is a non-issue — a lockout during a deliberate VPN hour is recovered by disconnecting the VPN — but if VPN-on ever becomes the default posture, pre-test this path.

### 3.5 fail2ban

Automated banning of brute-force SSH sources. Installed 2026-07-30 in response to the 335,715-attempt figure in §3.2 — not because key-only auth was failing (it was not), but because unthrottled sustained scanning is real CPU, log, and journal load with a single defensive layer behind it.

**Packages:** `fail2ban-1.1.0-6.el10_0` plus `fail2ban-server`, `fail2ban-firewalld`, `fail2ban-selinux`, `fail2ban-sendmail`, all from EPEL.

**Config:** `/etc/fail2ban/jail.local`

```
[DEFAULT]
banaction = firewallcmd-ipset
bantime   = 1h
findtime  = 10m
maxretry  = 5
bantime.increment = true
bantime.factor    = 2
bantime.maxtime   = 1w
ignoreip = 127.0.0.1/8 ::1 203.0.113.55 2001:db8:2::/64
backend  = auto

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/secure
maxretry = 5
findtime = 10m
bantime  = 1h
```

**Behavior:** 5 failures within 10 minutes → 1 hour ban. Repeat offenses double the ban (1h → 2h → 4h → 8h…) capped at one week, so no IP is banned permanently by exponential drift.

**`banaction = firewallcmd-ipset` is deliberate, not default.** This box uses firewalld, not raw iptables. `ipset` (v7.22) is installed, making the ipset-backed action viable and preferred — ipset lookups are O(1) regardless of ban-list size, while rich-rules degrade as the list grows. **A fail2ban install using an iptables banaction on this box would appear healthy, log bans correctly, and block nothing** — it would be issuing rules to a ruleset that is not in use.

📝 **`logpath` is effectively inert.** With `backend = auto`, fail2ban selected **journald**, not the file. Live status shows `Journal matches: _SYSTEMD_UNIT=sshd.service + _COMM=sshd + _COMM=sshd-session`. This is normal and arguably preferable (immune to log rotation/truncation), but a reader should not assume `/var/log/secure` is what is being read.

✅ **Reboot survival verified 2026-07-30.** After a VPS restart, fail2ban came back `active (running)` and `enabled` at 23:08:08 CEST with no manual intervention, reporting `Server ready`. `ipset list -n` returned `f2b-sshd`, confirming enforcement — not merely service health — persisted across the cycle. Effectiveness at ~12h after install: 84 total banned, 17 currently banned.

🚨 **`ignoreip` contains two dynamic addresses and will go stale.**
- `203.0.113.55` — the Fedora workstation's public IPv4 as of 2026-07-30. Residential ISP; not contractually static.
- `2001:db8:2::/64` — the Pixel 9's mobile carrier IPv6 prefix. The /64 (not the full address) was used deliberately, because IPv6 privacy extensions rotate the low 64 bits frequently while the prefix is comparatively stable. Still not guaranteed permanent.

If either changes and is not updated, the operator's own connections lose ignore protection and become bannable. **The VNC console (§3.4) is the recovery path if that happens — subject to its six-hour idle constraint.** No drift detection currently exists (§12).

🚨 **New coupling as of 2026-08-08:** the workstation now carries a NordVPN client (§6.6). The `203.0.113.55` exemption above holds during VPN sessions **only** because NordVPN's allowlist splits `198.51.100.20/32` out of the tunnel, keeping SSH-to-VPS on the residential source IP. If that allowlist entry is ever removed, workstation SSH traffic during a VPN session arrives from a shared, recycled NordVPN exit IP — bannable like any other source, and plausibly *already banned* given the volume of credential-stuffing this box attracts. Do not remove that allowlist entry without re-reading this paragraph.

**Additionally closed self-ban vector (2026-08-08):** before `IdentitiesOnly yes` was added to the Fedora SSH config (§6.1), the client could offer agent keys ahead of the pinned IdentityFile; each rejected offer logged a `Failed publickey` on the VPS and counted toward `maxretry=5`/`findtime=10m`. The config change closes it.

🚨 **DO NOT remove `fail2ban-sendmail`.** Earlier documentation listed this as an optional cosmetic cleanup, on the reasoning that it drags in a dormant `exim` MTA and ~65 perl packages for an email-alert feature that is unused here. **That recommendation is withdrawn.**

A `dnf remove fail2ban-sendmail` attempted 2026-08-04 resolved to a **70-package transaction that included the `fail2ban` meta-package itself** under "Removing dependent packages." Notably `fail2ban-server` was *not* in that list. On this box the packaging inverts the expected relationship: `fail2ban` behaves as a meta-package depending on `fail2ban-sendmail`, rather than the plugin depending on the base. Whether removing the meta-package would strip configuration or action files the running daemon relies on — including the `firewallcmd-ipset` action that makes bans actually reach the firewall — is **unknown and was not tested.**

The transaction was aborted before any package was touched. `exim` is `disabled`, `inactive (dead)`, and creates no listener; that is a small and already-contained cost. Risking the box's primary brute-force control to remove it is a bad trade. Anyone revisiting this should begin with `dnf repoquery --requires fail2ban` to map the actual dependency chain before proposing a removal.

**Verification commands (require root):**

```bash
systemctl status fail2ban
```
```bash
fail2ban-client status sshd
```
```bash
ipset list -n
```

**Expected:** service `active (running)` and `enabled` · jail `sshd` present with a nonzero banned count · `f2b-sshd` present in the ipset list.

**An `active (running)` service alone does not prove enforcement.** The ipset check is the one that confirms bans reach the firewall.

### 3.6 System timezone

🚨 **The VPS system clock runs Europe/Berlin (CEST).** The owner is in US Central (CDT) — a **seven-hour offset**. Every timestamp produced on this box (systemd journals, `last`/`lastb`, file mtimes, cron scheduling, `systemctl status` output) is Berlin time unless explicitly converted.

Practical consequences:
- Cron entries are scheduled in Berlin time. A job intended for 8 PM Central must be written as 3 AM Berlin the following day.
- A session log dated by local Central time may not match file timestamps created during that session, off by up to a day. This is expected, not a discrepancy to investigate.
- Contabo panel timestamps (e.g. Auto Backup listings) are displayed in **CDT**, not Berlin — so panel times and shell times for the same event will differ. Do not treat that as an inconsistency.

Verify with `timedatectl`.

---

## 4. CLAUDE CODE (VPS)

### 4.1 What it is and how it runs here

Anthropic's agentic coding/administration tool. Interactive terminal application: the operator types natural-language requests; Claude Code proposes file edits and shell commands and executes them **on the host** after approval. It is not a daemon, has no network listener, no bot integration, and no OpenAI-compatible API — sessions exist only while an operator is attached over SSH.

- **Install:** official native installer (`curl -fsSL https://claude.ai/install.sh | bash`) run as `minncore`
- **Location:** `/home/minncore/.local/bin/claude` (launcher symlink into `~/.local/share/claude/versions/`)
- **Version:** 2.1.216 at install (2026-07-21); 2.1.220 observed 2026-07-28. Native installs self-update in the background — the running version will drift upward without operator action.
- **Diagnostics:** `claude --version` · `claude doctor` (read-only health/config check)
- **Launch:** as minncore, from the directory to work in: `claude`

### 4.2 Authentication

Logged in via **Claude Pro subscription OAuth** (2026-07-21). Over SSH the login flow prints a URL to open in any browser, returning a code to paste back. No credential file is managed by the operator; Claude Code holds its own OAuth state under `~/.claude/`.

Switching to API-key billing later is supported (set `ANTHROPIC_API_KEY`; Claude Code prompts once to approve it) but bills through an Anthropic Console account — separate from the Pro subscription.

**Model selection is configurable per-session and persistently.** `/config` writes to `.claude/settings.json`. Contents as of 2026-07-30: the permissions block (§4.3) alongside `"model": "sonnet"` and `"effortLevel": "medium"`. Model choice is an operational preference, not a security control — but `settings.json` holds more than the permission block, so any drift check must account for legitimate model writes.

### 4.3 Security model — deliberate, with eyes open

This deployment replaces Hermes's container-isolation model with a different, consciously-chosen one: **host execution as a non-root user, gated by per-action human approval.**

- Runs as `minncore`, not root. Blast radius of any approved mistake is one user account plus whatever sudo is explicitly granted (§4.5).
- ⚠️ **Known soft spot:** minncore is in the `docker` group, which is root-equivalent in capability (a docker-group member can mount the host filesystem into a container). This is accepted for operational practicality — it is accident protection, not a hard security boundary. Anyone hardening further would remove docker group and run docker commands as root manually.
- **Permission mode: manual/default (ask-first).** Reads execute freely; every write, edit, and shell mutation stops for an explicit approval prompt. The Allow/Ask/Deny rule lists are deliberately empty — nothing is pre-approved.
- ✅ **Mode persistence verified 2026-07-30**, and re-verified after `/config` model writes: `{"permissions":{"defaultMode":"default"}}` survives alongside the model keys.
- **Approval discipline: choose "Yes" (once), never "Yes, and don't ask again."** Every approval prompt offers a broadening option that pre-approves a pattern for the rest of the session. Taking it converts an explicit gate into a judgment-based one. The narrow choice costs a keystroke; the broad one costs the boundary. The gate has demonstrably blocked a write when declined (2026-07-30).
- The earlier "auto" mode (a classifier model auto-approves actions it judges safe) was evaluated and **rejected** for this box: judgment-based gating is a softer boundary than explicit approval on a sudo-capable host.
- 🚨 **`--dangerously-skip-permissions` is banned on this machine.** It removes all gating. There is no legitimate use for it here.

### 4.4 Operating conventions

- One session = one attached terminal. Nothing runs when no one is connected (contrast: Hermes ran 24/7).
- Scheduled/automated work belongs in **host cron as plain scripts**, not in agent sessions. Cron is scheduled in Berlin time (§3.6).
- Claude Code reads files in the trusted workspace freely. Secrets on this box live outside `/home/minncore` where practical (see §9); do not copy them into the workspace.
- **Trust is granted per working directory.** `/home/minncore` and `~/docs` are trusted (2026-07-21/25). Launching from an untrusted directory prompts for approval; that prompt is the boundary working, not an obstacle.
- **Large-file transfer: use Claude Code's Write tool, never a chat-pasted heredoc.** Chat UI code blocks can truncate mid-scroll without warning; a broken `<< 'EOF'` delimiter causes bash to execute document contents as commands. This happened 2026-07-25 (§13), and truncation recurred 2026-07-30.
- **An agent's completion summary is not proof.** Verify written artifacts independently (`wc -l`, `head`, `tail`, `grep`) rather than trusting a reported line count. This rule has repeatedly caught real problems and has also produced false alarms in the other direction — verification resolves both.
- 🚨 **Confirm which machine you are on before writing files.** Two Claude Code installs exist (§4.7) with separate filesystems and separate `.claude/skills/` directories. A file intended for one will silently land on the other if the wrong session is used. This has happened once (2026-08-04, §13).
- 🚨 **A versioned baseline is a snapshot, not a live tracker.** Agents will treat §12 as a task list and attempt to close items in place upon completing them — this happened twice on 2026-08-04 and produced two divergent documents both labeled v16 (§13). The rule: an item drafted as open in vN **stays open in vN**, even if it becomes true partway through the same session. It closes in vN+1, produced as a full replacement document. If mid-session work makes something in the current version stale, say so and hold it for the next version; never go back and alter the version already produced. The same applies to session logs after `EXIT` — continuing work is an addendum, never a rewrite.

### 4.5 The sudo boundary and the scoped NOPASSWD carve-out

🚨 **Claude Code cannot answer interactive password prompts. This is structural, not a misconfiguration.**

Root-caused 2026-07-30 by two independent commands run inside a live session:

- `tty` → `not a tty` (exit 1). Claude Code's Bash tool executes with **no controlling terminal**; stdin is not attached to a terminal device. Sudo therefore has nowhere to prompt.
- `sudo -n whoami` → `sudo: a password is required` (exit 1). No cached credentials, no blanket NOPASSWD.

**Sudo itself is healthy for `minncore`.** Verified the same day from an interactive shell: `su - minncore` → `sudo whoami` prompted normally and returned `root`. The limitation belongs to Claude Code's execution environment, not to the account's sudoers configuration.

**Consequence:** any command requiring root must either be on the scoped allowlist below, or be run directly by the operator. A non-allowlisted `sudo` command inside a Claude Code session fails fast rather than hanging — an acceptable failure mode.

**The carve-out:** `/etc/sudoers.d/minncore-readonly`, mode **0440** (required — sudo silently ignores group- or world-writable sudoers files).

```
minncore ALL=(root) NOPASSWD: /usr/sbin/ss -tlnp
minncore ALL=(root) NOPASSWD: /usr/bin/firewall-cmd --list-all
minncore ALL=(root) NOPASSWD: /usr/bin/lastb
```

**Design rationale — read this before adding anything to it:**

- **Three commands, all read-only, all non-mutating.** Nothing that changes state is on this list.
- **Arguments are locked, not just binaries.** `/usr/bin/lastb` with no trailing arguments permits `sudo lastb` and nothing else — not `sudo lastb -f /other/file`.
- **Full paths are mandatory.** A bare command name would resolve through the caller's `$PATH`.
- **`docker ps` is deliberately absent** — minncore's `docker` group membership already covers it.
- **`ss -tlnp` is on the list despite running without root.** Without privilege it succeeds but silently returns an empty Process column, which defeats its purpose in an audit. Silent degradation, not failure.
- **`fail2ban-client` and `ipset` are deliberately NOT on the list.** A decision was taken 2026-08-02 to leave them off even though it renders the `fail2ban-check` Skill (§4.6) incomplete on its own. Convenience for a Skill is not sufficient justification to broaden a security carve-out; those two checks remain operator-run as root.

**Verified working from Claude Code's non-tty shell 2026-07-30:** all three ran via `sudo -n` with no prompt and full output, including a populated Process column for `ss`.

**Do not broaden this file casually.** A narrow diagnostic allowlist becoming a blanket grant by accretion is the predictable failure mode.

### 4.6 On-disk instruction files, reference copies, and Skills

**Instruction and reference files under `/home/minncore`:**

- **`/home/minncore/CLAUDE.md`** (**43 lines** as of 2026-08-04) — auto-loaded into every session started from `/home/minncore` or below. Scoped strictly to infrastructure facts and hard security bans; contains no interaction-style instruction. Includes: a pointer to the baseline copy, execution-context facts for this account (`/root` unreadable, sudo cannot prompt, the three-command carve-out), the §9 security ban list, current secret locations, and — added 2026-08-04 — four v16 facts: the Fedora Claude Code install exists with a separate skills directory (confirm host before writing files); the VNC six-hour idle constraint; `fail2ban-client`/`ipset` deliberately off the sudoers allowlist (decision, not oversight); and never remove `fail2ban-sendmail` (placed as a hard ban).

- **`/home/minncore/docs/MASTER_BASELINE.md`** — a verbatim on-disk copy of the baseline, so Claude Code can read infrastructure facts directly. ✅ **Holds v16 (824 lines) as of 2026-08-04.** ⚠️ Now one version stale against this document; updating it to v17 is the standing first open item (§12).

✅ **`CLAUDE.md` demonstrably affects live behavior — two independent confirmations.**
1. 2026-07-30: during the sudo investigation, Claude Code volunteered that the tty result was *"the same reason sudo can't prompt for a password here, as noted in CLAUDE.md"* — citing the file unprompted in ordinary work.
2. 2026-08-04: asked to write a `fedora-health-audit` Skill while connected to the VPS, Claude Code wrote the file but immediately flagged that `CLAUDE.md` describes the machine as `contabo-vps`, not a Fedora workstation, and asked whether a different host was intended. It was — the Skill was written to the wrong machine and had to be removed. **The file caught an operator error the operator had not caught.**

**Known limit:** `CLAUDE.md` is context, not enforcement. Claude Code reads it and generally follows it, but nothing mechanically compels compliance. Anything that must be unconditional would need a hook (a shell command bound to a lifecycle event), which is not currently configured.

**Known behavior:** Claude Code treats §12 of the on-disk baseline as a live task tracker and will move to close items on completing them. That is correct for a tracker and wrong for a versioned snapshot (§4.4). It attempted this twice on 2026-08-04. A line to this effect belongs in `CLAUDE.md` if the pattern recurs.

**Skills — repeatable bounded procedures.**

Skills live at `~/.claude/skills/<name>/SKILL.md`, **per machine** — the VPS and Fedora installs have entirely separate skills directories, and a Skill wanted on both must be written twice.

**VPS Skills (2 as of 2026-08-04):**

| Skill | Purpose |
|---|---|
| `vps-health-audit` | The §10 read-only health audit, built and proven Session #8 |
| `fail2ban-check` | The three §3.5 verification commands, built and proven 2026-08-02 |

**Fedora Skills (4, all registered and enabled — verified via the Skills picker 2026-08-05):**

| Skill | Purpose |
|---|---|
| `fedora-health-audit` | Workstation-side read-only health audit |
| `fedora-update-summary` | Summarize pending/applied system updates |
| `running-snapshot` | Capture current running-state snapshot |
| `settings-drift-check` | Detect unexpected changes in Claude Code `settings.json` (e.g. silent `env` overrides, §4.7) |

🚨 **Skills register at session start, not live.** A Skill written during a session will not be invocable until a fresh session is started. Confirmed twice.

**What Skills are for, and the boundary they respect.** A Skill is operator-invoked, scoped and named in advance, and runs its bounded steps without re-prompting at each sub-step — then stops. Nothing it does exceeds what the operator approved by invoking it. This is deliberately distinct from a standing grant or a judgment-based auto-approval mode, both of which remain rejected (§4.3, §9 rule 6).

The `fail2ban-check` run of 2026-08-02 demonstrated the pattern working correctly, including at its limits: step 1 passed, steps 2 and 3 hit the sudo boundary and were reported as documented limitations rather than treated as failures or worked around, and the Skill stopped without chaining into anything else.

### 4.7 Claude Code on the Fedora workstation

A **second, entirely independent** Claude Code install runs on the Fedora 44 workstation. It shares no state, no configuration, and no Skills with the VPS instance.

- **Version:** 2.1.221 (observed 2026-08-02)
- **Config:** `/home/Minn-Corp/.claude/settings.json`
- **Skills directory:** `/home/Minn-Corp/.claude/skills/` — **four Skills, all registered** (§4.6 table; verified 2026-08-05)
- **Auth:** ✅ **Claude Pro subscription OAuth**, verified by session header reading `Sonnet 5 with medium effort · Claude Pro` (2026-08-02)

🚨 **Formerly routed through a third-party API proxy — now removed.** Until 2026-08-02, `settings.json` contained a top-level `env` block hardcoding `ANTHROPIC_BASE_URL` to `https://thirdparty-aggregator.example` and an `ANTHROPIC_AUTH_TOKEN` for that service (§5). Because it was a config-level override rather than a launch flag, **every Fedora Claude Code session — including all file reads and project context — routed through third-party infrastructure.**

Removal was performed as a structural JSON edit (Python's `json` module popping the `env` key), not a regex or `sed` substitution, to avoid producing invalid JSON. Backup retained at `~/.claude/settings.json.bak.20260802`.

Two observations from the removal, both worth knowing:
- The subsequent launch reported **`Not logged in`**, proving the proxy had been this install's *only* authentication path — there was no OAuth session underneath it. `/login` was completed normally.
- The pre-existing `"model": "sonnet[1m]"` key **disappeared** during login. The `[1m]` variant appears to have been proxy-specific and did not survive the transition to Pro OAuth. This is not an error; the model key is simply absent and Claude Code defaults.

Current contents after cleanup: `effortLevel` and `theme` only.

📝 **Config-level `env` overrides are silent and total.** Nothing in normal operation announces that a base URL has been redirected. The `settings-drift-check` Skill (§4.6) now exists on this machine as the detection mechanism; no equivalent exists yet on the VPS and no run cadence has been established (§12).

---

## 5. OPEN WEBUI

- **Image:** `ghcr.io/open-webui/open-webui:main` · **Publish:** `127.0.0.1:3000:8080` — never 0.0.0.0 (a prior 0.0.0.0 bind exposed all configured API keys)
- **Volume:** `ai-stack_open-webui-data` · **Network:** `bridge` only (172.17.0.x)
- **Health:** `curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000` → `200`, verified 2026-08-04

**Configured API connections** (panel-observed 2026-07-26):

| Endpoint | Purpose | Status |
|---|---|---|
| `https://generativelanguage.googleapis.com/v1beta/openai` | Gemini | Inherited, unverified |
| `https://api.venice.ai/api/v1` | Venice | Inherited, unverified |
| `https://api.anthropic.com/v1` | Anthropic | Inherited, unverified |
| `https://inference-api.nousresearch.com/v1` | Nous direct | Inherited, unverified; account now free-tier |
| `https://thirdparty-aggregator.example` | Third-party aggregator — **identified, see below** | Functional; usage-restricted by policy |
| `http://172.17.0.1:11435` | Ollama | Returned 200 (2026-07-08 era); listener confirmed present 2026-07-30 |
| `http://172.17.0.1:8001/v1` | llama.cpp via nginx | ⚠️ Previously 🔴 502. The nginx listener on 8001 **is** present (2026-07-30) — the failure is behind nginx, not a missing listener. Uninvestigated (§12). |

### `thirdparty-aggregator.example` — identified, with a usage policy

A **third-party multi-provider API aggregator, purchased through a discount reseller marketplace** (name withheld in this public version). It was added deliberately by the owner; it is not an inherited Hermes-era artifact.

**Observed state 2026-08-02** (vendor dashboard): 84 requests · 6.9M tokens · $5.03 consumed · **$103.83 balance remaining** · average response time ~28.95s. Platforms offered include Claude, OpenAI, Gemini, grok, and Antigravity; only Claude shows traffic.

🚨 **Standing usage policy — treat as untrusted transit.**

The service functions, and nothing here asserts that it is fraudulent. But its provenance is opaque by nature: discounted API resale can rest on legitimate volume arrangements or on access obtained through compromised accounts or ToS-violating sharing, and from the buyer's side those are indistinguishable until the key stops working. More concretely, a proxy sees everything that passes through it in plaintext.

- ✅ **Acceptable:** throwaway experimentation, bulk or low-stakes generation, prompt testing, model comparison — anything the owner would not mind a stranger reading.
- 🚨 **Not acceptable:** infrastructure documentation, this baseline, session logs, credentials or paths, personal content, or any Claude Code project context. The removal from Fedora Claude Code (§4.7) was done for exactly this reason.
- ⚠️ **Continuity is not guaranteed.** These arrangements tend to end abruptly rather than gracefully. Nothing load-bearing should depend on it.
- ⚠️ **The ~29s average response time** is notably slower than direct API access. Not suitable for latency-sensitive use.

**Minor:** dead `watchtower` label on the container — strip on next recreate (deliberately deferred; recreating a container solely for a cosmetic label is not worth the churn). The `zen_perlman` stopped container has been **confirmed absent** (2026-08-04); the 2026-07-21 removal did land.

---

## 6. FEDORA WORKSTATION

Fedora 44 (Wayland/KDE Plasma) · hostname **`minn-corp-workstation`** · user `Minn-Corp` · Ryzen 9 5900X · RX 6700 XT 12 GB · 48 GB RAM · shell zsh. **Hardware ceiling: single GPU, 12 GB VRAM.**

Public IPv4 as of 2026-07-30: `203.0.113.55` (residential, not contractually static — see §3.5 `ignoreip` caveat).

Also runs its own Claude Code install (§4.7), a manually-invoked NordVPN client (§6.6), and Open Notebook under Podman (§6.7).

### 6.1 SSH configuration

`~/.ssh/config` contains exactly two blocks. The `contabo` block was hardened 2026-08-08:

```
Host contabo
    HostName 198.51.100.20
    User root
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

Host asus-router
    GSSAPIAuthentication no
    HostName 192.168.50.1
    User router-admin
    Port 22
    IdentityFile ~/.ssh/id_ed25519_asus
    IdentitiesOnly yes
```

**`IdentitiesOnly yes` on the `contabo` block (added 2026-08-08):** prevents the SSH client from offering agent-held keys ahead of the pinned IdentityFile. Without it, each rejected agent-key offer logs a `Failed publickey` on the VPS and counts toward fail2ban's `maxretry=5`/`findtime=10m` — a quiet self-ban vector. Verified with `ssh -G contabo`: `identitiesonly yes`, single identityfile (`~/.ssh/id_ed25519`). Matches the convention already used on the `asus-router` block.

`User root` on the `contabo` block is why `ssh contabo` lands as root. Backup of the pre-cleanup config: `~/.ssh/config.bak.20260721`.

To reach the operator account instead, connect explicitly: `ssh minncore@198.51.100.20` (verified 2026-08-04).

### 6.2 Shell config

Cleaned 2026-07-21 (backup: `~/.zshrc.bak.20260721`). Retained: `export PATH="$HOME/.local/bin:$PATH"` and `alias contabo`.

The `alias vps` Cloudflare browser-SSH shortcut was removed 2026-08-04 (formerly line 263) — it was never used; `ssh contabo` is the actual workflow.

### 6.3 The Fedora-side cloudflared tunnel

A second, independent cloudflared daemon runs on Fedora, publishing `router.minn-corp.example` → the Asus router at `https://192.168.50.1:8443` behind the "Router Admin Access" policy (owner's admin email + FIDO2, verified 2026-07-13). **Access apps are reachable from any device** (including the Pixel 9) as long as this daemon runs.

**Honest framing:** the router admin page is no longer publicly scannable, but attack surface shifted rather than vanished — Cloudflare identity/session management and the Fedora cloudflared daemon are the new perimeter.

**VPN interaction:** during a NordVPN session (§6.6), this daemon's path to `192.168.50.1` is preserved by the `192.168.50.0/24` allowlist entry. Without that entry, `router.minn-corp.example` would go dark for the duration of the VPN session, presenting as a Cloudflare origin error rather than an obvious VPN side effect.

### 6.4 Desktop configuration

✅ **Default browser fixed 2026-08-04.** Previously pointed at a nonexistent `/usr/bin/google-chrome-beta`, breaking URL auto-open. The prior description was stale in two ways: the path was broken *and* the browser was wrong — the installed browser is Chrome **Stable** at `/usr/bin/google-chrome`, not Beta.

Set with `xdg-settings set default-web-browser google-chrome.desktop`; verified by `xdg-settings get default-web-browser` returning `google-chrome.desktop`.

📝 Two Chrome desktop entries exist in `/usr/share/applications/`: `google-chrome.desktop` and `com.google.Chrome.desktop`. `flatpak list` confirms **no** Flatpak Chrome is installed, so the reverse-DNS-named entry is not a parallel Flatpak install. Its origin is unexplained. Harmless; noted for accuracy.

📝 **Autostart state (verified 2026-08-08):** `~/.config/autostart/` contains only `remmina-applet.desktop`. No NordVPN autostart entry exists at any level — not in user or system XDG autostart, not as a user systemd unit, not in the (nonexistent) `~/.config/autostart-scripts/`. A NordVPN GUI appearing at login is therefore attributable to **KDE session restore** (the app was open at logout) pending behavioral confirmation; the fix is closing the GUI before shutdown, or setting the session to start empty.

### 6.5 Local LLM inference

Independent of the VPS. Containers managed by Docker.

**Images present (2026-07-30):** `containrrr/watchtower:latest` · `ghcr.io/ggerganov/llama.cpp:server-rocm` (20.7 GB) · `local/llama:server-vulkan` · `ollama/ollama:rocm`.

**Model weights:** `/home/Minn-Corp/llama-models` — GGUF files, host directory, bind-mounted into the container. **Not stored inside any container or image.** Container and image deletion cannot touch them.

**`swap-llama` — model swap script** (on `$PATH`; locate with `which swap-llama`). Stops and removes the running `llama-server` container, then launches a new one against the requested GGUF. Usage: `swap-llama <model-filename.gguf>`; bare invocation lists available models.

Key parameters: container `llama-server` · image `local/llama:server-vulkan` · entrypoint `/app/llama-server` · `--device=/dev/dri` · mount `${MODELS_DIR}:/models:z` · server args `--host 0.0.0.0 --port 8080 --n-gpu-layers 99`.

🚨 **Two `0.0.0.0` values in this script do different jobs. Do not "fix" both.**
- `-p "127.0.0.1:${HOST_PORT}:${CONTAINER_PORT}"` — the **host publish**. Narrowed from an unqualified `-p 8000:8080` to loopback-only on 2026-07-30. Correct as-is.
- `--host 0.0.0.0` in `SERVER_ARGS` — llama-server's bind **inside the container's own network namespace**. This must remain `0.0.0.0` or Docker's port mapping cannot reach it and the service breaks entirely.

**Access:** llama.cpp's own built-in web UI at `http://127.0.0.1:8000/` in a browser on the same machine. Not wired into any Open WebUI instance.

📝 Because `swap-llama` removes the container on each swap, `docker ps -a` returns empty between runs. That is the script working, not a missing container.

📝 **This machine's llama.cpp server is unrelated to the VPS's Open WebUI.** They are separate systems that both happen to serve models.

### 6.6 NordVPN

Installed 2026-08-08. **Operating posture: manually invoked, short-duration sessions (~1 hour), never always-on.** This posture is the load-bearing fact — it is why the surrounding infrastructure needed minimal accommodation. If the posture ever changes to default-on, re-read the VPN caveats in §3.4 (VNC recovery path untested from an exit IP) and §3.5 (ignoreip coupling) before flipping the switch.

**Packages:** `nordvpn 5.3.0-1` + `nordvpn-gui 5.3.0-1` (x86_64). User `Minn-Corp` added to group `nordvpn` by package %post. Updates arrive via normal `dnf upgrade` through `/etc/yum.repos.d/nordvpn.repo` (2 repos, x86_64 + noarch).

**Install workaround — verification never weakened.** The official `nordvpn-release` wrapper RPM fails on rpm 4.20+/DNF5 with `does not verify: no digest` — an upstream packaging bug (the wrapper ships without file digests). Rather than using `--nodigest`/`--nopgpcheck`, the repo file and GPG key were **extracted from the wrapper via `rpm2cpio`**, installed to their real paths (`/etc/yum.repos.d/nordvpn.repo`, `/etc/pki/rpm-gpg/RPM-GPG-KEY-NordVPN`), the key imported normally, and `gpgcheck=1` left in force. The key was verified identical across three independent sources (the wrapper RPM, `repo.nordvpn.com/gpg/nordvpn_public.asc`, and `repodata/repomd.xml.key`): fingerprint `BC5480EFEC5C081CE5BCFBE26B219E535C964CA1`, `NordVPN <admin@nordvpn.com>`. Every package installed after that point was verified against a triply-cross-checked key.

**Settings (verified 2026-08-08 via `nordvpn settings`):**

- Technology NORDLYNX · Firewall enabled (mark 0xe1f1) · Routing enabled
- **Auto-connect: disabled** — booting the machine never establishes a tunnel
- **Kill Switch: disabled — deliberate.** With the kill switch on, a VPN drop would blackhole all traffic including SSH to the VPS, leaving the allowlist as the only path through. Off is the safer default for the primary administrative client. Do not "harden" this without re-reading the tradeoff.
- **DNS: disabled** — NordVPN does not override resolvers. Relevant because `minn-corp.example` resolution runs through Cloudflare; a forced VPN DNS would be a plausible future cause of "Access apps stopped working."
- LAN Discovery: disabled (superseded by the narrow subnet allowlist below)
- **Allowlisted subnets:**
  - `198.51.100.20/32` — keeps SSH-to-VPS out of the tunnel, on the residential source IP. **This entry is what preserves the fail2ban `ignoreip` exemption during VPN sessions (§3.5). Do not remove it.** Without it, VPS SSH would arrive from shared, recycled NordVPN exit IPs — bannable, and plausibly already present in `f2b-sshd` given the credential-stuffing volume this box attracts.
  - `192.168.50.0/24` — keeps the home LAN reachable: `ssh asus-router` and the Fedora cloudflared origin path to `192.168.50.1:8443` (§6.3). Chosen over `lan-discovery enable` deliberately: the narrow subnet matches the posture of every other control on this stack, where enabling LAN discovery would open all RFC1918 ranges to satisfy a need in one.

**systemd units (verified 2026-08-08):**

| Unit | State |
|---|---|
| `nordvpn.service` | masked, disabled |
| `nordvpnd.service` | **enabled** — the daemon; holds no tunnel at boot |
| `nordvpnd.socket` | **enabled** — socket activation |
| `nordvpnd-killswitch.service` | static (inert; kill switch off) |

🚨 **Socket-activation trap:** disabling `nordvpnd.service` alone is meaningless — the enabled socket restarts the daemon on first CLI contact, making the disable appear to have silently failed. Fully stopping the daemon at boot requires disabling **both** the service and the socket. Current decision: leave both enabled. The idle daemon holds no tunnel and touches no routes; `Auto-connect: disabled` is the control that matters.

**No autostart entries exist for the GUI** (§6.4). A tray icon at login is session-restore behavior, not a launch mechanism.

### 6.7 Open Notebook

Open-source AI research/notebook application, installed 2026-08-08. **Entirely workstation-local; nothing on the VPS was changed and nothing is exposed to the internet.**

- **Runtime:** Podman (`podman compose`), project directory `~/open-notebook`
- **Access:** `http://127.0.0.1:8502` — loopback-only, reachable from this machine only
- **Boot behavior:** does **not** auto-start. If the page won't load after a reboot: `cd ~/open-notebook` then `podman compose up -d`. This is the fix in the large majority of cases.
- **Local-model dependency:** AI features backed by the local model require llama.cpp to be running first (`swap-llama <model-name>`, §6.5). Without it the app itself works, but local-model AI features do not respond.
- **Runbook:** `~/open-notebook/RUNBOOK.md` on this machine — full instructions including troubleshooting.
- **Isolation:** Open WebUI (VPS) and the existing local AI setup are unaffected and unrelated.

⚠️ **Unverified (§12, §14):** whether Open Notebook stores remote provider API keys (if so, the location belongs in the §9 secrets table, and it must not be pointed at `thirdparty-aggregator.example` per §9 rule 11), and whether the Podman deployment is rootless. Rootless Podman would make §9 rule 2's intent structurally true for this workload rather than a rule to remember — directly relevant to the §12 Podman-migration evaluation, which now has a live reference workload on this machine.

---

## 7. NETWORK SUMMARY (VPS)

**Expected listeners** (`ss -tlnp` — verified 2026-07-30):

| Port | Bind | Process |
|---|---|---|
| 22 | `0.0.0.0` + `[::]` | sshd |
| 80 | `0.0.0.0` | nginx |
| 8001 | `172.17.0.1` | nginx (docker gateway) |
| 11435 | `172.17.0.1` | nginx (docker gateway) |
| 3000 | `127.0.0.1` | docker-proxy → open-webui |
| 20241 | `127.0.0.1` | cloudflared metrics |

**Ports 8642, 9119, and 9090 must be ABSENT.** Any of them listening is a finding (8642/9119 = Hermes remnant; 9090 = Cockpit re-enabled). Confirmed absent 2026-07-30.

**No new listener from fail2ban or exim.** fail2ban has no network service; exim is disabled. If port 25 ever appears, exim has been enabled — investigate.

**Docker networks:** `bridge` (172.17.0.0/16) plus default `host`/`none`.

---

## 8. CLOUDFLARE ZERO TRUST

Domain `minn-corp.example`. VPS IP never in DNS; all traffic via tunnels. Tunnel `contabo-vps` is systemd-managed on the VPS.

**cloudflared (VPS):** version `2026.7.3` (built 2026-07-23), service `active (running)` and `enabled`, up since 2026-07-30 23:08:37 CEST. Verified 2026-08-04.

📝 **Unexplained transient, 2026-08-04 02:04–02:05 CEST:** the journal recorded repeated `ERR "Unable to reach the origin service. The service may be down..."` entries over roughly one minute. A six-hour lookback taken at 22:10 CEST that same day returned **zero** occurrences, so the condition was not ongoing. Open WebUI answered HTTP 200 on localhost and nginx had been running continuously since 2026-08-02 with no restart in between. Cause unidentified; apparently self-resolved. Recorded so that a recurrence is recognized as a pattern rather than treated as novel.

**Published routes:**

| Hostname | Target |
|---|---|
| `ssh.minn-corp.example` | `ssh://localhost:22` — Browser SSH |
| `web.minn-corp.example` | `http://localhost:3000` — Open WebUI |
| *catch-all* | `http_status:404` — fail-closed |

**Access applications — 5 total, verified in dashboard 2026-08-04:**

| Application | Destination | Type | Policy |
|---|---|---|---|
| `router` | `router.minn-corp.example` | Self-hosted | Router Admin Access |
| `ssh` | `ssh.minn-corp.example` | Browser SSH | Admin Access |
| `llama-backend` | `llama-backend.minn-corp.example` | Self-hosted | Open WebUI Service Auth |
| `ai-backend` | `ai-backend.minn-corp.example` | Self-hosted | Open WebUI Service Auth |
| `web` | `web.minn-corp.example` | Self-hosted | Admin Access |

✅ **No `hermes` Access application exists.** Verified 2026-08-04.

**Access policies — 3 total, verified 2026-08-04:**

| Policy | Action | Rules | Used by |
|---|---|---|---|
| `Router Admin Access` | Allow | 1 | 1 application |
| `Admin Access` | Allow | 1 | 3 applications |
| `Open WebUI Service Auth` | Service Auth | 1 | 2 applications |

**`Admin Access` detail** (policy ID `00000000-0000-0000-0000-000000000000`, created 2026-05-06, last updated 2026-06-30):
- **Rule:** `Include → Emails → admin@minn-corp.example`
- **Session duration:** same as application session timeout
- **Consuming applications (3):** `ssh`, `web`, and **`App Launcher`** (`minn-corp-example.cloudflareaccess.com`)

📝 **The third consumer is the App Launcher**, Cloudflare's Access portal. It is a legitimate application object that counts toward the policy's usage total but does not appear in the main self-hosted Applications list. A prior apparent discrepancy between "used by 3" and two visible applications is explained by this and is **not** an anomaly.

**Not yet inspected:** rule contents of `Router Admin Access` and `Open WebUI Service Auth`.

**ai-backend / llama-backend** are fronted by nginx with Cloudflare Access service-token (JWT) validation, not tunnel ingress rules; the tunnel stays outbound-only. Healthy end-to-end signature (verified 2026-07-14): `curl -sk -o /dev/null -w "%{http_code}\n" --max-time 5 https://ai-backend.minn-corp.example/` returns **403** (Access correctly rejecting an unauthenticated call).

---

## 9. SECURITY POLICY

Standing rules. Not negotiable, no convenience exceptions. (Conversational-context security rules — never pasting secrets into chat, declining autonomous privilege — live in RULES v1.3, which points here for everything system-specific.)

1. **Container ports publish to `127.0.0.1` only.** Never 0.0.0.0. Applies to the Fedora workstation as well as the VPS. Open Notebook's `127.0.0.1:8502` (§6.7) conforms.
2. **Never mount `/var/run/docker.sock`** into any container; never `--network host`; never `--cap-add NET_ADMIN`/`SYS_ADMIN`.
3. **Secrets never appear in documentation, chat, or tickets.** Current secret locations:

   | Secret | Location |
   |---|---|
   | Cloudflare Access service credentials | `/etc/nginx/conf.d/cf-access-proxy.conf` — grep only, never cat |
   | Open WebUI provider API keys | inside the `ai-stack_open-webui-data` volume / admin UI |
   | Telegram bot token (live) | `/home/minncore/scripts/telegram-token.txt` — stored as a `TELEGRAM_BOT_TOKEN=` prefixed line (§11) |
   | Telegram bot token (stale copy) | `/root/salvage/telegram-token.txt` — duplicate, disposition undecided (§12) |
   | Claude Code OAuth state (VPS) | `/home/minncore/.claude/` (managed by the tool) |
   | Claude Code OAuth state (Fedora) | `/home/Minn-Corp/.claude/` (managed by the tool) |
   | NordVPN account session (Fedora) | `~/.config/nordvpn/` (mode 700, managed by the daemon) |
   | Open Notebook provider keys (if any) | 🔴 UNKNOWN — not yet audited (§12) |
   | Contabo panel / VNC console credentials | Contabo panel Secret Management; VNC password also retrieved and stored operator-side (§3.4) |
   | thirdparty-aggregator.example API token | vendor dashboard; removed from Fedora `settings.json` 2026-08-02 (backup file `~/.claude/settings.json.bak.20260802` still contains it) |

4. **Any credential exposed anywhere it shouldn't be is burned** — rotate immediately. (Happened twice in the Hermes era.)
5. **Never `setenforce 0`** — fix SELinux labels/booleans correctly.
6. **Claude Code: never `--dangerously-skip-permissions`.** Manual ask-first mode is the standing posture on the VPS; the Allow list stays empty. At approval prompts choose "Yes," never "Yes, and don't ask again."
7. **`docker` group membership = root-equivalent.** Grant to no additional users. minncore's membership is a documented, accepted exception.
8. **Credential transfer server-to-server = `scp` only**, never chat.
9. **The sudoers carve-out at `/etc/sudoers.d/minncore-readonly` is read-only by design** (§4.5). Additions require deliberate review, full binary paths, and locked arguments. Nothing state-changing. Convenience for tooling is not a sufficient reason to broaden it.
10. **`sudoers.d` files must be mode 0440** and edited via `visudo -f`, never a plain editor. Validate the whole set with `visudo -c` after any change.
11. **Third-party API proxies are untrusted transit** (§5). No infrastructure documentation, credentials, personal content, or agent project context routes through them. Verify agent configuration files for `env` overrides periodically — a base-URL redirect is silent and total. The `settings-drift-check` Skill (Fedora, §4.6) is the current detection mechanism; extend to the VPS.
12. **The NordVPN allowlist entries (§6.6) are security config, not convenience config.** The `198.51.100.20/32` entry preserves the fail2ban `ignoreip` exemption during VPN sessions; removing it exposes VPS SSH to shared-exit-IP banning. Do not remove either entry without reading §3.5 and §6.6.

---

## 10. OPERATIONAL PROCEDURES

### Standard read-only health audit (VPS shell)

```bash
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
```
```bash
df -h /
```
```bash
free -h
```
```bash
uptime
```
```bash
ss -tlnp
```
```bash
firewall-cmd --list-all
```
```bash
fail2ban-client status sshd
```
```bash
wc -l /root/.ssh/authorized_keys /home/minncore/.ssh/authorized_keys
```
```bash
last reboot | head -5
```

**Expected:** only open-webui Up, loopback-bound · disk under 75% · firewalld ssh+dhcpv6-client only · fail2ban sshd jail active · both authorized_keys files = 2 lines · no listeners on 8642/9119/9090.

**As `minncore`, three of these need `sudo`** and are covered by the carve-out (§4.5): `sudo ss -tlnp`, `sudo firewall-cmd --list-all`, `sudo lastb`. `fail2ban-client` and `ipset` are **not** on the allowlist — run them as root.

Available as the `vps-health-audit` Skill (§4.6).

### Claude Code health

```bash
claude doctor
```
Run as the relevant operator account, on the relevant machine. Read-only diagnostics: install health, settings validation, update status.

### fail2ban enforcement check

```bash
ipset list -n
```
Expect `f2b-sshd`. This is the check that proves bans reach the firewall; service status alone does not. Partially available as the `fail2ban-check` Skill (§4.6) — the Skill covers service status only and correctly reports the two root-required checks as blocked.

### External reachability (from Fedora, never from the VPS — and not from inside a VPN session)

```bash
nc -zv 192.0.2.10 63138
```
```bash
curl -sk -o /dev/null -w "%{http_code}\n" --max-time 5 https://ai-backend.minn-corp.example/
```
Expected: Connected · 403. See §3.4 on the VNC idle constraint before interpreting a refusal as a fault. If NordVPN is connected (§6.6), disconnect first — these checks have only ever been validated from the residential IP, and a VPN-path failure would be uninterpretable.

### Open WebUI reachability (VPS shell)

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000
```
Expected: `200`. Useful for isolating whether a Cloudflare origin error is the application or the path to it.

### Firmware monitor manual test

Forces a Telegram delivery without waiting for an upstream release. The state file self-heals to the real tag as part of the run, so no cleanup is required.

```bash
echo "TEST-FORCE-0" > /home/minncore/scripts/state/asuswrt-merlin-last-tag.txt
```
```bash
/home/minncore/scripts/check-asuswrt-release.sh
```
Expected: the release message printed to the terminal **and** delivered to the Minn-Core Telegram bot within seconds. Terminal output without a Telegram message means the delivery path is broken (§11).

### Failed-login review

```bash
lastb | head -20
```
Scanner noise is normal and heavy (§3.2). With fail2ban running, investigate *successful* logins from unrecognized sources rather than failure volume.

### Open Notebook restart after reboot (Fedora)

```bash
cd ~/open-notebook && podman compose up -d
```
Then browse `http://127.0.0.1:8502`. For local-model AI features, start llama.cpp first: `swap-llama <model-filename.gguf>`. Full troubleshooting: `~/open-notebook/RUNBOOK.md`.

---

## 11. SCHEDULED JOBS, SCRIPTS, AND SALVAGE

### The ASUS firmware monitor

**Script:** `/home/minncore/scripts/check-asuswrt-release.sh`
**Cron (minncore's crontab):** `0 3 * * * /home/minncore/scripts/check-asuswrt-release.sh` — 3 AM **Berlin** time, which is 8 PM US Central the prior day (§3.6).
**State file:** `/home/minncore/scripts/state/asuswrt-merlin-last-tag.txt`
**Token:** `/home/minncore/scripts/telegram-token.txt`

Queries the GitHub REST API for the latest `gnuton/asuswrt-merlin.ng` release tag (custom firmware for the home ASUS RT-AX92U mesh), compares against the stored last-seen tag, and alerts via the Telegram Bot API only on change. Uses `BASH_SOURCE` to resolve its own directory, so it functions correctly under cron's working directory.

🚨 **This script silently failed to deliver for weeks. Root-caused and fixed 2026-08-02.**

The token file stores the credential as a `TELEGRAM_BOT_TOKEN=…` prefixed line, but the script originally read it with a bare `cat`, injecting the prefix into the API URL and producing a malformed request on **every scheduled run since the Session #6 re-home.** The failure was structurally invisible: the Telegram `curl` sent output to `/dev/null` with no status check, empty API responses took an `exit 0` path, and nothing was logged anywhere. The fetch, parse, and state-update halves of the script worked correctly throughout — only delivery was broken.

**The fix** (line 27 onward) reads only the first line and strips the prefix if present, then fails loudly rather than silently if the result is empty:

```bash
TELEGRAM_TOKEN=$(head -1 "$TELEGRAM_TOKEN_FILE" | sed 's/^TELEGRAM_BOT_TOKEN=//')
if [ -z "$TELEGRAM_TOKEN" ]; then
  echo "ERROR: telegram token file empty or unreadable" >&2
  exit 1
fi
```

This accepts either a bare token or a `KEY=value` line, so a future paste in either format is safe. Note that the script runs under `set -euo pipefail`: a token file that exists but is unreadable will fail at the `head` call with a specific `head: cannot open …` error before the guard runs. That was judged acceptable — the raw error names the file and the operation, which is more diagnostic than a generic message, and adding `|| true` to route around it would reintroduce failure suppression into a script being hardened against exactly that.

✅ **Delivery verified end-to-end 2026-08-02.** A forced test produced a real message in the Minn-Core Telegram bot at 19:47:52 local, including a GitHub preview card. Independent corroboration: the bot's chat history showed **no traffic at all** between `/start` on 2026-07-22 and that test — confirming the cron had genuinely never delivered.

📝 **The lesson this cost.** Every observable indicator said the system was healthy: the script existed, cron was scheduled, cron ran, the state file updated with real tags. The single thing that mattered had never once worked. Verify the *outcome* of an alerting path, not the pipeline leading to it.

### Salvage directory

`/root/salvage/` (created 2026-07-21) preserves two Hermes-era artifacts, both now superseded by the live copies under `/home/minncore/scripts/`:

- **`check-asuswrt-release.sh`** — the pre-re-home version, which does **not** contain the token-format fix above.
- **`telegram-token.txt`** — a duplicate of the live bot token.

Its disposition remains undecided (§12). Two copies of a secret is one more than necessary, and the salvaged script is now actively wrong.

---

## 12. OPEN ITEMS

| # | Item | Why it matters | Effort |
|---|---|---|---|
| 1 | **Update `~/docs/MASTER_BASELINE.md` to v17** | Claude Code reads that copy as fact. It currently holds v16. Use Claude Code's Write tool, not a heredoc (§4.4), and confirm the session is attached to the **VPS**. | 15 min |
| 2 | **Update `~/CLAUDE.md` for anything v17 changes** | Candidates: the tracker-vs-snapshot rule (§4.4) as an explicit line, and any secret-location changes. Provide an explicit item list — a broad "update for v17" instruction has already once produced zero change while appearing successful (§13). | 10 min |
| 3 | **Audit Open Notebook for stored provider keys** | If it holds remote API keys, the location belongs in §9's table — and it must not be pointed at `thirdparty-aggregator.example` (§9 rule 11). Also determine whether the Podman deployment is rootless. | 15 min |
| 4 | **Portfolio sanitization pass** | Produce a public-safe version of RULES/BASELINE/session logs for GitHub: strip IPs, the panel email, the Telegram chat ID, hostnames as needed. Distinct task from any baseline version; sanitize *from* v17. Supports the owner's IT career-transition goal — the documentation discipline itself is the portfolio artifact. | 1–2 h |
| 5 | **Decide the VNC posture** | Accept reboot-on-demand, or establish a keep-warm practice so the console does not wedge before it is needed (§3.4). This is a decision, not a task, and it has now carried across three baseline versions. | 10 min |
| 6 | **`ignoreip` staleness review mechanism** | Both non-loopback entries are dynamic addresses (§3.5). No drift detection exists; the recovery path from a self-ban has its own constraint; and the NordVPN coupling (§3.5, §6.6) adds a second way the exemption can silently stop protecting. Oldest unaddressed item with a real failure mode. | 30 min |
| 7 | **Settings drift check — VPS side + run cadence** | The `settings-drift-check` Skill exists on Fedora only (§4.6). The VPS install has no equivalent, and no schedule or habit exists for running the Fedora one. The incident class it detects (silent `env` override, §4.7) already happened once. | 20 min |
| 8 | **Decide `/root/salvage/` disposition** | Holds a duplicate Telegram token and a copy of the firmware script that lacks the delivery fix. Actively misleading now, not just redundant. | 10 min |
| 9 | **Investigate llama.cpp 8001 → 502** | Listener exists; failure is behind nginx. Carried. Not security-relevant; deprioritized accordingly. | Unknown |
| 10 | **Inspect remaining Access policy rule contents** | `Router Admin Access` and `Open WebUI Service Auth` — `Admin Access` is confirmed (§8), these two are not. | 10 min |
| 11 | **Audit Claude Code auto-memory (`/memory`)** | It saves what it judges worth keeping, with no operator input. Worth knowing what is in there — on both machines. | 15 min |
| 12 | **Identify the source of the 2026-08-04 02:04 cloudflared origin errors** | Self-resolved and not recurring, so low priority — but unexplained. `docker logs open-webui --since …` for that window is the next step if it recurs (§8). | Unknown |
| 13 | **Explore Open WebUI Prompts** | Workspace → Prompts supports saved templates invoked by slash command, with `{{variable}}` substitution and version history. Directly useful for repeated operational framings. | 30 min |
| 14 | **Evaluate Podman migration for Open WebUI** | Native to the RHEL family; no root daemon and no `docker.sock`, which makes §9 rule 2 structurally true instead of a rule to remember. **Status change:** Podman now runs a real workload on the Fedora workstation (Open Notebook, §6.7) — the evaluation has a live reference implementation it previously lacked. | Unknown |
| 15 | **Re-verify Auto Backup restore capability once Contabo clears the US-central warning** | Backups are being created; restores are provider-flagged as possibly unavailable (§2). A backup that cannot restore is not a backup. | 5 min |
| 16 | **VNC idle-stability test** | Distinct from #5: characterize the six-hour boundary more precisely if it ever matters. Low priority. | 30 min waiting |
| 17 | **Strip watchtower label on next open-webui recreate** | Cosmetic. Deliberately deferred — do not recreate the container solely for this. | 5 min |
| 18 | **Explain the second Chrome desktop entry on Fedora** | `com.google.Chrome.desktop` exists with no corresponding Flatpak. Harmless, unexplained (§6.4). | 5 min |
| 19 | **Confirm the NordVPN tray icon is session-restore behavior** | All launch mechanisms ruled out by inspection (§6.4); confirmation is behavioral — close the GUI, reboot, observe. If it returns anyway, resume unit-hunting. | 5 min |

**Closed in v17:** update on-disk baseline to v16 (done 2026-08-04, 824 lines) · update `~/CLAUDE.md` with the four v16 facts (done 2026-08-04, 43 lines) · place `fedora-health-audit` on Fedora · place the three additional Fedora Skills (all four verified registered 2026-08-05).

**Explicitly closed and not to be reopened casually:**
- `fail2ban-sendmail` removal — **withdrawn** (§3.5). Do not attempt without first mapping the dependency chain.
- Broadening `minncore-readonly` to complete the `fail2ban-check` Skill — **declined** by decision 2026-08-02 (§4.5).

---

## 13. INCIDENT HISTORY

Kept because each produced a rule still in force.

| Date | Incident | Resulting rule |
|---|---|---|
| Pre-v8 | Open WebUI bound to 0.0.0.0, exposing all configured API keys | Loopback-only publishes, always |
| Pre-v8 | Hermes API key exposed in chat | Any exposed credential is burned; rotate immediately |
| 2026-07-02 | Nous refresh token revoked after an external process called the refresh endpoint | Single-consumer credentials get exactly one consumer |
| 2026-07-11 | `hermes setup` wizard silently switched providers, breaking Telegram | Verify config after any wizard; prefer scoped subcommands |
| 2026-07-11 | Hermes fabricated "93% disk critical" and a nonexistent audit (actual: 52%) | Agent self-reports are hypotheses; verify with real commands |
| 2026-07-11 | Cockpit found listening on 0.0.0.0:9090, enabled by installer | Audit default-enabled services on minimal installs |
| 2026-07-12 | GitHub PAT pasted into chat as plaintext CLI argument | Unmasked CLI secrets never get echoed; clean history |
| 2026-07-12 | Agent reported private repo "doesn't exist (404)" — it existed | A 404 is not proof of absence |
| 2026-07-14 | Contabo Auto Backup degraded; restores unavailable for days | Verify restore path before destructive work |
| 2026-07-16 | Hermes proposed container→host SSH / docker.sock mount; refused | Isolation boundaries are not negotiable with the thing being isolated |
| 2026-07-19 | Operator asked whether host execution could be approval-gated for Hermes; declined | An approval gate controls timing, not capability |
| 2026-07-21 | zshrc range-edit based on filtered grep output broke an unrelated function; two backup restores required | Before line-range edits: view full unfiltered context, verify between edits |
| 2026-07-21 | Hermes decommissioned; Claude Code deployed under non-root user with ask-first gating | Host-agent trust granted deliberately: non-root, empty allowlists, manual mode, bypass flag banned |
| 2026-07-25 | Heredoc paste of a 400-line document truncated mid-scroll; bash executed document contents as commands | Large files transfer via the agent's Write tool, never a chat-pasted heredoc |
| 2026-07-25 | Agent reported "407 lines" and "+498 lines" for the same write in one output | An agent's completion summary is not proof; verify the artifact independently |
| 2026-07-30 | `docker image prune -a` run on Fedora; removed one unused base image and forced a 20.7 GB re-pull | `prune -a` removes every image not backing a container. Bind-mounted host data is never at risk; images are. |
| 2026-07-30 | A `sed -i` one-liner replaced a line-continuation and dropped its trailing `\`, silently truncating a `docker run` command mid-flags | Verify a scripted in-place edit before executing the edited file, not after it fails |
| 2026-07-30 | `dnf install fail2ban` reported "No match for argument"; the failure was not read, and the next 20 minutes proceeded as though the install had succeeded | Read installer output before moving on. On Rocky, check EPEL before concluding a package does not exist. |
| 2026-07-30 | A chat-pasted SKILL.md truncated mid-word; the agent flagged the suspicious ending rather than accepting it | Verify every transferred document's ending, not just its length |
| 2026-08-02 | The ASUS firmware monitor had never delivered a single Telegram alert since re-home; the token file's `KEY=` prefix was being injected into the API URL, and `-o /dev/null` hid the resulting error | Verify the outcome of an alerting path, not the pipeline. A component reporting success at every observable layer can still be structurally incapable of its one job. |
| 2026-08-04 | `dnf remove fail2ban-sendmail` resolved to 70 packages including the `fail2ban` meta-package itself; aborted before execution | Read the full dnf transaction list every time. A cosmetic cleanup is never worth risking a primary security control. |
| 2026-08-04 | A Fedora-scoped Skill was written to the VPS; the agent caught the host mismatch by citing `CLAUDE.md` | With two agent installs, confirm which machine a session is attached to before writing files |
| 2026-08-04 | While writing MASTER_BASELINE v16 to disk, an agent also edited the just-drafted v16 in place — closing an open item and renumbering §12 — producing two divergent documents both labeled v16. Repaired surgically; the agent then caught a second contradiction it had introduced and asked before acting. It attempted an in-place close twice in one day. | A versioned baseline is a snapshot, not a live tracker. Items drafted open in vN stay open in vN and close in vN+1, always as a full replacement document (§4.4). |
| 2026-08-08 | A two-step command chained with `;` instead of `&&` created a stray malformed `.desktop` file when its first step (`cp` of a nonexistent source) failed silently under `2>/dev/null` — the second step ran regardless | Gate dependent steps on success (`&&`, not `;`), and do not combine silent-failure redirection with unconditional follow-on steps. Same failure family as the 2026-07-30 `sed -i` incident. |

---

## 14. VERIFICATION STATUS

**Verified by live command output or dashboard inspection:**

| Claim | Date |
|---|---|
| §3.1–3.3 hardware, SSH config, firewall layers | 2026-07-11/14 (carried) |
| firewalld zone config (ssh + dhcpv6-client only) | 2026-07-30 |
| §3.2 lastb volume — 335,715 failed attempts since 2026-07-02 | 2026-07-30 |
| §3.2 direct SSH as `minncore` succeeds; no SELinux context problem | 2026-08-04 |
| §3.4 VNC endpoint unchanged; credentials retrieved and stored | 2026-08-02 |
| §3.4 VNC reachable — `nc -zv` → `Connected` | 2026-08-02 |
| §3.4 six-hour idle-to-wedged behavior | Owner first-party observation, 2026-08-02 |
| §3.5 EPEL enabled; fail2ban 1.1.0-6 installed with dependencies | 2026-07-30 |
| §3.5 fail2ban `active (running)`, enabled, `Server ready` after reboot | 2026-07-30 23:08 CEST |
| §3.5 `ipset list -n` → `f2b-sshd` after reboot (enforcement confirmed) | 2026-07-30 |
| §3.5 `fail2ban` meta-package appears as a dependent of `fail2ban-sendmail` | 2026-08-04, dnf transaction list read and aborted |
| §4.3 `settings.json` permission mode persisted across `/config` writes | 2026-07-30 |
| §4.5 `tty` → not a tty; `sudo -n whoami` → password required | 2026-07-30, inside Claude Code |
| §4.5 all three carve-out commands run via `sudo -n` from Claude Code | 2026-07-30 |
| §4.6 `CLAUDE.md` cited unprompted (sudo context) | 2026-07-30 |
| §4.6 `CLAUDE.md` caught a wrong-host file write | 2026-08-04 |
| §4.6 `CLAUDE.md` now 43 lines; four v16 facts verified by content grep, one hit each | 2026-08-04 |
| §4.6 `fail2ban-check` Skill registered and ran end-to-end with correct boundary handling | 2026-08-02 |
| §4.6 VPS skills directory contains exactly `fail2ban-check` and `vps-health-audit` | 2026-08-04 |
| §4.6 all four Fedora Skills present and registered (`✓ on` in the Skills picker) | 2026-08-05 |
| §4.6 on-disk `~/docs/MASTER_BASELINE.md` holds v16, 824 lines, header confirmed; §12 divergence repaired and re-verified by grep | 2026-08-04 |
| §4.7 Fedora `settings.json` `env` block present, then removed; file contents after | 2026-08-02, `cat` before and after |
| §4.7 Fedora Claude Code reported `Not logged in` after removal, then `Claude Pro` after `/login` | 2026-08-02, session header |
| §5 Open WebUI HTTP 200 on localhost:3000 | 2026-08-04 |
| §5 `thirdparty-aggregator.example` identity, balance, usage figures | 2026-08-02, vendor dashboard |
| §5 `zen_perlman` container absent | 2026-08-04 |
| §6.1 `IdentitiesOnly yes` on the `contabo` block; `ssh -G contabo` shows identitiesonly yes, single identityfile | 2026-08-08 |
| §6.2 `alias vps` removed from `.zshrc`, verified absent | 2026-08-04 |
| §6.4 default browser set to `google-chrome.desktop`; no Flatpak Chrome present | 2026-08-04 |
| §6.4 `~/.config/autostart/` contains only `remmina-applet.desktop`; no NordVPN entry in any autostart location; no user systemd nordvpn units | 2026-08-08 |
| §6.5 Fedora image list, model directory, `swap-llama` contents and loopback fix | 2026-07-30 |
| §6.6 NordVPN GPG key identical across three sources, fingerprint recorded; gpgcheck=1 in force | 2026-08-08, install session |
| §6.6 `nordvpn settings` full output: NordLynx, firewall on, auto-connect off, kill switch off, DNS off, LAN discovery off | 2026-08-08 |
| §6.6 allowlist contains `198.51.100.20/32` and `192.168.50.0/24` | 2026-08-08, CLI confirmation of both adds |
| §6.6 unit states: `nordvpn.service` masked · `nordvpnd.service` enabled · `nordvpnd.socket` enabled | 2026-08-08, `systemctl list-unit-files` |
| §7 listener table; 8642/9119/9090 absent | 2026-07-30 |
| §8 cloudflared v2026.7.3, `active (running)`, enabled | 2026-08-04 |
| §8 five Access applications; no hermes application | 2026-08-04, dashboard |
| §8 three policies with rule counts; `Admin Access` rule content and three consumers | 2026-08-04, dashboard |
| §11 cron entry, script contents, `BASH_SOURCE` fix present | 2026-08-02 |
| §11 token file carries `TELEGRAM_BOT_TOKEN=` prefix (format check, secret not revealed) | 2026-08-02 |
| §11 fix applied; exactly one `TELEGRAM_TOKEN=` assignment; `bash -n` clean | 2026-08-02 |
| §11 real Telegram message delivered and received | 2026-08-02 19:47:52 |
| §2 Contabo panel 2FA active | 2026-07-30, panel |
| §2 Auto Backup enabled, 10 backups, daily, most recent 2026-08-04 01:53 CDT | 2026-08-04, panel |
| §2 disk 38% used (37G/99G) | 2026-07-30 |
| nginx `active (running)` since 2026-08-02 | 2026-08-04 |
| Hermes teardown complete | 2026-07-21 |
| Claude Code 2.1.220 (VPS) / 2.1.221 (Fedora) | 2026-07-28 / 2026-08-02 |

**NOT verified — treat as unknown:**

- **Auto Backup restore capability.** Backup creation is confirmed; Contabo's own panel warns restores may be unavailable in US-central. Never tested on this instance.
- Whether removing the `fail2ban` meta-package would break enforcement — deliberately untested, and the reason the removal was abandoned.
- Root cause of the 2026-08-04 02:04 CEST cloudflared origin errors.
- `Router Admin Access` and `Open WebUI Service Auth` policy rule contents (names, counts, and consuming applications confirmed; rule internals not).
- Contabo cloud firewall coverage of the VNC console IP.
- `thirdparty-aggregator.example` — the provenance of its underlying API access, and whether it retains request content.
- Open WebUI's non-Hermes provider connections (Gemini, Venice, Anthropic, Nous) — inherited, never re-tested.
- llama.cpp 8001 → 502 root cause (listener confirmed present; failure is behind nginx).
- Cloudflare tunnel *routes* — presumed live; last end-to-end tested 2026-07-21. The connector is healthy, which is not the same claim.
- Whether `ignoreip` addresses in §3.5 are still current at time of reading.
- Contents of Claude Code's auto-memory on either machine.
- Origin of the `com.google.Chrome.desktop` entry on Fedora.
- **Home-LAN reachability under an active NordVPN tunnel** (§6.6). The `192.168.50.0/24` allowlist entry is confirmed present; the `ip route get` check was run with the tunnel down and proves nothing about connected-state routing. Cheap to confirm during the next VPN session.
- **VNC console reachability from a NordVPN exit IP** (§3.4). Not required under the current manual/short-session posture; becomes required if the posture ever changes to default-on.
- Whether Open Notebook stores provider API keys, and whether its Podman deployment is rootless (§6.7).
- Whether the NordVPN GUI tray icon at login is in fact KDE session restore — all launch mechanisms ruled out by inspection; behavioral confirmation pending (§12 #19).

**A note on this document's reliability.** Multiple claims in earlier versions survived revisions purely by repetition until a command disproved them, and v16 itself was the subject of the divergence incident — briefly existing as two different documents under one version label. This version was produced strictly as a full replacement per the Authoring Contract, with the in-place-editing prohibition now recorded as a standing rule (§4.4) precisely because the failure mode has been observed, not merely theorized. "Not in the baseline" means unverified, not false, in **both** directions. Verify before acting on anything here not confirmed recently.

---

**Companion documents:** `RULES v1.3` (static session preamble; Hermes-era instruction removed, six-context execution labeling, conversational security core) · `MINNCORE SESSION LOG #1` through `#9` + `#9 ADDENDUM A` (append-only history and reasoning trail).
