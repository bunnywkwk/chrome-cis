# Build guide: `chrome_cis`, batch by batch

Every task file of the role, in the order to type it, with a test after each batch. The code here is **copied from
files that passed** `yamllint` + `ansible-lint` (profile `production`) and ran in containers (Rocky Linux 8.10, 9.8,
10.2, ansible-core 2.16.19): results at the end. Same approach as the `mongodb8_cis` rebuild guide.

**Already written by Claude (don't retype):** `meta/main.yml`, `defaults/main.yml`, `.yamllint`, `.ansible-lint`,
`.gitignore`, and the **simple** part of `vars/main.yml` (settings, URLs, paths, the 118-rule list). **You type:** the
five files under `tasks/` and the **4 computed variables** in `vars/main.yml` (batches 2b and 4a).
The complete `vars/main.yml` (as tested) is kept in [reference/vars-main-complete.yml](reference/vars-main-complete.yml)
to compare against.

**After every batch** (inside the role folder):
```bash
cd ~/ansible-cis/chrome_cis && yamllint . && ansible-lint --offline
```
Linting complains about missing files until batch 5 is done (`main.yml` imports all four); that's expected.

| Batch | File | What it adds | Decisions |
|-------|------|--------------|-----------|
| 1 | `tasks/main.yml` | running order | — |
| 2 | `tasks/prelim.yml` | checks, Chrome detection, clean skip | D10, D12 |
| 2b | `vars/main.yml`: `chrome_cis_installed`, `_installed_versions` | which `google-chrome-*` packages are installed | D10 |
| 3 | `tasks/install.yml` | opt-in install: key → repo → package | D9 |
| 4a | `vars/main.yml`: `chrome_cis_rule_status`, `chrome_cis_policies` | the 4 questions per rule → the JSON content | D3–D8 |
| 4 | `tasks/policy.yml` | writes `chrome_cis.json` | D2–D7 |
| 5 | `tasks/report.yml` | per-rule status + conflicts in other files | D7, D8 |

## How the role decides what to write (read once before batch 4)

The logic lives in `vars/main.yml` (you type it in batches 2b and 4a), so the tasks stay short:

```
chrome_cis_rules (118 rules + 2 allowlists, vars/main.yml)
        │  for each entry:
        │    na defined?                         → status na          (never written, D8)
        │    its level switched off?             → status off_level   (D4)
        │    chrome_cis_rule_<id> false?         → status off_toggle  (D5; 4.12 uses 4.1.1's toggle)
        │    site rule with no value (null / [])? → status site_unset  (D6)
        │    otherwise                           → status applied
        ▼
chrome_cis_rule_status   list of {id, policy, value, status, label}
        ▼  keep "applied", turn into {policy: value}
chrome_cis_policies      → written as JSON by batch 4, printed by batch 5
```
These variables are **lazy**: Ansible works them out each time they're used, so after the install step refreshes the
package list, `chrome_cis_installed` is up to date without any extra task.

---

## Batch 1: `tasks/main.yml`

```yaml
---
- name: PRELIM | Checks and Chrome detection
  ansible.builtin.import_tasks:
    file: prelim.yml
  tags:
    - always

- name: INSTALL | Google Chrome (opt-in)
  when: chrome_cis_install
  ansible.builtin.import_tasks:
    file: install.yml
  tags:
    - install

- name: POLICY | Write the CIS managed policy file
  ansible.builtin.import_tasks:
    file: policy.yml
  tags:
    - policy

- name: REPORT | Per-rule status
  ansible.builtin.import_tasks:
    file: report.yml
  tags:
    - always
```

| Line | Does | Why |
|------|------|-----|
| `import_tasks` | pulls in the file when the play starts (static) | `when:` and `tags:` apply to every imported task; `--list-tasks` shows the whole role |
| prelim `tags: always` | checks and detection run whatever tags are picked | later steps need them |
| install `when: chrome_cis_install` | only on request | D9 opt-in |
| report `tags: always` | every run ends with the per-rule status | D8: all 118 rules accounted for |

Rules are data, not tasks, so there are no per-rule tags (`rule_2.3.3`). Switch rules with variables
(`chrome_cis_rule_2_3_3: false`); tags are `install`, `policy` (and `always`).

---

## Batch 2: `tasks/prelim.yml`

```yaml
---
- name: "PRELIM | AUDIT | Check ansible-core version"
  ansible.builtin.assert:
    that:
      - ansible_version.full is version('2.16.1', '>=')
    fail_msg: "chrome_cis needs ansible-core >= 2.16.1 (found {{ ansible_version.full }})."
    quiet: true

- name: "PRELIM | AUDIT | Check the target OS is supported"
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] == 'RedHat'
      - ansible_facts['distribution_major_version'] in chrome_cis_supported_os_majors
      - ansible_facts['architecture'] == chrome_cis_supported_arch
    fail_msg: >-
      Unsupported target: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}
      {{ ansible_facts['architecture'] }}. Supported: RHEL-family {{ chrome_cis_supported_os_majors | join('/') }},
      {{ chrome_cis_supported_arch }}.
    quiet: true

- name: "PRELIM | AUDIT | Check the role settings"
  ansible.builtin.assert:
    that:
      - chrome_cis_channel in chrome_cis_channels
      - chrome_cis_safe_browsing_protection_level in [none, 1, 2]
      - chrome_cis_variations in [none, 0]
      - chrome_cis_proxy_mode in [none, 'direct', 'system']
    fail_msg: >-
      Check chrome_cis_channel ({{ chrome_cis_channels | join('/') }}),
      chrome_cis_safe_browsing_protection_level (1 or 2), chrome_cis_variations (0)
      and chrome_cis_proxy_mode ("direct" or "system").
    quiet: true

- name: "PRELIM | AUDIT | Gather installed packages"
  ansible.builtin.package_facts:
    manager: rpm

- name: "PRELIM | AUDIT | Stop when Chrome is not installed and install is off"
  when:
    - chrome_cis_installed | length == 0
    - not chrome_cis_install
  block:
    - name: "PRELIM | AUDIT | Stop when Chrome is not installed and install is off | Report"
      ansible.builtin.debug:
        msg: >-
          No google-chrome-* package on {{ inventory_hostname }} and chrome_cis_install is false:
          nothing to harden.

    - name: "PRELIM | AUDIT | Stop when Chrome is not installed and install is off | End this host"
      ansible.builtin.meta: end_host
```

| Task | Does | Why |
|------|------|-----|
| Check ansible-core version | stop if < 2.16.1 | D12, same floor as `meta/main.yml` |
| Check the target OS | RedHat family, major 8/9/10, x86_64 | D12: Google's repo is x86_64 only |
| Check the role settings | channel is known; the 3 site values with fixed choices are valid (`none` = not set) | catch typos before anything changes; 2.17 must never be `auto_detect` (D6) |
| Gather installed packages | `package_facts` → `ansible_facts['packages']` | detect, don't assume; feeds `chrome_cis_installed` (vars) |
| Stop when not installed and install off | message + `meta: end_host` | clean skip for this host only, `failed=0` (D10) |

**Test:** VM at `pre-chrome` (no Chrome), defaults → *"No google-chrome-\* package … nothing to harden"*, `failed=0`.

---

## Batch 2b: `vars/main.yml`, installed packages

Type this where the file says `# >>> You type here: chrome_cis_installed …`. Prelim (batch 2) needs it to decide
"installed or not", so do it before testing batch 2.

```yaml
# Installed google-chrome-* packages, e.g. ["google-chrome-stable 155.0.8059.39"] (D10)
chrome_cis_installed: >-
  {{ ansible_facts['packages'] | default({}) | dict2items | selectattr('key', 'match', '^google-chrome-')
     | map(attribute='value') | map('first') | map(attribute='name') | list }}
chrome_cis_installed_versions: >-
  {%- set out = [] -%}
  {%- for name in chrome_cis_installed -%}
  {%-   set _ = out.append(name ~ ' ' ~ ansible_facts['packages'][name][0]['version']) -%}
  {%- endfor -%}
  {{ out }}
```

**`chrome_cis_installed`: one filter chain, read left to right**

| Step | Turns | Into (example) |
|------|-------|----------------|
| `ansible_facts['packages']` | all RPMs from `package_facts` | `{'bash': [{…}], 'google-chrome-stable': [{'name': 'google-chrome-stable', 'version': '155.0.8059.39', …}], …}` |
| `\| default({})` | nothing yet (facts not gathered) | `{}` instead of an error |
| `\| dict2items` | the dict | `[{key: 'bash', value: [...]}, {key: 'google-chrome-stable', value: [...]}, …]` |
| `\| selectattr('key', 'match', '^google-chrome-')` | keep items whose key **starts with** `google-chrome-` | only Chrome packages (any channel) |
| `\| map(attribute='value')` | take the `value` of each | `[[{name: …, version: …}]]` (a list per package: RPM can hold several versions) |
| `\| map('first')` | first entry of each list | `[{name: 'google-chrome-stable', version: '155…'}]` |
| `\| map(attribute='name') \| list` | the names | `['google-chrome-stable']` |

Python: `[pkgs[0]['name'] for key, pkgs in facts['packages'].items() if key.startswith('google-chrome-')]`

**`chrome_cis_installed_versions`: a small Jinja loop**

| Line | Does |
|------|------|
| `{%- set out = [] -%}` | empty list (Python `out = []`) |
| `{%- for name in chrome_cis_installed -%}` | for each installed Chrome package |
| `{%- set _ = out.append(name ~ ' ' ~ …['version']) -%}` | add `"google-chrome-stable 155.0.8059.39"`; `set _ =` is how Jinja calls a method without printing |
| `{{ out }}` | the result |

`>-` folds the lines into one value; the `-` in `{%-`/`-%}` removes the spaces between them, so the result is a clean list.

**Test:** with batch 2, a VM without Chrome still prints the skip message (`chrome_cis_installed` = `[]`).

---

## Batch 3: `tasks/install.yml`

```yaml
---
- name: "INSTALL | Import Google's package signing key"
  ansible.builtin.rpm_key:
    key: "{{ chrome_cis_repo_key_url }}"
    state: present

- name: "INSTALL | Add the Google Chrome repository"
  ansible.builtin.yum_repository:
    name: google-chrome
    file: google-chrome
    description: google-chrome
    baseurl: "{{ chrome_cis_repo_baseurl }}"
    enabled: true
    gpgcheck: true
    gpgkey: "{{ chrome_cis_repo_key_url }}"

- name: "INSTALL | Install google-chrome-{{ chrome_cis_channel }}"
  ansible.builtin.dnf:
    name: "google-chrome-{{ chrome_cis_channel }}"
    state: present

- name: "INSTALL | Refresh installed packages"
  ansible.builtin.package_facts:
    manager: rpm
```

| Task | Does | Why |
|------|------|-----|
| Import the signing key | `rpm_key` from Google's URL | key **first**, so dnf verifies the package; the RPM's own import fails under dnf (T-C1) |
| Add the repository | `yum_repository` → `/etc/yum.repos.d/google-chrome.repo` (`file:`), section `[google-chrome]` (`name:`) | same file name and section as Google's, so there's one repo definition; Google's daily cron leaves it alone (it only recreates a missing file, D9 Revised) |
| Install `google-chrome-<channel>` | `dnf` | `chrome_cis_channel` picks the package (D10) |
| Refresh installed packages | `package_facts` again | the report shows the version just installed |

**Test:** VM at `pre-chrome`, `chrome_cis_install: true` → Chrome installed (`google-chrome --version`),
`rpm -q gpg-pubkey --qf '%{SUMMARY}\n' | grep -i google` shows Google's key (fixes T-C1).

---

## Batch 4a: `vars/main.yml`, rule status and policy set

Type this at the end of the file, where it says `# >>> You type here: chrome_cis_rule_status …`. This is the "4
questions per rule" from the top of this guide ([implementation-options.md](implementation-options.md), option A).

```yaml
# Status of every entry for this run: applied | off_level | off_toggle | site_unset | na (D4-D8)
chrome_cis_rule_status: >-
  {%- set out = [] -%}
  {%- for r in chrome_cis_rules -%}
  {%-   if r.na is defined -%}
  {%-     set st = 'na' -%}
  {%-     set val = none -%}
  {%-   else -%}
  {%-     set level_on = (r.level == 1 and chrome_cis_level_1) or (r.level == 2 and chrome_cis_level_2) -%}
  {%-     set toggle_on = lookup('ansible.builtin.vars', 'chrome_cis_rule_' ~ ((r.toggle | default(r.id)) | replace('.', '_'))) -%}
  {%-     set val = lookup('ansible.builtin.vars', r.var) if r.var is defined else r.value -%}
  {%-     if not level_on -%}
  {%-       set st = 'off_level' -%}
  {%-     elif not toggle_on -%}
  {%-       set st = 'off_toggle' -%}
  {%-     elif r.var is defined and (val is none or (val is not string and val is iterable and val | length == 0)) -%}
  {%-       set st = 'site_unset' -%}
  {%-     else -%}
  {%-       set st = 'applied' -%}
  {%-     endif -%}
  {%-   endif -%}
  {%-   set _ = out.append({'id': r.id, 'policy': r.policy | default(''), 'value': val, 'status': st,
                           'companion': r.companion | default(false),
                           'label': r.id ~ ' ' ~ (r.policy | default(r.na | default('')))}) -%}
  {%- endfor -%}
  {{ out }}

# The JSON content: policy name -> value for every applied entry (4.1.1 and 4.12 give the same key)
chrome_cis_policies: >-
  {{ chrome_cis_rule_status | selectattr('status', 'eq', 'applied')
     | items2dict(key_name='policy', value_name='value') }}
```

**`chrome_cis_rule_status`, line by line**

| Line | Does | Python |
|------|------|--------|
| `set out = []` | result list | `out = []` |
| `for r in chrome_cis_rules` | each of the 120 entries | `for r in rules:` |
| `if r.na is defined` → `st = 'na'` | question 1: N/A | `if 'na' in r:` |
| `set level_on = (r.level == 1 and chrome_cis_level_1) or (r.level == 2 and chrome_cis_level_2)` | is this rule's level switched on? | |
| `set toggle_on = lookup('ansible.builtin.vars', 'chrome_cis_rule_' ~ (…))` | read the toggle **by a name built from the id**: `'2.3.3'` → `chrome_cis_rule_2_3_3`; `r.toggle` lets 4.12 use 4.1.1's | `settings['chrome_cis_rule_' + id.replace('.', '_')]` |
| `set val = lookup('ansible.builtin.vars', r.var) if r.var is defined else r.value` | SITE rule → the user's value; otherwise the CIS value | |
| `if not level_on` → `off_level` | question 2 | |
| `elif not toggle_on` → `off_toggle` | question 3 | |
| `elif r.var is defined and (val is none or (… length == 0))` → `site_unset` | question 4: SITE value is `null` or an empty list. `val is not string` is there because a string is also "iterable" | |
| `else` → `applied` | passes all 4 | |
| `out.append({…, 'label': r.id ~ ' ' ~ …})` | one dict per entry; `label` = `"3.12 MetricsReportingEnabled"` or `"1.19 not in Chrome 155 policy list…"` for the report | `out.append({...})` |

**`chrome_cis_policies`**

| Step | Does |
|------|------|
| `selectattr('status', 'eq', 'applied')` | keep only rules that passed |
| `items2dict(key_name='policy', value_name='value')` | list of dicts → one dict `{policy: value}` = the JSON content. 4.1.1 and 4.12 give the same key, so it appears once |

Python: `{s['policy']: s['value'] for s in status if s['status'] == 'applied'}`

**Quick test without a VM** (from the role folder, prints the counts):
```bash
cat > /tmp/count.yml <<'EOF2'
- hosts: localhost
  gather_facts: false
  vars_files: [defaults/main.yml, vars/main.yml]
  tasks:
    - ansible.builtin.debug:
        msg: "{{ chrome_cis_rule_status | map(attribute='status') | list | community.general.counter }} policies={{ chrome_cis_policies | length }}"
EOF2
ansible-playbook -i localhost, -c local /tmp/count.yml
```
Expected with defaults: `{'applied': 65, 'site_unset': 9, 'off_level': 28, 'na': 16, 'off_toggle': 2} policies=65`
(120 entries: the 2 allowlists are counted here; the report leaves them out → 118). Compare your file with the reference if not.

---

## Batch 4: `tasks/policy.yml`

```yaml
---
- name: "POLICY | PATCH | Create the managed policy folder"
  ansible.builtin.file:
    path: "{{ chrome_cis_policy_dir }}"
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: "POLICY | PATCH | Write {{ chrome_cis_policy_file }}"
  ansible.builtin.copy:
    dest: "{{ chrome_cis_policy_dir }}/{{ chrome_cis_policy_file }}"
    content: "{{ chrome_cis_policies | to_nice_json(sort_keys=true) }}\n"
    owner: root
    group: root
    mode: "0644"
```

| Task | Does | Why |
|------|------|-----|
| Create the managed policy folder | `/etc/opt/chrome/policies/managed`, root `0755` | the RPM doesn't create it (D-2) |
| Write `chrome_cis.json` | `copy` with `content:` = `chrome_cis_policies` as JSON | one file the role owns (D2); `copy` only rewrites on drift, and `--check --diff` shows which policies would change (D3); `sort_keys` keeps the order stable, so a rerun is `changed=0` |

No handler, no restart: Chrome picks the file up at its next start (D11).

**Test:** `sudo cat /etc/opt/chrome/policies/managed/chrome_cis.json`; in the desktop, `chrome://policy` →
*Reload policies* → every row **OK**. Rerun → `changed=0`.

---

## Batch 5: `tasks/report.yml`

```yaml
---
- name: "REPORT | AUDIT | Find other policy files"
  ansible.builtin.find:
    paths: "{{ chrome_cis_policy_dir }}"
    patterns: "*.json"
    excludes: "{{ chrome_cis_policy_file }}"
  register: discovered_other_policy_files

- name: "REPORT | AUDIT | Read other policy files"
  ansible.builtin.slurp:
    src: "{{ item.path }}"
  loop: "{{ discovered_other_policy_files.files }}"
  loop_control:
    label: "{{ item.path }}"
  register: discovered_other_policy_raw

- name: "REPORT | AUDIT | Find CIS policies set differently in other files"
  ansible.builtin.set_fact:
    discovered_policy_conflicts: >-
      {%- set out = [] -%}
      {%- for f in discovered_other_policy_raw.results -%}
      {%-   set other = f.content | b64decode | from_json -%}
      {%-   for name, value in other.items() if name in chrome_cis_policies and value != chrome_cis_policies[name] -%}
      {%-     set _ = out.append(f.item.path ~ ': ' ~ name ~ ' = ' ~ (value | to_json)
                                 ~ ' (CIS: ' ~ (chrome_cis_policies[name] | to_json) ~ ')') -%}
      {%-   endfor -%}
      {%- endfor -%}
      {{ out }}

- name: "REPORT | AUDIT | Per-rule status"
  vars:
    rules: "{{ chrome_cis_rule_status | rejectattr('companion') }}"
  ansible.builtin.debug:
    msg:
      benchmark: "{{ chrome_cis_benchmark }}"
      chrome: "{{ chrome_cis_installed_versions }}"
      policy_file: "{{ chrome_cis_policy_dir }}/{{ chrome_cis_policy_file }}"
      applied: "{{ rules | selectattr('status', 'eq', 'applied') | map(attribute='label') | list }}"
      off_by_level: "{{ rules | selectattr('status', 'eq', 'off_level') | map(attribute='id') | list }}"
      off_by_toggle: "{{ rules | selectattr('status', 'eq', 'off_toggle') | map(attribute='id') | list }}"
      site_value_not_set: "{{ rules | selectattr('status', 'eq', 'site_unset') | map(attribute='label') | list }}"
      not_applicable: "{{ rules | selectattr('status', 'eq', 'na') | map(attribute='label') | list }}"
      conflicts_in_other_files: "{{ discovered_policy_conflicts }}"
      totals: >-
        applied {{ rules | selectattr('status', 'eq', 'applied') | list | length }},
        off {{ rules | selectattr('status', 'in', ['off_level', 'off_toggle']) | list | length }},
        site not set {{ rules | selectattr('status', 'eq', 'site_unset') | list | length }},
        N/A {{ rules | selectattr('status', 'eq', 'na') | list | length }},
        total {{ rules | list | length }}
```

| Task | Does | Why |
|------|------|-----|
| Find other policy files | every `*.json` in `managed/` except ours | the role never edits other teams' files (D2), it reports |
| Read other policy files | `slurp` each one | read-only |
| Find CIS policies set differently | for each other file, any policy we enforce with a **different** value | catches e.g. `HSTSPolicyBypassList: ["intranet"]` (D7) or `DownloadRestrictions: 0`; Chrome's result with two values from the same source is unpredictable |
| Per-rule status | groups rule IDs by status, plus Chrome version, conflicts and totals | every run accounts for all 118 rules (D8); `rejectattr('companion')` leaves out the two allowlists so counts are per CIS rule |

**Test:** put a file with a conflicting value next to ours and rerun:
```bash
echo '{"DownloadRestrictions": 0}' | sudo tee /etc/opt/chrome/policies/managed/zz-test.json
```
→ `conflicts_in_other_files` lists it. Remove it afterwards (`sudo rm …/zz-test.json`).

---

## Results of the container test (Claude, before handing over)

Containers: Rocky Linux 8.10, 9.8, 10.2 (podman), ansible-core 2.16.19, Chrome 155.0.8059.39 installed by the role.

| # | Test | Result |
|---|------|--------|
| L1–L3 | `yamllint .`, `ansible-lint --offline` (production), `--syntax-check` | Pass: pass (one finding fixed first: `no-handler` on "Refresh installed packages" → now runs every time) |
| T1 | no Chrome, defaults | Pass: "No google-chrome-* package … nothing to harden", `changed=0 failed=0` on all 3 |
| T2 | `chrome_cis_install: true`, defaults | Pass: `changed=5 failed=0` on all 3: Google key imported (`rpm -q gpg-pubkey` shows *Google Inc. (Linux Packages Signing Authority)*, fixes T-C1), repo added, `google-chrome-stable 155.0.8059.39` installed, `chrome_cis.json` with **65** policies; report `applied 65, off 28, site not set 9, N/A 16, total 118` |
| T3 | rerun of T2 | Pass: `changed=0` on all 3 |
| T4 | Level 2 + 4.4 on + 2.3.3 on with an extension allowlist + password manager `false` + proxy `system` | Pass: `changed=1` (the JSON), report `applied 82, off 11, site not set 9, N/A 16, total 118`; rerun `changed=0` |
| T5 | other file `zz-test.json` with `DownloadRestrictions: 0` and `HSTSPolicyBypassList: ["intranet"]` | Pass: `conflicts_in_other_files` lists both, with the CIS value next to each |
| T6 | `--check --diff` on a compliant host | Pass: `changed=0` on all 3 |

**Not covered by containers (no desktop):** `chrome://policy` showing the role's file OK. D-4 already proved all 101
policies OK with the same values on the VMs; the VM run confirms it for the role's own file.


Containers prove the Ansible side (install, file content, idempotence, check mode, report). They have no desktop, so
`chrome://policy` is checked on the VMs (D-4 already proved all 101 policies OK with the same values).
