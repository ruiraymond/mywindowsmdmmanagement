# Windows MDM Management Plan and Implementation Guide

This guide defines a complete Microsoft endpoint management model for **Windows 10/11** across three tiers:
- **Organization** (corporate-managed)
- **Family** (shared household-managed)
- **Personal** (single-user managed)

It is designed for secure defaults, standardized app lifecycle workflows, least privilege, and practical operations.

---

## Quick Start for New Administrators

1. **Define tenant structure and groups**
   - Create Entra ID groups: `MDM-Org`, `MDM-Family`, `MDM-Personal`, `MDM-Pilot`.
   - Scope policies, scripts, and app assignments by group.

2. **Enroll pilot devices first**
   - Start with 5–10 devices per tier in `MDM-Pilot`.
   - Validate compliance, software install success, and user experience.

3. **Deploy baseline profiles**
   - Security baseline
   - Compliance policy
   - Update rings
   - Endpoint protection/firewall
   - App deployment + remediation scripts

4. **Deploy required software packages**
   - Google Chrome
   - Brave Browser
   - VLC media player
   - Discord

5. **Monitor and remediate**
   - Use install detection rules and remediation scripts for drift.
   - Track failed installs by error code and retry logic.

6. **Move from pilot to broad rollout**
   - Expand assignments in phases after success criteria are met.

---

## 1) Tiered Policy Matrix

| Control Area | Organization Tier | Family Tier | Personal Tier |
|---|---|---|---|
| Security Baseline | CIS-aligned hardening; BitLocker required; Defender AV + EDR required; SmartScreen enforced | BitLocker strongly recommended; Defender AV required; SmartScreen enabled | Defender AV + Firewall on; BitLocker recommended based on hardware |
| App Control | Allow-list business and approved consumer apps only; optional WDAC/AppLocker | Block high-risk apps; allow trusted app store + approved list | Advisory controls; malware/risk-based blocks only |
| Update Cadence | Windows quality updates: 7-day deferral; feature updates: 30-60 days; emergency patch ring | Quality updates: 14-day deferral; feature updates: 60-90 days | Quality updates: 7-21 days; feature updates user-scheduled within policy window |
| User Permissions | Standard user only; local admin via just-in-time break-glass workflow | Parent/owner admin, daily users standard | Local admin allowed only for owner account; separate non-admin daily account recommended |
| Browser Policy | Forced safe browsing, extension restrictions, password manager policy, homepage/search control | Safe browsing and anti-phishing on; moderate extension controls | Privacy-focused defaults with user override on low-risk settings |
| Compliance Enforcement | Conditional access block on non-compliant devices | Warn + limited access for non-compliant | Notify-only or soft block based on chosen risk level |
| Telemetry | Required security telemetry; minimized optional diagnostics | Basic telemetry with privacy prompts | Minimal telemetry with user opt-in where possible |

---

## 2) Device Compliance Profiles (Windows 10/11)

## 2.1 Core Compliance Requirements by Tier

| Setting | Organization | Family | Personal |
|---|---|---|---|
| Disk Encryption | BitLocker required on OS + fixed drives | BitLocker strongly recommended | Recommend BitLocker if TPM + secure boot available |
| AV/EDR | Defender AV real-time + cloud protection + EDR required | Defender AV required, EDR optional | Defender AV required |
| Firewall | Enabled for domain/private/public profiles | Enabled all profiles | Enabled all profiles |
| SmartScreen | Required for apps/files + browser protection | Enabled | Enabled |
| Least Privilege | No standing local admin | Parent/owner admin only | Owner may retain admin; daily account standard |
| Browser Hardening | Enforce secure DNS, phishing protection, extension controls | Enforce anti-phishing and safe browsing | Apply privacy + anti-phishing defaults |

## 2.2 Recommended Baseline Settings

```powershell
# PowerShell sample baseline checks (can be used in proactive remediation detection)
$checks = [ordered]@{
  BitLockerOSDriveEncrypted = (Get-BitLockerVolume -MountPoint "C:").ProtectionStatus -eq 'On'
  DefenderRealtimeEnabled   = (Get-MpComputerStatus).RealTimeProtectionEnabled
  FirewallEnabled           = ((Get-NetFirewallProfile | Where-Object {$_.Enabled -eq 'True'}).Count -eq 3)
  SmartScreenExplorer       = (Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" -Name SmartScreenEnabled -ErrorAction SilentlyContinue).SmartScreenEnabled
}
$checks
```

---

## 3) Application Lifecycle Management (Chrome, Brave, VLC, Discord)

> Use **WinGet via Intune script/package** where possible for standardization. For strict enterprise packaging, use MSI/EXE as fallback with detection rules.

## 3.1 Google Chrome

- **Package ID:** `Google.Chrome`
- **Install method:** WinGet (preferred), MSI fallback
- **Silent install:**
```powershell
winget install --id Google.Chrome --exact --silent --accept-package-agreements --accept-source-agreements
```
- **Silent uninstall:**
```powershell
winget uninstall --id Google.Chrome --exact --silent
```
- **MSI fallback (example):**
```powershell
msiexec /i "GoogleChromeStandaloneEnterprise64.msi" /qn /norestart
msiexec /x "{PRODUCT-CODE}" /qn /norestart
```
- **Update strategy:** auto-update enabled + monthly validation ring
- **Version pinning/rollback:**
  - Pin by deploying known-good MSI version to pilot/broad groups.
  - Rollback by uninstalling current and redeploying prior tested MSI.
- **Telemetry/privacy hardening:** enforce Safe Browsing, disable third-party sign-in prompts where not needed, control sync and extension installation via policy.

## 3.2 Brave Browser

- **Package ID:** `Brave.Brave`
- **Install method:** WinGet or enterprise MSI
- **Silent install:**
```powershell
winget install --id Brave.Brave --exact --silent --accept-package-agreements --accept-source-agreements
```
- **Silent uninstall:**
```powershell
winget uninstall --id Brave.Brave --exact --silent
```
- **MSI fallback:**
```powershell
msiexec /i "BraveBrowserStandaloneEnterprise64.msi" /qn /norestart
```
- **Update strategy:** allow Brave updater; validate major version changes in pilot
- **Version pinning/rollback:** host tested MSI versions internally; assign older package to rollback group
- **Telemetry/privacy hardening:** enforce aggressive tracker blocking level, disable rewards/wallet/crypto features unless explicitly approved, control extension installs.

## 3.3 VLC Media Player

- **Package ID:** `VideoLAN.VLC`
- **Install method:** WinGet or EXE
- **Silent install:**
```powershell
winget install --id VideoLAN.VLC --exact --silent --accept-package-agreements --accept-source-agreements
```
- **Silent uninstall:**
```powershell
winget uninstall --id VideoLAN.VLC --exact --silent
```
- **EXE fallback:**
```powershell
vlc-<version>-win64.exe /S
```
- **Update strategy:** quarterly review unless critical CVE
- **Version pinning/rollback:** keep approved installer archive; redeploy prior version package
- **Telemetry/privacy hardening:** disable metadata/network privacy options where required; block unnecessary plugin install; restrict file associations in shared devices.

## 3.4 Discord

- **Package ID:** `Discord.Discord`
- **Install method:** WinGet (user-context behavior may vary), EXE fallback
- **Silent install:**
```powershell
winget install --id Discord.Discord --exact --silent --accept-package-agreements --accept-source-agreements
```
- **Silent uninstall:**
```powershell
winget uninstall --id Discord.Discord --exact --silent
```
- **EXE fallback (example):**
```powershell
DiscordSetup.exe /S
```
- **Update strategy:** allow app self-update but monitor version drift and enterprise risk
- **Version pinning/rollback:** limited native support; enforce by controlled redeployment and app execution policy in Organization tier
- **Telemetry/privacy hardening:** disable auto-start if unnecessary, restrict rich presence/integrations on Organization devices, document acceptable-use policy.

## 3.5 Reusable Detection Script (Intune Win32 / Remediation)

```powershell
param(
  [string]$AppDisplayName
)

$installed = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*, HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
  Where-Object { $_.DisplayName -like "*$AppDisplayName*" }

if ($installed) {
  Write-Output "Detected: $($installed.DisplayName) $($installed.DisplayVersion)"
  exit 0
}

Write-Output "Not detected: $AppDisplayName"
exit 1
```

---

## 4) Intune-Style Configuration Templates

## 4.1 Tier Variable Template (assignment model)

```json
{
  "tiers": [
    { "name": "Organization", "group": "MDM-Org", "complianceMode": "Strict" },
    { "name": "Family", "group": "MDM-Family", "complianceMode": "Moderate" },
    { "name": "Personal", "group": "MDM-Personal", "complianceMode": "Balanced" }
  ]
}
```

## 4.2 Proactive Remediation - Install Missing App

```powershell
# remediation-install-app.ps1
param(
  [Parameter(Mandatory = $true)][string]$WingetId
)

$winget = "$env:ProgramFiles\WindowsApps\Microsoft.DesktopAppInstaller_8wekyb3d8bbwe\winget.exe"
if (-not (Test-Path $winget)) { $winget = "winget" }

& $winget install --id $WingetId --exact --silent --accept-package-agreements --accept-source-agreements
exit $LASTEXITCODE
```

## 4.3 Windows Update Ring Guidance

- **Organization:**
  - Quality updates deferral: 7 days
  - Feature updates deferral: 30–60 days
  - Deadline: 7 days
- **Family:**
  - Quality deferral: 14 days
  - Feature deferral: 60–90 days
- **Personal:**
  - Quality deferral: 7–21 days
  - Feature update on user-selected maintenance windows

## 4.4 Browser Hardening Baseline (policy intent)

- Enforce anti-phishing/safe browsing
- Restrict extension installation to approved list (strict for Organization)
- Disable insecure password storage prompts where enterprise password manager is used
- Force HTTPS-first mode where available
- Disable unnecessary telemetry toggles where policy allows

---

## 5) Incident Response and Exception Handling Flow

## 5.1 Failed/Unmanaged Install Workflow

1. **Detect**: Failed install from MDM report (error code, device, user, package).
2. **Classify**:
   - Transient (network, timeout, lock)
   - Dependency/prerequisite missing
   - Policy conflict (app block, rights issue)
   - Unsupported device state
3. **Automated retry**:
   - Retry 3 times over 24 hours.
   - Switch to fallback installer (MSI/EXE) if WinGet repeatedly fails.
4. **Containment**:
   - If required security app failed, mark device non-compliant.
   - Restrict conditional access (Organization strict; Family warning-first).
5. **Manual remediation**:
   - Remote script to clear cache/temp installer artifacts.
   - Re-evaluate policy assignment and user permissions.
6. **Exception path**:
   - Time-bound exception (7/14/30 days based on tier risk).
   - Document owner, reason, compensating controls.
7. **Closeout**:
   - Record root cause category.
   - Add improvement to SOP/playbook.

## 5.2 Exception Register Fields

- Device ID / User / Tier
- App or policy affected
- Business justification
- Risk level + compensating controls
- Approval authority
- Expiration date
- Final resolution date

---

## 6) Reporting and Audit Checklist

## 6.1 Operational Reporting (weekly)

- Device inventory by tier and ownership
- Compliance status by control category
- App deployment success/failure rates
- Failed patch/install trend and top error codes
- Devices not reporting > 7 days

## 6.2 Monthly Audit Summary

- Patch compliance (quality + feature)
- Drift detection (policy vs actual state)
- Encryption/AV/firewall/SmartScreen coverage
- High-risk exception count and overdue exceptions
- Unsupported OS/build versions
- App version spread for Chrome/Brave/VLC/Discord

## 6.3 Drift Detection Signals

- Local admin membership changed unexpectedly
- Required app removed or downgraded
- Browser policy keys modified outside MDM
- Security services disabled (Defender, firewall, SmartScreen)

---

## 7) Phased Rollout Plan

## Phase 0: Preparation (Week 0-1)
- Define tier groups, naming conventions, tagging.
- Package apps and create detection/remediation scripts.
- Build baseline policies and compliance rules.

## Phase 1: Pilot (Week 2-3)
- 5-10 devices per tier.
- Success criteria:
  - >95% app install success
  - >95% compliance convergence within 48 hours
  - No critical user-impacting issues

## Phase 2: Broad Rollout (Week 4-6)
- Expand to 40-60% devices; stagger by department/household segment.
- Daily monitoring of install/compliance errors.
- Enforce conditional access gates for Organization.

## Phase 3: Full Coverage + Steady State (Week 7+)
- 100% target coverage (except approved exceptions).
- Transition to SOP-driven operations and monthly audits.

---

## 8) Maintenance SOP

## Weekly
- Review failed installs/remediations.
- Validate critical app versions.
- Triage non-compliant devices.
- Verify alerting and report freshness.

## Monthly
- Patch and feature update compliance review.
- Review exception register and expire stale exceptions.
- Validate browser hardening and extension controls.
- Sample 5-10 devices per tier for configuration drift.

## Quarterly
- Re-baseline security policies (new threats/features).
- Validate rollback packages and recovery process.
- Reassess tier restrictions and least-privilege model.
- Conduct tabletop incident response drill for MDM outage/app failure.

---

## 9) Governance Notes

- Keep **Organization tier strict by default** with documented exception handling.
- Keep **Family tier balanced** with safety-first controls and usability.
- Keep **Personal tier flexible** while preserving baseline security hygiene.
- Prefer automation with auditable scripts and deterministic detection rules.
