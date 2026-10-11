# Root Cause Analysis  
## Windows 11 feature-update rollback caused by legacy IXXAT VCI V3 boot driver

**Document purpose:** Prevent repeated Windows 11 feature-update failures on engineering and service laptops.

**Affected failure:** Windows 11 feature upgrade from 23H2 to later releases.

**Primary error:** `0xC1900101 - 0x40017`

**Primary component:** IXXAT VCI V3 `vcisrv.sys`

**Root-cause confidence:** High

**Final verified target:** Windows 11 Enterprise 26H2, build `26300.9457`

---

## 1. Executive summary

A long-lived engineering laptop repeatedly failed Windows 11 feature updates.

The Windows installation had existed since the Windows 7 generation. It had been migrated through multiple Windows versions and multiple laptop hardware generations. It contained many industrial automation, PLC, HMI, CAN, VPN, virtualization, and support applications.

More than 20 update attempts were made. Many attempts required several hours. Some attempts ran overnight. Extensive troubleshooting included BitLocker changes, TPM work, service changes, driver removal, and removal of industrial applications.

The final recurring failure was:

```text
0xC1900101 - 0x40017

SECOND_BOOT / BOOT
FirstBoot / End
```

Windows loaded the new operating system, failed to complete the new-OS boot sequence, retried the boot, and then restored Windows 11 23H2.

The high-confidence root cause was an old **IXXAT VCI V3 kernel driver**:

```text
Service:         vcisrv
Driver:          C:\Windows\System32\drivers\vcisrv.sys
Product:         VCI 3
Product version: 3.5.4.4543
File date:       2016
Service type:    KERNEL_DRIVER
Start type:      BOOT_START
Load group:      Base
Vendor:          IXXAT Automation GmbH
```

The laptop could run Windows 11 23H2 with this driver. Therefore, normal operation on 23H2 did not show the compatibility problem.

The driver was disabled before the next Windows boot:

```powershell
sc.exe config vcisrv start= disabled
```

The laptop then:

1. Booted Windows 11 23H2 normally.
2. Upgraded successfully to Windows 11 25H2.
3. Upgraded successfully from 25H2 to Windows 11 26H2.

The final 26H2 Setup log reported:

```text
SetupHost: Reporting post reboot event -> [0x0, 0x0]

SetupPlatform: Phase "Pre-OOBE" completed successfully.
SetupPlatform: Phase "Post-OOBE" completed successfully.
SetupPlatform: Phase "Post-OOBEBoot" completed successfully.
SetupPlatform: Phase "OOBEBoot" completed successfully.

Execution phase [Post OOBE] exiting with HRESULT [0x00000000]
```

The final operating system is:

```text
Windows 11 Enterprise
DisplayVersion: 26H2
Build:          26300.9457
```

---

# 2. Root cause

## 2.1 Primary root cause

The most probable root cause of the repeated `0xC1900101-0x40017` FirstBoot failures was the legacy **IXXAT VCI V3 `vcisrv.sys` boot-start driver**.

The installed version was:

```text
VCI 3
3.5.4.4543
```

HMS currently lists this exact VCI V3 version for **Windows 8, Windows 7, Windows XP, Windows 2000, and Windows Vista**. HMS lists VCI V4 as the driver for Windows 10 and Windows 11. HMS also states that VCI V3 must not be installed on Windows 10 systems with Secure Boot enabled and directs users to VCI V4. [HMS Networks](https://www.hms-networks.com/p/1-01-0088-90002-usb-to-can-compact-rj45?utm_source=chatgpt.com)

Microsoft documents `0xC1900101-0x40017` as a feature-update boot failure for which incompatible drivers are a known cause. [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/windows-10-upgrade-resolution-procedures?utm_source=chatgpt.com)

There is no HMS document that we found that specifically says:

> VCI V3 causes `0xC1900101-0x40017` when Windows 11 23H2 is upgraded to 25H2 or 26H2.

Therefore, do not present that exact failure mode as a published HMS known issue.

The conclusion in this RCA is based on the affected computer's repeatable A/B result.

---

# 3. Evidence for the root cause

The evidence is stronger than a simple correlation.

### Test A — EaseUS removed, VCI V3 still enabled

A stale EaseUS storage filter was found and neutralized.

The following drivers were disabled:

```text
EUDCPDC
EUEDKDC
```

`EUDCPDC` was also removed from the Windows Volume-class `LowerFilters` value.

The clean value became:

```text
fvevol
iorate
rdyboost
```

Windows 11 25H2 was then attempted again.

**Result:**

```text
FAILED
0xC1900101 - 0x40017
FirstBoot / End
```

Therefore, the EaseUS filter did not explain the remaining FirstBoot failure.

### Test B — VCI V3 disabled

The following legacy service was then found:

```text
SERVICE_NAME: vcisrv
TYPE:         KERNEL_DRIVER
START_TYPE:   BOOT_START
GROUP:        Base
```

The driver file reported:

```text
Product:         VCI 3
ProductVersion:  3.5.4.4543
FileVersion:     1.0.4.4543
File date:       2016
```

The driver was disabled:

```powershell
sc.exe config vcisrv start= disabled
```

The computer was restarted.

Windows 11 23H2 started normally.

### Test C — 23H2 to 25H2

No additional broad driver or industrial-software cleanup was done.

**Result: Windows 11 25H2 installed successfully.**

### Test D — 25H2 to 26H2

The same driver remained disabled.

Correct Windows 11 **Business Editions / Enterprise** installation media was used.

**Result: Windows 11 26H2 installed successfully.**

This two-stage success substantially increases confidence that `vcisrv` was the remaining FirstBoot blocker.

---

# 4. Failure mechanism

The exact kernel-level failure mechanism was not captured.

The failed 25H2 and 26H2 installations did **not** produce a useful bugcheck that directly named `vcisrv.sys`.

The failed NewOS boot sequence showed this general behavior:

```text
NewOS launched
↓
NewOS did not complete boot
↓
No crash detected
↓
Windows Recovery retried
↓
Maximum recovery attempts reached
↓
Rollback
↓
0xC1900101 - 0x40017
```

The failure occurred early enough that the failed NewOS did not produce useful normal Windows event-log records for the failed boots.

Therefore:

**Do not state that a specific instruction or exception inside `vcisrv.sys` was proven.**

State that the legacy VCI V3 boot driver was the **high-confidence causal component based on controlled isolation and successful subsequent upgrades**.

---

# 5. Fleet-prevention procedure

Before a Windows 11 feature update, check engineering and service computers for IXXAT VCI.

Run:

```powershell
sc.exe qc vcisrv
```

If the service exists, check the file:

```powershell
$p = 'C:\Windows\System32\drivers\vcisrv.sys'

if (Test-Path $p) {
    Get-Item $p | Select-Object `
        FullName,
        @{n='Product';e={$_.VersionInfo.ProductName}},
        @{n='ProductVersion';e={$_.VersionInfo.ProductVersion}},
        @{n='FileVersion';e={$_.VersionInfo.FileVersion}},
        LastWriteTime
}
```

A computer requires review if the result is similar to:

```text
Product:        VCI 3
ProductVersion: 3.x
Service:        vcisrv
START_TYPE:     BOOT_START
```

### Recommended disposition

**Preferred action:** Upgrade the applicable IXXAT software and hardware environment to a supported VCI V4 release.

HMS currently identifies VCI V4 as the Windows 10/11 driver family. [HMS Networks](https://www.hms-networks.com/p/1-01-0088-90002-usb-to-can-compact-rj45?utm_source=chatgpt.com)

Do not assume that all old IXXAT hardware or all old engineering applications support VCI V4.

Confirm the hardware and software dependency before removal.

---

# 6. Temporary upgrade workaround

Use this only after the responsible engineering owner confirms that VCI3 is not required during the upgrade.

Back up the service configuration:

```powershell
reg.exe export `
  "HKLM\SYSTEM\CurrentControlSet\Services\vcisrv" `
  "C:\Temp\vcisrv-before.reg" /y
```

Disable the driver:

```powershell
sc.exe config vcisrv start= disabled
```

Restart Windows.

Verify:

```powershell
sc.exe query vcisrv
sc.exe qc vcisrv
```

The expected result is:

```text
STATE      : STOPPED
START_TYPE : DISABLED
```

Confirm that the existing Windows installation starts normally.

Then start the feature update.

**CAUTION:** Do not manually delete `vcisrv.sys` as the first action.

**CAUTION:** Do not disable VCI3 on a production engineering computer until its CAN-interface dependency is known.

---

# 7. Actions that did not solve the final failure

Do not start with broad system removal or broad service disabling.

During this investigation, the following actions either did not solve the final FirstBoot failure or were not shown to be causal:

- Disable or change BitLocker configuration.
- Clear or reset the TPM as a generic repair.
- Remove TwinCAT.
- Remove Siemens/SIMATIC engineering components.
- Remove large groups of industrial network drivers.
- Disable large groups of services.
- Remove virtualization components without specific evidence.
- Remove VPN software without specific evidence.
- Remove all old-looking third-party drivers.
- Repeatedly run SFC/DISM when component corruption is not indicated.

These actions can make an engineering laptop unusable for normal work and can add new variables to the investigation.

**Do not treat the presence of a driver name in `setupact.log` as proof that the driver caused the failure.**

---

# 8. TwinCAT and Siemens findings

TwinCAT and Siemens components appeared repeatedly in Windows Setup logs.

This did not prove that they caused the rollback.

The investigation initially gave too much weight to these entries.

No controlled evidence identified TwinCAT as the cause of the final FirstBoot problem.

No controlled evidence identified the SIMATIC network protocol drivers as the cause of the final FirstBoot problem.

After the investigation changed focus to the VCI3 boot driver, the upgrade succeeded without another broad TwinCAT or Siemens purge.

**Fleet recommendation:** Do not remove supported industrial engineering software only because its migration entries appear in Windows Setup logs.

---

# 9. Separate TPM migration problem

This computer also had an **earlier and different failure**.

An earlier 24H2 attempt failed in:

```text
SAFE_OS
MIGRATE_DATA
```

with:

```text
0x8007042B - 0x2000D
```

The logs identified:

```text
Microsoft-Windows-TPM-Driver-WMI

Source migration unit ... is not supported on the destination machine

Source migration unit is critical, arbitration will fail

V2V Arbitration failed
Last error: 0x00000032
```

This failure occurred before the later `0xC1900101-0x40017` FirstBoot problem.

Later 25H2/26H2 media passed this migration stage.

Therefore:

> **Do not combine the TPM V2V failure and the VCI3 FirstBoot failure into one root cause.**

They were two different failure families on the same long-lived Windows installation.

---

# 10. EaseUS finding

The system also contained stale EaseUS Todo Backup drivers:

```text
EUDCPDC.sys
EUEDKDC.sys
```

`EUDCPDC` was configured as a `BOOT_START` driver and was present in the Volume-class `LowerFilters` chain.

The configuration was cleaned.

The final system state is:

```text
EUDCPDC : STOPPED / DISABLED
EUEDKDC : STOPPED / DISABLED

Volume LowerFilters:
fvevol
iorate
rdyboost
```

This was valid system cleanup.

However, Windows 25H2 still failed after the EaseUS components were neutralized and before VCI3 was disabled.

Therefore:

> **EaseUS was stale boot-path residue, but it was not sufficient to explain the final recurring FirstBoot failure.**

---

# 11. Final verification

The final successful system reports:

```text
DisplayVersion     : 26H2
CurrentBuild       : 26300
CurrentBuildNumber : 26300
UBR                : 9457
```

Final relevant driver state:

```text
vcisrv   : STOPPED / DISABLED
EUDCPDC  : STOPPED / DISABLED
EUEDKDC  : STOPPED / DISABLED
```

Final Volume filters:

```text
fvevol
iorate
rdyboost
```

Windows Setup reported:

```text
Post reboot event -> [0x0, 0x0]

Pre-OOBE completed successfully
Post-OOBE completed successfully
Post-OOBEBoot completed successfully
OOBEBoot completed successfully

Post OOBE HRESULT = 0x00000000
```

This confirms a successful 26H2 upgrade and finalization.

---

# 12. Recommended IT policy

For engineering computers with a long Windows migration history, add a **legacy boot-driver inventory** to the feature-update readiness process.

Give special attention to:

```text
Type  = KERNEL_DRIVER
Start = BOOT_START or SYSTEM_START
```

Do not classify a driver as compatible only because the existing Windows release boots with it.

A legacy boot driver can operate normally for years and still prevent the next Windows release from completing Setup.

For IXXAT specifically:

> **Block or review Windows 11 feature-update deployment when VCI V3 is detected.**

Determine whether the computer requires IXXAT CAN hardware.

Move to VCI V4 when the hardware and engineering application support it.

HMS lists VCI V3 `3.5.4.4543` for legacy Windows releases and lists VCI V4 for Windows 10/11. [HMS Networks](https://www.hms-networks.com/p/1-01-0088-90002-usb-to-can-compact-rj45?utm_source=chatgpt.com)

---

# 13. Recommended help-desk search terms

Add these terms to the internal knowledge base:

```text
IXXAT
VCI3
VCI 3
vcisrv
vcisrv.sys
3.5.4.4543
0xC1900101
0x40017
SECOND_BOOT
FirstBoot
Windows 11 23H2
Windows 11 24H2
Windows 11 25H2
Windows 11 26H2
feature update rollback
```

Suggested article title:

**Windows 11 Feature Update Rollback 0xC1900101-0x40017 — Legacy IXXAT VCI3 / vcisrv.sys**

---

## Final RCA statement

> **The high-confidence root cause of the recurring Windows 11 FirstBoot rollback was a legacy IXXAT VCI V3 3.5.4.4543 `vcisrv.sys` kernel driver configured as BOOT_START. The existing Windows 11 23H2 installation tolerated the driver during normal operation, but later Windows feature upgrades repeatedly failed with `0xC1900101-0x40017`. After `vcisrv` was disabled and the computer was verified to boot normally, the upgrade to Windows 11 25H2 succeeded. The subsequent upgrade from 25H2 to Windows 11 26H2 also succeeded with the driver still disabled. HMS lists this VCI V3 release for legacy Windows versions and identifies VCI V4 as the Windows 10/11 driver family. The exact kernel failure mechanism was not captured, so the driver should be described as the high-confidence causal component rather than as a vendor-confirmed 25H2/26H2 defect.**
