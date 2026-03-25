# GPOHardening
ps1 Scripts for GPO hardening (CIS)
CIS AD GPO Hardening Script
CIS Benchmark hardening for Active Directory environments via PowerShell + GPO.
Targets

Windows Server 2019 / 2022 (Domain Controllers, Member Servers)
Windows 10 / Windows 11 (Workstations)

Baseline
CIS Benchmarks — Level 1 and Level 2 (selectable per run)
Prerequisites

Must run as Domain Admin from a domain-joined machine
RSAT tools installed:

powershell  Add-WindowsFeature RSAT-AD-Tools, GPMC
  # or on workstations:
  Add-WindowsCapability -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0 -Online
  Add-WindowsCapability -Name Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0 -Online

PowerShell 5.1 or later
Must be run elevated (Run as Administrator)


Quick Start
1. Audit mode (no changes — readiness check)
powershell.\Invoke-CISHardening.ps1 -AuditMode
2. Preview all changes (WhatIf)
powershell.\Invoke-CISHardening.ps1 -WhatIf
3. Apply Level 1 hardening
powershell.\Invoke-CISHardening.ps1 -Level L1
4. Apply Level 2 hardening with ASR in Block mode
powershell.\Invoke-CISHardening.ps1 -Level L2 -ASRMode Block
5. Apply specific modules only
powershell.\Invoke-CISHardening.ps1 -Level L1 -Modules AccountPolicy,AuditPolicy,SecurityOptions
6. Rollback all changes
powershell.\Invoke-CISHardening.ps1 -Rollback -RollbackPath .\CISHardening_Output\Rollback\20241201_143022
7. Rollback a specific module
powershell.\Invoke-CISHardening.ps1 -Rollback -RollbackPath .\CISHardening_Output\Rollback\20241201_143022 -RollbackModule TLSHardening

Directory Structure
CISHardening\
├── Invoke-CISHardening.ps1          # Main entry point
├── README.md
│
├── Modules\
│   ├── Core\
│   │   ├── Logging.psm1             # Centralised logging
│   │   ├── RoleDetection.psm1       # DC / MS / WS detection
│   │   └── Snapshot.psm1            # Pre-flight snapshot + rollback
│   │
│   ├── AccountPolicy\               # CIS 1.1-1.3 — Password, lockout, Kerberos
│   ├── AuditPolicy\                 # CIS 17.x — Advanced audit (53 subcategories)
│   ├── SecurityOptions\             # CIS 2.3.x — UAC, SMB, LM auth, LDAP, NTLM
│   ├── UserRights\                  # CIS 2.2.x — Privilege assignments
│   ├── TLSHardening\                # SChannel — TLS 1.2/1.3, cipher suites
│   ├── NetworkHardening\            # LLMNR, NetBIOS, SMBv1, NTLM, routing
│   ├── WinRMHardening\              # CIS 18.9.102.x — WinRM auth hardening
│   ├── ServicesHardening\           # CIS 5.x — Disable legacy services
│   ├── RegistryHardening\           # CIS 18.x — VBS, Credential Guard, PS logging
│   ├── FirewallPolicy\              # CIS 9.x — All 3 firewall profiles
│   ├── DefenderPolicy\              # CIS 18.10.43.x — AV + 16 ASR rules
│   ├── PrivilegedAccess\            # DC-only — Restricted Groups, delegation audit
│   └── HardeningReport\             # Compliance snapshot — Pass/Fail per control
│
├── Profiles\
│   ├── DomainController.json        # DC module list + policy values
│   ├── MemberServer.json
│   └── Workstation.json
│
└── CISHardening_Output\             # Created at runtime
    ├── hardening_<timestamp>.log
    ├── compliance_report_<ts>.json
    └── Rollback\
        └── <timestamp>\
            ├── manifest.json        # Tracked GPO GUIDs for rollback
            ├── GPOBackup\           # Backup-GPO output
            ├── secedit_pre.inf      # Local security DB snapshot
            └── auditpol_pre.csv     # Audit policy snapshot

Module Reference
ModuleCIS SectionRolesReboot?AccountPolicy1.1, 1.2, 1.3AllNoAuditPolicy17.xAllNoSecurityOptions2.3.xAllNoUserRights2.2.xAllNoTLSHardening18.4.x / SCHANNELAllYESNetworkHardening18.5.xAllNoWinRMHardening18.9.102.xAllNoServicesHardening5.xAllNoRegistryHardening18.x / MSSAllPartial*FirewallPolicy9.xAllNoDefenderPolicy18.10.43.xAllNoPrivilegedAccessAD-tierDC onlyNoHardeningReport—AllNo
*RegistryHardening: Credential Guard (LsaCfgFlags) and VBS require reboot + UEFI hardware

Rollback Architecture
Three-layer rollback system captured before any changes:

GPO backup — Backup-GPO -All to Rollback\<ts>\GPOBackup\
secedit snapshot — secedit /export to secedit_pre.inf
auditpol snapshot — auditpol /backup to auditpol_pre.csv

Every GPO GUID created by the script is tracked in manifest.json.
Rollback calls Remove-GPLink + Remove-GPO per tracked GUID, then
restores secedit and auditpol snapshots.
Note: AD object changes (Protected Users group, Restricted Groups)
are NOT automatically rolled back — document current state before Phase 6.

Deployment Phases (recommended)
PhaseModulesAudit Period1Logging, AuditPolicy, FirewallPolicyNone2AccountPolicy, SecurityOptions, UserRightsNotify users 2 weeks prior3LLMNR/NetBIOS (NetworkHardening), WinRMHardeningDNS audit complete4ServicesHardening, RegistryHardeningSpooler survey done5DefenderPolicy (-ASRMode Audit)14 days audit5bDefenderPolicy (-ASRMode Block)After ASR audit complete6SMBv1, TLSHardeningSChannel audit 7+ days7PrivilegedAccessService account mapping done

GPO Naming Convention
All GPOs created by this script follow the pattern:
CIS-<Role>-<Module>-<Level>
Examples:

CIS-DomainController-AccountPolicy-L1
CIS-MemberServer-TLSHardening-L1
CIS-Workstation-DefenderPolicy-L2


Important Caveats

TLS changes require a reboot — schedule a maintenance window
ASR Block mode will break RMM agents — use Audit mode first (14 days minimum)
SMBv1 disable will break legacy NAS/MFP devices — audit first with Get-SmbSession
Print Spooler script pre-checks for active queues and skips if found on Member Servers
Credential Guard requires UEFI Secure Boot + VT-x/AMD-V hardware
Protected Users group and Restricted Groups require manual AD population — the script documents the required action but does not blindly remove group members


Version History
VersionDateNotes1.02024Initial release — 13 modules, 158+ CIS controls
