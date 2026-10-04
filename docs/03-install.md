# Install

1. Open **Azure Portal → Log Analytics workspace → Workbooks → New**.
2. Switch to the Advanced Editor (`</>` icon).
3. Paste the contents of [`workbook/IntuneHunter.workbook`](workbook/IntuneHunter.workbook).
4. Apply, then **Save**.
5. On first load, update the workbook's **three** resource pickers (Intune
   logs, Entra ID logs, Windows Update reports) to point at your own
   workspace(s). This repo ships with all six subscription/workspace
   parameters set to placeholder values (`<your-subscription-id>` /
   `<your-resource-group>` / `<your-workspace-name>`) that must be replaced,
   either in the portal's resource pickers or by editing the JSON before
   import. If everything lives in one workspace, all three buckets can point
   at the same place.
