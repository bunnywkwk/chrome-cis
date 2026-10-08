# Work log: chrome_cis

One entry per working day: what was covered, why, and where the result is. For the full detail follow the links.

## 2026-10-07: planning and discovery start

| # | What we did | Why | Result / where |
|---|-------------|-----|----------------|
| 1 | Read the CIS Google Chrome Benchmark v3.0.0 (PDF + spreadsheet) and wrote a plain-language summary of all 118 rules | understand what CIS asks before designing anything (workflow phase 1); the benchmark is the source of truth | [benchmark-summary.md](benchmark-summary.md): 88 L1 + 30 L2; each rule's policy, value, rationale, user impact |
| 2 | Found that the benchmark is written for **Windows Group Policy** | it has no Linux commands; we must translate each rule to its Chrome policy name, which Chrome on Linux reads from JSON | benchmark-summary §1 (F2, F3) |
| 3 | Decided the target: **Google Chrome on RHEL 8/9/10**; Chromium later | CIS has a Chrome benchmark, none for Chromium | benchmark-summary §4 |
| 4 | Decided Windows-only rules are `na_os` (not implemented, still listed) | the role runs on RHEL only; all 118 CIS IDs stay accounted for | benchmark-summary §4 |
| 5 | Decided levels: both implemented, L1 on / L2 off by default; risky rules get their own toggle `false` | L2 = "limited functionality" (blocks mic, camera, uploads, screen share); risk toggles are separate from levels, as in `mongodb8_cis` | benchmark-summary §4 |
| 6 | Decided install is opt-in; candidate pattern = data-driven (one policy list, one JSON file) | ~100 rules use the same mechanism, so one task per rule would be repetition | benchmark-summary §4 |
| 7 | Listed the rules where the site chooses a value (~15) | same idea as MongoDB's opt-in site variables: report by default, apply only when set | benchmark-summary §5 |
| 8 | Took snapshot `pre-chrome` on the 3 VMs | rollback point for Chrome testing; MongoDB stays as it is | — |
| 9 | Checked the Chrome package on the workstation (read-only) | learn where Chrome reads policies and what its RPM does, to plan the VM checks | [platform-notes W-1](platform-notes.md#w-1-package-facts-from-the-workstation-almalinux-10-google-chrome-stable-15408037857): `/etc/opt/chrome/policies`; RPM adds its own repo + daily cron |
| 10 | **Discovery step 1** (read-only) on rhel8/9/10 | confirm OS, GUI, SELinux, session type, and that nothing Chrome-related is there yet | [platform-notes D-1](platform-notes.md#d-1-step-1-read-only-baseline-2026-10-07), evidence 01–03 in [test-results.md](test-results.md): same baseline on all 3; GNOME's native-messaging dir already exists (affects 2.5.1) |
| 11 | **Discovery step 2**: installed Chrome by hand on rhel8/9/10 | see what an install adds (repo, key, cron, dirs) so the role's opt-in install copies it; check that current Chrome runs on RHEL 8 | [platform-notes D-2](platform-notes.md#d-2-step-2-install-by-hand-2026-10-07): Chrome 155 on all 3, same repo file; RPM does **not** create `policies/managed/`; Google key **not** imported ([T-C1](troubleshooting.md#t-c1-google-chrome-rpm-key-1-import-failed-during-install)) |
| 12 | **Discovery step 3**: test policy file + `chrome://policy` on rhel8/9/10 | prove Chrome on RHEL reads a policy file (the whole role depends on it) and which rules are Windows-only/obsolete | [platform-notes D-3](platform-notes.md#d-3-step-3-policy-read-test--windows-only-check-pending), evidence 04–06: control policy **OK** on all 3 (SELinux fine); all 9 suspected rules **Error** → not implementable |
| 13 | Checked **all 118** rules against Google's official policy list for Chrome 155 | the VM test proved 9 failures but not the reason, and only covered rules I suspected | 16 N/A (5 more than suspected: 2.3.4, 2.3.5, 2.23, 2.29, 3.13); 102 implemented |
| 14 | Wrote the **requirements matrix** | checklist for building and for the compliance test | [cis-requirements.md](cis-requirements.md) |
| 15 | Checked "Chrome Enterprise" and channels | the task mentions enterprise and beta | same RPM from Google's repo (stable/beta/unstable/canary); enterprise = policy-managed ([platform-notes W-2](platform-notes.md#w-2-channels-and-chrome-enterprise-2026-10-07)) |
| 16 | **Full test**: all 101 applicable policies at CIS values on rhel8/9/10 | prove names **and** values before writing code | all **OK** on all 3; only `ProxyMode` labelled Deprecated ([D-4](platform-notes.md#d-4-step-4-full-test-of-all-applicable-policies-2026-10-07)); proof files in [evidence-files/](evidence-files/) |

**Why we check every rule on RHEL by hand:** the benchmark is written for Windows Group Policy (PDF printed p. 8–9, viewer p. 9–10) and only
gives Windows registry checks. CIS itself says "Adjustments/tailoring… will be needed" outside that setup. So each
rule's Linux equivalent (the Chrome policy) must be proven on each RHEL version, not assumed.

| 17 | Wrote the **design decisions** (draft) | every build choice with its reason and evidence, reviewed before code | [design-decisions.md](design-decisions.md): D1–D14 |

| 18 | Renamed the folder to `chrome_cis`; Claude wrote `meta/`, `defaults/`, `vars/` (118-rule list) and the **build guide** | the user types `tasks/` from the guide | [build-guide.md](build-guide.md): full path tested in Rocky 8/9/10 containers: install, 65-policy file, rerun `changed=0`, Level 2, conflicts, check mode |

| 19 | Wrap-up: updated root `CLAUDE.md` (chrome_cis active), copied it to `docs/CLAUDE.md`, wrote the **session handoff** | continue on another machine or in a new session with the same context | [session-handoff.md](session-handoff.md), [CLAUDE.md](CLAUDE.md) |

## 2026-10-08: typed code, first VM runs

| # | What we did | Why | Result / where |
|---|-------------|-----|----------------|
| 20 | The user typed `tasks/` and the 4 computed vars; Claude reviewed and fixed 14 typing errors | the role must match the tested code | lint passes; Rocky 9 container: install + L2 `changed=5`, rerun `changed=0`, 79 policies |
| 21 | First VM run (L2 + site values), then the full benchmark | prove the role on real hosts | `applied 83 ... total 118`; with 4.7 on, every site "can't be reached" ([user-view.md](user-view.md)) |
| 22 | RHEL 8 run without the 2.16 venv failed at `package_facts` | newer ansible-core picks Python 3.12 on RHEL 8 | [troubleshooting.md](troubleshooting.md) T-C2 |
| 23 | **Risky rules now `true` by default** (user decision) | role = full CIS; `group_vars` lists only the site's exceptions | [design-decisions.md](design-decisions.md) D5 revised; defaults give 66 policies |
| 24 | Wrote [user-view.md](user-view.md): what a Chrome user notices, risky rules and their effect | evidence from the browser, not only the file | to fill "seen" on the VMs |

**Next:** RHEL 8/10 runs with the 2.16 venv; `chrome://policy` screenshots; mark user-view rows "seen".
