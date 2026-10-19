# Setup Guide

This guide rebuilds the AI Invoice Automation pipeline from scratch. It assumes you have:

- An **Azure subscription** (Azure for Students works).
- A **Microsoft 365 work or school account** with OneDrive for Business, Excel Online, Outlook and Power Automate. This guide uses `you@school.edu` as the example account.

Microsoft moves buttons often. If a button is not where this guide says, search for its exact name on the page.

**Excel rule:** keep `Invoices.xlsx` closed while the flows run. Excel files can stay locked for a few minutes after a connector writes to them, and editing by hand at the same time causes conflicts.

```
PDF uploaded to /Invoices/Incoming
  -> Logic App: trigger -> Get file content -> Analyze_invoice -> For each -> Compose JSON -> Create file in /Invoices/Processed
  -> Power Automate: When a file is created (/Invoices/Processed)
       -> Parse JSON -> Add a row into a table (Status Pending)
       -> Amount > 1000 ? approval -> Update a row Approved or Rejected
       -> else Update a row Approved
       -> Send an email (V2)
```

---

## Part 1. Document Intelligence

1. In the Azure portal select **Create a resource**, search **Document Intelligence**, select **Create**.
2. Fill in:
   - **Resource group:** for example `rg-invoice-pipeline` (a resource group's region does not limit the resources inside it).
   - **Region:** one your subscription allows. Canada Central was blocked for this service on Azure for Students, so this project used **East US**.
   - **Name:** something unique, like `yourname-docintel-01`.
   - **Pricing tier:** **Free F0** (500 pages a month, first two pages of each PDF, files under 4 MB).
3. **Review + create**, then **Create**, then **Go to resource**.
4. Open **Keys and Endpoint** and copy **KEY 1** and the **Endpoint** into a private notes file.

**If validation fails with "disallowed by policy":** open **Policy > Assignments**, open the allowed locations assignment, check its parameter values for the allowed regions, and pick one of those.

Treat the key like a password. Never commit it, screenshot it or paste it in chat.

## Part 2. OneDrive folders and the Excel table

1. In OneDrive for Business create this layout under **My files**:

```
Invoices/
  Invoices.xlsx    (table: InvoiceTable)
  Incoming/        (PDFs go here)
  Processed/       (the Logic App writes JSON files here)
```

2. Open `Invoices.xlsx` (you can start from `data/Invoices_sample.xlsx` and delete the data rows). Make sure:
   - The table is named **InvoiceTable** (check on the **Table Design** tab).
   - The headers are exactly `Vendor`, `InvoiceId`, `DueDate`, `Amount`, `Confidence`, `Status`.
   - Column C (DueDate) is formatted as **Text**, so dates do not turn into serial numbers like `46326`.
3. Close the tab.

## Part 3. The Logic App (Azure)

### 3.1 Create it

1. Portal search **Logic apps**, select **Add**, choose **Consumption (Multi-tenant)**.
2. Use the same resource group and the **same region as Document Intelligence**. Name it something like `la-invoice-reader`. Log analytics **No**.
3. **Review + create**, **Create**, **Go to resource**, then open the **designer** under **Development Tools** and choose a blank workflow.

In the designer, the **lightning icon** inserts outputs from earlier steps and the **fx** icon opens the expression editor. Save often.

### 3.2 Trigger and Get file content

1. **Add a trigger**: **OneDrive for Business > When a file is created (properties only)**. Do not pick the personal OneDrive connector or anything marked [DEPRECATED]. Sign in with `you@school.edu`.
2. **Folder:** `/Invoices/Incoming` (pick it with the folder icon). Leave **Split On** turned on. Set the check interval to **3 Minute**. Every check is a billed call, so do not go lower.
3. Add **OneDrive for Business > Get file content**. **File:** the trigger's file **Id**. **Infer Content Type:** Yes.

Using the properties-only trigger plus a separate Get file content step fixed a `BadRequest` error where the AI step received no file content.

The trigger ignores files over 50 MB and does not count files moved inside OneDrive as new. Always upload a fresh PDF.

### 3.3 Analyze the invoice

1. Add **Azure AI Document Intelligence > Analyze Document for Prebuilt or Custom models (v4.x API)**.
2. Connection: authentication type **API Key**, paste your Endpoint and KEY 1.
3. Parameters:
   - **API version:** `2024-11-30`
   - **Model Identifier:** `prebuilt-invoice`
   - **Document/Image File Content:** **File content** from **Get file content**
4. Rename the step to `Analyze_invoice`. The expressions below use this exact name.

### 3.4 For each loop

Add **Control > For each**. In **Select An Output From Previous Steps**, use **fx**:

```
body('Analyze_invoice')?['analyzeResult']?['documents']
```

Check the loop's real name in **Code view**. This guide assumes `For_each`. If yours is `For_each_1`, change it in every expression below.

### 3.5 Compose the JSON (inside the loop)

Add **Data Operations > Compose**, rename it `Build_invoice_json`, and set **Inputs** with **fx** to this one-line expression:

```
setProperty(setProperty(setProperty(setProperty(setProperty(json('{}'), 'Vendor', items('For_each')?['fields']?['VendorName']?['content']), 'InvoiceId', items('For_each')?['fields']?['InvoiceId']?['content']), 'DueDate', formatDateTime(items('For_each')?['fields']?['DueDate']?['valueDate'], 'yyyy-MM-dd')), 'Amount', coalesce(items('For_each')?['fields']?['AmountDue']?['valueCurrency']?['amount'], items('For_each')?['fields']?['InvoiceTotal']?['valueCurrency']?['amount'])), 'Confidence', coalesce(items('For_each')?['fields']?['AmountDue']?['confidence'], items('For_each')?['fields']?['InvoiceTotal']?['confidence']))
```

It starts with an empty object and adds one field at a time. `coalesce(a, b)` means "use a, and if it is empty use b", because the model may return the total as AmountDue or InvoiceTotal.

### 3.6 Create file (inside the loop)

Add **OneDrive for Business > Create file**:

- **Folder Path:** `/Invoices/Processed`
- **File Name** (fx):

```
concat(coalesce(items('For_each')?['fields']?['InvoiceId']?['content'], 'unknown'), '_', formatDateTime(utcNow(), 'yyyyMMddHHmmss'), '.json')
```

- **File Content** (fx):

```
string(outputs('Build_invoice_json'))
```

### 3.7 Test the Logic App

1. Make sure it is **Enabled**, then **Run > Run** in the designer.
2. Upload `sample_invoices/INV-1002.pdf` to **Incoming**.
3. Within about 3 minutes, **Overview > Run history** should show a green run and **Processed** should contain a file like:

```
{"Vendor":"Lakeshore Cloud Hosting Inc.","InvoiceId":"INV-1002","DueDate":"2026-11-02","Amount":1384.25,"Confidence":0.43}
```

| Problem | Fix |
|---|---|
| Error on `formatDateTime` | The due date was empty. Use `items('For_each')?['fields']?['DueDate']?['valueDate']` without formatDateTime |
| "cannot be evaluated" or "property doesn't exist" | A step name does not match. Check **Code view** |
| Amount is `null` | Find the field with `valueCurrency` in the Analyze_invoice raw output and use its name |
| 401 or "invalid subscription key" | Edit the API connection and re-paste the key and full endpoint (with `https://` and the trailing `/`) |
| Nothing happens | Upload a new file. Files uploaded before the Logic App was saved and enabled are ignored |

## Part 4. The Power Automate flow

Why Power Automate: some Microsoft 365 tenants require admin consent before Azure Logic Apps can use the Excel Online connector. Power Automate runs inside the tenant, so Excel works there.

### 4.1 Trigger

1. At https://make.powerautomate.com, signed in as `you@school.edu`, select **My flows > New flow > Automated cloud flow**.
2. Name it `Invoice JSON to Excel with approval`. Trigger: **OneDrive for Business > When a file is created**.
3. **Folder:** `/Invoices/Processed`. **Infer Content Type:** No.

### 4.2 Parse JSON

Add **Data Operations > Parse JSON**.

- **Content** (fx):

```
json(base64ToString(triggerBody()?['$content']))
```

- **Schema:**

```json
{
  "type": "object",
  "properties": {
    "Vendor": { "type": ["string", "null"] },
    "InvoiceId": { "type": ["string", "null"] },
    "DueDate": { "type": ["string", "null"] },
    "Amount": { "type": ["number", "null"] },
    "Confidence": { "type": ["number", "null"] }
  }
}
```

If Parse JSON says "$content not found", add OneDrive **Get file content** after the trigger (File = the trigger's file Id) and use `json(base64ToString(body('Get_file_content')?['$content']))` instead.

**Important:** in every later step, take values from Parse JSON with these expressions (use **fx**), not from the lightning list of another step:

| Field | Expression |
|---|---|
| Vendor | `body('Parse_JSON')?['Vendor']` |
| InvoiceId | `body('Parse_JSON')?['InvoiceId']` |
| DueDate | `body('Parse_JSON')?['DueDate']` |
| Amount | `body('Parse_JSON')?['Amount']` |
| Confidence | `body('Parse_JSON')?['Confidence']` |

Values picked from the **Add a row into a table** output (the green Excel icon) come back blank, which caused empty emails and failed row lookups in testing.

### 4.3 Add a row into a table

Add **Excel Online (Business) > Add a row into a table**:

- **Location:** OneDrive for Business. **Document Library:** OneDrive.
- **File:** `/Invoices/Invoices.xlsx`. **Table:** `InvoiceTable`.
- Vendor, InvoiceId, DueDate, Amount, Confidence: the Parse JSON expressions above.
- **Status:** type `Pending`.

### 4.4 Condition: over $1,000?

Add **Control > Condition**. First box (fx):

```
float(string(body('Parse_JSON')?['Amount']))
```

Operator **is greater than**, value `1000`.

### 4.5 True branch: ask for approval

1. Add **Start and wait for an approval**. Type **Approve/Reject - First to respond**.
   - **Title:** `Approve invoice ` (with a trailing space) then the InvoiceId expression.
   - **Assigned to:** `you@school.edu`.
   - **Details:** `Vendor: `, `Amount: `, `Due date: ` with the matching expressions.
2. Under it add another **Condition**: **Outcome** (from the approval step) **is equal to** `Approve`.
3. Inner **True:** **Excel Online (Business) > Update a row**:
   - Same Location, Document Library, File and Table as 4.3.
   - **Key Column:** `InvoiceId`. Double check this. Setting it to Status makes every lookup fail with "No row was found".
   - **Key Value:** `body('Parse_JSON')?['InvoiceId']`
   - **Status:** `Approved`. Leave other columns empty (empty columns are not changed).
4. Inner **False:** the same **Update a row** with Status `Rejected`.

### 4.6 False branch: approve automatically

In the outer **False** add the same **Update a row** with Status `Approved`.

### 4.7 Summary email

**Below** the outer Condition (not inside a branch), add **Office 365 Outlook > Send an email (V2)**:

- **To:** `you@school.edu`
- **Subject:** `Invoice processed: ` then the InvoiceId expression.
- **Body:** `Vendor: `, `Amount: `, `Due date: `, `AI confidence: ` each followed by its expression.

Save, and fix anything the Flow checker marks red.

## Part 5. Test

1. Empty the **Incoming** and **Processed** folders (keep the folders). Delete the data rows in `Invoices.xlsx` (keep the header and the table) and close it.
2. Upload the five PDFs from `sample_invoices/`. You can upload them one at a time (easier to debug) or together (each file gets its own run). Select **Run** on the Logic App to skip the wait.
3. INV-1002 and INV-1004 are over $1,000 and send approval requests (Outlook, or **Approvals** in Power Automate). Approve INV-1002 and reject INV-1004.
4. Expected table:

| Vendor | InvoiceId | DueDate | Amount | Status |
|---|---|---|---:|---|
| Niagara Office Supply Ltd. | INV-1001 | 2026-10-31 | 395.48 | Approved |
| Lakeshore Cloud Hosting Inc. | INV-1002 | 2026-11-02 | 1384.25 | Approved |
| St. Catharines Print Co. | INV-1003 | 2026-11-04 | 203.4 | Approved |
| Welland Tech Repair | INV-1004 | 2026-11-07 | 1145.82 | Rejected |
| Thorold Coffee Catering | INV-1005 | 2026-11-09 | 723.2 | Approved |

Total **3,852.15**, plus five summary emails.

Notes:

- Excel can take up to 30 seconds to show a change.
- Uploading the same invoice twice creates two rows with the same InvoiceId, and **Update a row** only changes the first one. Delete duplicates while nothing is running.
- If a run fails, open it from **28-day run history**, select the red step, and read its inputs and outputs. **Resubmit** reruns it with the same data.

## Part 6. Export and turn things off

- **Export the Logic App:** **Development Tools > Logic app code view**, copy the JSON into `workflows/logic-app.json`, and scrub it first (see `workflows/README.md`).
- **Export the flow:** **My flows > ... > Export > Package (.zip)**, save as `workflows/power-automate-flow.zip`.
- **Pause the Logic App:** **Overview > Disable** (each trigger check is a billed call).
- **Turn off the flow:** open it and select **Turn off**.
- **Delete everything in Azure** (only after exporting): **Resource groups > rg-invoice-pipeline > Delete resource group**.

## References

| Topic | Link |
|---|---|
| Document Intelligence with Logic Apps | https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/tutorial/logic-apps?view=doc-intel-4.0.0 |
| Logic Apps expression functions | https://learn.microsoft.com/en-us/azure/logic-apps/expression-functions-reference |
| Logic Apps loops | https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-control-flow-loops |
| OneDrive for Business connector | https://learn.microsoft.com/en-us/connectors/onedriveforbusiness/ |
| Excel Online (Business) connector | https://learn.microsoft.com/en-us/connectors/excelonlinebusiness/ |
| Approvals in Power Automate | https://learn.microsoft.com/en-us/power-automate/modern-approvals |
