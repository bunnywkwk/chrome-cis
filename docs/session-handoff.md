# Session handoff: chrome_cis

**Read this first when you continue on another machine or in a new Claude session.** It says where the work stopped,
how we think about it, and what to do next. Last update: **2026-10-07**.

## 1. Where we are

| Phase ([role-workflow](../../docs/role-workflow.md)) | Status | File |
|-------|--------|------|
| 1. Read the benchmark | ✅ | [benchmark-summary.md](benchmark-summary.md) |
| 2. Requirements matrix | ✅ 118 rules: 102 applicable, 16 N/A, with evidence | [cis-requirements.md](cis-requirements.md) |
| 3. Discovery on RHEL 8/9/10 | ✅ | [platform-notes.md](platform-notes.md) |
| 4. Design decisions | ✅ D1–D14 | [design-decisions.md](design-decisions.md) |
| 5. Build | ⏳ Claude wrote `meta/`, `defaults/`, `vars/`; **you type `tasks/`** from the guide | [build-guide.md](build-guide.md) |
| 6. Validate | ⏳ guide code passed lint + container tests; your typed files still to lint | build-guide "Results" |
| 7–8. Test project + VM runs | ❌ next | — |
| 9–11. Compliance record, evidence, wrap-up | ❌ | — |

Daily history: [work-log.md](work-log.md). Errors met: [troubleshooting.md](troubleshooting.md). Every file explained:
[code-walkthrough.md](code-walkthrough.md).

## 2. Next steps, in order

1. Type batches 1–5 from [build-guide.md](build-guide.md) into `tasks/` (main, prelim, install, policy, report).
2. After each batch, inside `chrome_cis/`: `yamllint . && ansible-lint --offline` (errors about missing task files are
   expected until batch 5).
3. Test project (separate repo, like `mongodb8-cis-test`): inventory with rhel8/9/10, one `group_vars` file, `site.yml`.
4. VM runs, each VM reverted to snapshot **`pre-chrome`** first: defaults (skip message) → `chrome_cis_install: true`
   → rerun `changed=0` → Level 2 → `chrome://policy` screenshot (all rows OK).
5. Compliance record + README (batch 6), then Chromium.

## 3. The mindset (why the role looks like this)

| Principle | In this role |
|-----------|--------------|
| **The benchmark is the source of truth** | only the 118 CIS rules; never extra Chrome policies; CIS IDs never renumbered |
| **Evidence, not assumption** | every "works on RHEL" / "N/A" claim is backed by Google's policy template for Chrome 155 (Ctrl+F search text per rule) **and** a VM test. Base proof on that template, not on new sources |
| **The benchmark is written for Windows** | PDF printed p. 8–9 (viewer p. 9–10): Group Policy / registry. On RHEL the same **policy names** go into a JSON file in `/etc/opt/chrome/policies/managed/` |
| **Data, not 100 tasks** | all rules use one mechanism, so they're a list in `vars/main.yml`; one `copy` writes the JSON (D3) |
| **Safe by default** | L1 on, L2 off; 12 risky rules off with a `# WARNING:` saying what breaks; site values report-only until set |
| **Users choose, never weaken** | they can switch rules off/on or set site values; they can't change a CIS value |
| **All 118 accounted for** | N/A rules stay in the list with a reason and appear in every report |
| **Don't over-engineer** | simple inputs only (one value or a flat list); no restart of users' browsers; no SELinux tasks (not needed) |
| **You type tasks, Claude writes the rest** | Claude gives code block by block with what + why; you must be able to defend every line |

## 4. Reading `defaults/main.yml` and `vars/main.yml`

**`defaults/main.yml` = what a user may change** (set in `group_vars`):

| Group | Variables | Default |
|-------|-----------|---------|
| Install | `chrome_cis_install`, `chrome_cis_channel` | `false`, `stable` |
| Profiles | `chrome_cis_level_1`, `chrome_cis_level_2` | `true`, `false` |
| Site values (11 SITE rules + 2 allowlists) | e.g. `chrome_cis_proxy_mode`, `chrome_cis_http_allowlist` | `null` / `[]` = report only |
| Rule toggles | `chrome_cis_rule_<id>` (101) | `true`; 12 risky ones `false` with the reason in the comment |

**`vars/main.yml` = internal, don't override:**

| Variable | Is |
|----------|----|
| `chrome_cis_rules` | the 118 rules (+2 allowlist entries): `{id, level, policy, value}`; `var:` = site variable; `na:` = not applicable + reason; `toggle:` = use another rule's toggle (4.12 → 4.1.1); `companion:` = allowlist next to a blocklist |
| `chrome_cis_rule_status` | for each entry: `applied` / `off_level` / `off_toggle` / `site_unset` / `na` |
| `chrome_cis_policies` | the applied entries as `{policy: value}` = the JSON content |
| `chrome_cis_installed(_versions)` | `google-chrome-*` packages found by `package_facts` |

With defaults: **65** policies written, report `applied 65, off 28, site not set 9, N/A 16, total 118`.

## 5. Facts to remember

| Fact | Where proven |
|------|--------------|
| Chrome 155.0.8059.39 on RHEL 8.10 / 9.7 / 10.2: same behaviour, no OS branches | platform-notes D-1..D-4 |
| The RPM doesn't create `policies/managed/`; the role does | D-2 |
| Installing by hand doesn't import Google's key (rpm lock under dnf) → role imports it first | troubleshooting T-C1 |
| "Chrome Enterprise" on Linux = the same RPM + policies; beta/unstable/canary are in the same repo | platform-notes W-2 |
| All 101 applicable policies OK in `chrome://policy`; `ProxyMode` is "Deprecated" but works | D-4 |
| "Disabled" exception lists written as `[]` (user decision): same effect as absent, visible, enforced | design-decisions D7 |
| Some policies load only at Chrome start; the role never restarts Chrome | D11 |

**Still open (small):** desktop session type (X11/Wayland) only matters for testing 4.1.1/4.12; that beta reads the
same policy folder (expected, to verify); 5 N/A rules not yet loaded on a VM (`evidence-files/cis-not-applicable-14.json`).

## 6. Environment

| Item | Value |
|------|-------|
| VMs (Proxmox) | rhel8 `192.168.20.50`, rhel9 `192.168.20.30`, rhel10 `192.168.20.40`, "Server with GUI", MongoDB on them; SSH key `~/.ssh/id_ed25519_cis` |
| Snapshot | **`pre-chrome`** on all 3 (before any Chrome work). They currently have Chrome + test policy files from discovery → revert before role tests |
| Control node | ansible-core 2.16 venv `~/.ansible-2.16.1-env` ([control-node-setup](../../docs/control-node-setup.md)); no collections needed for this role |
| Benchmark files | `cis-pdf/`, `cis-spreadsheet/` at the repo root; **never commit them** (`.gitignore` blocks `*.pdf`, `*.xlsx`) |
| Standup notes | [docs/standup.md](../../docs/standup.md) (short, one entry per day) |

### Optional: repeat Claude's container test

Proves the Ansible side without the VMs (no desktop, so no `chrome://policy`). Needs `podman` and the
`containers.podman` collection (test only, not a role dependency):

```bash
ansible-galaxy collection install containers.podman -p ~/chrome-cis-ctest/collections
for v in 8 9 10; do podman run -d --name cis$v docker.io/rockylinux/rockylinux:$v sleep infinity; done
cd ~/chrome-cis-ctest
cat > inv.ini <<'EOF'
[containers]
cis8 ansible_python_interpreter=/usr/libexec/platform-python
cis9 ansible_python_interpreter=/usr/bin/python3
cis10 ansible_python_interpreter=/usr/bin/python3
[containers:vars]
ansible_connection=containers.podman.podman
ansible_become=false
EOF
printf -- '---\n- name: Test chrome_cis\n  hosts: all\n  roles:\n    - role: chrome_cis\n' > site.yml
ANSIBLE_ROLES_PATH=~/ansible-cis ANSIBLE_COLLECTIONS_PATH=./collections \
  ansible-playbook -i inv.ini site.yml -e '{"chrome_cis_install": true}'      # then rerun: changed=0
podman rm -f cis8 cis9 cis10                                                   # clean up
```
