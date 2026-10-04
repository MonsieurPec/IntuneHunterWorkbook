# Requirements

The workbook reads from three independent workspace "buckets", each with its
own Subscription + Workspace picker. All three default to the same workspace
if that's where everything lives; point any bucket elsewhere independently if
your logs are split across workspaces or subscriptions.

**None of the three log sources below are on by default.** Each one requires
a one-time, manual configuration step in its own admin center before any data
reaches Log Analytics. See [Logs and diagnostic
settings](#logs-and-diagnostic-settings).

## Table of contents

- [Logs and diagnostic settings](#logs-and-diagnostic-settings)
- [Licensing & cost](#licensing--cost)

### Logs and diagnostic settings

- **Intune logs**: a Log Analytics workspace receiving Intune diagnostic
  settings, configured in **Intune admin center → Reports → Diagnostics
  settings** (official guide: [Route logs to Azure Monitor using Microsoft
  Intune](https://learn.microsoft.com/en-us/intune/governance/integrate-azure-monitor)):
  - [`IntuneAuditLogs`](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/intuneauditlogs)
    and [`IntuneDevices`](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/intunedevices)
    (core tabs). Enable the **AuditLogs** and **IntuneDevices** categories.
  - [`IntuneDeviceComplianceOrg`](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/intunedevicecomplianceorg)
    (Device compliance category, needed for the Compliance and All devices
    tabs). Enable the **DeviceComplianceOrg** category.
  - [`IntuneOperationalLogs`](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/intuneoperationallogs)
    (needed for Autopilot builds). Enable the **OperationalLogs** category.
    This one is easy to miss: it is a separate checkbox from AuditLogs in the
    same Diagnostics settings blade, and the Autopilot builds view stays
    empty without it.

- **Entra ID logs**: the same or a linked workspace receiving Entra ID
  `AuditLogs` (needed for the LAPS, PIM and BitLocker tabs only; every other
  tab works without it), configured in **Microsoft Entra admin center → Monitoring &
  health → Diagnostic settings** (official guides: [How to configure
  Microsoft Entra diagnostic
  settings](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-configure-diagnostic-settings)
  and [Integrate Microsoft Entra logs with Azure Monitor
  logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-integrate-activity-logs-with-azure-monitor-logs)).

- **Windows Update reports**: `UCClient`, `UCUpdateAlert` and `UCDOStatus`
  from [Windows Update for Business
  reports](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-overview),
  enrolled separately per tenant from the Azure Portal and pointed at a
  workspace (needed for Windows Update errors and the whole Delivery
  Optimization tab only). See [Prerequisites for Windows Update for Business
  reports](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-prerequisites)
  and [Enable Windows Update for Business
  reports](https://learn.microsoft.com/en-us/windows/deployment/update/wufb-reports-enable).
  Enrollment can take up to 24 hours before the first data appears. Not
  available in GCC High or DoD tenants.

- Reader access to every workspace involved, to import and use the workbook:
  the [Monitoring Reader built-in
  role](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles#monitoring-reader)
  is sufficient; see [Manage access to log data and workspaces in Azure
  Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access)
  for details. This role exposes the full detail behind this workbook's
  queries — including LAPS recovery requests, PIM role activations and
  UPNs — to anyone who holds it on the workspace, with no finer-grained
  filtering. Restrict it to security/IT teams rather than granting it
  broadly.

### Licensing & cost

Creating, viewing and editing Azure Workbooks carries **no charge of its
own**, and there is no charge for querying data already ingested into a Log
Analytics workspace ([Azure Workbooks
overview](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/workbooks-overview);
confirmed by Microsoft support in [this Microsoft Q&A
thread](https://learn.microsoft.com/en-us/answers/questions/219617/pricing-for-using-azure-workbooks-and-dashboards):
*"there are no charges for querying the data captured by Application Insights
or Log Analytics"*). The only real costs are the ones you'd already be paying
independently of this workbook:

- **Log Analytics ingestion and retention** for whichever tables above you
  route into a workspace. See [Azure Monitor
  pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/).

- **Azure Storage or Event Hubs**, only if you choose either as an *additional*
  destination for the Intune diagnostic settings (routing to Log Analytics
  alone is enough for this workbook). Cost estimates for both are in the
  [Intune Azure Monitor integration
  guide](https://learn.microsoft.com/en-us/intune/governance/integrate-azure-monitor#cost-considerations).

The one thing that *does* carry a separate cost is turning a workbook into a
scheduled report: the **Workbook automation** feature that emails a
workbook's results on a recurring schedule runs on Azure Logic Apps under the
hood, which bills per execution (see [Automate custom reports with
Application Insights
data](https://learn.microsoft.com/en-us/azure/azure-monitor/app/automate-custom-reports)
and [Azure Logic Apps
pricing](https://azure.microsoft.com/en-us/pricing/details/logic-apps/)).
This workbook does not configure or require that automation; opening and
using it in the portal stays free.
