# Platform notes: Google Chrome on RHEL 8, 9, 10

**What:** facts about how Chrome is installed and configured on each target OS, with the exact commands used.
**Why:** CLAUDE.md "Evidence over assumption": paths, packages and OS behaviour are checked on the real targets before
the role is designed. Differences between OS versions become version gates; no difference found = no gate.

Test VMs (Proxmox, "Server with GUI", MongoDB already installed, snapshot `pre-chrome` taken before this work):
rhel8 = 192.168.20.50, rhel9 = 192.168.20.30, rhel10 = 192.168.20.40.

## D-1. Step 1: read-only baseline (2026-10-07)

Commands (run on each VM, nothing changed):

```bash
cat /etc/redhat-release; uname -r; rpm -q glibc
systemctl get-default
getenforce
for s in $(loginctl list-sessions --no-legend | awk '{print $1}'); do loginctl show-session "$s" -p Name -p Type -p Class; done
rpm -q google-chrome-stable; ls /etc/yum.repos.d/ | grep -i google
ls -laZ /etc/opt/chrome 2>&1
```

| Check | Why we check it | RHEL 8 | RHEL 9 | RHEL 10 |
|-------|-----------------|--------|--------|---------|
| OS release | confirm host ↔ OS mapping (the KVM lab once had them swapped) | 8.10 (Ootpa) | 9.7 (Plow) | 10.2 (Coughlan) |
| Kernel | record of the tested platform | 4.18.0-553 | 5.14.0-611 | 6.12.0-211 |
| glibc | Chrome is a prebuilt binary; it needs a minimum glibc. RHEL 8 has the oldest | 2.28 | 2.34 | 2.39 |
| Boot target | Chrome is a GUI app; `chrome://policy` (CIS's own check) needs a desktop | graphical.target | graphical.target | graphical.target |
| SELinux | the policy file must get a label Chrome can read; Enforcing is the real-world case | Enforcing | Enforcing | Enforcing |
| Login sessions | X11 or Wayland changes how screen capture behaves (rules 4.1.1/4.12) | gdm greeter = wayland; frqadmin = tty (SSH) | same | same, plus systemd `manager` sessions (newer systemd) |
| Chrome installed | is there something to harden, or is install needed | no | no | no |
| Google repo | does the role need to add it for opt-in install | no | no | no |
| `/etc/opt/chrome` | where Chrome reads policies (`/etc/opt/chrome/policies`, see W-1) | exists, only `native-messaging-hosts/`, label `etc_t` | same | same |

Evidence: [01-rhel8](evidence-images/01-rhel8-discovery-readonly-ok.png),
[02-rhel9](evidence-images/02-rhel9-discovery-readonly-ok.png),
[03-rhel10](evidence-images/03-rhel10-discovery-readonly-ok.png).

### What step 1 tells us

1. **Same baseline on all three**: GUI, SELinux Enforcing, no Chrome, no Google repo. No version difference yet.
2. **`/etc/opt/chrome/native-messaging-hosts/` exists before Chrome is installed.** On the workstation (AlmaLinux 10)
   it belongs to `gnome-browser-connector` (`rpm -qf` → `gnome-browser-connector-42.1-9.el10`; files
   `org.gnome.browser_connector.json`, `org.gnome.chrome_gnome_shell.json`): the GNOME Shell extensions integration.
   **Relevant to 2.5.1** (`NativeMessagingBlocklist = ["*"]`): it would block this GNOME integration in Chrome.
   To confirm on the VMs in step 2 (`rpm -qf`).
3. **Desktop session type not known yet:** nobody was logged in to the desktop (only the gdm login screen, which runs on
   Wayland). Re-check after logging in at the Proxmox console (step 2).
4. `/etc/opt/chrome` is labelled `etc_t` (normal for `/etc`); check that a new `policies/managed/` file gets the same.

## W-1. Package facts from the workstation (AlmaLinux 10, google-chrome-stable 154.0.8037.57)

Read-only, product-level facts used to plan steps 2–3. To be confirmed on each VM.

| Fact | Command | Result |
|------|---------|--------|
| Policy base path compiled into Chrome | `grep -a -o -E '/etc/opt/chrome[a-zA-Z/_-]*' /opt/google/chrome/chrome \| sort -u` | `/etc/opt/chrome/policies`, `/etc/opt/chrome/native-messaging-hosts` |
| The RPM does not create `policies/managed/` | `ls -la /etc/opt/chrome` | only `native-messaging-hosts/` |
| The RPM writes the repo file itself | `rpm -q --scripts google-chrome-stable` | creates `/etc/yum.repos.d/google-chrome.repo` (baseurl `https://dl.google.com/linux/chrome/rpm/stable/x86_64`, `gpgcheck=1`, key `https://dl.google.com/linux/linux_signing_key.pub`) and imports key `gpg-pubkey-d38b4796-570c8cd3` |
| A daily cron keeps the repo file | `cat /etc/cron.daily/google-chrome` | re-creates the repo config ("since we cannot do this during the installation…") |
| Install may create an enrollment dir | `rpm -q --scripts` | `mkdir -p /etc/opt/chrome/policies/enrollment` (conditional: **not** created on the workstation nor on the VMs, see D-2) |

**Planning impact:** the role creates `/etc/opt/chrome/policies/managed/` itself; an opt-in install should use the
same repo file name/content as Google's, or the cron job and the role will rewrite each other.

## W-2. Channels and "Chrome Enterprise" (2026-10-07)

**Why:** the task mentions "Chrome Enterprise" and other versions (beta); find out whether they are different software.

| Check | Command / source | Result |
|-------|------------------|--------|
| Packages in Google's RHEL repo | `curl https://dl.google.com/linux/chrome/rpm/stable/x86_64/repodata/repomd.xml` → its `primary.xml.gz`, list `<name>`/`<version>` | `google-chrome-stable` 155.0.8059.39, `google-chrome-beta` 156.0.8078.4, `google-chrome-unstable` (dev) 157.0.8081.0, `google-chrome-canary` 157.0.8089.0. **No separate "enterprise" package** |
| Chrome Enterprise download page | `https://chromeenterprise.google/download/` (page metadata) | "official MSI, PKG, and DMG installers… across Windows, macOS, and Linux fleets… stable and beta channel bundles, including the Chrome ADM/ADMX templates… for centralized GPO and cloud-based policy management". The special installers are Windows (MSI) and macOS (PKG/DMG); Linux uses the same RPM repo |

**What it tells us:** "Google Chrome Enterprise" on RHEL is the same `google-chrome-*` RPM we installed, managed with
enterprise policies (our JSON file) and optionally Google's cloud console. Channels are separate packages that can sit
side by side → the role can select one with a single variable. **To verify** when we get there: that beta/unstable read
the same `/etc/opt/chrome/policies/managed/` folder (install one on a VM and repeat the `chrome://policy` test).

## W-4. Google's documentation for policies on Linux (2026-10-07)

**Why:** the RPM doesn't create `policies/`, so the folder layout must come from Google, not from guessing.

| Source | Quote |
|--------|-------|
| Chrome Enterprise Help, "2. Set policies" (Linux), `https://support.google.com/chrome/a/answer/9027408` | "Managed and recommended policies must have their respective folders in the file system. **Create the following directories if they do not already exist**: `mkdir /etc/opt/chrome/policies`, `mkdir /etc/opt/chrome/policies/managed`, `mkdir /etc/opt/chrome/policies/recommended`" |
| same page | managed = "required and … mandated by an admin. Make sure that these files are not writable and, therefore, cannot be overridden by non-admin users"; recommended = "can be changed by the users" |
| same page | "Be careful not to set the same policy in more than one file. If you do, it's unclear which of the values you specify will be applied." |
| same page | "Chrome browser amalgamates all of the individual files in the policies folders and applies all the settings. Note: If there are multiple files with conflicting values set for a specific policy, the behavior is undefined." |
| same page, *Verify the configuration* | "Users need to restart Chrome browser for policies to take effect … go to chrome://policy. Click Reload policies" |
| Chromium "Linux Quick Start", `https://www.chromium.org/administrators/linux-quick-start/` | "For Google Chrome, these two sets live at … `/etc/opt/chrome/policies/managed/` … `/etc/opt/chrome/policies/recommended/` … Create these directories if they do not already exist" and "(remember that the paths differ for Chromium)" |
| same Chromium page | "You can spread your policies over multiple JSON files. Chrome will read and apply them all. However, you should not be setting the same policy in more than one file. If you do, it is undefined which of the values you specified prevails." |

**What it confirms:** the role must create `policies/managed/` (D2); root-owned, not writable by users (D2: `0755`/`0644`
root); conflicts between files are undefined, which is why the report checks other files (D7); policies apply at
the next Chrome start (D11); `chrome://policy` is Google's own verification (D-3, D-4). Chromium uses a different
path (for later).

## D-2. Step 2: install by hand (2026-10-07)

Commands (each VM; changes the VM, rollback = snapshot `pre-chrome`):

```bash
sudo dnf install -y https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
google-chrome --version
cat /etc/yum.repos.d/google-chrome.repo
rpm -q gpg-pubkey --qf '%{VERSION}-%{RELEASE} %{SUMMARY}\n' | grep -i google
ls -l /etc/cron.daily/google-chrome
sudo ls -laZR /etc/opt/chrome
```

| Check | Why we check it | RHEL 8 | RHEL 9 | RHEL 10 |
|-------|-----------------|--------|--------|---------|
| Install works | current Chrome is a prebuilt binary; RHEL 8 has the oldest glibc (2.28) | Yes | Yes | Yes |
| Chrome version | record of the tested version (benchmark tested v120) | 155.0.8059.39 | 155.0.8059.39 | 155.0.8059.39 |
| Extra packages pulled | what an install adds to the host | liberation fonts, `vulkan-loader`, `mesa-vulkan-drivers` | liberation fonts | liberation fonts, `xdg-utils` |
| Repo file | the role's opt-in install must match it | `/etc/yum.repos.d/google-chrome.repo`, identical on all 3 (same as W-1) | same | same |
| Google signing key imported | without it, `dnf` can't verify later Chrome updates from the repo | **no** (see T-C1) | **no** | **no** |
| Daily cron | it rewrites the repo file and retries the key | `/etc/cron.daily/google-chrome` present | present | present |
| `/etc/opt/chrome` after install | does the RPM create `policies/`? | unchanged: only `native-messaging-hosts/` (2 GNOME files), `etc_t` | same | same |

Evidence: terminal output pasted in the session (2026-10-07); key lines quoted above and in
[troubleshooting T-C1](troubleshooting.md#t-c1-google-chrome-rpm-key-1-import-failed-during-install).

### What step 2 tells us

1. **No OS-version gate needed for install**: same Chrome version, same repo file, same result on 8, 9 and 10.
2. **The Google key is not imported by the RPM when installed with dnf** (the RPM's own script can't get the rpm lock
   that dnf holds). The role's opt-in install must import the key itself (`ansible.builtin.rpm_key`) **before** adding
   the repo and installing, so the package is signature-checked.
3. **The role must create `/etc/opt/chrome/policies/managed/`**: the RPM doesn't (confirmed on all 3).
4. Still open: `rpm -qf /etc/opt/chrome/native-messaging-hosts` and the desktop session type (moved to step 3).

## D-3. Step 3: policy read test + Windows-only check (pending)

Planned commands (each VM; log in to the desktop at the Proxmox console first):

```bash
# a) who owns the GNOME native-messaging files (rule 2.5.1), and the desktop session type (rules 4.1.1/4.12)
rpm -qf /etc/opt/chrome/native-messaging-hosts
for s in $(loginctl list-sessions --no-legend | awk '{print $1}'); do loginctl show-session "$s" -p Name -p Type -p Class; done

# b) test policy file: 1 control policy + the win?/obsolete candidates, all at their CIS values
sudo mkdir -p /etc/opt/chrome/policies/managed
sudo tee /etc/opt/chrome/policies/managed/discovery-test.json <<'JSON'
{
  "MetricsReportingEnabled": false,
  "ThirdPartyBlockingEnabled": true,
  "RendererAppContainerEnabled": true,
  "ChromeCleanupReportingEnabled": false,
  "CloudAPAuthEnabled": true,
  "RemoteAccessHostAllowUiAccessForRemoteAssistance": false,
  "CloudPrintProxyEnabled": false,
  "ExtensionManifestV2Availability": 3,
  "FirstPartySetsEnabled": false,
  "CertificateTransparencyEnforcementDisabledForLegacyCas": []
}
JSON
sudo ls -laZ /etc/opt/chrome/policies /etc/opt/chrome/policies/managed
```

c) In the desktop: Chrome → `chrome://policy` → **Reload policies** → screenshot.

### Results a) and b) (2026-10-07)

| Check | Why we check it | RHEL 8 | RHEL 9 | RHEL 10 |
|-------|-----------------|--------|--------|---------|
| Owner of `/etc/opt/chrome/native-messaging-hosts` | rule 2.5.1 (`NativeMessagingBlocklist = ["*"]`) would block this; know what users lose | `chrome-gnome-shell-42.1-1.el8` | `chrome-gnome-shell-42.1-1.el9` | `gnome-browser-connector-42.1-9.el10` (renamed package, same 2 files) |
| Desktop session type | X11 vs Wayland changes screen capture (4.1.1/4.12) | **not seen**: only gdm greeter (wayland) + SSH tty; no desktop login at the time | same | same (+ systemd `manager` sessions) |
| Label of new `policies/` + `managed/` + file | Chrome must be able to read the file under SELinux Enforcing | `etc_t` (user `unconfined_u` because created by hand with sudo) | same | same |

**What it tells us:**
1. 2.5.1 blocks GNOME's "install GNOME Shell extensions from the browser" integration on all 3 → note it in the rule's
   `# WARNING:` (L2 rule, toggle off by default anyway). The role doesn't touch these files.
2. SELinux: the new dir and file get the normal `/etc` type `etc_t`, same as the rest of `/etc/opt/chrome`. The SELinux
   *user* part (`unconfined_u` vs `system_u`) doesn't affect access. **No SELinux handling needed** in the role (to
   confirm by `chrome://policy` showing OK).
3. Session type still open: log in at the console, then re-run the `loginctl` loop.

### Result c): `chrome://policy` (2026-10-07)

Evidence: [04-rhel8](evidence-images/04-rhel8-discovery-policy-page-ok.png),
[05-rhel9](evidence-images/05-rhel9-discovery-policy-page-ok.png),
[06-rhel10](evidence-images/06-rhel10-discovery-policy-page-ok.png). Chrome 155.0.8059.39, identical on all 3.

| Policy | CIS ID | Why it was in the test | RHEL 8 | RHEL 9 | RHEL 10 |
|--------|--------|------------------------|--------|--------|---------|
| `MetricsReportingEnabled` | 3.12 | control: proves Linux reads the file | **OK** | **OK** | **OK** |
| `ThirdPartyBlockingEnabled` | 1.19 | suspected Windows-only | Error | Error | Error |
| `RendererAppContainerEnabled` | 2.30 | suspected Windows-only | Error | Error | Error |
| `ChromeCleanupReportingEnabled` | 3.6 | suspected Windows-only | Error | Error | Error |
| `CloudAPAuthEnabled` | 2.10.1 | suspected Windows-only | Error | Error | Error |
| `RemoteAccessHostAllowUiAccessForRemoteAssistance` | 2.8.2 | suspected Windows-only | Error | Error | Error |
| `CloudPrintProxyEnabled` | 2.7.1 | suspected retired | Error | Error | Error |
| `ExtensionManifestV2Availability` | 2.3.6 | suspected obsolete (MV2 removed) | Error | Error | Error |
| `FirstPartySetsEnabled` | 2.9.1 | suspected renamed/removed | Error | Error | Error |
| `CertificateTransparencyEnforcementDisabledForLegacyCas` | 1.10 | suspected removed | Error | Error | Error |

Source `Platform`, applies to `Machine`, level `Mandatory` for all rows: Chrome treats the file as machine-wide,
mandatory policy (users can't override), which is what CIS wants (benchmark-summary F4).

**What it tells us:**
1. **The core mechanism works on RHEL 8, 9 and 10**: a JSON file in `/etc/opt/chrome/policies/managed/` is applied
   as mandatory machine policy. SELinux (`etc_t`, Enforcing) does not block it → no SELinux handling in the role.
2. **All 9 suspected rules are rejected by Chrome 155 on RHEL** → none of them can be applied; they will not be
   implemented. Reason per row (Windows-only vs removed from Chrome) still to read from each row's expanded message,
   to label them `na_os` or `obsolete` correctly.
3. No difference between OS versions → no version gate.

### Result d): error reason and session type (2026-10-07)

**Error reason:** every rejected row expands to **"Unknown policy"** (user's check, all 3 OS). Chrome on Linux
doesn't know these names, either because they exist only in Windows builds or because they were removed after Chrome
v120 (the benchmark's test version). The message doesn't say which, and it doesn't matter for the role: Chrome on RHEL
can't apply them. Classification: see the next section (the reasons below were first guessed from the benchmark text, then checked).

### Result e): Google's policy list for Chrome 155 (2026-10-07, revised classification)

**Why:** the VM test proves *that* a policy fails, not *why*, and it only covered the 9 policies I suspected. Google's
official list says, per policy and per platform, where it is supported, for the exact Chrome version on the VMs.

```bash
curl -sS -o policy_templates.zip https://dl.google.com/dl/edgedl/chrome/policy/policy_templates.zip
unzip -q policy_templates.zip VERSION common/html/en-US/chrome_policy_list.html   # VERSION: 155.0.8059.40
# per CIS policy name: find name="<Policy>" and read its "Supported on:" lines (script run in the scratchpad)
```

| Result | CIS IDs | Notes |
|--------|---------|-------|
| Supported on Linux | 101 policies (102 rules: 4.1.1 and 4.12 share one) | `ProxyMode` (2.17) is marked **Deprecated** (successor `ProxySettings`): still works, design note for 2.17 |
| **Windows only** | 2.8.2, 2.10.1, 2.30, **3.13** | 3.13 `SafeBrowsingForTrustedSourcesEnabled` was not on the suspect list |
| **Not in the list (removed from Chrome)** | 1.10, 1.19, **2.3.4**, **2.3.5**, 2.3.6, 2.7.1, 2.9.1, **2.23**, **2.29**, 3.6 | Bold = not on the suspect list |
| Not a Chrome policy | 2.1.1, 2.1.2 | Google Update (Windows/macOS updater) |

**Revised:** my first reasons were partly wrong: 1.19 and 3.6 were Windows-only **and** have since been removed (per-rule search text in [cis-requirements.md](cis-requirements.md#evidence-for-the-16-not-applicable-rules-search-it-yourself)). And there are
**16** N/A rules, not 11: five more (2.3.4, 2.3.5, 2.23, 2.29, 3.13) found only by the list. All 9 VM-tested rules
agree with the list. Full per-rule result: [cis-requirements.md](cis-requirements.md).

**Session type:** `echo $XDG_SESSION_TYPE` returned `tty` (rhel9, rhel10) and empty (rhel8): it was run in a shell
that isn't the desktop session (SSH), so the answer is still open. It only matters for testing the L2 screen-capture
rules (4.1.1/4.12); Chrome enforces `ScreenCaptureAllowed` itself on either display server. Low priority.

**Policy precedence (seen on `chrome://policy`):** shows which policy *source* wins when the same policy is set in more
than one place (e.g. cloud management vs platform). Here the only source is **Platform / Machine** (our JSON file), so
it wins. Not shown there but relevant: Chrome merges every `*.json` file in `managed/`; the role should own one file
and its AUDIT should report other files that set the same policy (to verify when designing the AUDIT step).

## D-4. Step 4: full test of all applicable policies (2026-10-07)

**Why:** Google's catalogue says the 101 policies exist on Linux, but only one had been proven on the VMs, and a correct
name with a wrong value type or out-of-range value also gives an error. One file with every applicable policy at its
CIS value tests names **and** values at once, before any code is written.

Files (kept as proof, see [evidence-files/](evidence-files/)):

| File | Contents | Expected on `chrome://policy` |
|------|----------|-------------------------------|
| `cis-applicable-101.json` | the 101 policies of the 102 applicable rules (4.1.1 and 4.12 share `ScreenCaptureAllowed`), CIS values; SITE rules with harmless examples (`example.com`, `ProxyMode: "system"`); "keep unset" rules as `[]` to test the name only | all **OK** |
| `cis-not-applicable-14.json` | the 14 N/A rules that are Chrome policy names (2.1.1/2.1.2 are Google Update registry values with no JSON form) | all **Error: Unknown policy** |

Commands (each VM):

```bash
scp -i ~/.ssh/id_ed25519_cis cis-applicable-101.json frqadmin@<vm>:/tmp/        # from the workstation
sudo rm -f /etc/opt/chrome/policies/managed/discovery-test*.json
sudo install -m 0644 -o root -g root /tmp/cis-applicable-101.json /etc/opt/chrome/policies/managed/cis-all-test.json
# desktop: close Chrome fully, reopen, chrome://policy -> Reload policies -> sort by Status
```

| Result | RHEL 8 | RHEL 9 | RHEL 10 |
|--------|--------|--------|---------|
| 101 policies | all **OK** | all **OK** | all **OK** |
| `ProxyMode` | OK, labelled **Deprecated** | same | same |

Evidence: user's check on all 3 VMs (2026-10-07), recorded here **without screenshots** (101 rows don't fit one
screen). `cis-not-applicable-14.json` was not loaded as a whole; 9 of its 14 were tested in D-3 (Unknown policy).

**What it tells us:**
1. All 102 applicable rules work on RHEL 8, 9 and 10 with the CIS values: names, types and values. Nothing else fails
   besides the 16 N/A.
2. `ProxyMode` (2.17) works but is deprecated (successor `ProxySettings`): design note.
3. Some policies only apply at the next Chrome start (catalogue: "Dynamic Policy Refresh: No", e.g.
   `MetricsReportingEnabled`, `DiskCacheSize`) → the role writes the file; it does not restart users' browsers.

## W-3. What else is in Google's policy zip (2026-10-07)

`policy_templates.zip` (Chrome 155) contains only `VERSION`, `common/html/<lang>/chrome_policy_list.html`,
`common/html/<lang>/chrome_policy_atomic_groups_list.html` and `windows/adm*` (Group Policy templates). **No Linux
template:** on Linux each policy's JSON key is the "Mac/Linux preference name" shown in `chrome_policy_list.html`.

**Atomic policy groups** (`chrome_policy_atomic_groups_list.html`): "groups of policies that depend on each other…
only values coming from the highest priority source will be applied. Values coming from a lower priority source in the
same group will be ignored." Groups that contain CIS policies include `Proxy` (2.17), `Extensions` (2.3.x,
`ExtensionInstallAllowlist`/`Blocklist`), `RemoteAccess` (2.8.x), `SafeBrowsing` (1.2.x).
**Design impact:** none while our JSON file is the only source. If a site also uses cloud management, a group must
come from one source: e.g. the extension allowlist (2.3.3 site value) must be in the same file as the blocklist,
which the role does.

## W-5. Older Chrome versions: what can be installed (2026-10-09)

**Why:** organizations may run an older Chrome than the newest one; check what can still be installed and whether
the CIS policies work there. Refines W-2 ("one version per channel"): that is true for the repo **index**, not for
the files on the server.

| Check | Command | Result |
|-------|---------|--------|
| Real released versions | `curl -s 'https://versionhistory.googleapis.com/v1/chrome/platforms/linux/channels/stable/versions?pageSize=400'` | newest build per major, e.g. 155.0.8059.39, 154.0.8037.97, ..., 144.0.7559.132, 143.0.7499.192 |
| Is the RPM still on Google's server? ([evidence-files/chrome-rpm-availability-2026-10-09.txt](evidence-files/chrome-rpm-availability-2026-10-09.txt)) | `curl -sI -o /dev/null -w '%{http_code}' https://dl.google.com/linux/chrome/rpm/stable/x86_64/google-chrome-stable-<version>-1.x86_64.rpm` for **every** stable build of majors 138-146 | majors 138-142: none (0 of 4-6 builds); 143: only **143.0.7499.40** (1 of 5); 144-146: all builds. **Oldest downloadable: `143.0.7499.40`** |
| Oldest Chrome version any CIS policy needs | "since version" per policy in Google's policy template for Chrome 155 (`chrome_policy_list.html`) | **115** (2.3.7 `ExtensionUnpublishedAvailability`, 2.26 `GoogleSearchSidePanelEnabled`); all others older |

**What it means:**
- About the last 12-13 major versions (roughly one year) can be installed by direct URL, e.g.
  `dnf install https://dl.google.com/linux/chrome/rpm/stable/x86_64/google-chrome-stable-143.0.7499.40-1.x86_64.rpm`.
  Google does not remove whole majors in order (143 kept its first build, dropped the later ones), so check the exact
  build. Snapshot of 2026-10-09; Google decides how long files stay.
- **Revised 2026-10-09 (same day):** a first check of only the newest build per major said "144 is the oldest"; the
  full check of every build found 143.0.7499.40.
- Every downloadable version supports all 101 CIS policies (all exist since 115 or earlier).
- The role's install uses `state: present`: an existing older Chrome is kept, never upgraded. A plain `dnf update`
  upgrades it to the newest build in Google's repo unless the site pins the version.
- **To verify on a VM (user test):** install 143.0.7499.40 (or a 144 build) by URL, run the role with `chrome_cis_install: false`, check the
  report version, `chrome://policy` (all OK), rerun `changed=0`, and that `chrome_cis_install: true` keeps 144.
