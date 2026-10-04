# Reading notes

**Navigation**

- Per-tab descriptions live in **About > Navigation**, not at the top of each tab. The data tabs open straight onto their content.
- Four top-level tabs: Overview, Audit, Inventory, About. Audit and Inventory carry their own sub-tab strip; Device inventory carries a third level.
- `TimeRange` is declared at the top level and is therefore global. `Actor` and `Operation` are also declared at the top level, so they still apply to every audit query, but the parameter block is only *displayed* on the Audit tab through `conditionalVisibility`. Declaring them inside the Audit group instead would scope them to that group and break every query outside it.
- KPI strips render at tile size 1 (medium). Size 4 (tiny) clips the number and produces a vertical scrollbar inside the strip.

**Data scope**

- `Sync*` operations (Autopilot, DEP, MultiToken) are excluded from the Overview, Configuration, Top actions and Raw audit log tabs: high volume, no audit value.
- Empty actors are filtered out on the LAPS tab (they correspond to automatic password rotation, not to an administrative recovery).
- The BitLocker tab's local **Actor type** filter defaults to `User`, hiding `Application`/service-account key reads (backup, key-escrow) unless explicitly selected. Unlike LAPS, actors are never empty here — an unresolved `InitiatedBy` falls back to `"Unknown"` rather than being filtered out.
- `Delete ManagedDeviceMobileAppConfiguration` is explicitly excluded wherever `Delete ManagedDevice` is matched: it is a configuration object, not a device.
- The `DeviceManagementAPIVersion` modified property is filtered out of configuration changes: it is present on almost every event and carries no analytical value.

- Columns built with `make_set()` are joined into a plain string and given an explicit `customColumnWidthSetting`. A raw dynamic array renders as `["A","B",...]`, and the brackets and quotes waste about a fifth of the visible width before the cell is ellipsised.

**Counting**

- KPI tiles use `dcount(AuditEventId)` (HyperLogLog, roughly 0.8% default accuracy). Trend charts use `count()` on raw rows. Small gaps between a tile and its chart are expected. Use `dcount(x, 4)` if an exact count matters more than query cost.
- Events are de-duplicated by `AuditEventId` because `mv-expand` on `Targets` then `ModifiedProperties` multiplies rows per event.

**Filters**

- The **Actor** and **Operation** filters are global and propagate to every tab.
- The **Operation** filter targets `IntuneAuditLogs` operations only and is therefore not applied to the LAPS or BitLocker tabs, which read `AuditLogs`. The **Actor** filter applies normally to both.

**Device name resolution**

- `IntuneDevices` is a snapshot table (one row per device per day). It is de-duplicated with `arg_max` per `DeviceId` **before** any join, over a fixed 30-day lookback that is deliberately independent of the selected time range: a one-hour range would otherwise contain no snapshot row and resolve nothing.
- On device action events, `TargetDisplayNames[0]` carries the **action label**, not the device name. The real device name comes from the inventory join and is kept in `DeviceName`, while the raw audit label is kept in `AuditTargetLabel`.
- The join is a **left outer** on purpose: a device wiped and then deleted disappears from the inventory, and an inner join would silently hide exactly the events that matter. Unresolved rows are kept and flagged in the `NameResolution` column. `Not in inventory` usually means the device was deleted after the action, or last checked in more than 30 days ago.

**Thresholds**

- Wipe/Retire burst detection: 3 or more actions per actor within 15 minutes.
- LAPS harvesting detection: 5 or more distinct devices per actor within 1 hour.
- BitLocker harvesting detection: 5 or more distinct devices per actor within 1 hour, same as LAPS.
- All thresholds are arbitrary starting points and are meant to be tuned against your own baseline. They are set directly in the corresponding queries.

**Known gaps**

- `Target.GroupId` is a raw GUID. The group display name is not present in `IntuneAuditLogs`. Resolving it requires joining an Entra ID group export (`externaldata` on a blob, or a custom table).
- The `Result` column used on the LAPS tab comes from the standard Entra ID `AuditLogs` schema. If it is absent or named differently in your export, only the "Failed attempts" tile and the `Result` column are affected.
- The BitLocker tab resolves its target device by `parse`-ing a `device: '<guid>'` fragment out of `AdditionalDetails[0].value`, since (unlike LAPS) `TargetResources` doesn't carry it. If Microsoft rewords that message, the parse returns nothing and the device name/serial join silently stops resolving.
- The BitLocker tab never extracts the recovery key ID or the requester's IP address, by design.
- Honeycomb (hive cluster) was considered for the Top actions tab but not implemented: its `graphSettings` JSON keys are not documented, and hand-writing them was judged too risky. The tile grid is the functional equivalent.

## Table of contents

- [Device inventory and Delivery Optimization](#device-inventory-and-delivery-optimization)
- [Version](#version)
- [Maintenance](#maintenance)

---

## Device inventory and Delivery Optimization

**Extra tables required**

These two tabs read beyond `IntuneAuditLogs` / `IntuneDevices` / `AuditLogs`. If a section is empty, check the corresponding feed first, in the ingestion tile at the top of the workbook.

- `IntuneOperationalLogs` — Intune diagnostic settings, *Operational logs* category. Needed for Autopilot builds.
- `IntuneDeviceComplianceOrg` — Intune diagnostic settings, *Device compliance* category. Needed for the Compliance section, and for the `ComplianceState`/`IsCompliant` columns on All devices.
- `UCClient`, `UCUpdateAlert`, `UCDOStatus` — [Windows Update for Business reports](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-overview), enrolled separately from the Azure portal and pointed at this workspace. Needed for Windows Update errors and the whole Delivery Optimization tab. Not available in GCC High or DoD tenants.

**Time semantics**

- Audit tabs are event streams: everything honours `{TimeRange}`.
- Inventory tabs mostly read **snapshot** tables, one row per device per day. Sections that describe a *current state* (All devices tiles/table, Windows versions, Disk, Mobile OS currency, Compliance tiles) use a fixed **30-day** lookback, deliberately independent of the selected range, so that a one-hour range still resolves something.
- Sections that are genuinely time-series (All devices trend, Autopilot builds, version evolution, compliance trend, update errors, Delivery Optimization) do honour `{TimeRange}`.
- WUfB reports only refresh devices seen in the **last 28 days**. A 90-day range on the Delivery Optimization tab will not surface older data.

**De-duplication**

Every snapshot read goes through `arg_max(TimeGenerated, ...)` on a stable key before any join or count:

- `IntuneDevices` and `IntuneDeviceComplianceOrg` on `DeviceId`, not on `DeviceName` or `SerialNumber`. Device names are not unique and are sometimes reported as `User deleted for this device`; serial numbers are blank on many virtual machines.
- `UCClient` on `AzureADDeviceId`.
- Trend charts bin to one day and take the last record per device *within that day*, otherwise a day carrying two snapshots would double every point on the curve.
- All devices is the only tab that actually joins `IntuneDevices` to `IntuneDeviceComplianceOrg` (elsewhere they're only ever deduplicated separately, never joined). Each side keeps its own established dedup: explicit-column `arg_max` for `IntuneDevices`, wildcard `arg_max(TimeGenerated, *)` for `IntuneDeviceComplianceOrg`, same as the standalone Compliance tab.

**Version handling**

- Windows releases are mapped from the **third octet** of `OSVersion` (`10.0.<build>.<revision>`), not from a `startswith` on the whole string. Monthly revisions therefore never break the mapping. Build 28000 is Windows 11 **26H1**, a new-device-only release, which is why it sorts above 26H2 (26300). LTSC cannot be told apart from `OSVersion` alone: 17763, 19044 and 26100 each cover both a semi-annual and an LTSC release.
- Mobile OS versions are ranked with `parse_version()`, never by string comparison. Lexicographically `"9.5"` sorts above `"18.5"`, which would mark a whole model as current on a two-year-old release.
- OS currency uses the tenant as its own reference: the newest version *seen on any device of the same model* is treated as the target. This needs no external version feed, but it silently reports 100 % if no device of that model has ever been updated.

**Thresholds and constants**

- Disk: `Critical` below 10 GB free, `Warning` below 25 GB. Set as `let` constants in each Disk query.
- `StorageUnitToGB = 1024.0`. Microsoft documents `StorageFree` / `StorageTotal` only as `long`, with no unit. This tenant reports MB. If free space comes out absurdly large, switch the constant to `1073741824.0`. Same constant, same assumption, on the All devices tab's `Storage` column.
- Delivery Optimization P2P includes `BytesFromIntPeers`. Most published queries count only LAN and group peers, which understates peering wherever `DODownloadMode` is 3.

**Known gaps**

- `GroupID` in `UCDOStatus` is an opaque hash of the DO group. It cannot be resolved to a site, subnet or policy name from this table.
- Autopilot builds only cover the ESP path. A device provisioned without an Enrollment Status Page emits no `DeviceEnrollmentSuccess` scope and will not appear.
- The Autopilot detail table keeps the latest build per device; the tiles and trend count every build event. A device rebuilt three times appears once in the table and three times in the counters.
- ESP success is matched on `Status =~ "Completed"`, the only value Microsoft documents for the `EspSlaIsDeviceSyncDone` event; every other status is bucketed as `Failure`.

## Version

**20261004**

Updated by hand whenever the workbook changes. Not semantic versioning — just
a same-day-comparable stamp. If you're looking at two copies of this file
side by side, this line is the fastest way to tell which one is newer. See
`docs/11-changelog.md` for what changed at each version.

---

## Maintenance

This workbook JSON is maintained directly — edits are made in place, either
in the Azure Workbook editor or in the JSON itself. There is no separate
build step or generator to run; the JSON you're looking at is the source of
truth.

Before sharing this JSON outside the tenant, replace the subscription/resource-group/workspace identifiers with placeholders in **all 7 places** they appear: `fallbackResourceIds`, plus the `value` field of each of the 6 workspace-scope parameters (`IntuneSubscription`, `IntuneWorkspace`, `EntraSubscription`, `EntraWorkspace`, `WufbSubscription`, `WufbWorkspace`). Before sharing, verify with `grep -c '/subscriptions/<your-subscription-id>' workbook/IntuneHunter.workbook` — it should return `8`: the 7 real placeholders, plus one self-match, since the workbook's own Maintenance text carries this exact command too.
