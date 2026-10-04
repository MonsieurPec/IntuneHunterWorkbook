# Known limitations

## Table of contents

- [`Target.GroupId` is a raw GUID.](#targetgroupid-is-a-raw-guid)
- [Honeycomb was considered and dropped.](#honeycomb-was-considered-and-dropped)
- [Two columns are schema assumptions, not verified against every tenant.](#two-columns-are-schema-assumptions-not-verified-against-every-tenant)
- [BitLocker's device link is parsed from free text, not a structured field.](#bitlockers-device-link-is-parsed-from-free-text-not-a-structured-field)
- [KPI tiles use `dcount()`, trend charts use `count()`.](#kpi-tiles-use-dcount-trend-charts-use-count)
- [Disk status assumes an undocumented storage unit.](#disk-status-assumes-an-undocumented-storage-unit)
- [LTSC editions are not distinguishable from `OSVersion` alone.](#ltsc-editions-are-not-distinguishable-from-osversion-alone)
- [Windows Update for Business reports refresh on a 28-day window and are unavailable in GCC High / DoD.](#windows-update-for-business-reports-refresh-on-a-28-day-window-and-are-unavailable-in-gcc-high--dod)
- [`CacheNodeDetails`'s sub-fields (MCC server usage table) aren't itemised in the `UCDOStatus` schema reference.](#cachenodedetailss-sub-fields-mcc-server-usage-table-arent-itemised-in-the-ucdostatus-schema-reference)
- [`Autopilot reset` is not matched by the `DestructiveOps` list.](#autopilot-reset-is-not-matched-by-the-destructiveops-list)
- [PIM's "off hours" detection is fixed to UTC.](#pims-off-hours-detection-is-fixed-to-utc)
- [Delivery Optimization's `GroupID` is an opaque hash.](#delivery-optimizations-groupid-is-an-opaque-hash)
- [The Autopilot ESP failure/success value isn't publicly documented by Microsoft.](#the-autopilot-esp-failuresuccess-value-isnt-publicly-documented-by-microsoft)

<a id="targetgroupid-is-a-raw-guid"></a>
- **`Target.GroupId` is a raw GUID.**
  - The group's display name is not present in `IntuneAuditLogs`. Resolving it requires joining an Entra ID group export (an `externaldata()` block against a blob, or a custom ingested table), not included here.

<a id="honeycomb-was-considered-and-dropped"></a>
- **Honeycomb was considered and dropped.**
  - It's the closest built-in Workbook visualization to a treemap-style density view, but its `graphSettings` JSON keys are not documented in a way that could be reproduced with confidence.
    The Top-actions tab uses an icon-and-gradient tile grid as a functional
    substitute.

<a id="two-columns-are-schema-assumptions-not-verified-against-every-tenant"></a>
- **Two columns are schema assumptions, not verified against every tenant.**
  - `IntuneDevices.LastContact` (used on the Wipe and Device actions tabs) and `AuditLogs.Result` (used on the LAPS tab, standard Entra ID schema). If either is absent or named differently in your export, only the tile or column relying on it is affected; everything else keeps working.

<a id="bitlockers-device-link-is-parsed-from-free-text-not-a-structured-field"></a>
- **BitLocker's device link is parsed from free text, not a structured field.**
  - The `Read BitLocker key` audit event doesn't put the target device in `TargetResources` the way LAPS does. The BitLocker tab's `AadDeviceId` comes from a `parse` on `AdditionalDetails[0].value` looking for `device: '<guid>'`. If Microsoft changes that message's wording, the parse silently returns nothing and the device name/serial join stops resolving; the row still appears, with the raw device ID falling back to empty instead of `<unresolved>`.

<a id="kpi-tiles-use-dcount-trend-charts-use-count"></a>
- **KPI tiles use `dcount()`, trend charts use `count()`.**
  - `dcount()` is HyperLogLog-based (roughly 0.8% default error). Small discrepancies between
    a tile's total and its own trend chart are expected and not a bug.

<a id="disk-status-assumes-an-undocumented-storage-unit"></a>
- **Disk status assumes an undocumented storage unit.**
  - Microsoft does not document the unit of `StorageFree` / `StorageTotal` (the schema just says
    `long`). This tenant reports MB, so `StorageUnitToGB = 1024.0`; if another tenant's values come out roughly a million times too large, the export is in bytes and the constant should be `1073741824.0`. The All devices tab's `Storage` column makes the same assumption with the same constant.

<a id="ltsc-editions-are-not-distinguishable-from-osversion-alone"></a>
- **LTSC editions are not distinguishable from `OSVersion` alone.**
  - Build 17763 covers both 1809 and LTSC 2019, 19044 both 21H2 and LTSC 21H2, and
    26100 both 24H2 and LTSC 2024. The Windows versions tab cannot tell them
    apart.

<a id="windows-update-for-business-reports-refresh-on-a-28-day-window-and-are-unavailable-in-gcc-high--dod"></a>
- **Windows Update for Business reports refresh on a 28-day window and are
  unavailable in GCC High / DoD.**
  - A time range beyond 28 days will not surface older data on the Windows Update errors or Delivery Optimization tabs, and both tabs are empty by design in GCC High/DoD tenants.

<a id="cachenodedetailss-sub-fields-mcc-server-usage-table-arent-itemised-in-the-ucdostatus-schema-reference"></a>
- **`CacheNodeDetails`'s sub-fields (MCC server usage table) aren't itemised
  in the `UCDOStatus` schema reference.**
  - They're verified by construction against Microsoft's own `DeliveryOptimization.workbook` instead of an
    independent schema page.

<a id="autopilot-reset-is-not-matched-by-the-destructiveops-list"></a>
- **`Autopilot reset` is not matched by the `DestructiveOps` list** on the
  Wipe / Retire / Reset tab. If your tenant emits it under a distinct
  operation name, add it manually; until then it won't appear on that tab
  even though it clears the primary user and the Entra device owner.

<a id="pims-off-hours-detection-is-fixed-to-utc"></a>
- **PIM's "off hours" detection is fixed to UTC** (before 07:00 or after
  19:00, or a weekend).
  - If your admins sit in a single non-UTC timezone, shift `TimeGenerated` before `hourofday()` rather than reading the tile as-is.

<a id="delivery-optimizations-groupid-is-an-opaque-hash"></a>
- **Delivery Optimization's `GroupID` is an opaque hash**,
  - not resolvable to a site name from that table alone, the same limitation as
    `Target.GroupId` on the Assignments tab, just on a different field.

<a id="the-autopilot-esp-failuresuccess-value-isnt-publicly-documented-by-microsoft"></a>
- **The Autopilot ESP failure/success value isn't publicly documented by
  Microsoft.**
  - Only `Status =~ "Completed"` on the `EspSlaIsDeviceSyncDone` event is confirmed; the failure-rate metrics bucket everything else as  `Failure` rather than matching an exact failure string, so a future
    Microsoft schema change adding a third status (e.g. a "Blocked" or
    "Timeout" state) would still count correctly as a failure, just without
    its own label.

See the workbook's own **About > Reading notes** tab (mirrored in
[05-reading-notes.md](05-reading-notes.md)) for the same information surfaced
inside the tool itself, and **About > Sources** for the full attribution,
mirrored in [10-credits.md](10-credits.md).
