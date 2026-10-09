# Design decisions: chrome_cis

Every decision has **Decision**, **Why** and **Evidence**. Evidence IDs point to [platform-notes.md](platform-notes.md)
(W-n = workstation/package facts, D-n = discovery on the VMs) and [cis-requirements.md](cis-requirements.md).
Status: **draft for review, 2026-10-07**. Changes later are marked "Revised" with the reason, never silently rewritten.

Scope: **Google Chrome (`google-chrome-*` RPM) on RHEL 8, 9, 10**, CIS Google Chrome Benchmark **v3.0.0** (118 rules;
102 applicable on RHEL, 16 not applicable). Chromium comes later as a separate step.

---

## D1. One role for RHEL 8, 9 and 10, no OS branches

- **Decision:** one role; no `when:` on the OS version anywhere except the supported-OS check in `prelim.yml`.
- **Why:** an OS branch is only justified by a real difference; none was found.
- **Evidence:** D-1..D-4: same Chrome 155 build, same repo file, same policy folder, same SELinux label (`etc_t`), and all
  101 policies OK on 8.10, 9.7 and 10.2.

## D2. Mechanism: one managed-policy JSON file owned by the role

- **Decision:** the role writes **one** file, `/etc/opt/chrome/policies/managed/chrome_cis.json` (root:root `0644`; folder
  `0755`), and nothing else in that folder.
- **Why:** this is how Chrome on Linux takes enterprise policies; "managed" = mandatory, users can't override (what CIS
  wants, benchmark-summary F4). One file the role owns = easy to audit and to remove; other teams can keep their own
  files next to it.
- **Evidence:** W-4 (Google's Linux policy docs: "Create the following directories if they do not already exist … /etc/opt/chrome/policies/managed"); W-1 (path compiled into Chrome); D-3/D-4: file read and shown as *Platform / Machine / Mandatory*, OK on
  all 3 OS; the RPM doesn't create `policies/managed/` (D-2), so the role creates it.

## D3. Rules as data, one task writes the file (data-driven)

- **Decision:** all 118 rules live in `vars/main.yml` as a list (`id`, `level`, `policy`, `value`, `kind`, `reason` for
  N/A). One task filters the list by the user's choices (D4–D7) and writes the JSON with `ansible.builtin.copy` and
  `content: "{{ ... | to_nice_json }}"`.
- **Why:** ~100 rules use the **same** mechanism (policy name = value). One task per rule would be ~100 copies of the
  same code. CLAUDE.md, app roles: "single declarative artifact → rules as data, rendered by one task". `copy` is
  declarative: it compares and only changes the file on drift (no separate AUDIT step needed, CLAUDE.md AUDIT→PATCH
  rule 3), and `--check --diff` shows exactly which policies would change.
- **Evidence:** cis-requirements.md (every applicable rule = one policy + value); D-4 (one file with 101 policies works).
- **Not adopted** (at the time): one task per rule (MongoDB style: there, rules used different modules/restarts); a
  looping `set_fact` task; a Jinja template.
- **Revised 2026-10-09: one task per rule, Lockdown layout** (branch `lockdown`). The team review asked for the
  Ansible Lockdown / `mongodb8_cis` structure: `tasks/section_<n>/main.yml` + `cis_<n>.x.yml`, one task per CIS rule
  with the standard name (`"<ID> | PATCH | <CIS title>"`), its own `when:` (level + rule toggle; section switch on the
  import) and tags (`level1`/`level2`, `automated`/`manual`, `patch`, `rule_<id>`, topic), so a rule can be read, run
  (`--tags rule_2.3.3`) and skipped (`--skip-tags rule_2.3.3`) on its own.
  - **How a rule applies:** the rule task evaluates its `when:` and, if it applies, adds its policy to
    `chrome_cis_policies` with `set_fact` + `combine`. `post.yml` writes **one** file, `chrome_cis.json`, with
    `ansible.builtin.copy` (declarative: unchanged content = no change, `--check --diff` shows the diff). Same pattern
    as Lockdown RHEL9-CIS auditd rules (rule tasks set a fact, one task writes the file).
  - **Full run vs limited run:** `prelim.yml` starts `chrome_cis_policies` **empty on a full run**, so the file always
    matches the current settings (a rule switched off is removed on the next run); with `--tags`/`--skip-tags` it
    starts from the **current file**, so only the selected rules change (`ansible_run_tags`, `ansible_skip_tags`).
  - **Not adopted:** one policy file per rule (`managed/cis_<id>.json`, ~101 files; user preferred one file).
  - The data-driven variants (rule list + Jinja loop, `set_fact` loop, baseline files) were built and tested on
    branches `staging`, `staging-option-b`, `staging-option-c`; kept as history.
## D4. Levels gate the rules (L1 on, L2 off)

- **Decision:** `chrome_cis_level_1: true`, `chrome_cis_level_2: false`. A rule is written only if its level is on.
  Level 2 = L1 + L2.
- **Why:** CIS Profile Definitions (PDF printed p. 14, viewer p. 15): L2 "may negatively inhibit the utility" (blocks mic, camera,
  uploads, screen share). Same as `mongodb8_cis` D18 and CLAUDE.md.

## D5. Every rule has its own toggle; risky ones default to off

- **Decision:** `chrome_cis_rule_<id>` per applicable rule in `defaults/main.yml`, default `true`. **13 risky rules
  default to `false`** with a `# WARNING:`: 2.2.3, 2.3.3, 2.4.1, 2.5.1, 2.12, 2.18, 3.1.1, 4.1.1, 4.3, 4.4, 4.5, 4.7,
  and 4.12 (same policy as 4.1.1, follows 4.1.1's toggle).
- **Why:** level = CIS's profile; the toggle = our safety switch for things that break users' work (e.g. 2.3.3 removes
  every installed extension, 4.4 kills web-call audio). Same rule as `mongodb8_cis` (2.1, 2.2 are L1 but off).
  Per-rule variables (not only a skip list) so users can switch a rule **on** by name and see all rules in defaults.
- **Evidence:** benchmark-summary §3 (Impact column of each rule).
- **Not adopted:** a global "disruptive" switch (removed in `mongodb8_cis`, D6 there: hides which rules it covers).
- **Revised 2026-10-08 (user decision): risky rules now default to `true`.** The role = full CIS by default; the
  site turns off what it can't accept in `group_vars` (`chrome_cis_rule_4_7: false`).
  - **Why:** `group_vars` then holds only the site's exceptions, the same list as the compliance record's exceptions
    table; nothing is silently skipped by the role. Still safe by default for most of them: 11 of the 12 are Level 2,
    and Level 2 is off by default. With plain defaults only **2.3.3** (L1, removes every extension not in
    `chrome_cis_extension_allowlist`) is a risky rule that applies.
  - **Kept:** the `# WARNING:` comment on each toggle, and the effect of each one in
    [user-view.md](user-view.md) (Risky rules table).
  - **Evidence:** 4.7 on the full-benchmark run 2026-10-08: every site "can't be reached" (no DoH server). The others:
    CIS Impact section + Google's policy description; to be seen on the VMs.
  - Deviates from the Lockdown convention ("disruptive = off") used in `mongodb8_cis`, which keeps it.

## D6. SITE rules: report by default, applied only when the site sets a value

- **Decision:** the 11 SITE rules get one variable each, default `null` (scalar) or `[]` (list) = **report only**:

  | CIS | Variable | Accepts |
  |-----|----------|---------|
  | 1.2.2 | `chrome_cis_safe_browsing_protection_level` | `1` (standard) or `2` (enhanced) |
  | 1.9 | `chrome_cis_variations` | `0` (all variations, CIS value) |
  | 2.6.1 | `chrome_cis_password_manager_enabled` | `true` / `false` |
  | 2.8.1 | `chrome_cis_remote_access_connections` | `true` / `false` (CIS: `false`) |
  | 2.8.3 | `chrome_cis_remote_access_client_domains` | list of domains |
  | 2.17 | `chrome_cis_proxy_mode` | `direct` or `system` (never `auto_detect`) |
  | 2.27 | `chrome_cis_http_allowlist` | list of hosts |
  | 4.2.3 / 4.2.4 | `chrome_cis_clipboard_allowed_urls` / `_blocked_urls` | list of URLs |
  | 4.2.7 / 4.2.8 | `chrome_cis_window_management_allowed_urls` / `_blocked_urls` | list of URLs (L2) |

  Two allowlists belong to risky rules: `chrome_cis_extension_allowlist` (2.3.3) and
  `chrome_cis_native_messaging_allowlist` (2.5.1), default `[]` = block all.
- **Why:** CLAUDE.md principle 5, and simple inputs only (as in `mongodb8_cis`): every input is one value or a flat list. `null` (not `false`)
  means "not decided", because `false` is a real answer for 2.6.1 and 2.8.1.
- **2.17 simplified:** only `direct`/`system`. `fixed_servers`/`pac_script` need extra nested values (server, PAC URL);
  a site that needs them sets `ProxyMode` in its own file (README). `ProxyMode` is marked Deprecated by Google but
  still works and is what CIS checks (D-4).
- **Allowlists go in the same file as their blocklist:** Chrome's *atomic groups* take a whole group from one source
  (W-3), so a blocklist here plus an allowlist elsewhere could drop the allowlist.

## D7. "Disabled" exception lists: written as an empty list `[]`

- **Decision:** the 9 rules where CIS wants an exception list *Disabled* (1.2.1, 1.11, 1.12, 1.25, 1.26, 1.27, 1.29,
  2.2.5, 2.25) are written as `[]`. A rule **passes when the policy is absent or an empty list** in every file in
  `managed/`. The report flags another file that sets real entries (e.g. `["intranet"]`).
- **Why:** an empty list = no exceptions, the same effect as absent (Google's template: e.g. 1.2.1 "Leaving the policy
  unset means default Safe Browsing protection applies to all resources"). Writing it makes the rule **visible** in
  `chrome://policy` and, as mandatory machine policy, keeps lower-priority sources (e.g. cloud) from adding exceptions.
- **CIS deviation (documented):** the CIS audit says the registry path "will not exist" when Disabled: that is how
  Windows stores a Disabled list. On RHEL an empty list has the same effect, so `[]` counts as Disabled.
- **Evidence:** D-4: all 9 written as `[]` showed **OK** on RHEL 8, 9 and 10.
- **Known gap (accepted 2026-10-07, user decision):** the report compares only policies the role **writes**. A CIS
  policy the role does not apply (rule off, Level 2 off, SITE value not set, e.g. 2.17) but another file sets (e.g.
  `ProxyMode: "auto_detect"`) is not reported; Chrome uses that value because it is the only one. Covered case: both
  files set the same policy (Google: "the behavior is undefined", W-4) → listed in `conflicts_in_other_files`.
  Possible later fix: also list CIS policies set elsewhere while not applied (a few lines in `report.yml`).
- **Revised 2026-10-07 (before review):** first draft left them unset; changed on the user's decision for visible,
  enforced assurance.
- **Revised 2026-10-09 (user decision):** the check for other policy files (`conflicts_in_other_files`) was
  **removed**: it is not a CIS recommendation, so it is out of scope (over-engineered add-on). The "Known gap" above no
  longer applies; other files in `managed/` are the site's responsibility.

## D8. Not-applicable rules stay in the list, never written, always reported

- **Decision:** the 16 N/A rules are entries with `kind: na` and a `reason`; they are never written, and every run
  reports them as `N/A: <reason>`.
- **Why:** all 118 CIS IDs are accounted for in code, docs and run output; `chrome://policy` stays free of "Unknown
  policy" errors, so any error there is a real problem (no noise).
- **Evidence:** cis-requirements.md, "Evidence for the 16 not-applicable rules" (policy template search + VM test).

- **Revised 2026-10-09 (user decision):** the 14 N/A rules that have a Chrome policy are now normal rule tasks
  (tag `not_applicable`) with a switch, grouped in `defaults/main.yml` under "Not applicable on
  RHEL / current Chrome" (removed from Chrome / Windows only). **On by default** (user decision, same day): the site
  sets `false` in `group_vars` where its Chrome version or Chromium doesn't support them. Why: the role now installs other Chrome versions and
  Chromium (D15); a site can switch one on and check `chrome://policy`. 2.1.1/2.1.2 (Google Update) have no Chrome
  policy and stay comments. The `chrome_cis_not_applicable` list and its report line were removed.
## D9. Install is opt-in; key first, then repo, then package

- **Decision:** `chrome_cis_install: false`. When `true`: import Google's key (`ansible.builtin.rpm_key`), add the repo
  (`ansible.builtin.yum_repository`, same name and content as Google's `/etc/yum.repos.d/google-chrome.repo`), install
  `google-chrome-<channel>` (`ansible.builtin.dnf`).
- **Why:** key first so the package is signature-checked and the RPM's own key import (which fails under dnf) isn't
  needed. Same file name and section (`google-chrome.repo`, `[google-chrome]`) as Google's, so there is only one repo
  definition.
- **Revised 2026-10-07:** the draft said "same content" so Google's daily cron wouldn't rewrite the file. Not exact:
  Ansible writes `baseurl = …` (spaces), Google writes `baseurl=…`. They still don't fight: the cron
  (`/etc/cron.daily/google-chrome`) only recreates the file if it is **missing** (`verify_install`: `[ -f
  "$YUM_REPO_FILE" ]`) and only edits lines starting exactly with `baseurl=` (`sed -i -e "s,^baseurl=.*,…"`), which
  ours don't. Container test T3: rerun after install `changed=0`.
- **Evidence:** D-2, T-C1 (key not imported when installed by hand); W-1 (RPM writes the repo file + daily cron);
  cron script read on the workstation (google-chrome-stable 154); container test T3 (2026-10-07, work-log item 18).

## D10. Channel by one variable; detect, don't assume; no version gate

- **Decision:** `chrome_cis_channel: stable` (`beta`, `unstable`, `canary`) → package `google-chrome-<channel>`.
  `prelim.yml` uses `package_facts` to find installed `google-chrome-*` packages and reports their versions. Nothing
  installed and install off → clear message, role ends for that host (no failure). No Chrome version gate.
- **Why:** Chrome policies are identified by name across versions, and Chrome ignores names it doesn't know (D-3), so
  a version gate would add nothing. The matrix records the version it was proven on (155).
- **Evidence:** W-2 (all channels in Google's repo, no separate "enterprise" package).
- **To verify when tested:** beta/unstable read the same `/etc/opt/chrome/policies/managed/` (W-2).

## D11. No browser restart

- **Decision:** no handler restarts Chrome. README: settings apply at the next Chrome start (`chrome://policy` →
  *Reload policies* applies the dynamic ones at once).
- **Why:** Chrome is a user app, not a service; closing it would lose users' work. Some policies only load at start
  ("Dynamic Policy Refresh: No" in the policy template).
- **Evidence:** D-4.

## D12. Supported targets and control node

- **Decision:** RHEL 8/9/10 and compatible (Rocky, Alma, Oracle), **x86_64 only**; `prelim.yml` asserts both.
  `min_ansible_version: "2.16.1"`; only `ansible.builtin` modules → **no collections needed**.
- **Why:** Google's RPM repo is x86_64 only (repo URL, W-1). Same control node as `mongodb8_cis` (ansible-core 2.16
  works on RHEL 8's Python).
- **Evidence:** W-1, W-2; `docs/control-node-setup.md`.

## D13. No SELinux handling

- **Decision:** none.
- **Evidence:** D-3: new folder/file get `etc_t` and Chrome reads them under Enforcing on all 3 OS.

## D14. Role and variable names

- **Decision:** role folder `chrome_cis` (renamed from `chrome-cis` on 2026-10-07), variable prefix `chrome_cis_`.
- **Why:** variable names can't contain `-`; ansible-lint's `role-name` rule rejects hyphens in role names; matches
  `mongodb8_cis`. No version number in the name (unlike `mongodb8_cis`): one CIS Chrome benchmark covers all Chrome
  versions (D10).

---

## D15. Browser choice and exact version (2026-10-09)

- **Decision:** two settings in `defaults/`: `chrome_cis_browser: chrome` (`chrome` or `chromium`) and
  `chrome_cis_version: ""` (empty = newest; or an exact version). `vars/main.yml` has one small lookup per browser
  (`chrome_cis_browsers`: package name + policy folder); package detection, install and the policy folder follow it.
  - **Chrome:** newest = `google-chrome-<channel>` from Google's repo; exact version = the RPM file on Google's server
    (`<repo>/google-chrome-<channel>-<version>-1.x86_64.rpm`), because the repo index lists only the newest build.
  - **Chromium:** `chromium` (or `chromium-<version>`) from **EPEL**; enabling EPEL is the site's prerequisite (on RHEL
    it also needs CodeReady Builder), not done by the role. Policy folder `/etc/chromium/policies/managed`.
- **Why:** organizations run Chrome or Chromium and sometimes a fixed, older version; one variable each is enough
  (CLAUDE.md app roles: "variants via a lookup dict", "version selection is one variable").
- **Evidence:** EPEL package and folder: `dnf repoquery chromium` → `chromium 154.0.8037.97-1.el10_2 epel`,
  `dnf repoquery -l chromium` → `/etc/chromium/policies` (workstation, EL10). Old Chrome RPMs: platform-notes W-5.
- **Limits:** the role never downgrades (`state: present`); an exact version only installs if Google (about one
  year back) or EPEL still has it; a later `dnf update` upgrades unless the site pins it. For Chromium there is no CIS
  benchmark: the role is "aligned with the CIS Google Chrome Benchmark"; which rules work on Chromium is to be
  confirmed with `chrome://policy` on a VM (CLAUDE.md "No benchmark for the target", rule mapping later).

## Not adopted

| Idea | Why not |
|------|---------|
| Write the 16 N/A policies too | they do nothing on Linux and fill `chrome://policy` with errors (D8) |
| Chrome Enterprise Core (cloud) enrollment | not a CIS rule, needs a Google admin account; our file wins by default anyway (`CloudPolicyOverridesPlatformPolicy` unset) |
| `policies/recommended/` folder | users could override; CIS wants enforced settings |
| Extra non-CIS Chrome policies in this role | CLAUDE.md principle 2; sites can add their own file |
| Users changing a CIS value (e.g. a weaker `DownloadRestrictions`) | they can turn a rule off (visible in the report), not weaken it |
| Restarting Chrome after a change | D11 |
| Lockdown Goss audit framework | CLAUDE.md: only if asked |
