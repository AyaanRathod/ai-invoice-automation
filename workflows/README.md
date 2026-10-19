# Workflows

The exported workflow definitions will be added here:

| File | What it is |
|---|---|
| `logic-app.json` | The Azure Logic App definition (trigger, Get file content, Analyze_invoice, For each, Compose, Create file) |
| `power-automate-flow.zip` | The Power Automate flow package (Parse JSON, Excel rows, approval, email) |

## How to export them

**Logic App**

1. Open the Logic App in the Azure portal.
2. Go to **Development Tools > Logic app code view**.
3. Copy all of the JSON and save it here as `logic-app.json`.

**Power Automate flow**

1. Go to https://make.powerautomate.com and open **My flows**.
2. Select the **...** next to `Invoice JSON to Excel with approval`.
3. Select **Export > Package (.zip)**, give it a name, and select **Export**.
4. Save the download here as `power-automate-flow.zip`.

## Scrub before you commit

Before committing `logic-app.json`, search it and replace:

- Your **subscription ID** (in paths like `/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/...`) with `<subscription-id>`
- **Connection IDs** and resource group paths under `$connections` (for example `.../resourceGroups/rg-invoice-pipeline/providers/Microsoft.Web/connections/...`) with `<connection-id>`
- Any **email address** with `you@school.edu`
- Any **tenant ID** with `<tenant-id>`

The Document Intelligence key is stored in the API connection, not in the workflow file. Still, search the file for the first few characters of your key to be sure it is not there.

To import into your own account, you will need to recreate the connections (OneDrive for Business, Document Intelligence, Excel Online, Approvals, Office 365 Outlook) and point them at your own folders and workbook.
