# CIS requirements matrix: Google Chrome on RHEL 8, 9, 10

**What:** one row per recommendation of the CIS Google Chrome Benchmark v3.0.0 (all 118): the policy and value CIS
requires, whether it can work on RHEL, and what the role will do. **Why:** this is the checklist for building **and**
for the compliance test later; a rule is done only when its row is built and verified on RHEL 8, 9 and 10.
Plain-language rationale and user impact per rule: [benchmark-summary.md](benchmark-summary.md).

## Totals

| | Rules | Level 1 | Level 2 |
|---|---|---|---|
| In the benchmark | 118 | 88 | 30 |
| **Not applicable on RHEL** (N/A) | 16 | 13 | 3 |
| **Implemented by the role** | **102** | 75 | 27 |

How rules and policies add up (a *rule* is a CIS recommendation; a *policy* is a Chrome setting name):

| Group | Policies | Rules |
|-------|----------|-------|
| Applicable ([evidence-files/cis-applicable-101.json](evidence-files/cis-applicable-101.json)) | 101 | 102 (4.1.1 and 4.12 are both `ScreenCaptureAllowed`) |
| N/A, Chrome policy names ([evidence-files/cis-not-applicable-14.json](evidence-files/cis-not-applicable-14.json)) | 14 | 14 |
| N/A, Google Update settings, not Chrome policies (2.1.1, 2.1.2) | 0 | 2 |
| **Total** | **115** | **118** |

Of the 102: 69 PATCH, 9 PATCH "empty list = Disabled", 13 PATCH risky (on by default, site may turn off; D5 revised), 11 SITE (site decides the value).

## How applicability was decided (evidence, not assumption)

The benchmark only gives Windows registry checks (PDF printed p. 8–9, viewer p. 9–10: "written for Microsoft Windows Active Directory
domain-joined systems using Group Policy… Adjustments/tailoring… will be needed"). Each rule was therefore checked in
two independent ways:

| # | Check | Source | Result |
|---|-------|--------|--------|
| E1 | **Google's official policy list for the same Chrome version** ("Supported on" per policy) | `https://dl.google.com/dl/edgedl/chrome/policy/policy_templates.zip`, `VERSION` = 155.0.8059.40, file `common/html/en-US/chrome_policy_list.html` (downloaded 2026-10-07) | 101 Chrome policies "Google Chrome (Linux)"; 4 Windows-only; 10 not in the list (removed from Chrome); 2 are Google Update, not Chrome |
| E2 | **Test on the VMs**: policies written into `/etc/opt/chrome/policies/managed/*.json`, then `chrome://policy` | D-3 (screenshots 04–06) and D-4 (user check, no screenshots); files in [evidence-files/](evidence-files/); Chrome 155.0.8059.39 on RHEL 8.10 / 9.7 / 10.2 | **all 101 applicable policies OK** with CIS values on all 3 OS (`ProxyMode` labelled Deprecated); 9 N/A policies tested → **Error: Unknown policy** |

E1 covers all 118 rules. E2 proves all 102 applicable rules on the real hosts, and 9 of the 16 N/A rules. The other 5
N/A rules (2.3.4, 2.3.5, 2.23, 2.29, 3.13) can be confirmed by loading `evidence-files/cis-not-applicable-14.json`.


## Evidence for the 16 not-applicable rules (search it yourself)

**Basis:** Google's policy template for Chrome **155** (the version on the VMs), file
`common/html/en-US/chrome_policy_list.html` from `https://dl.google.com/dl/edgedl/chrome/policy/policy_templates.zip`
(`VERSION`: 155.0.8059.40). Open it in a browser, press **Ctrl+F**, search the text in column 2:

- found, and **"Supported on:"** lists only *Google Chrome (Windows)* → Windows only;
- **no match** → the policy is not in Chrome 155 any more (removed since CIS tested Chrome 120).

Second check: the VM test (D-3) wrote the policy into the JSON file → `chrome://policy` said **Unknown policy** (confirmed).

| CIS | Search text (Ctrl+F) | What the 155 template shows | Meaning | VM |
|-----|----------------------|-----------------------------|---------|----|
| 1.10 | `CertificateTransparencyEnforcementDisabledForLegacyCas` | **no match** | removed from Chrome | Unknown policy |
| 1.19 | `ThirdPartyBlockingEnabled` | **no match** | removed from Chrome (was Windows-only) | Unknown policy |
| 2.3.4 | `DefaultThirdPartyStoragePartitioningSetting` | **no match** | removed from Chrome | not tested |
| 2.3.5 | `ThirdPartyStoragePartitioningBlockedForOrigins` | **no match** | removed from Chrome | not tested |
| 2.3.6 | `ExtensionManifestV2Availability` | **no match** | removed (Manifest V2 retired) | Unknown policy |
| 2.7.1 | `CloudPrintProxyEnabled` | **no match** | removed (Cloud Print retired) | Unknown policy |
| 2.8.2 | `RemoteAccessHostAllowUiAccessForRemoteAssistance` | Supported on: Google Chrome (Windows) since version 55 | Windows only | Unknown policy |
| 2.9.1 | `FirstPartySetsEnabled` | **no match** | removed from Chrome | Unknown policy |
| 2.10.1 | `CloudAPAuthEnabled` | Supported on: Google Chrome (Windows) since version 111 | Windows only | Unknown policy |
| 2.23 | `EnforceLocalAnchorConstraintsEnabled` | **no match** | removed from Chrome | not tested |
| 2.29 | `InsecureHashesInTLSHandshakesEnabled` | **no match** | removed from Chrome | not tested |
| 2.30 | `RendererAppContainerEnabled` | Supported on: Google Chrome (Windows) since version 104 | Windows only | Unknown policy |
| 3.6 | `ChromeCleanupReportingEnabled` | **no match** | removed from Chrome (was Windows-only) | Unknown policy |
| 3.13 | `SafeBrowsingForTrustedSourcesEnabled` | Supported on: Google Chrome (Windows) since version 61 | Windows only | not tested |
| 2.1.1 | `Update{8A69D345` | **no match** | Google Update setting, not a Chrome policy. CIS 2.1.1 itself: "available only on Windows instances that are joined to a Microsoft® Active Directory® domain". On RHEL, Chrome updates come from the `google-chrome` dnf repo (platform-notes W-1) | — |
| 2.1.2 | `AutoUpdateCheckPeriodMinutes` | **no match** | same as 2.1.1 | — |

## Legend

- **Linux on Chrome 155**: "Linux" = supported on Linux (E1). "N/A" = cannot be applied on RHEL, with the reason.
- **Role**:
  - **PATCH**: the role writes the policy with the CIS value.
  - **PATCH: `[]`**: CIS wants the exception list *Disabled*; the role writes an empty list (same effect as absent,
    design-decisions D7) and reports another file in `managed/` that sets entries.
  - **PATCH, risky**: breaks things users rely on; own toggle, `true` by default (D5 revised 2026-10-08), `# WARNING:` in
    defaults; a site turns it off in `group_vars`.
  - **SITE**: Manual (or site-dependent) rule: report by default; applied only when the site sets the value.
  - **—**: not implemented (N/A).
- **Verify on the VM** (same for every row): `chrome://policy` → the policy shows the CIS value, Source *Platform*,
  Level *Mandatory*, Status **OK**; "Disabled" list rows show `[]`.

## Section 1: Enforced Defaults (30 rules, all L1 except 1.8; they lock Chrome's defaults)

| ID | Lvl / type | Policy = CIS value | Linux on Chrome 155 | Role |
|----|------|----------------|---------------------|------|
| 1.1.1 | L1 Auto | `AllowCrossOriginAuthPrompt = false` | Linux | PATCH |
| 1.2.1 | L1 Auto | `SafeBrowsingAllowlistDomains` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 1.2.2 | L1 Manual | `SafeBrowsingProtectionLevel = 1` (standard) or `2` (enhanced) | Linux | SITE: report; applies only if site value set |
| 1.3 | L1 Auto | `MediaRouterCastAllowAllIPs = false` | Linux | PATCH |
| 1.4 | L1 Auto | `BrowserNetworkTimeQueriesEnabled = true` | Linux | PATCH |
| 1.5 | L1 Auto | `AudioSandboxEnabled = true` | Linux | PATCH |
| 1.6 | L1 Auto | `PromptForDownloadLocation = true` | Linux | PATCH |
| 1.7 | L1 Auto | `BackgroundModeEnabled = false` | Linux | PATCH |
| 1.8 | **L2** Auto | `SafeSitesFilterBehavior = 1` | Linux | PATCH |
| 1.9 | L1 Manual | `ChromeVariations = 0` (all variations) | Linux | SITE: report; applies only if site value set |
| 1.10 | L1 Auto | `CertificateTransparencyEnforcementDisabledForLegacyCas` absent | N/A: not in Chrome 155 policy list (removed); VM: Unknown policy | — |
| 1.11 | L1 Auto | `CertificateTransparencyEnforcementDisabledForCas` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 1.12 | L1 Auto | `CertificateTransparencyEnforcementDisabledForUrls` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 1.13 | L1 Auto | `SavingBrowserHistoryDisabled = false` | Linux | PATCH |
| 1.14 | L1 Auto | `DNSInterceptionChecksEnabled = true` | Linux | PATCH |
| 1.15 | L1 Auto | `ComponentUpdatesEnabled = true` | Linux | PATCH |
| 1.16 | L1 Auto | `GloballyScopeHTTPAuthCacheEnabled = false` | Linux | PATCH |
| 1.17 | L1 Auto | `EnableOnlineRevocationChecks = false` | Linux | PATCH |
| 1.18 | L1 Auto | `CommandLineFlagSecurityWarningsEnabled = true` | Linux | PATCH |
| 1.19 | L1 Auto | `ThirdPartyBlockingEnabled = true` | N/A: not in Chrome 155 policy list (removed; was Windows-only); VM: Unknown policy | — |
| 1.20 | L1 Auto | `EnterpriseHardwarePlatformAPIEnabled = false` | Linux | PATCH |
| 1.21 | L1 Auto | `ForceEphemeralProfiles = false` | Linux | PATCH |
| 1.22 | L1 Auto | `ImportAutofillFormData = false` | Linux | PATCH |
| 1.23 | L1 Auto | `ImportHomepage = false` | Linux | PATCH |
| 1.24 | L1 Auto | `ImportSearchEngine = false` | Linux | PATCH |
| 1.25 | L1 Auto | `HSTSPolicyBypassList` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 1.26 | L1 Auto | `OverrideSecurityRestrictionsOnInsecureOrigin` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 1.27 | L1 Auto | `LookalikeWarningAllowlistDomains` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 1.28 | L1 Auto | `SuppressUnsupportedOSWarning = false` | Linux | PATCH |
| 1.29 | L1 Auto | `WebRtcLocalIpsAllowedUrls` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |

## Section 2: Attack Surface Reduction (49 rules)

| ID | Lvl / type | Policy = CIS value | Linux on Chrome 155 | Role |
|----|------|----------------|---------------------|------|
| 2.1.1 | L1 Auto | Google Update `Update{8A69…}` = 1 or 3 (auto updates) | N/A: Google Update (Windows/macOS updater), not a Chrome policy; RHEL updates via dnf | — |
| 2.1.2 | L1 Auto | Google Update `AutoUpdateCheckPeriodMinutes` ≠ 0 | N/A: Google Update (Windows/macOS updater), not a Chrome policy; RHEL updates via dnf | — |
| 2.2.1 | L1 Auto | `DefaultInsecureContentSetting = 2` | Linux | PATCH |
| 2.2.2 | **L2** Auto | `DefaultWebBluetoothGuardSetting = 2` | Linux | PATCH |
| 2.2.3 | **L2** Auto | `DefaultWebUsbGuardSetting = 2` | Linux | PATCH, risky |
| 2.2.4 | **L2** Auto | `DefaultNotificationsSetting = 2` | Linux | PATCH |
| 2.2.5 | L1 Auto | `PdfLocalFileAccessAllowedForDomains` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 2.3.1 | L1 Auto | `BlockExternalExtensions = true` | Linux | PATCH |
| 2.3.2 | L1 Auto | `ExtensionAllowedTypes = ["extension","hosted_app","platform_app","theme"]` | Linux | PATCH |
| 2.3.3 | L1 Auto | `ExtensionInstallBlocklist = ["*"]` | Linux | PATCH, risky |
| 2.3.4 | **L2** Auto | `DefaultThirdPartyStoragePartitioningSetting = 2` | N/A: not in Chrome 155 policy list (removed) | — |
| 2.3.5 | L1 Manual | `ThirdPartyStoragePartitioningBlockedForOrigins = [<urls>]` | N/A: not in Chrome 155 policy list (removed) | — |
| 2.3.6 | **L2** Auto | `ExtensionManifestV2Availability = 3` (forced only) | N/A: not in Chrome 155 policy list (removed); VM: Unknown policy | — |
| 2.3.7 | L1 Auto | `ExtensionUnpublishedAvailability = 1` | Linux | PATCH |
| 2.4.1 | **L2** Auto | `AuthSchemes = "ntlm,negotiate"` | Linux | PATCH, risky |
| 2.5.1 | **L2** Auto | `NativeMessagingBlocklist = ["*"]` | Linux | PATCH, risky |
| 2.6.1 | L1 Manual | `PasswordManagerEnabled` = explicitly set (CIS audit checks `0`) | Linux | SITE: report; applies only if site value set |
| 2.7.1 | L1 Auto | `CloudPrintProxyEnabled = false` | N/A: not in Chrome 155 policy list (removed); VM: Unknown policy | — |
| 2.8.1 | L1 Manual | `RemoteAccessHostAllowRemoteAccessConnections = false` | Linux | SITE: report; applies only if site value set |
| 2.8.2 | L1 Auto | `RemoteAccessHostAllowUiAccessForRemoteAssistance = false` | N/A: Windows-only (Google list: "Google Chrome (Windows)" only); VM: Unknown policy | — |
| 2.8.3 | L1 Manual | `RemoteAccessHostClientDomainList = [<domains>]` | Linux | SITE: report; applies only if site value set |
| 2.8.4 | L1 Auto | `RemoteAccessHostRequireCurtain = false` | Linux | PATCH |
| 2.8.5 | L1 Auto | `RemoteAccessHostFirewallTraversal = false` | Linux | PATCH |
| 2.8.6 | L1 Auto | `RemoteAccessHostAllowClientPairing = false` | Linux | PATCH |
| 2.8.7 | L1 Auto | `RemoteAccessHostAllowRelayedConnection = false` | Linux | PATCH |
| 2.9.1 | L1 Manual | `FirstPartySetsEnabled = false` | N/A: not in Chrome 155 policy list (removed); VM: Unknown policy | — |
| 2.10.1 | L1 Manual | `CloudAPAuthEnabled = true` | N/A: Windows-only (Google list: "Google Chrome (Windows)" only); VM: Unknown policy | — |
| 2.11 | L1 Auto | `DownloadRestrictions = 4` | Linux | PATCH |
| 2.12 | **L2** Auto | `SSLErrorOverrideAllowed = false` | Linux | PATCH, risky |
| 2.13 | L1 Auto | `DisableSafeBrowsingProceedAnyway = true` | Linux | PATCH |
| 2.14 | L1 Auto | `SitePerProcess = true` | Linux | PATCH |
| 2.15 | **L2** Auto | `ForceGoogleSafeSearch = true` | Linux | PATCH |
| 2.16 | L1 Auto | `RelaunchNotification = 2` | Linux | PATCH |
| 2.17 | L1 Auto | `ProxyMode` set and **not** `"auto_detect"` | Linux (ProxyMode marked Deprecated; successor ProxySettings) | SITE: report; applies only if site value set |
| 2.18 | **L2** Auto | `RequireOnlineRevocationChecksForLocalAnchors = true` | Linux | PATCH, risky |
| 2.19 | L1 Auto | `RelaunchNotificationPeriod = 86400000` (24 h) | Linux | PATCH |
| 2.20 | L1 Auto | `AllowWebAuthnWithBrokenTlsCerts = false` | Linux | PATCH |
| 2.21 | L1 Auto | `DomainReliabilityAllowed = false` | Linux | PATCH |
| 2.22 | L1 Auto | `EncryptedClientHelloEnabled = true` | Linux | PATCH |
| 2.23 | **L2** Auto | `EnforceLocalAnchorConstraintsEnabled = true` | N/A: not in Chrome 155 policy list (removed) | — |
| 2.24 | L1 Auto | `EnterpriseProfileCreationKeepBrowsingData = true` | Linux | PATCH |
| 2.25 | L1 Auto | `FileOrDirectoryPickerWithoutGestureAllowedForOrigins` absent | Linux | PATCH: `[]` (empty = Disabled); reports other files with entries |
| 2.26 | L1 Auto | `GoogleSearchSidePanelEnabled = false` | Linux | PATCH |
| 2.27 | L1 Manual | `HttpAllowlist = [<hosts>]` | Linux | SITE: report; applies only if site value set |
| 2.28 | L1 Auto | `HttpsUpgradesEnabled = true` | Linux | PATCH |
| 2.29 | L1 Auto | `InsecureHashesInTLSHandshakesEnabled = false` | N/A: not in Chrome 155 policy list (removed) | — |
| 2.30 | L1 Auto | `RendererAppContainerEnabled = true` | N/A: Windows-only (Google list: "Google Chrome (Windows)" only); VM: Unknown policy | — |
| 2.31 | L1 Auto | `StrictMimetypeCheckForWorkerScriptsEnabled = true` | Linux | PATCH |
| 2.32 | L1 Auto | `RemoteDebuggingAllowed = false` | Linux | PATCH |

## Section 3: Privacy (17 rules)

| ID | Lvl / type | Policy = CIS value | Linux on Chrome 155 | Role |
|----|------|----------------|---------------------|------|
| 3.1.1 | **L2** Auto | `DefaultCookiesSetting = 4` | Linux | PATCH, risky |
| 3.1.2 | L1 Auto | `DefaultGeolocationSetting = 2` | Linux | PATCH |
| 3.2.1 | L1 Auto | `EnableMediaRouter = false` | Linux | PATCH |
| 3.3 | L1 Auto | `PaymentMethodQueryEnabled = false` | Linux | PATCH |
| 3.4 | L1 Auto | `BlockThirdPartyCookies = true` | Linux | PATCH |
| 3.5 | **L2** Auto | `BrowserSignin = 0` | Linux | PATCH |
| 3.6 | L1 Auto | `ChromeCleanupReportingEnabled = false` | N/A: not in Chrome 155 policy list (removed; was Windows-only); VM: Unknown policy | — |
| 3.7 | L1 Auto | `SyncDisabled = true` | Linux | PATCH |
| 3.8 | L1 Auto | `AlternateErrorPagesEnabled = false` | Linux | PATCH |
| 3.9 | L1 Auto | `AllowDeletingBrowserHistory = false` | Linux | PATCH |
| 3.10 | L1 Auto | `NetworkPredictionOptions = 2` | Linux | PATCH |
| 3.11 | L1 Auto | `SpellCheckServiceEnabled = false` | Linux | PATCH |
| 3.12 | L1 Auto | `MetricsReportingEnabled = false` | Linux | PATCH |
| 3.13 | L1 Auto | `SafeBrowsingForTrustedSourcesEnabled = false` | N/A: Windows-only (Google list: "Google Chrome (Windows)" only) | — |
| 3.14 | **L2** Auto | `SearchSuggestEnabled = false` | Linux | PATCH |
| 3.15 | **L2** Auto | `TranslateEnabled = false` | Linux | PATCH |
| 3.16 | L1 Auto | `UrlKeyedAnonymizedDataCollectionEnabled = false` | Linux | PATCH |

## Section 4: Data Loss Prevention (19 rules)

| ID | Lvl / type | Policy = CIS value | Linux on Chrome 155 | Role |
|----|------|----------------|---------------------|------|
| 4.1.1 | **L2** Auto | `ScreenCaptureAllowed = false` | Linux | PATCH, risky |
| 4.2.1 | **L2** Auto | `DefaultSerialGuardSetting = 2` | Linux | PATCH |
| 4.2.2 | **L2** Auto | `DefaultSensorsSetting = 2` | Linux | PATCH |
| 4.2.3 | L1 Manual | `ClipboardAllowedForUrls = [<urls>]` | Linux | SITE: report; applies only if site value set |
| 4.2.4 | L1 Manual | `ClipboardBlockedForUrls = [<urls>]` | Linux | SITE: report; applies only if site value set |
| 4.2.5 | L1 Auto | `DefaultClipboardSetting = 2` | Linux | PATCH |
| 4.2.6 | **L2** Auto | `DefaultWindowManagementSetting = 2` | Linux | PATCH |
| 4.2.7 | **L2** Manual | `WindowManagementAllowedForUrls = [<urls>]` | Linux | SITE: report; applies only if site value set |
| 4.2.8 | **L2** Manual | `WindowManagementBlockedForUrls = [<urls>]` | Linux | SITE: report; applies only if site value set |
| 4.3 | **L2** Auto | `AllowFileSelectionDialogs = false` | Linux | PATCH, risky |
| 4.4 | **L2** Auto | `AudioCaptureAllowed = false` | Linux | PATCH, risky |
| 4.5 | **L2** Auto | `VideoCaptureAllowed = false` | Linux | PATCH, risky |
| 4.6 | L1 Auto | `UserFeedbackAllowed = false` | Linux | PATCH |
| 4.7 | **L2** Auto | `DnsOverHttpsMode = "secure"` | Linux | PATCH, risky |
| 4.8 | **L2** Auto | `AutofillAddressEnabled = false` | Linux | PATCH |
| 4.9 | L1 Auto | `AutofillCreditCardEnabled = false` | Linux | PATCH |
| 4.10 | L1 Auto | `ImportSavedPasswords = false` | Linux | PATCH |
| 4.11 | L1 Auto | `SyncTypesListDisabled = ["passwords"]` | Linux | PATCH |
| 4.12 | **L2** Auto | `ScreenCaptureAllowed = false` | Linux | PATCH, risky (follows 4.1.1: same policy) |

## Section 5: Forensics (3 rules)

| ID | Lvl / type | Policy = CIS value | Linux on Chrome 155 | Role |
|----|------|----------------|---------------------|------|
| 5.1 | **L2** Auto | `BrowserGuestModeEnabled = false` | Linux | PATCH |
| 5.2 | **L2** Auto | `IncognitoModeAvailability = 1` | Linux | PATCH |
| 5.3 | L1 Auto | `DiskCacheSize = 250609664` (~239 MB) | Linux | PATCH |
