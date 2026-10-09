# Session handoff: chrome_cis

**Read this first when you continue on another machine or in a new Claude session.** It says where the work stopped,
how we think about it, how Claude should work with you, and what to do next. Last update: **2026-10-08**.

New Claude session: say *"read `chrome_cis/docs/session-handoff.md` and `chrome_cis/docs/CLAUDE.md` first"*.

## 1. Where we are (2026-10-09)

The role is being **rebuilt in the Ansible Lockdown layout** (team review: follow Lockdown / `mongodb8_cis` best
practice). Branch **`lockdown`** (from `staging`). Design: [design-decisions.md](design-decisions.md) D3 revised.

| Phase | Status |
|-------|--------|
| Benchmark, requirements matrix, discovery, decisions D1–D14 | Done (still valid; D3 revised) |
| Data-driven builds A / B / C | Done and tested; kept on `staging`, `staging-option-b`, `staging-option-c` as history only |
| **Lockdown rebuild** | **Next**: build guide per section, you type it section by section |
| VM test, compliance record, README | after the rebuild |

## 2. Target structure

```
tasks/main.yml              prelim -> install -> section_1..5 -> post
tasks/prelim.yml            checks, package_facts, stop if Chrome missing, start the policy set
tasks/install.yml           unchanged
tasks/section_<n>/main.yml  imports the cis_<n>.x.yml files of that section
tasks/section_<n>/cis_*.yml one task per CIS rule, benchmark order
tasks/post.yml              write chrome_cis.json (one copy), report
```

Each rule (CLAUDE.md task template) evaluates itself and adds its policy; `post.yml` writes one file:

```yaml
- name: "1.1.1 | PATCH | Ensure 'Cross-origin HTTP Authentication prompts' is set to 'Disabled'"
  when:
    - chrome_cis_level_1
    - chrome_cis_rule_1_1_1
  tags: [level1, automated, patch, rule_1.1.1, http_auth]
  ansible.builtin.set_fact:
    chrome_cis_policies: "{{ chrome_cis_policies | combine({'AllowCrossOriginAuthPrompt': false}) }}"
```

- **Full run:** the policy set starts empty, so `chrome_cis.json` always matches the settings (rule off = removed).
  **`--tags` / `--skip-tags` run:** starts from the current file, only the selected rules change.
- **Site rules:** the task runs only when the site value is set (`is not none` / `| length > 0`); the report lists
  the ones not set. **N/A rules:** no task; listed in the report (`chrome_cis_not_applicable`). **4.12:** own task,
  same policy as 4.1.1, follows `chrome_cis_rule_4_1_1`.
- **Tailoring for users:** levels, `chrome_cis_section1..5`, one `chrome_cis_rule_<id>` per rule, site values; all in
  `group_vars`, as in `mongodb8_cis`.

## 3. Branches (`github.com/bunnywkwk/chrome-cis`)

| Branch | Holds |
|--------|-------|
| `lockdown` | **the rebuild** (new) |
| `staging` | Option A (rule list + Jinja loop), tested; used by the test project until `lockdown` is ready |
| `staging-option-b`, `staging-option-c` | Option B (`set_fact` loop), Option C (baseline files); history, untouched |

## 4. Next steps

1. Done 2026-10-09 (Claude wrote it at the user's request): all 5 sections, `main.yml`, prelim start, `post.yml`,
   `vars/` trimmed, section switches in `defaults/`. Lint passes; container tests in [work-log.md](work-log.md).
2. **You test on the VMs** from `pre-chrome`: test project `requirements.yml` → `version: lockdown`; `group_vars`
   unchanged (toggles + site values; add `chrome_cis_section<n>: false` to try a section switch).
3. README, compliance record.

## 5. The mindset (why the role looks like this)

| Principle | In this role |
|-----------|--------------|
| **The benchmark is the source of truth** | only the 118 CIS rules; never extra Chrome policies; CIS IDs never renumbered |
| **Evidence, not assumption** | every "works on RHEL" / "N/A" claim is backed by Google's policy template for Chrome 155 (Ctrl+F search text per rule) **and** a VM test. Don't add new sources mid-way |
| **The benchmark is written for Windows** | PDF printed p. 8–9 (viewer p. 9–10): Group Policy / registry. On RHEL the same **policy names** go into a JSON file in `/etc/opt/chrome/policies/managed/` |
| **One task per rule (Lockdown)** | each CIS rule is its own task with name, toggle, level and tags; it adds its policy, `post.yml` writes one `chrome_cis.json` (D3 revised 2026-10-09) |
| **Full CIS by default, exceptions in `group_vars`** (2026-10-08) | L1 on, L2 off; the 12 risky rules are **on** with a `# WARNING:` saying what breaks; the site turns off what it can't accept (`chrome_cis_rule_4_7: false`), so `group_vars` = the compliance record's exception list. Applies to future roles too (CLAUDE.md principle 3). `mongodb8_cis` keeps the old "risky = false" |
| **Site values** | rules where CIS lets the organization decide (Manual): `null`/`[]` = report only; set = written (D6) |
| **All 118 accounted for** | N/A rules stay in the list with a reason and appear in every report |
| **No Chrome version pinning** | Google's repo keeps only the latest build per channel; policies work by name on any version (D10). Channel = one variable |
| **Don't over-engineer** | simple inputs only (one value or a flat list); no browser restart; no SELinux tasks |

## 6. How Claude works with you (keep this the same on every machine)

| You prefer | So Claude |
|------------|-----------|
| Learning by typing | you hand-type `tasks/` (and the computed vars); Claude gives code in a **guide or in chat**, block by block, each with a one-line **what** and **why**; never writes your task files unless you ask. If you say "I'll do it", Claude gives code + steps, no files |
| Plain words | simple English, tables, short answers; you have a Python background, so Python comparisons help |
| Snippet questions | when you paste a snippet, explain only that syntax, 3–8 lines with a tiny example |
| Honest answers | Claude says when something isn't verified, when it was wrong, and pushes back with a reason (e.g. risky defaults), then follows your decision |
| Simple options | one value or a flat list per option; complex per-item inputs → report-only + a documented hand fix |
| Reverts | when you say "revert", Claude reverts fully, no arguing |
| Review | after you type a file, Claude reviews it against the tested code, fixes or lists typos, runs `yamllint` + `ansible-lint` |
| Evidence | you drop screenshots, say "check it"; Claude reads each image, checks the command is right, renames `NN-<os>-<scenario>-<result>.png`, indexes it in `test-results.md` |
| Docs | Claude writes all docs; every doc has what / why / evidence; decisions changed → marked "Revised <date>"; **no emojis** (Pass / Fail / N/A / Yes / No) |
| Standup | short entry per day in `docs/standup.md` |
| Safety | Claude never commits or pushes unless asked; never hardens your workstation (commands for it are given to you to run); never commits CIS PDFs/spreadsheets |
| Errors | every error gets a troubleshooting entry with the exact output, cause, fix, prevent |

## 7. `defaults/main.yml` and `vars/main.yml`

**`defaults/main.yml`** (unchanged by the rebuild): install + channel, `chrome_cis_level_1/2`, 11 site values + 2
allowlists (`null`/`[]` = report only), 101 `chrome_cis_rule_<id>` toggles (12 risky ones with `# WARNING:`, on by
default). **`vars/main.yml`**: internal values only (repo, policy folder, installed Chrome); the rule list and the
status loop go away in the rebuild.

## 8. Facts to remember

| Fact | Where proven |
|------|--------------|
| Chrome 155.0.8059.39 on RHEL 8.10 / 9.7 / 10.2: same behaviour, no OS branches | platform-notes D-1..D-4 |
| The RPM doesn't create `policies/managed/`; the role does | D-2 |
| Installing by hand doesn't import Google's key (rpm lock) → role imports it first | T-C1 |
| "Chrome Enterprise" on Linux = the same RPM + policies; all channels in the same repo, one build each | platform-notes W-2 |
| All 101 applicable policies OK in `chrome://policy`; `ProxyMode` "Deprecated" but works | D-4 |
| 4.7 DoH "secure" without `DnsOverHttpsTemplates` breaks all browsing | user-view.md, 2026-10-08 |
| Some policies load only at Chrome start; the role never restarts Chrome | D11 |

**Still open (small):** desktop session type (X11/Wayland) for 4.1.1/4.12; beta/canary read the same policy folder
(expected); 5 N/A rules not yet loaded on a VM; README (batch 6); maybe make 1.9 a fixed rule instead of a site
value (only one CIS value); maybe add a DoH site variable next to 4.7 (CIS note) later.

## 9. Environment

| Item | Value |
|------|-------|
| VMs (Proxmox) | rhel8 `192.168.20.50`, rhel9 `192.168.20.30`, rhel10 `192.168.20.40`, "Server with GUI"; SSH user `frqadmin`, key `~/.ssh/id_ed25519_cis` |
| Snapshot | **`pre-chrome`** on all 3; revert before each clean test |
| Control node | ansible-core 2.16 venv `~/.ansible-2.16.1-env` ([control-node-setup](../../docs/control-node-setup.md)); **activate it first** (T-C2); no collections needed |
| Lint | `yamllint . && ansible-lint --offline` inside `chrome_cis/` (uv tools) |
| Benchmark files | `cis-pdf/`, `cis-spreadsheet/` at the repo root; **never commit them** |
| Standup notes | [docs/standup.md](../../docs/standup.md) |
| Container test | Rocky 8/9/10 with podman + `containers.podman` (test only); recipe below |

### Optional: Claude's container test

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
