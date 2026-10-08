# Code walkthrough: chrome_cis

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

Order of the run: prelim → install (opt-in) → policy file → report.

| Key | Does | Why |
|-----|------|-----|
| `import_tasks` (static) | tasks are known when the play starts; tags and `when` apply to every imported task | `--list-tasks` / `--check` show the whole role |
| prelim `tags: always` | checks + Chrome detection run whatever tags the user picks | later steps need its facts (D10) |
| install `when: chrome_cis_install` | install only on request | D9 opt-in |
| report `tags: always` | every run ends with the per-rule status | D8: all 118 rules accounted for |

## defaults/main.yml

Every setting a user may change, grouped: install, profiles, site values, rule toggles.

| Key | Does | Why |
|-----|------|-----|
| `chrome_cis_install: false`, `chrome_cis_channel: stable` | opt-in install; which `google-chrome-*` package | D9, D10 |
| `chrome_cis_level_1: true`, `chrome_cis_level_2: false` | profile switches | D4 |
| site values `null` / `[]` | 11 SITE rules + 2 allowlists: report only until set | D6; `null` because `false` is a real answer |
| `chrome_cis_rule_<id>` (101) | one toggle per applicable rule; 12 risky ones `true` + `# WARNING` (site turns off in `group_vars`) | D5; 4.12 has none (follows 4.1.1) |

## vars/main.yml

Internal values and the rule logic (not meant to be overridden).

| Key | Does | Why |
|-----|------|-----|
| `chrome_cis_repo_baseurl`, `_repo_key_url` | Google's repo and key URLs | D9; one repo for all channels (W-2) |
| `chrome_cis_policy_dir`, `_policy_file` | `/etc/opt/chrome/policies/managed/chrome_cis.json` | D2 |
| `chrome_cis_installed`, `_installed_versions` | `google-chrome-*` packages found by `package_facts` | D10 detect, don't assume |
| `chrome_cis_rules` | all 118 rules + 2 allowlist entries: `value`, `var`, `na`, `toggle`, `companion` | D3, D6, D8 |
| `chrome_cis_rule_status` | per entry: `applied` / `off_level` / `off_toggle` / `site_unset` / `na` | one place decides; policy file and report both use it |
| `chrome_cis_policies` | applied entries as `{policy: value}` | the JSON content (D2) |
| lazy evaluation | recomputed each time it's used | after install refreshes `package_facts`, versions are current without extra tasks |

## tasks/prelim.yml

| Key | Does | Why |
|-----|------|-----|
| version / OS / settings asserts | stop early with a clear message | D12; invalid site values never reach the file |
| `package_facts` | list installed RPMs | D10 |
| block + `meta: end_host` | no Chrome and no install → skip this host, `failed=0` | D10 |

## tasks/install.yml

| Key | Does | Why |
|-----|------|-----|
| `rpm_key` → `yum_repository` → `dnf` | key, repo, package in that order | D9, T-C1 |
| `package_facts` again | refresh versions for the report | simpler than a conditional (ansible-lint `no-handler`) |

## tasks/policy.yml

| Key | Does | Why |
|-----|------|-----|
| `file` state directory | create `managed/` | RPM doesn't (D-2) |
| `copy` + `to_nice_json(sort_keys=true)` | write the policy JSON | declarative, stable order → idempotent; `--check --diff` shows changes (D3) |

## tasks/report.yml

| Key | Does | Why |
|-----|------|-----|
| `find` + `slurp` | read other `*.json` in `managed/` | report, never edit (D2, D7) |
| conflicts `set_fact` | CIS policies with a different value elsewhere | D7; same-source conflicts are unpredictable in Chrome |
| `debug` grouped by status | IDs per status + totals + Chrome version | D8 |
