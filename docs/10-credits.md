# Credits & sources

This workbook did not start from a blank page. It grew out of several published
approaches to querying `IntuneAuditLogs`, and one visual reference dashboard.
This file exists so that lineage is traceable and so that the people whose work
made this easier get credit for it.

## Authorship

Conceived by moniseur.pec.
Claude was used to optimize KQL queries, and build the JSON architecture. 


## Acknowledgments

Thanks to Pascal, Anthony, David and Le Tigre for their support and feedback along the way.

## KQL lineage

The configuration-change query in this workbook is the end point of a chain of
iterations, each one fixing a limitation of the previous approach:

- **[Shehan Perera, IntuneAuditLogs series (2022)](https://emsroute.com/2022/10/23/intuneauditlogs-1/)**
  (originally published on `shehanperera.com`, now hosted on `emsroute.com`).
  Source of the original approach: using the KQL `parse` operator positionally
  against the raw `Properties` string to pull out `Object`, `OperationName` and
  a first pass at `ModifiedProperties`. This workbook no longer uses positional
  `parse`; it was replaced by `parse_json()`, which does not depend on key
  order or on the absence of commas inside values, but the underlying idea of
  auditing Intune through `IntuneAuditLogs` in Log Analytics starts here.

- **[Oliver Kieselbach, Monitoring Intune policy configuration changes (2022)](https://oliverkieselbach.com/2022/07/19/monitoring-intune-policy-configuration-changes/)**.
  An independent take on the same problem, useful for cross-checking. It
  documents the same limitation later solved in this workbook: expanding
  `ModifiedProperties` automatically rather than eyeballing a JSON blob.

- **[Ugur Koc, Daily Intune monitoring report](https://ugurkoc.de/daily-intune-monitoring-report/)**.
  This is the direct technical ancestor of the query pattern actually used in
  this workbook: `parse_json()` combined with `mv-expand` on
  `Properties.Targets[*].ModifiedProperties`, extracting `Name` / `Old` / `New`
  as clean columns, and resolving the acting identity through
  `Properties.Actor.UPN` falling back to `Properties.Actor.ApplicationName`.
  One deliberate deviation from that article: this workbook expands the
  **entire** `Targets` array rather than `Targets[0]`. Real audit data pulled
  from a live tenant showed that the meaningful `ModifiedProperties` can sit on
  the second target rather than the first; `Targets[0]` alone silently drops
  those events.
  Ugur Koc is a Microsoft MVP for Intune and Security Copilot and maintains a
  wide set of free, open-source Intune tooling. See
  [ugurkoc.de](https://ugurkoc.de), [GitHub @ugurkocde](https://github.com/ugurkocde)
  and [LinkedIn](https://de.linkedin.com/in/ugurkocde).

- **[`parse` operator, Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/parseoperator)**.
  Reference documentation for the original positional-parsing approach.

## Autopilot ESP duration lineage

- **[Tom Wechsler, Microsoft_Endpoint_Manager, Kusto Query Language (KQL)](https://github.com/tomwechsler/Microsoft_Endpoint_Manager/blob/main/Kusto_Query_Language_(KQL).md)**.
  Independent reimplementation, inspired by seeing this approach: reading
  per-device ESP end-to-end duration and outcome from
  `IntuneOperationalLogs` / `ESPEnrollment` (`EventId ==
  "EspSlaIsDeviceSyncDone"`, `Scope == "DeviceEspE2E"`, `Properties.TimeDiff`
  as the duration and `Properties.Status` as the outcome). The queries
  behind the Autopilot failure-rate and average-duration additions were
  written independently for this workbook: aggregated **by day** instead of
  a single global average, scoped to `{TimeRange}` instead of a fixed
  30-day lookback, and bucketing every non-`Success` `Status` value as
  `Failure` rather than matching one exact failure string, since only the
  success value is documented by Microsoft.

## PIM lineage

- **[reprise99, Audit-NewPIMRoleActivated](https://github.com/reprise99/Sentinel-Queries/blob/main/Azure%20Active%20Directory/Audit-NewPIMRoleActivated.kql)**.
  Starting point for the PIM tab's target and detail extraction from
  `AuditLogs`. Extended here to resolve role, target user and scope by
  `type` rather than by array position (the `TargetResources` layout
  differs between role assignments and group assignments), and to coalesce
  the activation justification across its three possible schema locations
  (`AdditionalDetails["Justification"]`, `ResultReason`,
  `ResultDescription`) instead of reading a single one.
- **[Microsoft Entra audit log schema](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/auditlogs)**
  and **[View activity and audit history for Privileged Identity
  Management](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-how-to-use-audit-log)**,
  reference documentation for the `AuditLogs` schema and PIM's own audit
  history view.

## Presentation layer

- **[MSEndpointMgr, Intune Audit Dashboard (2022)](https://msendpointmgr.com/2022/10/28/msendpointmgr-intune-audit-dashboard/)**.
  This workbook's visual grammar is modeled on that dashboard: tab navigation
  built from a `LinkItem` bound to a parameter with conditional visibility on
  groups, KPI tiles with threshold-driven icons and gradient big numbers,
  per-value icons inside grids, colour-coded `make-series` area charts, and an
  ingestion-freshness indicator. None of that dashboard's own KQL was reused;
  only the presentation patterns were, rebuilt from scratch against this
  workbook's own queries.

## Device name resolution

The device-name resolution on the Wipe / Retire / Reset and Device actions tabs
is based on a working query of the repo owner's own, joining `IntuneAuditLogs`
to `IntuneDevices` on the device ID extracted from `TargetObjectIds[0]`. Two
changes were made on top of that original query: the inventory is
de-duplicated with `arg_max` **before** the join rather than after (so that
multiple actions on the same device are all preserved instead of collapsing to
one row), and the join was switched from inner to left outer (so that actions
on devices no longer present in the inventory (because they were deleted
after being wiped, for instance) stay visible instead of silently
disappearing).

## Device inventory and Delivery Optimization

The Inventory tab's queries (Windows versions, Disk status, Autopilot
builds, OS currency, Compliance, and Delivery Optimization) are adapted from
working queries of the repo owner's own. Changes made on top of those
originals:

- Snapshot de-duplication (`arg_max(TimeGenerated, ...)`) moved to run
  **before** every join and count, not after.
- `startswith` build matching replaced by an octet-based `case()` on
  `OSVersion`, so monthly revision bumps don't break the mapping.
- String version comparison replaced by `parse_version()` for OS currency,
  avoiding the lexicographic-sort bug where `"9.5"` would outrank `"18.5"`.
- De-duplication key switched from `SerialNumber` to `DeviceId` throughout:
  serial numbers aren't guaranteed unique across manufacturers and are
  sometimes blank on virtual machines.
- Hard-coded `GroupID` and `DeviceName endswith` development filters
  replaced by workbook parameters.
- `format_bytes()` kept only for table columns, since it returns a string
  and can't drive a chart or a KPI tile.
- The Delivery Optimization time filter corrected from
  `TimeGenerated < ago(7d)`, which selected everything *older* than seven
  days, to the global `{TimeRange}`.

**MCC server usage table.** Adapted from Microsoft's own
**[DeliveryOptimization.workbook, microsoft/Application-Insights-Workbooks](https://github.com/microsoft/Application-Insights-Workbooks/blob/master/Workbooks/UpdateCompliance/DeliveryOptimization/DeliveryOptimization.workbook)**,
the official community workbook behind the Windows Update for Business
reports Delivery Optimization tab, which is the source of the
`CacheNodeDetails` / `mv-expand` aggregation-by-cache-node pattern used
here. The **[UCDOStatus table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/ucdostatus)**
confirms `CacheNodeDetails` as a `dynamic` column ("an array of cache node
details ... including cache node identifier, date, and bytes served") but
doesn't itemise the sub-field names, which come from the workbook above
instead.

**UCClient / UCUpdateAlert join pattern.** Origin credited to **HTMD Blog**,
*List of Intune devices with patch deployment status and country details
using KQL queries* (article link not retained — add it here if you have it).

## Microsoft documentation

The workbook mechanics (tabs, tiles, time parameters, honeycomb, grid link
renderers, conditional visibility) were built and verified against:

- [Workbook visualizations](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-visualizations)
- [Tile visualizations](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-tile-visualizations)
- [Honeycomb visualizations](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-honey-comb)
- [Time parameters](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-time)
- [Workbook parameters](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-parameters)
- [Sample workbooks with links (the tab pattern)](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-samples)
- [Interactive reports (grid link renderers, conditional visibility)](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-interactive-reports)
- [IntuneDevices table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/intunedevices)
- [IntuneDeviceComplianceOrg table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/intunedevicecomplianceorg)
- [Windows Update for Business reports overview](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-overview)
  and the [UCDOStatus](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-schema-ucdostatus),
  [UCClient](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-schema-ucclient)
  and [UCUpdateAlert](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-schema-ucupdatealert)
  schemas
- [Windows 11 release information](https://learn.microsoft.com/en-us/windows/release-health/windows11-release-information)
  and [Windows 10 release information](https://learn.microsoft.com/en-us/windows/release-health/windows10-release-information),
  used to build and check the Windows versions build-number mapping
- [Delivery Optimization reference](https://learn.microsoft.com/en-us/windows/deployment/do/waas-delivery-optimization-reference)
  for `DODownloadMode` and `DOGroupIdSource`

## Multi-workspace parameterization

- **[Query across resources with Azure Monitor, Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cross-workspace-query)**.
  Source of the `workspace('identifier').table` syntax used to pin a
  specific table to a specific workspace inside a single query, wherever an
  item reads across two of the three workspace buckets.
- **[Cross workspace queries, Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/api/cross-workspace-queries)**.
  Confirms the accepted identifier formats (name, qualified name, workspace
  ID, Azure Resource ID) for `workspace()`.
- **[Azure Monitor workbook resource parameters, Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-resources)**.
  Subscription and Resource picker parameter types used for the six
  subscription/workspace parameters (Intune logs, Entra ID logs, Windows
  Update reports).
- **[microsoft/Application-Insights-Workbooks](https://github.com/microsoft/Application-Insights-Workbooks)**
  - specifically the Windows Update for Business reports hub workbook
  (`community-Workbooks/UpdateCompliance/UpdateComplianceHub`). Origin of
  the ARM-based subscription-then-workspace resolution pattern (a Resource
  Graph query filtered by `type == "microsoft.operationalinsights/workspaces"`,
  scoped via `crossComponentResources` to the selected subscription). Its
  Graph API auto-detection of a saved workspace mapping and its
  onboarding-state UI were **not** reused: both are specific to the Windows
  Update for Business reports enrollment flow in the M365 admin center and
  have no equivalent for a bespoke workspace.

## If you build on this

If you fork or reuse this workbook, please keep this file (or an equivalent
attribution) intact. None of the people listed above endorsed this specific
workbook; they published the ideas it is built on, in good faith, for the
community, and that's worth passing forward accurately.
