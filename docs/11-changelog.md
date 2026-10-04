# Changelog

Reconstructed from the workbook's development history. Dates mark when each
milestone was reached during development, not calendar-accurate release dates.

## v0.15 (20261004): Layout pass

- Layout only: `customWidth` values changed, no query, parameter or tile
  was touched.
- **Overview**: the four charts now sit on one 65/35 grid (trend and
  privileged access on the left, top actors and fleet split on the
  right). The eight KPI tiles are unchanged.
- **Audit > Assignments**: change breakdown and assignment trend are
  50/50 (was 40/60).
- **Audit > Device actions**: *Top 10 actors on devices* and *Top 10
  device operations* are 50/50 (was 35/65).
- Narrow tables placed side by side instead of stacked at full width:
  Windows version (release chart 40, breakdown 60, which also removes a
  60% empty gap), BitLocker 50/50, LAPS 40/60, Wipe (per-device 60,
  burst 40), PIM (justifications 50, burst 50) and Delivery Optimization
  (groups 65, MCC servers 35). Tables with 7 or more columns stay full
  width.
- `docs/images/overview.png` predates this layout and should be
  recaptured.

## v0.14 (20260929): Remove Device Lookup

- Removed the **All devices** tab's search-driven "Device Lookup" added in
  v0.13 (`params - device-search`, `text - alldevices-prompt`, `table -
  alldevices-results`, `text - alldevices-detail`): the feature was not
  working. The KPI tiles and the per-day-by-OS trend chart above it are
  unchanged.
- Updated `docs/04-navigation.md`, `README.md` and the workbook's own
  **About > Navigation** text to match: the All devices section no longer
  describes a lookup feature that isn't there.

## v0.13 (20260922c): Device Lookup on All devices

- Replaced the **All devices** tab's 1000-row `table - alldevices` grid
  (`showExportToExcel`, no filter gate) with a search-driven "Device
  Lookup": a `DeviceSearch` text parameter, a compact 50-row results grid
  matched on device name, serial number, UPN or Intune Device ID, and a
  vertical key/value detail card for the selected device. The KPI tiles
  and the per-day-by-OS trend chart above are unchanged.
- The results grid (`table - alldevices-results`) stays empty until
  `DeviceSearch` is non-empty (a `text - alldevices-prompt` item covers
  the empty state) — it never dumps the full fleet. Selecting a row
  exports `SelectedDeviceId` (Intune `DeviceId`) and `SelectedEntraId`
  (`ReferenceId`) via `exportedParameters`, the same
  `fieldName`/`parameterName`/`parameterType` mechanism already used by
  the Overview KPI tiles, just applied to a grid row-click instead of a
  tile click for the first time in this workbook.
- The detail card (`text - alldevices-detail`) is a KQL-fed `text`
  visualization (`markdown` style) rather than a grid, rendering device
  identity, enrollment, compliance and last check-in as a single
  key/value block, plus the 5 most recent `IntuneAuditLogs` remote
  actions against that device (matched the same way Wipe/Retire already
  does: `Properties` → `TargetObjectIds[0]`).
- Both new queries use a flat `ago(30d)` snapshot with no
  `timeContextFromParameter`/`{TimeRange}` reference at all — simpler
  than, and independent of, the `iif({TimeRange:start} < ago(30d), ...)`
  idiom used elsewhere in the workbook.
- Documentation describing the old full-listing behavior of this tab
  (`README.md`, `NAVIGATION.md`, `READING_NOTES.md`, `DESIGN_NOTES.md`,
  `REQUIREMENTS.md`) was flagged as stale rather than rewritten in this
  pass.

## v0.12 (20260922b): All devices tab

- Added the **All devices** section to Device inventory (inserted
  alphabetically first, before Autopilot builds): a flat, cross-OS
  inventory table, one row per device, with `ComplianceState`/`IsCompliant`
  joined in from `IntuneDeviceComplianceOrg`.
- This is the first place in the workbook where `IntuneDevices` and
  `IntuneDeviceComplianceOrg` are actually joined together — everywhere
  else they're deduplicated and read independently. Each side keeps its
  own established dedup convention rather than a new one being invented
  for the join (see Design notes).
- Added 5 KPI tiles counting distinct devices per major OS family
  (All devices, Windows, iOS/iPadOS, Android, macOS) and a line chart of
  device count per day per OS, reusing the existing "bin + per-day
  `arg_max` + summarize by category" idiom already used by Windows
  versions and Compliance.
- `Storage` is rendered in GB using the same `StorageUnitToGB = 1024.0`
  assumption as Disk status (see Known limitations).
- Updated `README.md`, `NAVIGATION.md`, `READING_NOTES.md`,
  `KNOWN_LIMITATIONS.md`, `DESIGN_NOTES.md` and `REQUIREMENTS.md` to match,
  and fixed a pre-existing drift in README's Device inventory sub-tab rows
  (out of alphabetical order, and "OS currency" instead of "Mobile OS
  currency") while touching that table.

## v0.11 (20260922a): BitLocker key recovery tracking

- Added the **BitLocker** section to the Audit tab (inserted alphabetically
  between Assignments and Configuration), reading recovery-key read
  requests from Entra ID `AuditLogs` (`Read BitLocker key`) — the same
  bucket LAPS already reads from.
- Unlike LAPS, this event doesn't carry a device display name in
  `TargetResources`; the device object ID is `parse`d out of
  `AdditionalDetails[0].value` (`device: '<guid>'`) and resolved against
  the `IntuneDevices` inventory, the same de-duplicated-`arg_max` pattern
  used on the Wipe and Device actions tabs.
- Added a local **Actor type** filter (`User` / `Application` / `Unknown`),
  defaulting to `User`, so backup or key-escrow service accounts reading
  keys in bulk don't drown out human activity; switch it to select all to
  include them.
- Same harvesting detection as LAPS: actors reading keys for 5+ distinct
  devices within a single hour are flagged in a dedicated table.
- Deliberately does **not** extract the recovery key ID or the requester's
  IP address (see Design notes) — device name and actor are enough for
  hunting.
- Added BitLocker's `Actor` extraction as a third branch of the global
  **Actor** filter's union query, so BitLocker requesters show up in that
  dropdown the same way LAPS requesters already do.
- Updated `README.md`, `NAVIGATION.md`, `READING_NOTES.md`,
  `KNOWN_LIMITATIONS.md`, `DESIGN_NOTES.md` and `REQUIREMENTS.md` to match,
  and re-aligned README's tab table to the workbook's actual alphabetical
  order (it had drifted out of sync since v0.10).

## v0.10 (20260921c): Overview KPI grid redesign

- Cut the Overview tab from 15 KPI tiles to 8, and switched from
  `customWidth: "20"` / no border (5 tiles/row, felt sparse with no
  boundary to anchor the eye) to `customWidth: "25"` / `showBorder: true`
  (a clean 4-per-row, 2-row grid).
- **Merged `kpi-ov-config` + `kpi-ov-assign` → "Config & assignment
  changes"**: dropped `"Assignment"` from `kpi-ov-config`'s
  `ExcludedOperationPatterns`, since the two tiles' filters were already
  mutually exclusive by construction (config excluded assignment ops,
  assign required them) — no double-counting risk.
- **Merged `kpi-ov-wipe` + `kpi-ov-delete` → "Destructive & deletion
  actions"**: combined both `where` filters with `or`. Both source tiles
  already pointed their click-through at the same **Wipe** section, so
  nothing changes there.
- **Merged `kpi-ov-pim` + `kpi-ov-pimweak` → "PIM weak-justification
  rate"**: replaced the two raw counts with a single `Weak / Total * 100`
  percentage tile, matching the existing rate-tile pattern (Windows 11
  share, Compliance rate). Raw PIM activation volume stays visible via the
  *Privileged access over time* chart and the PIM tab itself. Also
  collapses one more copy of the ~25-line `PIMActivations` preprocessing
  block duplicated across the PIM tab (see Design notes).
- **Dropped outright** (still fully covered on their own tabs, just not
  duplicated at glance-level on Overview): `kpi-ov-lapsdevices` ("Devices
  unlocked" — the LAPS tab already has a dedicated harvesting-detection
  table), `kpi-ov-windows` ("Windows devices" raw count — Win11 share
  carries the signal), `kpi-ov-do` (Delivery Optimization), `kpi-ov-wu`
  (Update errors).
- Updated the Overview tab description in `README.md` and bumped the
  version line to match.

## v0.9 (20260921b): Autopilot ESP refinements and cleanup

- ESP success match changed from `Status =~ "Success"` to
  `Status =~ "Completed"` across all ESP queries; `"Completed"` is the only
  value Microsoft documents for the `EspSlaIsDeviceSyncDone` event,
  everything else still buckets as `Failure`. README and the workbook's own
  About tab (Navigation and Reading notes) updated to match.
- The standalone "ESP duration detail" table was merged into the main
  **Autopilot builds** table; the failure-rate chart was rebuilt as a
  two-series (Success/Failure) line chart, and the average-duration chart
  restyled as a line chart.
- Documented the ESP duration/failure-rate feature in the workbook's About
  tab, which had never mentioned it: a new paragraph under Navigation's
  **Autopilot builds** entry, and a new bullet under Reading notes' *Known
  gaps*.
- KQL consistency/optimization pass across all ~149 query items:
  - Added `| where Properties has "EspSlaIsDeviceSyncDone"` before
    `todynamic(Properties)` in every standalone ESP query, pruning rows
    before JSON parsing instead of after.
  - Fixed the `kpi-ap-espvolume` tile label ("ESP events" → "ESP devices"):
    the query already dedups by `ManagedDeviceId`, so it counts distinct
    devices, not raw events.
  - Removed a dead, fully-unused `let ModelCurrency = ...` block from the
    OS currency mobile-devices detail table.
  - Added the `ScopeSet`/`ResourceName` computation to the PIM raw-events
    table, matching all 11 sibling PIM queries that already had it.
- Re-sanitized real tenant subscription/resource group/workspace
  identifiers that had crept back into the six workspace parameters and
  `fallbackResourceIds` from a fresh Azure Portal export, back to the
  placeholder values established in v0.7.

## v0.8 (20260919): Autopilot ESP duration & failure rate

- Added ESP (Enrollment Status Page) end-to-end duration and failure-rate
  tracking to the **Autopilot builds** section, reading
  `IntuneOperationalLogs` / `ESPEnrollment` events with `EventId ==
  "EspSlaIsDeviceSyncDone"` and `Scope == "DeviceEspE2E"`, a different
  event from the `DeviceProvisioningProgressEvent` / `DeviceEnrollmentSuccess`
  pair already used for the existing build-count KPIs, and the only one that
  carries a duration (`TimeDiff`) and an outcome (`Status`).
- New KPI tiles: **ESP events** (sample size), **ESP failures**, **ESP
  failure rate**, **Avg ESP duration (success)**.
- New trend charts: **ESP failure rate by day** and **Avg ESP duration by
  day (success)**.
- New tables: **ESP failure rate by day** (Day / Success / Failed / Total /
  Failure rate), **Avg ESP duration by day (success)** (Day / duration as
  `hh:mm:ss` / duration in minutes), and **ESP duration detail**, a
  per-device table with device name resolved, mirroring the existing
  Autopilot device-name-resolution pattern.
- Non-success `Status` values are bucketed as `Failure` rather than matched
  against an exact string, since only the success value is documented.
- Scoped to `{TimeRange}` throughout, consistent with the rest of the
  Autopilot section.

## v0.7 (20260919): Tenant redaction, JSON fix, documentation rewrite

- Fixed an invalid trailing comma in the workbook JSON (right before the
  closing brace) that would have blocked import into Azure Portal.
- Replaced the real tenant identifiers (subscription ID, resource group,
  workspace name) hard-coded into all six subscription/workspace parameters
  with placeholder values, ahead of publishing this repo. The workbook's own
  **About > Maintenance** note already called for this before sharing the
  JSON outside the tenant.
- Rewrote `README.md` to document the four-root-tab structure, the three
  workspace buckets, and the Inventory / PIM / Delivery Optimization
  additions from v0.6.

## v0.6 (20260904): Inventory tab, PIM, workspace buckets, About restructure

- Restructured the workbook into four root tabs: **Overview**, **Audit**,
  **Inventory**, **About**, replacing the previous flat tab strip. Audit
  now groups the existing audit-focused tabs; Inventory is new.
- Added the **PIM** tab: Entra ID Privileged Identity Management role
  activations, read from `AuditLogs` where `LoggedByService == "PIM"`.
  Resolves role/target by `type` rather than array position (layout differs
  between role and group assignments) and coalesces three possible
  justification fields (`AdditionalDetails["Justification"]`,
  `ResultReason`, `ResultDescription`). Flags justifications as `Provided`,
  `Too short` or `Missing`, and detects off-hours activations (before 07:00
  or after 19:00 UTC, or a weekend).
- Added the **Inventory** root tab:
  - **Device inventory**: Windows versions (build-derived, via the third
    `OSVersion` octet), Windows Update errors, Disk status, Autopilot
    builds, OS currency (via `parse_version()`), and Compliance.
  - **Delivery Optimization**: CDN/peer/Microsoft Connected Cache bandwidth
    split, including Internet peers in the peer-to-peer percentage, plus a
    per-cache-node breakdown table.
  - Most Inventory sections read a fixed 30-day snapshot lookback,
    independent of the global time range; genuinely time-series sections
    (Autopilot builds, OS version evolution, compliance trend, update
    errors, Delivery Optimization) honour it.
- Added source tables: `IntuneOperationalLogs` (Autopilot),
  `IntuneDeviceComplianceOrg` (Compliance), and the Windows Update for
  Business reports tables `UCClient`, `UCUpdateAlert`, `UCDOStatus`
  (enrolled separately from the Azure Portal; not available in GCC High or
  DoD tenants).
- Replaced the single subscription/workspace parameter pair with three
  independent buckets: **Intune logs**, **Entra ID logs**, **Windows
  Update reports**, each with its own Subscription + Workspace picker.
  Four items (the Actor list, LAPS, Delivery Optimization devices, and the
  ingestion-health check) read across two buckets in a single query using
  explicit `workspace('{Bucket}').Table` syntax.
- Moved every tab's description out of the data tab itself into a new
  **About > Navigation** reference page. The former single "Notes &
  sources" tab (added in v0.4) split into four: **Navigation**, **Data
  feeds**, **Reading notes**, **Sources**.
- Changed the version-stamp format to allow a same-day letter suffix
  (`20260904b`) for multiple same-day updates.

## v0.5 (20260820): Direct maintenance, versioning

- Retired the Python generator script from the repo. The workbook JSON is now
  the single source of truth, edited directly rather than regenerated from a
  build step.
- Added a version line (`yyyyMMdd` format) to the workbook's **Notes &
  sources** tab and to this README, kept in sync by hand going forward.

## v0.4: Notes tab, device name resolution

- Added a dedicated **Notes & sources** tab; reading notes and full source
  attribution moved out of a trailing text block into their own tab.
- **Wipe / Retire / Reset**: resolved the target device name via a
  de-duplicated, left-outer join against `IntuneDevices`, instead of showing
  the raw audit action label. Added a per-device summary table and an
  unresolved-device flag (`NameResolution`).
- Extended the destructive-action operation set beyond wipe/retire to include
  `freshStart`, `eraseDevice`, `remoteLock`, `resetPasscode`, `disable`.
- Reverted an accidental change that flattened every KQL query to a single
  line; queries are meant to stay multi-line for readability in the query
  editor.

## v0.3: English rebuild, tabs, KPI tiles, new sections

- Rebuilt the whole UI in English (titles, labels, computed values).
- Restructured the workbook from a single scrolling page into tabs (Overview,
  Top actions, Configuration, Assignments, Wipe/Retire/Reset, LAPS, Device
  actions, Raw audit log), following the tab-navigation pattern from the
  MSEndpointMgr Intune Audit Dashboard.
- Adopted that same dashboard's KPI tile style (threshold icons, gradient big
  numbers) and per-value icon grids across every tab.
- Added the **Top actions** tab: ranking, per-operation trend, and a focus
  filter to drill into a single operation's raw events.
- Added the **Device actions** tab, joining `IntuneAuditLogs` against a
  de-duplicated `IntuneDevices` snapshot, broken down by OS.
- Removed all `crossComponentResources` fields; the workbook now resolves its
  workspace solely through `fallbackResourceIds`.

## v0.2: LAPS tracking

- Added the **LAPS** section, reading Windows LAPS local administrator
  password recovery requests from Entra ID `AuditLogs` (a separate table from
  `IntuneAuditLogs`).
- Added harvesting detection: actors requesting 5 or more distinct devices'
  passwords within a single hour.

## v0.1: Initial KQL and workbook

- First working queries for configuration changes and assignment changes,
  built on `parse_json()` + `mv-expand` against `IntuneAuditLogs.Properties`,
  replacing an earlier positional-`parse` approach.
- Added the **Wipe / Retire / Reset** section with burst detection (3+
  actions per actor within 15 minutes).
- Excluded `Sync*` operations and `Delete ManagedDeviceMobileAppConfiguration`
  as noise/false positives across the relevant sections.
- Assembled into an importable Azure Workbook JSON, generated programmatically
  from a Python script rather than hand-written.
