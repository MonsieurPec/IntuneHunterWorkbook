# Design notes

## Table of contents

- [Why `IntuneDevices` is de-duplicated before joining, not after.](#why-intunedevices-is-de-duplicated-before-joining-not-after)
- [Why the device-inventory join is `leftouter`, not `inner`.](#why-the-device-inventory-join-is-leftouter-not-inner)
- [Why queries are authored multi-line.](#why-queries-are-authored-multi-line)
- [Why three independent workspace buckets instead of one workspace parameter.](#why-three-independent-workspace-buckets-instead-of-one-workspace-parameter)
- [Why the Windows build mapping is a `case()` on the third octet of `OSVersion`, not a `startswith` match.](#why-the-windows-build-mapping-is-a-case-on-the-third-octet-of-osversion-not-a-startswith-match)
- [Why OS currency compares versions with `parse_version()`, not string comparison.](#why-os-currency-compares-versions-with-parse_version-not-string-comparison)
- [Why the PIM tab resolves role and target by `type`, not by array position, and coalesces three justification fields.](#why-the-pim-tab-resolves-role-and-target-by-type-not-by-array-position-and-coalesces-three-justification-fields)
- [Why the Delivery Optimization peer-to-peer percentage includes Internet peers.](#why-the-delivery-optimization-peer-to-peer-percentage-includes-internet-peers)
- [Why the PIM tab's ~20-line preprocessing block is duplicated across all 12 PIM items instead of shared.](#why-the-pim-tabs-20-line-preprocessing-block-is-duplicated-across-all-12-pim-items-instead-of-shared)
- [Why the BitLocker tab never extracts the recovery key ID or the requester's IP address.](#why-the-bitlocker-tab-never-extracts-the-recovery-key-id-or-the-requesters-ip-address)
- [Why All devices re-derives `ComplianceState` instead of reusing the Compliance tab's query.](#why-all-devices-re-derives-compliancestate-instead-of-reusing-the-compliance-tabs-query)
- [Why the Disk status thresholds are duplicated across 7 items instead of a workbook parameter.](#why-the-disk-status-thresholds-are-duplicated-across-7-items-instead-of-a-workbook-parameter)

<a id="why-intunedevices-is-de-duplicated-before-joining-not-after"></a>
- **Why `IntuneDevices` is de-duplicated before joining, not after.**
  - `IntuneDevices` is a snapshot table: one row per device per day. Joining
    against it directly multiplies every audit event by the number of
    retained snapshot days. The lookup table is built with
    `arg_max(TimeGenerated, ...) by DeviceId` **before** the join, over a
    lookback window that is deliberately independent of the workbook's
    selected time range (a one-hour range would otherwise contain zero
    snapshot rows and resolve nothing).

<a id="why-the-device-inventory-join-is-leftouter-not-inner"></a>
- **Why the device-inventory join is `leftouter`, not `inner`.**
  - A device that was wiped and then deleted from Intune disappears from
    `IntuneDevices`. An inner join would silently drop exactly the audit
    events that matter most for a hunting workflow. The left outer join
    keeps every event; unresolved rows are flagged in a `NameResolution`
    column instead of vanishing.

<a id="why-queries-are-authored-multi-line"></a>
- **Why queries are authored multi-line.**
  - The `"query"` field inside a Workbook JSON item can contain literal
    newlines; this is valid JSON, and it renders as normal, readable
    multi-line KQL in the workbook's query editor. Flattening every query
    to one line was tried at one point and reverted: it made the JSON
    marginally smaller and the query editor meaningfully harder to read
    and edit.

<a id="why-three-independent-workspace-buckets-instead-of-one-workspace-parameter"></a>
- **Why three independent workspace buckets instead of one workspace
  parameter.**
  - Intune logs, Entra ID logs and Windows Update reports can live in
    different workspaces or even different subscriptions depending on the
    tenant. Five items read across two buckets in a single query (the
    Actor list, LAPS, BitLocker, Delivery Optimization devices, and the
    ingestion-health check): the secondary table is referenced explicitly
    with `workspace('{BucketWorkspace}').Table`, because
    `crossComponentResources` unions the *same* table across resources and
    cannot pull two *different* tables from two different workspaces into
    one query.

<a id="why-the-windows-build-mapping-is-a-case-on-the-third-octet-of-osversion-not-a-startswith-match"></a>
- **Why the Windows build mapping is a `case()` on the third octet of
  `OSVersion`, not a `startswith` match.**
  - `OSVersion` follows `10.0.<build>.<revision>`; matching on the build
    number survives monthly revision bumps that would break a `startswith`
    comparison. The map is an inline `case()` rather than a `let` lambda
    because a lambda body is wrapped in braces, and braces are the
    workbook's parameter-substitution syntax.

<a id="why-os-currency-compares-versions-with-parse_version-not-string-comparison"></a>
- **Why OS currency compares versions with `parse_version()`, not string
  comparison.**
  - Lexicographic sort ranks `"9.5"` above `"18.5"`, which would silently
    mark an entire device model as up to date on a version that is
    actually two years old.

<a id="why-the-pim-tab-resolves-role-and-target-by-type-not-by-array-position-and-coalesces-three-justification-fields"></a>
- **Why the PIM tab resolves role and target by `type`, not by array
  position, and coalesces three justification fields.**
  - `TargetResources` lays its entries out differently for role
    assignments versus group assignments, so position-based extraction is
    unreliable. The activation justification has three possible homes
    depending on tenant and event version
    (`AdditionalDetails["Justification"]`, `ResultReason`,
    `ResultDescription`); this workbook `coalesce`s all three rather than
    betting on one.

<a id="why-the-delivery-optimization-peer-to-peer-percentage-includes-internet-peers"></a>
- **Why the Delivery Optimization peer-to-peer percentage includes
  Internet peers.**
  - Most published queries count only LAN and group peers
    (`BytesFromPeers` + `BytesFromGroupPeers`), which understates peering
    wherever `DODownloadMode` is set to 3 (Internet). This workbook adds
    `BytesFromIntPeers` to the numerator.

<a id="why-the-pim-tabs-20-line-preprocessing-block-is-duplicated-across-all-12-pim-items-instead-of-shared"></a>
- **Why the PIM tab's ~20-line preprocessing block is duplicated across
  all 12 PIM items instead of shared.**
  - A Workbook JSON item can't reference a `let` defined in a sibling item,
    so every PIM tile, chart and table recomputes its own
    `PIMActivations` (role/target resolution plus the `Justification`
    coalesce) independently. The alternative would be a saved Log
    Analytics function deployed to the workspace and called from all 12
    items, which removes the duplication but adds a deployment dependency
    this repo doesn't manage or version. Kept as-is; worth revisiting if
    the PIM tab grows further.

<a id="why-the-bitlocker-tab-never-extracts-the-recovery-key-id-or-the-requesters-ip-address"></a>
- **Why the BitLocker tab never extracts the recovery key ID or the
  requester's IP address.**
  - The `Read BitLocker key` event exposes both in `AdditionalDetails`, and
    an earlier draft parsed and displayed them (mirroring how LAPS shows
    its `Result` column). Both were dropped by design: the key ID
    identifies exactly *which* secret was read, which this workbook has no
    business surfacing even to an admin looking at it on a screen, and the
    IP address is PII with no hunting value the device name and actor
    don't already provide. Device name and actor are enough to spot
    harvesting; recovering the actual key stays a deliberate, audited
    action in the Intune admin center, not a workbook query result.

<a id="why-all-devices-re-derives-compliancestate-instead-of-reusing-the-compliance-tabs-query"></a>
- **Why All devices re-derives `ComplianceState` instead of reusing the
  Compliance tab's query.**
  - A Workbook JSON item can't reference a `let` defined in a sibling item
    (the same constraint documented above for the PIM tab), so All
    devices' detail table computes its own `IntuneDeviceComplianceOrg`
    lookup rather than depending on whatever the Compliance tab already
    resolved. It's the first place in this workbook where `IntuneDevices`
    and `IntuneDeviceComplianceOrg` are actually joined — everywhere else
    they're deduplicated and read independently — but each side keeps its
    own established dedup convention (explicit-column `arg_max` for
    `IntuneDevices`, wildcard `arg_max(TimeGenerated, *)` for
    `IntuneDeviceComplianceOrg`) rather than inventing a third one for the
    join.

<a id="why-the-disk-status-thresholds-are-duplicated-across-7-items-instead-of-a-workbook-parameter"></a>
- **Why the Disk status thresholds are duplicated across 7 items instead of
  a workbook parameter.**
  - `CriticalGB = 10.0` and `WarningGB = 25.0` are repeated as identical
    `let` constants in `kpi-disk-critical`, `kpi-disk-warning`,
    `kpi-disk-ok`, `kpi-disk-total`, `chart - disk-status`,
    `chart - disk-lowest` and `table - disk`, rather than exposed as a
    workbook parameter like `TopN`. Keep all 7 in sync by hand if either
    threshold changes; there is no cross-item validation that would catch
    a missed one.
