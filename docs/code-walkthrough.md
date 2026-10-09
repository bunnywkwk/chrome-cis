# Code walkthrough: chrome_cis

**Revised 2026-10-09:** the role is being rebuilt in the Lockdown layout (one task per rule, `tasks/section_<n>/`,
D3 revised). Sections below for files that change are rewritten as each section is built.

One section per file: purpose in one line, then `Key | Does | Why`. Decisions (Dn) are in
[design-decisions.md](design-decisions.md) and are linked, not repeated.

## meta/main.yml

Galaxy metadata: who wrote the role, what it supports.

| Key | Does | Why |
|-----|------|-----|
| `role_name: chrome_cis` | role name used by Galaxy / `requirements.yml` | D14 (no hyphen) |
| `min_ansible_version: "2.16.1"` | refuses older ansible-core | D12; same control node as `mongodb8_cis` |
| `platforms: EL 8, 9, 10` | supported OS | D1, D12 |
| no `collections:` | only `ansible.builtin` modules are used | D12 |

## .ansible-lint, .yamllint, .gitignore

Lint rules and files git must ignore (same as `mongodb8_cis`).

| Key | Does | Why |
|-----|------|-----|
| `profile: production` | strictest ansible-lint profile | CLAUDE.md validation |
| `truthy: ["true", "false"]` | only real YAML booleans | CLAUDE.md: switches are unquoted booleans |
| `.gitignore`: `*.pdf`, `*.xlsx`, `cis-pdf/`, `cis-spreadsheet/` | CIS files never get committed | CIS terms forbid hosting them (lesson from `mongodb8_cis`) |

## tasks/main.yml

Orchestration, Lockdown order: prelim → install → section 1–5 → post.

| Key | Does | Why |
|-----|------|-----|
| `import_tasks` (static) | tasks known when the play starts; `when`/tags apply to every imported task | `--list-tasks`, `--check`, `--tags rule_x` see every rule |
| prelim, post `tags: always` | checks, policy-set start, file write and report run whatever tags are chosen | a `--tags rule_x` run still writes the file and reports (D3 revised) |
| install `when: chrome_cis_install` | install only on request | D9 opt-in |
| `when: chrome_cis_section<n>` | switch a whole CIS section off in `group_vars` | same as `mongodb8_cis` (variables, no section tags) |

## defaults/main.yml

Every setting a user may change, grouped: install, profiles, site values, rule toggles.

| Key | Does | Why |
|-----|------|-----|
| `chrome_cis_install: false`, `chrome_cis_channel: stable` | opt-in install; which `google-chrome-*` package | D9, D10 |
| `chrome_cis_level_1: true`, `chrome_cis_level_2: false` | profile switches | D4 |
| `chrome_cis_section1..5: true` | skip a whole CIS section | Lockdown section switches (D3 revised) |
| site values `null` / `[]` | 11 SITE rules + 2 allowlists: report only until set | D6; `null` because `false` is a real answer |
| `chrome_cis_rule_<id>` (101) | one toggle per applicable rule; 12 risky ones `true` + `# WARNING` (site turns off in `group_vars`) | D5; 4.12 has none (follows 4.1.1) |

## vars/main.yml

Internal values only; no rule list (D3 revised).

| Key | Does | Why |
|-----|------|-----|
| `chrome_cis_repo_baseurl`, `_repo_key_url` | Google's repo and key URLs | D9; one repo for all channels (W-2) |
| `chrome_cis_policy_dir`, `_policy_file` | `/etc/opt/chrome/policies/managed/chrome_cis.json` | D2 |
| `chrome_cis_installed` | `{name: version}` of installed `google-chrome-*` packages; `{}` = not installed | D10; prelim checks its length, report prints it |
| `chrome_cis_not_applicable` | the 16 N/A rules with reason | no task; report accounts for all 118 rules (D8) |

## tasks/prelim.yml

| Key | Does | Why |
|-----|------|-----|
| version / OS asserts | stop early with a clear message on an unsupported control node or target | D12 (the role-settings check was removed 2026-10-09: user decision) |
| `package_facts` | list installed RPMs | D10 |
| block + `meta: end_host` | no Chrome and no install → skip this host, `failed=0` | D10 |
| `slurp` current `chrome_cis.json`, `failed_when: false` | reads the file if it exists | start point for `--tags` / `--skip-tags` runs |
| `Start the policy set` | full run (`ansible_run_tags == ['all']`, no skip tags): `{}`; otherwise the current file | full run = file matches the settings (rule off is removed); limited run changes only the selected rules (D3 revised) |

## tasks/install.yml

| Key | Does | Why |
|-----|------|-----|
| `rpm_key` → `yum_repository` → `dnf` | key, repo, package in that order | D9, T-C1 |
| `package_facts` again | refresh versions for the report | simpler than a conditional (ansible-lint `no-handler`) |

## tasks/section_<n>/main.yml, cis_<n>.x.yml

One task per CIS rule, benchmark order, grouped by subsection (`cis_2.3.x.yml` = 2.3 Extensions; `cis_2.x.yml` = the
rules without a subsection). 104 tasks: 102 applicable rules (4.12 has its own task) + 2 allowlist steps.

| Key | Does | Why |
|-----|------|-----|
| `name: "<ID> \| PATCH \| <CIS title>"` | exact CIS title from the benchmark | CLAUDE.md task template |
| `when: chrome_cis_level_<n>`, `chrome_cis_rule_<id>` | level and rule toggle | D4, D5 |
| site rules: `when: <site var> is not none` / `\| length > 0` | applies only when the site set a value | D6 |
| `tags: level<n>, automated/manual, patch, rule_<id>, <topic>` | select or skip by level, rule or topic | Lockdown tags |
| `set_fact: chrome_cis_policies \| combine({'<Policy>': <CIS value>})` | adds this rule's policy | the file is written once in post (Lockdown auditd pattern) |
| 4.12 `when: chrome_cis_rule_4_1_1` | same policy as 4.1.1 | one switch for one policy |

## tasks/post.yml

Writes the file and reports.

| Key | Does | Why |
|-----|------|-----|
| `file` managed/ 0755 | creates the folder (the RPM doesn't) | D-2 |
| `copy` `content: chrome_cis_policies \| to_nice_json(sort_keys=true)` | one file; unchanged content = no change | idempotent, `--check --diff` (D2) |
| report | benchmark, Chrome version, file, number of policies written, levels, the 16 N/A rules | plain facts; the site's exceptions are in its `group_vars` (D8) |
