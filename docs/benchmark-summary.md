# CIS Google Chrome Benchmark v3.0.0: plain-language summary (planning)

**What:** every recommendation in the benchmark, with what it sets, why CIS wants it, what users will notice, and a
first planning note for the role. **Why:** to skim the benchmark before designing the role (workflow phase 1,
[role-workflow.md](../../docs/role-workflow.md)). This is a reading aid, not the requirements matrix: that comes next
(`cis-requirements.md`) after the open questions below are answered and discovery is done.

**Source:** `cis-spreadsheet/CIS_Google_Chrome_Benchmark_v3.0.0.xlsx` (sheet *Combined Profiles*: title, profile,
status, rationale, impact, audit, remediation, default value) and `cis-pdf/CIS_Google_Chrome_Benchmark_v3.0.0.pdf`
(Overview, pages 8–9). Rationale and impact are condensed into my own words; the CIS wording is in the spreadsheet.

## 1. Facts about this benchmark that shape the plan

| # | Fact | Evidence | What it means for us |
|---|------|----------|----------------------|
| F1 | **118 recommendations**: Level 1 = 88 (77 Automated, 11 Manual), Level 2 = 30 (28 Automated, 2 Manual) | spreadsheet count | far bigger than MongoDB (23), but almost every rule is the same kind of thing: one policy = one value |
| F2 | Written for **Windows Group Policy on AD-domain-joined machines**; every Audit is a registry value under `HKLM\SOFTWARE\Policies\Google\Chrome` | PDF printed p. 8 (viewer p. 9): "This Benchmark assumes the installation of the Google Chrome and Google Update ADMX/ADML templates into the Active Directory policy store"; printed p. 9 (viewer p. 10): "designed to use Windows Group Policy on a domain joined system to set the appropriate Windows registry values", "written for Microsoft Windows Active Directory domain-joined systems using Group Policy… Adjustments/tailoring to some recommendations will be needed" | no Linux commands in the benchmark. We translate each rule to its **Chrome policy name** (the registry value name) and check on each RHEL version that Chrome accepts it (platform-notes D-3) |
| F3 | On Linux, Chrome reads the **same policy names** from JSON files in a "managed policies" directory; `chrome://policy` shows the effective result on every OS | PDF p. 9 says CIS settings end up as Chrome "policies" visible in `chrome://policy`. Linux path to **verify in discovery** (expected `/etc/opt/chrome/policies/managed/` for Google Chrome, `/etc/chromium/policies/managed/` for Chromium) | strong hint for a **data-driven role**: one rules-as-data file rendered into one JSON policy file. To be confirmed and justified in `design-decisions.md` |
| F4 | **"Enforced Defaults"** (section 1): many rules set what Chrome already does by default. CIS sets them anyway so **the user can't change them** and scanners can check them | PDF p. 9 "Enforced Defaults" | most of section 1 has **no user impact**; it just locks the default |
| F5 | Policy value types: boolean (`0`/`1` in registry → `false`/`true` in JSON), integer (`2` = block, etc.), string (`"secure"`), list (`ExtensionInstallBlocklist: ["*"]`), and "must **not exist**" (rules saying 'Disabled' for a list policy) | Audit column of each rule | the data file needs per-rule type; "must not exist" rules = **don't write the key** (and remove it if present) |
| F6 | Some policies are for Google Update, Windows-only AD features, or retired Google services | rule text (e.g. 2.1.1 "available only on Windows instances that are joined to … Active Directory") | each rule gets a status `applies` / `na_os` / `manual`, **verified** against Google's policy list "Supported on" (CLAUDE.md "classify with evidence") |
| F7 | Level 2 **extends** Level 1 (High Security, "limited functionality") | Profile Definitions | same gating as `mongodb8_cis`: `level_1` on, `level_2` off by default |
| F8 | Tested against **Chrome v120** | PDF p. 8 | some policies may be renamed/removed in current Chrome; discovery checks each against the installed version |
| F9 | **4.1.1 and 4.12 are the same policy** (`ScreenCaptureAllowed = 0`) | Audit of both | one data entry, two IDs, or one entry with a note; never renumber |
| F10 | 1.26's audit path is `HKLM\SOFTWARE\Policies\Chrome\…` (missing `Google\`): a CIS typo | Audit of 1.26 | use the policy name `OverrideSecurityRestrictionsOnInsecureOrigin`; note it in the matrix |

## 2. How to read the tables

- **Policy = value**: the Chrome policy name and the value CIS requires, in JSON terms. `absent` = the policy must not be set.
- **Why (CIS rationale)**: one line, plain words.
- **Users notice**: from CIS *Impact*. "none (default)" = Chrome already behaves this way.
- **Plan**: first idea only, to be confirmed.
  - `set` = write the policy; low risk.
  - `set ⚠` = write it, but it can break things for users; toggle **off by default** with a `# WARNING:` (candidate).
  - `site` = Manual rule whose value the organisation decides: report by default, optional site variable (CLAUDE.md principle 5).
  - `report` = Manual rule, report only.
  - `win?` = probably Windows-only or retired; **verify** in discovery before classifying `na_os`.

---

## Section 1: Enforced Defaults (30 rules, all L1 except 1.8; they lock Chrome's defaults)

| ID | L / type | Policy = value | Why (CIS rationale) | Users notice | Plan |
|----|----------|----------------|---------------------|--------------|------|
| 1.1.1 | L1 Auto | `AllowCrossOriginAuthPrompt = false` | stops login pop-ups from third-party content on a page (phishing) | none (default) | set |
| 1.2.1 | L1 Auto | `SafeBrowsingAllowlistDomains` absent | no domains exempt from Safe Browsing warnings | none (default) | set (absent) |
| 1.2.2 | L1 Manual | `SafeBrowsingProtectionLevel = 1` (standard) or `2` (enhanced) | Safe Browsing blocks malicious sites/downloads; Google recommends 2, but 2 sends more data to Google | none at 1 | site (1 or 2) |
| 1.3 | L1 Auto | `MediaRouterCastAllowAllIPs = false` | Cast only to private IPs, not public ones | none (default) | set |
| 1.4 | L1 Auto | `BrowserNetworkTimeQueriesEnabled = true` | trusted network time helps certificate validation | none (default) | set |
| 1.5 | L1 Auto | `AudioSandboxEnabled = true` | audio runs sandboxed so a site can't abuse it | none (default) | set |
| 1.6 | L1 Auto | `PromptForDownloadLocation = true` | ask where to save each file: no silent drive-by downloads | save dialog on every download | set |
| 1.7 | L1 Auto | `BackgroundModeEnabled = false` | no apps/extensions keep running after Chrome closes | none (default) | set |
| 1.8 | **L2** Auto | `SafeSitesFilterBehavior = 1` | filter adult sites (more prone to malware) | adult content filtered; may leak typed text to Google's API | set |
| 1.9 | L1 Manual | `ChromeVariations = 0` (all variations) | Google ships fixes gradually via variations; keep them on | none (default) | site (or set) |
| 1.10 | L1 Auto | `CertificateTransparencyEnforcementDisabledForLegacyCas` absent | every CA must follow Certificate Transparency | none (default) | set (absent); policy may be removed in new Chrome → verify |
| 1.11 | L1 Auto | `CertificateTransparencyEnforcementDisabledForCas` absent | CT enforced for all certificates | none (default) | set (absent) |
| 1.12 | L1 Auto | `CertificateTransparencyEnforcementDisabledForUrls` absent | CT enforced for all URLs | none (default) | set (absent) |
| 1.13 | L1 Auto | `SavingBrowserHistoryDisabled = false` | history kept: evidence for investigations | none (default) | set |
| 1.14 | L1 Auto | `DNSInterceptionChecksEnabled = true` | detect DNS hijacking | none (default) | set |
| 1.15 | L1 Auto | `ComponentUpdatesEnabled = true` | keep Chrome components patched | none (default) | set |
| 1.16 | L1 Auto | `GloballyScopeHTTPAuthCacheEnabled = false` | HTTP auth credentials not shared across sites (leak, tracking) | none (default) | set |
| 1.17 | L1 Auto | `EnableOnlineRevocationChecks = false` | online OCSP/CRL soft-fail adds no real security; CRLSets used instead | none (default) | set |
| 1.18 | L1 Auto | `CommandLineFlagSecurityWarningsEnabled = true` | warn when Chrome runs with dangerous flags | none (default) | set |
| 1.19 | L1 Auto | `ThirdPartyBlockingEnabled = true` | block third-party code injection into Chrome | none (default) | set; **win?** |
| 1.20 | L1 Auto | `EnterpriseHardwarePlatformAPIEnabled = false` | extensions don't get hardware platform API | none (default) | set |
| 1.21 | L1 Auto | `ForceEphemeralProfiles = false` | profiles keep data on disk (forensics) | none (default) | set |
| 1.22 | L1 Auto | `ImportAutofillFormData = false` | no PII imported from another browser | none (default) | set |
| 1.23 | L1 Auto | `ImportHomepage = false` | no (possibly compromised) homepage imported | none (default) | set |
| 1.24 | L1 Auto | `ImportSearchEngine = false` | no malicious search engine imported | none (default) | set |
| 1.25 | L1 Auto | `HSTSPolicyBypassList` absent | no host exempt from HSTS (downgrade, cookie hijack) | none (default) | set (absent) |
| 1.26 | L1 Auto | `OverrideSecurityRestrictionsOnInsecureOrigin` absent | insecure origins always labelled insecure | none (default) | set (absent) |
| 1.27 | L1 Auto | `LookalikeWarningAllowlistDomains` absent | keep warnings for look-alike (phishing) domains | none (default) | set (absent) |
| 1.28 | L1 Auto | `SuppressUnsupportedOSWarning = false` | user is told when the OS is unsupported | none (default) | set |
| 1.29 | L1 Auto | `WebRtcLocalIpsAllowedUrls` absent | WebRTC doesn't expose internal IPs | none (default) | set (absent) |

## Section 2: Attack Surface Reduction (49 rules)

### 2.1 Updates (Google Update, Windows)

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 2.1.1 | L1 Auto | Google Update `Update{8A69…}` = 1 or 3 (auto updates) | apply security patches as soon as available | — | **win?** (benchmark: "only on Windows… joined to AD"). On RHEL updates come from the dnf repo → likely `na_os` |
| 2.1.2 | L1 Auto | Google Update `AutoUpdateCheckPeriodMinutes` ≠ 0 | updater must keep checking | — | **win?** same as 2.1.1 |

### 2.2 Content settings

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 2.2.1 | L1 Auto | `DefaultInsecureContentSetting = 2` | no HTTP content mixed into HTTPS pages | mixed-content pages partly break | set |
| 2.2.2 | **L2** Auto | `DefaultWebBluetoothGuardSetting = 2` | sites can't talk to Bluetooth devices | web Bluetooth stops working | set |
| 2.2.3 | **L2** Auto | `DefaultWebUsbGuardSetting = 2` | WebUSB could be abused for phishing that bypasses hardware 2FA | web USB stops; **may break some security-key flows** | set ⚠ |
| 2.2.4 | **L2** Auto | `DefaultNotificationsSetting = 2` | notifications may carry fake/malicious links | no site notifications | set |
| 2.2.5 | L1 Auto | `PdfLocalFileAccessAllowedForDomains` absent | sites can't open local files in the PDF viewer | open local PDFs manually | set (absent) |

### 2.3 Extensions

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 2.3.1 | L1 Auto | `BlockExternalExtensions = true` | only Web Store extensions, no side-loaded ones | can't install external extensions | set |
| 2.3.2 | L1 Auto | `ExtensionAllowedTypes = ["extension","hosted_app","platform_app","theme"]` | block misusable/deprecated app types | installed extensions of other types are **removed** | set (note: Google removed Chrome Apps; check values on current Chrome) |
| 2.3.3 | L1 Auto | `ExtensionInstallBlocklist = ["*"]` | block all extensions except an allowlist | **every installed extension is removed** unless allowlisted (e.g. password manager) | set ⚠ + site allowlist (`ExtensionInstallAllowlist`) |
| 2.3.4 | **L2** Auto | `DefaultThirdPartyStoragePartitioningSetting = 2` | defends against cross-site tracking/timing attacks | some sites using third-party access may break | set; value to verify (CIS text is unclear: "Enabled and Blocked") |
| 2.3.5 | L1 Manual | `ThirdPartyStoragePartitioningBlockedForOrigins = [<urls>]` | curated list instead of blocking all | varies per site | site (list) |
| 2.3.6 | **L2** Auto | `ExtensionManifestV2Availability = 3` (forced only) | old v2 extensions disabled unless forced by admin | v2 extensions disabled | set; may be obsolete in current Chrome (MV2 removed) → verify |
| 2.3.7 | L1 Auto | `ExtensionUnpublishedAvailability = 1` | no extensions removed from the Web Store (unpatched) | some extensions disabled | set |

### 2.4 – 2.10 Single-topic groups

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 2.4.1 | **L2** Auto | `AuthSchemes = "ntlm,negotiate"` | Basic/Digest send passwords (almost) in clear | legacy Basic-auth sites stop working | set ⚠ |
| 2.5.1 | **L2** Auto | `NativeMessagingBlocklist = ["*"]` | extensions can't talk to local programs unless allowlisted | e.g. desktop password-manager integration breaks | set ⚠ + site allowlist |
| 2.6.1 | L1 Manual | `PasswordManagerEnabled` = explicitly set (CIS audit checks `0`) | organisation decides whether the browser stores passwords | depends | site (true/false) |
| 2.7.1 | L1 Auto | `CloudPrintProxyEnabled = false` | no printing from unmanaged devices via Cloud Print | none | set; **win?/retired** (Google Cloud Print shut down) → verify |
| 2.8.1 | L1 Manual | `RemoteAccessHostAllowRemoteAccessConnections = false` | only approved remote-access tools; skip if Chrome Remote Desktop is approved | Chrome Remote Desktop disabled | site |
| 2.8.2 | L1 Auto | `RemoteAccessHostAllowUiAccessForRemoteAssistance = false` | remote users can't control elevated (admin) windows | none (default) | set; **win?** |
| 2.8.3 | L1 Manual | `RemoteAccessHostClientDomainList = [<domains>]` | only clients from your domains may connect | — | site (list) |
| 2.8.4 | L1 Auto | `RemoteAccessHostRequireCurtain = false` | person at the machine can see what the remote user does | none (default) | set |
| 2.8.5 | L1 Auto | `RemoteAccessHostFirewallTraversal = false` | no remote access across firewalls | remote access only from LAN | set |
| 2.8.6 | L1 Auto | `RemoteAccessHostAllowClientPairing = false` | PIN needed every time | PIN every connection | set |
| 2.8.7 | L1 Auto | `RemoteAccessHostAllowRelayedConnection = false` | no relay servers to bypass the firewall | — | set |
| 2.9.1 | L1 Manual | `FirstPartySetsEnabled = false` | sites can't declare "related" sites to share cookies | odd behaviour across related sites | site (or set); policy may be renamed (Related Website Sets) → verify |
| 2.10.1 | L1 Manual | `CloudAPAuthEnabled = true` | Microsoft cloud SSO without an extension | none | **win?** (Microsoft AD/Entra feature) |

### 2.11 – 2.32 Other attack-surface settings

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 2.11 | L1 Auto | `DownloadRestrictions = 4` | block downloads Safe Browsing flags as malicious | malicious downloads blocked | set |
| 2.12 | **L2** Auto | `SSLErrorOverrideAllowed = false` | users can't click through certificate errors | **internal sites with bad certs become unreachable** | set ⚠ |
| 2.13 | L1 Auto | `DisableSafeBrowsingProceedAnyway = true` | users can't click through Safe Browsing warnings | rare false positive blocks a real site | set |
| 2.14 | L1 Auto | `SitePerProcess = true` | each site in its own process (data isolation) | more memory | set |
| 2.15 | **L2** Auto | `ForceGoogleSafeSearch = true` | filter risky search results | filtered search | set |
| 2.16 | L1 Auto | `RelaunchNotification = 2` | after an update, force a relaunch so the patch takes effect | recurring relaunch prompt, then forced relaunch (tabs restored) | set |
| 2.17 | L1 Auto | `ProxyMode` set and **not** `"auto_detect"` | WPAD auto-detect can be abused to inject a rogue proxy | proxy no longer auto-discovered | site (`direct`, `fixed_servers`, `pac_script`, `system`); network-dependent ⚠ |
| 2.18 | **L2** Auto | `RequireOnlineRevocationChecksForLocalAnchors = true` | always check revocation for internal-CA certificates | **hard-fail if the OCSP/CRL server is down** | set ⚠ |
| 2.19 | L1 Auto | `RelaunchNotificationPeriod = 86400000` (24 h) | relaunch within a day of an update | reminder until relaunch | set |
| 2.20 | L1 Auto | `AllowWebAuthnWithBrokenTlsCerts = false` | no WebAuthn on sites with invalid TLS | none (default) | set |
| 2.21 | L1 Auto | `DomainReliabilityAllowed = false` | no reliability data sent to Google | none | set |
| 2.22 | L1 Auto | `EncryptedClientHelloEnabled = true` | hide the site name in the TLS handshake | none (default) | set |
| 2.23 | **L2** Auto | `EnforceLocalAnchorConstraintsEnabled = true` | enforce constraints in locally installed CAs | some internal sites may fail | set |
| 2.24 | L1 Auto | `EnterpriseProfileCreationKeepBrowsingData = true` | keep previous data when an enterprise profile is created | none | set |
| 2.25 | L1 Auto | `FileOrDirectoryPickerWithoutGestureAllowedForOrigins` absent | no file pickers opened without a user click | none (default) | set (absent) |
| 2.26 | L1 Auto | `GoogleSearchSidePanelEnabled = false` | no Google search side panel | side panel gone | set |
| 2.27 | L1 Manual | `HttpAllowlist = [<hosts>]` | internal HTTP-only servers keep working with HTTPS upgrades on | none | site (list) |
| 2.28 | L1 Auto | `HttpsUpgradesEnabled = true` | upgrade HTTP to HTTPS when possible | none (default) | set |
| 2.29 | L1 Auto | `InsecureHashesInTLSHandshakesEnabled = false` | no legacy (weak) hashes in TLS | very old sites blocked | set |
| 2.30 | L1 Auto | `RendererAppContainerEnabled = true` | stronger sandbox for renderer processes | none (default) | set; **win?** (AppContainer is a Windows sandbox) |
| 2.31 | L1 Auto | `StrictMimetypeCheckForWorkerScriptsEnabled = true` | worker scripts must use proper JS MIME types | none | set |
| 2.32 | L1 Auto | `RemoteDebuggingAllowed = false` | attack tools abuse remote debugging to steal data / inject code | `--remote-debugging-port` stops working (affects test automation, e.g. Selenium/Puppeteer) | set |

## Section 3: Privacy (17 rules)

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 3.1.1 | **L2** Auto | `DefaultCookiesSetting = 4` | cookies only for the session | **logged out of every site when the browser closes** | set ⚠ |
| 3.1.2 | L1 Auto | `DefaultGeolocationSetting = 2` | no site can track location (also leaks network info) | location features off | set |
| 3.2.1 | L1 Auto | `EnableMediaRouter = false` | no Google Cast of tabs/desktop to local devices | Cast icon gone | set |
| 3.3 | L1 Auto | `PaymentMethodQueryEnabled = false` | sites can't ask what payment methods are stored | — | set |
| 3.4 | L1 Auto | `BlockThirdPartyCookies = true` | stops cross-site tracking cookies | some embedded content/SSO may break | set |
| 3.5 | **L2** Auto | `BrowserSignin = 0` | no personal Google accounts in a corporate browser | no sign-in, no sync | set |
| 3.6 | L1 Auto | `ChromeCleanupReportingEnabled = false` | Chrome Cleanup doesn't report to Google | — | set; **win?** (Chrome Cleanup is Windows-only) |
| 3.7 | L1 Auto | `SyncDisabled = true` | browser data not synced to Google cloud | no sync | set |
| 3.8 | L1 Auto | `AlternateErrorPagesEnabled = false` | navigation errors not sent to a Google service | — | set |
| 3.9 | L1 Auto | `AllowDeletingBrowserHistory = false` | users can't erase evidence | can't clear history | set |
| 3.10 | L1 Auto | `NetworkPredictionOptions = 2` | no pre-connections to resources the user didn't choose | slightly slower page loads | set |
| 3.11 | L1 Auto | `SpellCheckServiceEnabled = false` | typed text not sent to Google spellcheck | local dictionary only | set |
| 3.12 | L1 Auto | `MetricsReportingEnabled = false` | no usage/crash data to Google | — | set |
| 3.13 | L1 Auto | `SafeBrowsingForTrustedSourcesEnabled = false` | intranet downloads not sent to Google | — | set |
| 3.14 | **L2** Auto | `SearchSuggestEnabled = false` | omnibox text (could be passwords) not sent while typing | no suggestions | set |
| 3.15 | **L2** Auto | `TranslateEnabled = false` | internal pages not sent to Google Translate | no translate | set |
| 3.16 | L1 Auto | `UrlKeyedAnonymizedDataCollectionEnabled = false` | no URL data collection | — | set |

## Section 4: Data Loss Prevention (19 rules)

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 4.1.1 | **L2** Auto | `ScreenCaptureAllowed = false` | sites can't capture the screen | **screen sharing in web meetings breaks** | set ⚠ (same policy as 4.12) |
| 4.2.1 | **L2** Auto | `DefaultSerialGuardSetting = 2` | sites can't use serial ports | — | set |
| 4.2.2 | **L2** Auto | `DefaultSensorsSetting = 2` | sites can't read sensors (profiling) | — | set |
| 4.2.3 | L1 Manual | `ClipboardAllowedForUrls = [<urls>]` | the only sites allowed to read the clipboard (with 4.2.5) | — | site (list) |
| 4.2.4 | L1 Manual | `ClipboardBlockedForUrls = [<urls>]` | sites never allowed to read the clipboard | — | site (list) |
| 4.2.5 | L1 Auto | `DefaultClipboardSetting = 2` | sites can't read the clipboard | paste of images/formatting in web apps may fail | set |
| 4.2.6 | **L2** Auto | `DefaultWindowManagementSetting = 2` | rogue sites can't open/move windows on other screens | — | set |
| 4.2.7 | **L2** Manual | `WindowManagementAllowedForUrls = [<urls>]` | sites allowed window management | — | site (list) |
| 4.2.8 | **L2** Manual | `WindowManagementBlockedForUrls = [<urls>]` | sites blocked from window management | — | site (list) |
| 4.3 | **L2** Auto | `AllowFileSelectionDialogs = false` | no upload/download/save dialogs (data leaving) | **no file uploads, no "save as"** | set ⚠ |
| 4.4 | **L2** Auto | `AudioCaptureAllowed = false` | sites can't use the microphone | **web calls lose audio** | set ⚠ |
| 4.5 | **L2** Auto | `VideoCaptureAllowed = false` | sites can't use the camera | **web calls lose video** | set ⚠ |
| 4.6 | L1 Auto | `UserFeedbackAllowed = false` | no data to Google via feedback | — | set |
| 4.7 | **L2** Auto | `DnsOverHttpsMode = "secure"` | encrypted DNS, no fallback | DNS fails if no DoH server; **bypasses corporate DNS filtering** | set ⚠ (needs `DnsOverHttpsTemplates` site value?) → verify |
| 4.8 | **L2** Auto | `AutofillAddressEnabled = false` | stolen machine can't leak stored addresses | no address autofill | set |
| 4.9 | L1 Auto | `AutofillCreditCardEnabled = false` | stored cards not harvestable | no card autofill | set |
| 4.10 | L1 Auto | `ImportSavedPasswords = false` | no passwords imported from other browsers | — | set |
| 4.11 | L1 Auto | `SyncTypesListDisabled = ["passwords"]` | passwords never synced to the cloud | — | set |
| 4.12 | **L2** Auto | `ScreenCaptureAllowed = false` | duplicate of 4.1.1 | as 4.1.1 | same entry as 4.1.1 |

## Section 5: Forensics (3 rules)

| ID | L / type | Policy = value | Why | Users notice | Plan |
|----|----------|---------------|-----|--------------|------|
| 5.1 | **L2** Auto | `BrowserGuestModeEnabled = false` | guest sessions delete their data (lost evidence) | no guest mode | set |
| 5.2 | **L2** Auto | `IncognitoModeAvailability = 1` | incognito hides activity from investigations | no incognito | set |
| 5.3 | L1 Auto | `DiskCacheSize = 250609664` (~239 MB) | enough cache kept for investigations | cache up to ~250 MB on disk | set |

---

## 3. What this means for the role (first read, not decided)

| Topic | Observation | Candidate approach (decide in `design-decisions.md`) |
|-------|-------------|-------------------------------------------------------|
| Config mechanism | ~100 of 118 rules are "policy X = value Y" | CLAUDE.md "single declarative artifact" pattern: rules as data in `vars/`, one task renders the managed-policy JSON; skip by rule ID |
| Absent-policy rules | 1.2.1, 1.10–1.12, 1.25–1.27, 1.29, 2.2.5, 2.25 | our JSON simply doesn't contain them; an AUDIT step reports if **another** file in the policy dir sets them |
| Risky rules (`set ⚠`) | 2.2.3, 2.3.3, 2.4.1, 2.5.1, 2.12, 2.17, 2.18, 3.1.1, 4.1.1/4.12, 4.3, 4.4, 4.5, 4.7 | own toggle `false` + `# WARNING:` (mostly L2; 2.3.3 and 2.17 are L1) |
| Manual with a site value (`site`) | 1.2.2, 1.9, 2.3.5, 2.6.1, 2.8.1, 2.8.3, 2.9.1, 2.17, 2.27, 4.2.3, 4.2.4, 4.2.7, 4.2.8 (+ allowlists for 2.3.3, 2.5.1) | report by default; optional site variable writes the policy. Lists are simple (list of strings), so they fit [keep-options-simple] |
| Not applicable on RHEL (confirmed 2026-10-07) | **16**: Windows-only 2.8.2, 2.10.1, 2.30, 3.13; removed from Chrome 1.10, 1.19, 2.3.4, 2.3.5, 2.3.6, 2.7.1, 2.9.1, 2.23, 2.29, 3.6; Google Update 2.1.1, 2.1.2 | not implemented; evidence: Google's Chrome 155 policy list + VM test ([cis-requirements.md](cis-requirements.md)) |
| Possibly obsolete in current Chrome | 1.10, 2.3.2 (Chrome Apps), 2.3.6 (MV2), 2.9.1 (First-Party Sets), 2.7.1 (Cloud Print) | check `chrome://policy` on the installed version: unknown policies show as errors |
| How to audit effective state | CIS's own check is `chrome://policy` | on RHEL: the JSON file + `chrome://policy` (screenshot/export) or a headless check → discovery |

## 4. Decided

| Date | Decision | Why |
|------|----------|-----|
| 2026-10-07 | **Target = Google Chrome (`google-chrome-stable`) on RHEL 8, 9 and 10.** Chromium comes later, as a separate step, only if the benchmark can be applied to it | CIS has a Chrome benchmark but none for Chromium; using the benchmark's own product avoids the "substitute benchmark" mapping for now |
| 2026-10-07 | **Windows-only rules are not implemented.** They are listed as `na_os` in the requirements matrix with evidence (Google's policy list "Supported on"), so the matrix still covers all 118 IDs | the role runs on RHEL only; CIS IDs are kept, never renumbered |
| 2026-10-07 | **Both levels are implemented.** Defaults: `level_1: true`, `level_2: false` (Level 2 = L1 + L2, switched on by the site) | CIS Profile Definitions (PDF p. 14): L1 = "practical and prudent… not inhibit the utility"; L2 = "security more critical than usability… may negatively inhibit the utility". L2 limits features for browser users; it does not lock admins out of the server |
| 2026-10-07 | **Per-rule risk toggles are separate from levels.** A rule that breaks things for users gets its own toggle `false` + `# WARNING:`, whatever its level (as `mongodb8_cis` 2.1/2.2, which are L1 but off). Chrome L1 candidates: 2.3.3 (removes all extensions), 2.17 (proxy) | level = CIS's security/usability profile; the risk toggle = our safety switch for the site |
| 2026-10-07 | **Install is opt-in** (`chrome_cis_install: false`). Not installed and install off → skip message, clean end | same as `mongodb8_cis`; CLAUDE.md "Install is opt-in" |
| 2026-10-07 | **Candidate pattern: data-driven** (one data list of policies in `vars/`, one task writes one JSON policy file), not one task per rule. To be confirmed in discovery and recorded in `design-decisions.md` | all ~100 applicable rules use the same mechanism (policy name = value); MongoDB rules each used different mechanisms |
| 2026-10-07 | **Test VMs = the same 3 Proxmox VMs** (rhel8/9/10, "Server with GUI", MongoDB already on them), new snapshot before Chrome work for rollback | GUI present, so `chrome://policy` can be checked on each OS; Chrome policies don't touch MongoDB |

## 5. Site choices (like MongoDB's opt-in site variables)

Rules where the **site** decides a value. Default = report only (show the current value, change nothing); set the
variable and the role writes it (CLAUDE.md principle 5). Everything else (~95 rules) is a fixed CIS value: no choice.

| Kind | IDs | Input |
|------|-----|-------|
| Pick one value (Manual) | 1.2.2 Safe Browsing level (1 or 2), 1.9 variations (0), 2.6.1 password manager (true/false), 2.8.1 Chrome Remote Desktop (true/false), 2.9.1 First-Party Sets (false) | one value |
| URL / domain list (Manual) | 2.3.5, 2.8.3, 2.27, 4.2.3, 4.2.4, 4.2.7 (L2), 4.2.8 (L2) | list of strings |
| Automated, but needs a site value | 2.17 proxy mode (`direct` / `system` / `fixed_servers` / `pac_script`, never `auto_detect`) | one value; toggle off until set |
| Allowlist next to a risky blocklist | 2.3.3 `ExtensionInstallAllowlist`, 2.5.1 `NativeMessagingAllowlist` | list of IDs (empty = block all) |
| Not applicable on RHEL (if confirmed) | 2.10.1 (Microsoft cloud sign-in) | — |

## 6. Open questions for you

1. **Site values for the lab:** extension allowlist (2.3.3), proxy mode (2.17), DoH server (4.7), password manager yes/no (2.6.1).


## 7. Next steps (workflow phases)

1. Answer the questions above.
2. Discovery per RHEL 8/9/10 → `platform-notes.md` (package, policy dir, SELinux, Wayland, how to read effective policy).
3. Verify every `win?` / obsolete rule → classification with evidence.
4. Requirements matrix `cis-requirements.md` (one row per rule, Verify command).
5. Design decisions, then build in batches.
