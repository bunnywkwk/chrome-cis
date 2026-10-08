# What a Chrome user notices after hardening

**What:** things to try in Chrome on a hardened host, and what you should see, by CIS rule.
**Why:** proves the policies work in the browser, not only that the file exists. Good screenshots for the test record.
**Evidence:** expected results come from Google's policy descriptions (policy template, Chrome 155); mark each one
"seen" in [test-results.md](test-results.md) once checked on a VM.

L2 = only with `chrome_cis_level_2: true`. "Full" = a risky rule: on by default (D5), unless `group_vars` turns it off.

## Risky rules: on by default, turn off in `group_vars`

The role applies full CIS. These rules break things users rely on; a site turns off the ones it can't accept, e.g.
`chrome_cis_rule_4_7: false`, and lists them as exceptions in its compliance record.

| Rule | Level | What breaks when on | Evidence |
|------|-------|---------------------|----------|
| 2.2.3 WebUSB blocked | L2 | USB devices / some security keys in web apps | expected (CIS Impact) |
| 2.3.3 Extensions blocklist `*` | **L1** | **every extension removed** unless in `chrome_cis_extension_allowlist` | expected |
| 2.4.1 Auth schemes ntlm,negotiate | L2 | sites with Basic/Digest login | expected |
| 2.5.1 Native messaging blocklist `*` | L2 | extensions talking to local apps (password managers) | expected |
| 2.12 No click-through on cert errors | L2 | sites with self-signed or expired certs | expected |
| 2.18 Online revocation for local CAs | L2 | internal-CA sites when the OCSP/CRL server is down | expected |
| 3.1.1 Session-only cookies | L2 | logged out of every site when Chrome closes | expected |
| 4.1.1 (+ 4.12) No screen capture | L2 | screen sharing in web meetings | expected |
| 4.3 No file dialogs | L2 | file upload and "Save as" | expected |
| 4.4 / 4.5 No mic / camera | L2 | web calls lose audio / video | expected |
| 4.7 DoH secure only | L2 | **no site loads** without a DoH server | **seen 2026-10-08** (see below) |

"Expected" = from the CIS Impact section and Google's policy description; change to "seen" with a screenshot once
checked on a VM.

## First look

| Where | You see |
|-------|---------|
| Chrome menu | building icon: "Managed by your organization" |
| `chrome://management` | browser is managed |
| `chrome://policy` | every CIS policy with status OK (`ProxyMode` shows "Deprecated", still works) |

## Account and privacy

| Try | Expect | Rule |
|-----|--------|------|
| Sign in to Chrome (profile icon) | not possible | 3.5 (L2) |
| Turn on Sync | disabled | 3.7 |
| Ctrl+Shift+N (Incognito) / Guest profile | not available | 5.2, 5.1 (L2) |
| Settings > Delete browsing data | history can't be deleted | 3.9 |
| Save an address or a card | not offered | 4.8 (L2), 4.9 |
| Save a password | follows `chrome_cis_password_manager_enabled` | 2.6.1 |
| Type in the address bar | no suggestions | 3.14 (L2) |
| Foreign-language page | no "Translate?" | 3.15 (L2) |
| Site asks for location / notifications | blocked, no prompt | 3.1.2, 2.2.4 (L2) |
| Login through another domain (Messenger via Facebook) | can fail: third-party cookies blocked | 3.4 |
| Log in, close Chrome, reopen | logged out | 3.1.1 (Full) |

## Security and downloads

| Try | Expect | Rule |
|-----|--------|------|
| `https://testsafebrowsing.appspot.com` > a malware link | red warning, no "Proceed anyway" | 2.13 |
| `https://expired.badssl.com` | certificate error, no way past it | 2.12 (Full) |
| `http://neverssl.com` | tries HTTPS first, warns before HTTP | 2.28 |
| Download a file | asks where to save | 1.6 |
| Download a flagged test file | blocked | 2.11 |
| Google search | SafeSearch locked on | 2.15 (L2) |

## Extensions and features

| Try | Expect | Rule |
|-----|--------|------|
| Install an extension from the Web Store | "Blocked by your administrator" | 2.3.3 (Full) |
| Cast button | gone | 3.2.1 |
| Close Chrome while a web app runs | Chrome really exits | 1.7 |
| Web call (e.g. meet.google.com): mic / camera / screen share | blocked | 4.4, 4.5, 4.1.1 (Full) |
| Attach a file on any site | no file dialog | 4.3 (Full) |
| Web app reads the clipboard | blocked | 4.2.5 |

## Rule 4.7: no website opens (Full)

**Seen 2026-10-08 on the full-benchmark run: every site showed "This site can't be reached".**

- **The rule:** `DnsOverHttpsMode = "secure"`. Chrome looks up site names **only** over encrypted DNS (DoH) and never
  falls back to normal DNS (CIS: *"will fail to resolve on error"*).
- **Why CIS wants it:** encrypted lookups hide visited sites from the network and stop DNS spoofing.
- **Why it broke:** Chrome also needs the address of a **DoH server** (`DnsOverHttpsTemplates`, e.g.
  `https://dns.google/dns-query{?dns}`). CIS only recommends it in a note and gives no value, so the role doesn't set it.
  The lab DNS speaks only normal DNS, so every lookup failed.
- **Lab decision:** `chrome_cis_rule_4_7: false` in `group_vars`, recorded as an accepted exception (no DoH server in the lab).
  A real site would set its own DoH server; that's an organization choice, not a CIS value.
