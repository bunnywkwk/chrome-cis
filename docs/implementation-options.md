# Implementation options: how the rules become the policy file

**What:** the four ways the role could turn the 118 CIS rules + the user's settings into
`/etc/opt/chrome/policies/managed/chrome_cis.json`, explained in simple words, with the one we chose.
**Why:** to understand (and defend) why the role has a rule list in `vars/main.yml` and only short tasks.
Decision record: [design-decisions.md](design-decisions.md) D3.

## The problem every option solves

```
 INPUT 1: the rules (from CIS)           INPUT 2: the user's settings (defaults + group_vars)
 3.12  MetricsReportingEnabled = false   chrome_cis_level_1: true     chrome_cis_rule_3_12: true
 4.4   AudioCaptureAllowed     = false   chrome_cis_level_2: false    chrome_cis_rule_4_4: false
 2.17  ProxyMode = <site value>          chrome_cis_proxy_mode: null
 1.19  N/A (removed from Chrome)
                     \                          /
                      ▼                        ▼
            for EACH rule ask 4 questions, in this order:
            1. Is it N/A?                     → yes: skip   ("na")
            2. Is its level switched on?      → no:  skip   ("off_level")
            3. Is its own toggle on?          → no:  skip   ("off_toggle")
            4. SITE rule without a value?     → yes: skip   ("site_unset")
               otherwise                      →      WRITE  ("applied")
                              │
                              ▼
         OUTPUT: chrome_cis.json  {"MetricsReportingEnabled": false}
                 + a report that says why each other rule was skipped
```

All four options ask the same 4 questions. They differ in **where** the questions are asked and **how many tasks** it
takes.

---

## Option A: Jinja loop in `vars/main.yml` (chosen)

**In simple words:** the role has a "calculator" variable. You hand it the rule list and the settings; it gives back a
list with a status for every rule. The tasks only **use** the answer: one writes the applied ones, one prints them all.

```
vars/main.yml                                   tasks/
┌───────────────────────────────┐
│ chrome_cis_rules (118 rules)  │
│        │ loop + 4 questions   │
│        ▼                      │
│ chrome_cis_rule_status ───────┼──────────► report.yml: print status per rule
│        │ keep "applied"       │
│        ▼                      │
│ chrome_cis_policies ──────────┼──────────► policy.yml: copy → chrome_cis.json
└───────────────────────────────┘
```

```yaml
# vars/main.yml (shortened)
chrome_cis_rule_status: >-
  {%- for r in chrome_cis_rules -%} ... 4 questions ... {%- endfor -%}

# tasks/policy.yml: one task writes everything
- ansible.builtin.copy:
    dest: /etc/opt/chrome/policies/managed/chrome_cis.json
    content: "{{ chrome_cis_policies | to_nice_json(sort_keys=true) }}\n"
```

| Pros | Cons |
|----|----|
| the decision is made in **one place**; the file and the report can never disagree | the logic is in Jinja inside `vars/`, harder to read the first time |
| tasks are short (one `copy`, one `debug`) | you don't type the loop yourself |
| quiet output; `--check --diff` shows exactly which policies change | |

---

## Option B: Ansible `loop` + `set_fact` in a task

**In simple words:** the same 4 questions, but asked by a **task** that runs once per rule. Each round, if the rule
passes, it is added to a growing dictionary. A second task writes the dictionary.

```
tasks/policy.yml
┌───────────────────────────────────────────┐
│ set_fact  (loop: chrome_cis_rules)        │   round 1: 1.1.1 → add
│   when: 4 questions pass                  │   round 2: 1.2.1 → add
│   policies = policies + {policy: value}   │   round 3: 1.2.2 → skip (site value not set)
│                                           │   ... 120 rounds, each printed
├───────────────────────────────────────────┤
│ copy → chrome_cis.json                    │
└───────────────────────────────────────────┘
```

```yaml
- name: "POLICY | Build the policy set"
  when: <the 4 questions as conditions>
  ansible.builtin.set_fact:
    chrome_cis_policies: "{{ chrome_cis_policies | default({}) | combine({item.policy: item.value}) }}"
  loop: "{{ chrome_cis_rules }}"
  loop_control:
    label: "{{ item.id }}"
```

| Pros | Cons |
|----|----|
| the loop is a **visible task** you type; looks like normal Ansible | ~120 lines of output per run (one per rule) |
| easy to follow one rule at a time in the output | the report needs the same 4 questions again (second loop) or a second list, so the logic exists twice |
| | `when:` with 4 conditions on lookups gets long |

---

## Option C: template file (`templates/chrome_cis.json.j2`)

**In simple words:** write the JSON file as a "fill-in-the-blanks" template. The loop and the 4 questions live inside
the template; the `template` module fills it in on the host.

```
templates/chrome_cis.json.j2              tasks/policy.yml
┌──────────────────────────────┐
│ {                            │
│ {% for r in rules %}         │
│ {% if 4 questions pass %}    │ ◄──────── template: src=chrome_cis.json.j2
│   "{{ r.policy }}": ...,     │                     dest=.../chrome_cis.json
│ {% endif %}{% endfor %}      │
│ }                            │
└──────────────────────────────┘
```

| Pros | Cons |
|----|----|
| the familiar "template" pattern (like an nginx.conf.j2) | same Jinja as A, plus an extra file and folder |
| | writing valid JSON by hand in a template is fiddly (commas, quotes, `true` vs `True`) |
| | the report still needs the statuses, so the logic is duplicated |

---

## Option D: one task per rule (the `mongodb8_cis` style)

**In simple words:** like MongoDB, every CIS rule gets its own task with its own `when:` and tags.

```
tasks/section_1/cis_1.1.1.yml  → write AllowCrossOriginAuthPrompt into the file
tasks/section_1/cis_1.2.1.yml  → write SafeBrowsingAllowlistDomains into the file
... ~100 files / tasks, all doing the same thing with a different name and value
```

| Pros | Cons |
|----|----|
| each rule visible on its own, with its own tags (`--skip-tags rule_2.3.3`) | ~100 near-identical tasks: copy-paste code |
| same layout as `mongodb8_cis` | ~100 edits to **one** JSON file: slow, and hard to keep the file valid and idempotent |
| | in MongoDB it was right because each rule used a **different** mechanism (DB query, config key, file permission, restart); here every rule is "policy name = value" |

---

## Side by side

| | A (chosen) | B | C | D |
|---|---|---|---|---|
| Where the 4 questions are asked | `vars/main.yml`, once | a looping task | the template | each task's `when:` |
| Tasks you type for the policies | 1 | 2 | 1 | ~100 |
| Output per run | short | ~120 lines | short | ~100 tasks |
| File and report use the same decision | Yes | No (logic twice) | No (logic twice) | Yes |
| Per-rule tags | No (use toggles) | No | No: | Yes |
| Fits CLAUDE.md "single artifact → rules as data, one task" | Yes | Yes | Yes | No |

**Why A:** one place decides, both the file and the report read that decision, and the tasks stay small enough to type
and defend line by line. It passed the container tests (build-guide "Results").
