# Session handoff: chrome_cis

**Read this first when you continue on another machine or in a new Claude session.** It says where the work stopped,
how we think about it, how Claude should work with you, and what to do next. Last update: **2026-10-08**.

New Claude session: say *"read `chrome_cis/docs/session-handoff.md` and `chrome_cis/docs/CLAUDE.md` first"*.

## 1. Where we are

| Phase ([role-workflow](../../docs/role-workflow.md)) | Status | File |
|-------|--------|------|
| 1. Read the benchmark | Done | [benchmark-summary.md](benchmark-summary.md) |
| 2. Requirements matrix | Done: 118 rules, 102 applicable, 16 N/A, with evidence | [cis-requirements.md](cis-requirements.md) |
| 3. Discovery on RHEL 8/9/10 | Done | [platform-notes.md](platform-notes.md) |
| 4. Design decisions | Done: D1–D14 (D5 revised 2026-10-08: risky rules on) | [design-decisions.md](design-decisions.md) |
| 5. Build, **Option A** | Done on branch `staging`: you typed `tasks/` + computed vars; Claude fixed 14 typing errors; lint passes | [build-guide.md](build-guide.md) |
| 6. Validate | Done: Rocky 9 container, install + L2 `changed=5`, rerun `changed=0` | build-guide "Results" |
| 7–8. Test project + VM runs | In progress (see 3) | `~/chrome_cis_test` |
| Options B, C, D | **Next**: build each on its own branch to compare (see 4) | [implementation-options.md](implementation-options.md) |
| 9–11. Compliance record, evidence, wrap-up | Not started | — |

Daily history: [work-log.md](work-log.md) (items 1–24). Errors: [troubleshooting.md](troubleshooting.md) (T-C1, T-C2).
Every file explained: [code-walkthrough.md](code-walkthrough.md). What a user notices: [user-view.md](user-view.md).

## 2. Git branches (`github.com/bunnywkwk/chrome-cis`)

| Branch | Holds | State |
|--------|-------|-------|
| `main` | older state | behind `staging` |
| `staging` | **Option A**, complete, lint passes; risky rules `true` (D5 revised); `user-view.md` | used by the test project |
| `staging-option-b` | Option B: `vars/` without the Jinja loop, `docs/build-guide-option-b.md`, `tasks/rules.yml` **half typed** | commit 2c3cb74 "option b started" |

**`tasks/rules.yml` on `staging-option-b` has 3 typos to fix** when you continue (compare with the guide):
- `item(item.level == 1 ...` → `(item.level == 1 ...`
- `lookup('ansible/builtin.vars', ...` → `lookup('ansible.builtin.vars', ...`
- task name `" RULES | AUDIT | Start  with no policies"` → no leading space, single space

Then type the rest of task 2 (line 4 of `when:`, `set_fact`, `loop`, `loop_control`), batch B2 (`main.yml`) and B3
(`report.yml`).

## 3. VM test state (2026-10-08)

- **Test project:** `~/chrome_cis_test` (same layout as `mongodb8-cis-test`): `ansible.cfg`, `requirements.yml` (role
  `chrome-cis` from branch **`staging`**), `sysconfig/inventory.yml` (group `rhel-os`: rhel8/9/10),
  `sysconfig/group_vars/rhel-os.yml`, `playbooks/site.yml`.
- **group_vars now:** install on, **channel `canary`**, Level 1 + 2, all 11 site values set (lists with
  `lab.example.com` demo values), `chrome_cis_rule_4_7: false` (exception). Change the channel back to `stable` for
  the compliance runs.
- **Seen so far:** L2 + 5 site values → `applied 83, off 13, site not set 6, N/A 16, total 118`. Full benchmark with
  4.7 on → **every site "can't be reached"** (no DoH server) → 4.7 is now an exception. Messenger login can fail
  with 3.4 (third-party cookies blocked), expected.
- **RHEL 8 failed** at `package_facts` because the venv wasn't active (T-C2). Always:
  `source ~/.ansible-2.16.1-env/bin/activate` and check `ansible --version` = core 2.16.x.
- **Screenshots:** `~/chrome_cis_test/docs/evidence/image.png` and `image copy.png` are the same T-C2 error. Real
  evidence goes to `chrome_cis/docs/evidence-images/` named `NN-<os>-<scenario>-<result>.png` (next free: 07).
- **To do:** each VM from snapshot `pre-chrome` → run → rerun `changed=0` → `chrome://policy` and
  `chrome://management` screenshots → try the rows of [user-view.md](user-view.md) and mark them "seen".

## 4. The 4 options plan (what you continue at home)

All four turn the same rule list into the same `chrome_cis.json`; they differ in **where the 4 questions are asked**
(N/A? level on? toggle on? site value set?). See [implementation-options.md](implementation-options.md).

| Option | Where the questions are | Branch | State |
|--------|-------------------------|--------|-------|
| A | Jinja loop in `vars/main.yml` | `staging` | **done**, tested |
| B | `when:` list on a looping `set_fact` task | `staging-option-b` | guide done, you're typing |
| C | inside a template `templates/chrome_cis.json.j2` | create `staging-option-c` from `staging` | ask Claude for `build-guide-option-c.md` |
| D | one task per rule (MongoDB style), each with `when:` and tags | create `staging-option-d` from `staging` | ask Claude for `build-guide-option-d.md` |

**How each option is proven (same test for all):** the option must produce **exactly the same `chrome_cis_policies` /
JSON as Option A**. Claude's quick check: a localhost playbook that loads `defaults/` + Option A's `vars/`, keeps A's
result, runs the new option's tasks, and `assert`s both are equal, in 4 scenarios (defaults: 66 policies; Level 2: 90;
L2 + site values + 4.7 off + extension allowlist: 94; 2.3.3 off + allowlist: 65). Only `set_fact`/`assert`, nothing
installed. Then a real run: rerun on a host that Option A already hardened must give `changed=0` (same file).

**Known trade-offs to write down for the comparison:** B loses the "why off" detail in the report (applied / not
applied / N/A only) and prints 120 loop lines; C needs careful JSON (commas, `true`/`True`), use `to_json` per value;
D is ~100 near-identical tasks editing one JSON file (hard to keep idempotent; maybe one `copy` at the end).

## 5. The mindset (why the role looks like this)

| Principle | In this role |
|-----------|--------------|
| **The benchmark is the source of truth** | only the 118 CIS rules; never extra Chrome policies; CIS IDs never renumbered |
| **Evidence, not assumption** | every "works on RHEL" / "N/A" claim is backed by Google's policy template for Chrome 155 (Ctrl+F search text per rule) **and** a VM test. Don't add new sources mid-way |
| **The benchmark is written for Windows** | PDF printed p. 8–9 (viewer p. 9–10): Group Policy / registry. On RHEL the same **policy names** go into a JSON file in `/etc/opt/chrome/policies/managed/` |
| **Data, not 100 tasks** | all rules use one mechanism, so they're a list in `vars/main.yml`; one `copy` writes the JSON (D3) |
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

## 7. Reading `defaults/main.yml` and `vars/main.yml` (branch `staging`, Option A)

**`defaults/main.yml` = what a user may change** (set in `group_vars`):

| Group | Variables | Default |
|-------|-----------|---------|
| Install | `chrome_cis_install`, `chrome_cis_channel` (stable, beta, unstable, canary) | `false`, `stable` |
| Profiles | `chrome_cis_level_1`, `chrome_cis_level_2` | `true`, `false` |
| Site values (11 SITE rules + 2 allowlists) | e.g. `chrome_cis_proxy_mode`, `chrome_cis_http_allowlist` | `null` / `[]` = report only |
| Rule toggles | `chrome_cis_rule_<id>` (101) | all `true`; 12 risky ones carry a `# WARNING:` |

**`vars/main.yml` = internal, don't override:**

| Variable | Is |
|----------|----|
| `chrome_cis_rules` | the 118 rules (+2 allowlist entries): `{id, level, policy, value}`; `var:` = site variable; `na:` = not applicable + reason; `toggle:` = use another rule's toggle (4.12 → 4.1.1); `companion:` = allowlist next to a blocklist |
| `chrome_cis_installed` | names of installed `google-chrome-*` packages (logic: "is Chrome here?") |
| `chrome_cis_installed_versions` | same with versions (report only) |
| `chrome_cis_rule_status` (A only) | per entry: `applied` / `off_level` / `off_toggle` / `site_unset` / `na` |
| `chrome_cis_policies` (A: var, B: fact) | the applied entries as `{policy: value}` = the JSON content |

With defaults: **66** policies, report `applied 66, off 27, site not set 9, N/A 16, total 118`.

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
