# Navigation

Every tab and sub-tab of this workbook, with the description that used to sit at
the top of each one. The data tabs now open straight onto their content; this
page is the single place where the intent, the scope and the caveats of each
section are documented.

- **Overview** and **Audit** read the audit streams: `IntuneAuditLogs` and the Entra ID `AuditLogs` directory audit.
- **Inventory** reads the snapshot exports: `IntuneDevices`, `IntuneOperationalLogs`, `IntuneDeviceComplianceOrg` and the Windows Update for Business reports tables.
- **About** holds this page, the reading notes and the sources.

Parameter scope: `TimeRange` is global. `Actor` and `Operation` are also declared
globally, so they still constrain every audit query, but the block is only
displayed on the Audit tab. Sub-tab filters (`TopN`, `FocusOperation`, `OS`,
`MobileOS`, `ComplianceOS`, `DOGroupId`, `DOContentType`) are local to their own
section.

**Workspace scope.** Every table this workbook reads is grouped into one of three buckets, each with its own Subscription + Workspace picker at the top of the page: **Intune logs** (`IntuneAuditLogs`, `IntuneDevices`, `IntuneDeviceComplianceOrg`, `IntuneOperationalLogs`), **Entra ID logs** (`AuditLogs`), and **Windows Update reports** (`UCClient`, `UCUpdateAlert`, `UCDOStatus`). All three default to this tenant's own workspace, so nothing changes if you never touch them; another organization whose logs are split across separate workspaces (or subscriptions) can repoint any bucket independently. Five items read across two buckets in a single query (`Actor`, LAPS, BitLocker, DO devices, ingestion health): the secondary table is referenced explicitly with `workspace('{BucketWorkspace}').Table`, since cross-component resources union the *same* table across resources rather than let a join pull two *different* tables from two different workspaces.

## Table of contents

- [Overview](#overview)
- [Audit](#audit)
  - [Assignments](#assignments)
  - [BitLocker](#bitlocker)
  - [Configuration](#configuration)
  - [Device actions](#device-actions)
  - [LAPS](#laps)
  - [PIM](#pim)
  - [Raw audit log](#raw-audit-log)
  - [Top actions](#top-actions)
  - [Wipe / Retire / Reset](#wipe--retire--reset)
- [Inventory](#inventory)
  - [Delivery Optimization](#delivery-optimization)
  - [Device inventory](#device-inventory)
    - [All devices](#all-devices)
    - [Autopilot builds](#autopilot-builds)
    - [Compliance](#compliance)
    - [Disk status](#disk-status)
    - [Mobile OS currency](#mobile-os-currency)
    - [Windows Update errors](#windows-update-errors)
    - [Windows versions](#windows-versions)
- [About > Navigation](#about--navigation)
- [About > Data feeds](#about--data-feeds)
- [About > Reading notes](#about--reading-notes)
- [About > Sources](#about--sources)

---

### Overview

Volume of administrative actions over **{TimeRange:label}**. Counters are de-duplicated by `AuditEventId`.

---

### Audit

Everything sourced from `IntuneAuditLogs` and, for LAPS and BitLocker, the Entra ID `AuditLogs` directory audit. The **Actor** and **Operation** filters at the top of the page belong to this tab and propagate to every section below.

---

#### Assignments

Group added to or removed from a policy or an app. `Target.GroupId` is a raw GUID: the group display name is not present in this log. Join with an Entra ID export to resolve it.

---

#### BitLocker

BitLocker recovery-key read requests, from the `AuditLogs` table (Entra ID directory audit, `Read BitLocker key`). Unlike LAPS, this event does not carry a device display name, only the device's Entra object ID; it is resolved against the `IntuneDevices` inventory the same way the Wipe and Device actions tabs resolve theirs. The bottom table flags actors who read keys for 5 or more distinct devices within a single hour, the same harvesting signal as LAPS.

The local **Actor type** filter (`User` / `Application` / `Unknown`) defaults to `User`, so automated backup or key-escrow service accounts reading keys in bulk don't drown out human activity. Switch it to select all to include them.

Neither the recovery key ID nor the requester's source IP address is extracted by this tab's queries.

> The **Operation** filter does not apply to this tab: it targets `IntuneAuditLogs` operations. The **Actor** filter applies normally.

---

#### Configuration

Policy and setting modifications (Create / Update / Delete / Patch), excluding assignments, `Sync*` jobs, device actions, `decrypt`, certificates and Autopilot noise. The `DeviceManagementAPIVersion` property is filtered out: present on almost every event, no analytical value.

---

#### Device actions

Actions pushed to managed devices, enriched with the `IntuneDevices` inventory. The inventory is a snapshot table (one row per device per day) and is de-duplicated with `arg_max` before the join: without that, every audit event would be multiplied by the number of retained snapshot days. The join is a left outer so that actions on devices no longer present in the inventory remain visible.

**What this tab matches**

Every `IntuneAuditLogs` operation whose name contains `ManagedDevice`, minus
`Delete ManagedDeviceMobileAppConfiguration`, which is a configuration object rather
than a device. In practice that covers three families.

| Family | Typical operations | Meaning |
|---|---|---|
| Object lifecycle | `Create ManagedDevice`, `Delete ManagedDevice` | Enrolment and removal of the device record. |
| Object edits | `Update ManagedDevice`, `Patch ManagedDevice` | Changes to the device record itself: rename, primary user reassignment, device category, ownership, notes. |
| Remote commands | `wipe`, `retire`, `cleanWindowsDevice`, `remoteLock`, `resetPasscode`, `rebootNow`, `syncDevice`, and the rest of the action surface | A command pushed to the device. The audit event records that the command was *submitted*, not that it succeeded on the endpoint. |

**Reading caveats**

- **`Sync*` is not excluded here**, unlike the Overview, Configuration, Top actions and
  Raw audit log tabs. Sync is by far the highest-volume device operation, so it will
  dominate the *Top 10 device operations* chart unless you filter it out with the
  global **Operation** picker.
- `ResultType` reflects the **service-side** outcome. A device that never checks in
  again keeps a pending command indefinitely, and a Retire or Wipe stays pending until
  the MDM certificate expires, up to a year after enrolment.
- Actions the user triggers themselves from the Company Portal appear with no
  initiating admin. In the Intune admin center these show an empty *Initiated By*; here
  they fall back to the raw `Identity` value.

**References**

- [Device actions overview, per-platform matrix and bulk actions](https://learn.microsoft.com/en-us/intune/device-management/actions/)
- [Sync](https://learn.microsoft.com/en-us/intune/device-management/actions/sync) - [Restart](https://learn.microsoft.com/en-us/intune/device-management/actions/restart) - [Rename](https://learn.microsoft.com/en-us/intune/device-management/actions/rename) - [Collect diagnostics](https://learn.microsoft.com/en-us/intune/device-management/actions/collect-diagnostics)
- [Rotate BitLocker keys](https://learn.microsoft.com/en-us/intune/device-management/actions/rotate-bitlocker-keys) - [Run remediation](https://learn.microsoft.com/en-us/intune/device-management/actions/run-remediation) - [Locate device](https://learn.microsoft.com/en-us/intune/device-management/actions/locate)
- [Quick scan](https://learn.microsoft.com/en-us/intune/device-management/actions/quick-scan) - [Full scan](https://learn.microsoft.com/en-us/intune/device-management/actions/full-scan)
- [Troubleshoot device actions](https://learn.microsoft.com/en-us/troubleshoot/mem/intune/device-management/troubleshoot-device-actions)

---

#### LAPS

Windows LAPS local administrator password recovery requests, from the `AuditLogs` table (Entra ID directory audit, `Recover device local administrator password`). The bottom table flags actors who requested passwords for 5 or more distinct devices within a single hour (threshold adjustable in the query), useful for spotting password harvesting or lateral movement recon.

> The **Operation** filter does not apply to this tab: it targets `IntuneAuditLogs` operations. The **Actor** filter applies normally.

---

#### PIM

Entra ID Privileged Identity Management role activations, from `AuditLogs` where
`LoggedByService == "PIM"`. The tab answers two questions: who elevated into which role,
and whether they gave a usable reason for it.

**Field resolution.** Two things in this schema are index-dependent and were rewritten:

- The role sits in `TargetResources`, but **not always at index 0**: the array layout
  differs between role assignments and group assignments. Role, target user and scope are
  resolved by `type` (`Role`, `User`, everything else) with `make_set_if`, not by position.
- The justification has three possible homes depending on tenant and event version:
  `AdditionalDetails["Justification"]`, `ResultReason`, or `ResultDescription`. Published
  queries pick one and each is right somewhere. This tab `coalesce`s all three rather than
  betting on one, and `AdditionalDetails` is folded into a bag with `make_bag` so the
  activation window, ticket reference and source IP are read **by key**, not by array index.

**Justification quality** is bucketed as `Provided`, `Too short` (under 15 characters) or
`Missing`. The threshold is a `let` at the top of each query. The *Justifications, grouped*
grid normalises reasons to lowercase and collapses punctuation, so `RFC 1234`, `rfc-1234`
and `rfc 1234` land on one row: that is where recurring copy-paste justifications become
visible.

**Off hours** means before 07:00 or after 19:00 **UTC**, or a weekend. If your admins sit
in a single timezone, shift `TimeGenerated` before `hourofday()` rather than reading the
tile as-is.

**Scope notes**

- The role list is a workbook parameter built from your own data, not a hard-coded list.
  Leave it on the *All* pill to see every role.
- The global **Actor** filter applies to the requestor. The global **Operation** filter does
  not: it targets `IntuneAuditLogs` operations, and this tab reads `AuditLogs`.
- The KPI tiles and grids cover completed activations only. The *PIM operation mix* chart
  is deliberately wider: it shows every PIM event in range, so requested, approved, denied
  and deactivation events stay visible next to the completions.

**References**

- [Microsoft Entra audit log schema](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/auditlogs)
- [View activity and audit history for Privileged Identity Management](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-use-audit-log)
- [Activate Microsoft Entra roles in PIM](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-activate-role)
- Query lineage: [reprise99, Audit-NewPIMRoleActivated](https://github.com/reprise99/Sentinel-Queries/blob/main/Azure%20Active%20Directory/Audit-NewPIMRoleActivated.kql), the starting point for the target and detail extraction

---

#### Raw audit log

Every `IntuneAuditLogs` event in range, without normalisation or exclusion, apart from `Sync*` operations. Use the column filters for ad hoc search, or the Excel export for offline analysis.

Azure Monitor workbooks cap grid results at 2,000 rows. On a large tenant with a wide time range, the grid can silently truncate at that limit with no visible warning; narrow **{TimeRange:label}** or use the column filters to stay under it rather than assuming an empty result set means no matching events.

---

#### Top actions

Ranking of Intune operations over **{TimeRange:label}**, excluding `Sync*` operations (service volume, no audit value). Pick an operation in the **Focus** filter to drill into the underlying events.

---

#### Wipe / Retire / Reset

Destructive and remote actions on managed devices, plus device object deletions in Intune (for reference: a deletion is not necessarily a wipe).

**Device name resolution.** On these events `TargetDisplayNames[0]` carries the *action* label, not the device name, so the device name is resolved by joining `TargetObjectIds[0]` against a de-duplicated `IntuneDevices` snapshot (30-day lookback, `arg_max` per `DeviceId`). The join is a **left outer**: a device wiped then deleted disappears from the inventory, and an inner join would silently hide exactly the events that matter. Unresolved rows are kept and flagged in the `NameResolution` column.

The bottom table flags actors reaching 3 or more Wipe/Retire actions within a 15-minute window (threshold adjustable in the query), useful for spotting a compromised account or an unintended bulk wipe.

**What each action actually does**

The `DestructiveOps` list in these queries is deliberately a **superset**: it collects
every *sensitive* remote action, not only the ones that destroy data. Three of them
remove nothing at all and are here purely for correlation, because a passcode reset
followed twenty minutes later by a wipe reads very differently from either on its own.

| Shown as | Matched on | Data loss | What Intune actually does |
|---|---|---|---|
| Wipe | `wipe`, `eraseDevice` | Total | Factory reset. Removes all data, apps and settings. On Windows, enrolment can optionally be retained. |
| Retire | `retire` | Company data only | Removes company data, apps and settings, leaves personal data intact. The device leaves management at its next check-in, so it stays visible in Intune until then. |
| Device deletion | `Delete ManagedDevice` | Company data only | Removes the device from Intune immediately and triggers a Retire on the client. A status of *Completed* means the server side is done, not the device. |
| Fresh Start | `cleanWindowsDevice`, `freshStart` | Apps, optionally user data | Reinstalls the latest Windows and removes manufacturer-installed apps. Used to de-bloat or repurpose a device rather than to secure it. |
| Remote lock | `remoteLock` | **None** | Locks the device screen. Reversible. Requires a passcode policy assigned to the device. |
| Passcode reset | `resetPasscode` | **None** | Issues a new passcode so a locked-out user gets back in without a wipe. On Android, distinguishes a device-level reset from a work-profile reset. |
| Disable | `disable` | **None** | Blocks the device, typically tied to activation lock. Reversible. |

**Two operational facts worth knowing when reading this tab**

- **Retire, Wipe and Delete take precedence over every other pending action.** A device
  carrying several pending commands executes only one of those three and silently drops
  the rest. A remote lock logged just before a wipe may therefore never have run.
- Intune enforces **daily tenant limits**: 500 Wipe, 1,000 Retire, 1,000 Delete,
  cumulative across the admin center, bulk actions and Graph. A burst that stalls at a
  round number is likely to be a quota, not a failure.

**Known gap.** *Autopilot reset* is not matched by the operation names above. If your
tenant emits it under a distinct operation, add it to `DestructiveOps`; until then it
will not appear on this tab even though it removes personal files, apps and settings,
and clears the primary user and the Entra device owner.

**References**

- [Device actions overview, per-platform matrix and daily limits](https://learn.microsoft.com/en-us/intune/device-management/actions/)
- [Wipe](https://learn.microsoft.com/en-us/intune/device-management/actions/wipe) - [Retire](https://learn.microsoft.com/en-us/intune/device-management/actions/retire) - [Delete](https://learn.microsoft.com/en-us/intune/device-management/actions/delete)
- [Fresh Start](https://learn.microsoft.com/en-us/intune/device-management/actions/fresh-start) - [Autopilot reset](https://learn.microsoft.com/en-us/intune/device-management/actions/autopilot-reset)
- [Reset passcode](https://learn.microsoft.com/en-us/intune/device-management/actions/reset-passcode)
- [Troubleshoot device actions](https://learn.microsoft.com/en-us/troubleshoot/mem/intune/device-management/troubleshoot-device-actions), for pending states and MDM certificate expiry

---

### Inventory

Fleet state rather than administrative activity. Sources: `IntuneDevices`, `IntuneOperationalLogs`, `IntuneDeviceComplianceOrg` and the Windows Update for Business reports tables. The **Actor** and **Operation** filters do not apply here; each section carries its own filters where relevant.

---

#### Delivery Optimization

Bandwidth split between CDN, peers and Microsoft Connected Cache, from `UCDOStatus` (Windows Update for Business reports).

**P2P here includes Internet peers.** The percentage is `(BytesFromPeers + BytesFromGroupPeers + BytesFromIntPeers) / Total`, where the denominator also carries `BytesFromCDN` and `BytesFromCache`. Counting only LAN and group peers, as most published queries do, understates peering wherever `DODownloadMode` is set to 3 (Internet).

`GroupID` is an opaque hash derived from the Delivery Optimization group (`DOGroupID` policy, AD site, domain, or subnet depending on `DOGroupIdSource`). It cannot be resolved to a site name from this table: pick one in the filter below and correlate with your own DO policy assignment.

**MCC server usage.** The *MCC server usage* table breaks the same data down by `CacheNodeId`, the Microsoft Connected Cache server that served the content, using the `CacheNodeDetails` dynamic array (cache node identifier, date and bytes served, one entry per node a device downloaded from). Adapted from Microsoft's own `DeliveryOptimization.workbook`, which uses the same `mv-expand` pattern to aggregate by cache node. `CacheNodeDetails`'s own sub-fields (`Id`, `Date`, `Bytes`) aren't individually itemised in the `UCDOStatus` schema reference beyond "an array of cache node details", so they're verified by construction against that workbook rather than an independent schema page.

**Ingestion caveat.** WUfB reports refresh a full snapshot every 24 h for devices seen in the last **28 days**, so a time range beyond 28 days will not return older data, and the same device can be re-ingested on consecutive days even with no new download. Read the percentages as the shape of the traffic, not as an exact byte ledger.

---

#### Device inventory

Fleet state derived from the Intune device and compliance exports, the Intune operational log, and Windows Update for Business reports. Unlike the audit tabs, most sections here read a **fixed 30-day inventory lookback** that is deliberately independent of **{TimeRange:label}**: these are snapshot tables (one row per device per day), so a short range would contain no snapshot at all. Sections that are genuinely time-series (Autopilot builds, version evolution, update errors) do honour the global time range.

> The global **Actor** and **Operation** filters do not apply here: they target `IntuneAuditLogs`, and none of these sections read that table.

---

##### All devices

KPI tiles and a trend chart. The KPI tiles count distinct devices per major OS family (Windows, iOS/iPadOS, Android, macOS); devices on any other or unrecognised OS are still counted in the "All devices" total and visible in the chart, just without their own tile. The chart plots device count per day per OS, from a de-duplicated `IntuneDevices` snapshot.

---

##### Autopilot builds

Successful Autopilot provisioning completions over **{TimeRange:label}**, from `IntuneOperationalLogs` / `ESPEnrollment`. Only `DeviceProvisioningProgressEvent` events with scope `DeviceEnrollmentSuccess` and `IsAutopilot = true` are kept, so one row equals one completed build, not one ESP step.

The device name is not present in the operational log: it is resolved by a left outer join against a de-duplicated `IntuneDevices` snapshot (30-day lookback). A device built and then deleted stays visible, flagged `<unresolved ...>`.

The **detail table** keeps only the **latest build per device** (`arg_max`), while the KPI tiles and the trend chart count **every build event**: a device rebuilt three times shows once in the table and three times in the counters.

This section also tracks the Enrollment Status Page (ESP) itself: per-build end-to-end duration and success/failure outcome, read from the same `EspSlaIsDeviceSyncDone` event within `ESPEnrollment`. Success is matched on `Status =~ "Completed"`, the only value Microsoft documents; anything else is bucketed as `Failure`.

---

##### Compliance

Compliance state from `IntuneDeviceComplianceOrg`, de-duplicated on `DeviceId` (not `DeviceName`, which is not unique and is sometimes reported as `User deleted for this device`) over a 30-day lookback.

`InGracePeriodUntil` is a **string** in this schema. It is cast with `todatetime()` and a device counts as in grace only when the date is in the future *and* below year 9000: Intune uses `9999-12-31` as the sentinel for "no grace period", which would otherwise make every device look like it is in grace.

The trend chart honours **{TimeRange:label}** and de-duplicates one row per device per day; the tiles and the breakdown read the 30-day snapshot.

---

##### Disk status

Free space on Windows devices, from the latest `IntuneDevices` snapshot per device over a 30-day lookback. De-duplication is done on `DeviceId` rather than `SerialNumber`: serial numbers are not guaranteed unique across manufacturers and are sometimes blank on virtual machines.

**Thresholds:** `Critical` below 10 GB free, `Warning` below 25 GB, `OK` above. Both are `let` constants at the top of each query, alongside `StorageUnitToGB`. That last one matters: Microsoft does not document the unit of `StorageFree` / `StorageTotal` (the schema just says `long`). This tenant reports **MB**, hence `1024.0`. If your Free_GB values come out roughly a million times too large, the export is in bytes and the constant should be `1073741824.0`.

---

##### Mobile OS currency

Answers a single question: **is this device on the newest OS version any device of the same model has reported?** The tenant is used as its own reference, so no external version feed is needed, at the cost of one assumption: at least one device per model must already be up to date.

Versions are ranked with `parse_version()`, not by string comparison. That matters: lexicographically `"9.5"` sorts above `"18.5"`, which would silently mark an entire model as up to date on a version two years old.

Scope: last record per `DeviceId` over 30 days, `DeviceState = Managed` only. Use the **Operating system** filter below to restrict to Android and iOS/iPadOS; Windows is included by default but the per-model logic is far less meaningful there (use the Windows versions section instead).

---

##### Windows Update errors

Active `Error` alerts from **Windows Update for Business reports** (`UCUpdateAlert`), enriched with device context from `UCClient`. One row per device and per alert subtype, keeping the most recent occurrence.

`UCClient` is a daily snapshot, so it is de-duplicated with `arg_max` per `AzureADDeviceId` before the join, over a fixed 30-day lookback. WUfB reports only refresh devices seen in the **last 28 days**, so a time range beyond that returns nothing older, and `Recommendation` is the remediation text Microsoft ships with the alert, not a local judgement.

---

##### Windows versions

Feature-update spread across the Windows fleet. The release name is derived from the **third octet** of `OSVersion` (`10.0.<build>.<revision>`) rather than a `startswith` match, so monthly revisions never break the mapping. The build map is an inline `case()` in each query rather than a `let` lambda: a lambda body is wrapped in braces, and braces are the workbook parameter-substitution syntax.

Build 28000 is **Windows 11 26H1**, a new-device-only release that is *not* offered as an in-place upgrade, which is why its build number sits above 26H2 (26300). LTSC editions cannot be distinguished from `OSVersion` alone: 17763 covers both 1809 and LTSC 2019, 19044 both 21H2 and LTSC 21H2, 26100 both 24H2 and LTSC 2024.

KPI tiles and the distribution chart read the **30-day inventory snapshot**; the evolution chart honours **{TimeRange:label}** and de-duplicates one row per device per day before counting, otherwise multiple snapshots in a single day would inflate every point.

---

### About > Navigation

This page, mirrored inside the workbook itself: the same per-tab intent, scope and
caveats, kept in one place instead of repeated at the top of every tab.

---

### About > Data feeds

Ingestion freshness for every table the workbook reads, over a fixed seven-day window
independent of `{TimeRange}`. A section that renders empty is almost always a feed
problem rather than a query problem, so check here first.

`Feed lagging` means the most recent row is more than three days old. Tables that have
never ingested anything are dropped from the list entirely rather than shown as `n/a`.

---

### About > Reading notes

Cross-cutting notes that don't belong to any single tab: counting semantics, filter
scope, de-duplication and time-range conventions used throughout the workbook.
Mirrored in [Reading notes](05-reading-notes.md).

---

### About > Sources

Full attribution for every query this workbook builds on, plus the queries and
presentation patterns that are the repo owner's own. Mirrored in
[Credits & sources](10-credits.md).

