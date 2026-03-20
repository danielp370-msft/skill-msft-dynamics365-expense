---
name: dynamics365-expense
description: Create and submit expense reports in Microsoft Dynamics 365 Finance & Operations via Playwright browser automation. Use this skill when asked to add expenses, create expense reports, attach receipts, or submit for reimbursement.
license: MIT
---

# Dynamics 365 Expense Management

## Getting Started

### Step 1: Navigate to the base URL

```
browser_navigate → https://myexpense.operations.dynamics.com
```

The system auto-redirects to `?cmp=XXXX&mi=DefaultDashboard` using the logged-in user's default company. No need to hardcode company codes.

### Step 2: Discover Environment Details

On the **Dashboard** page, read these elements to learn the user's context:

| What | Where to Find |
|------|---------------|
| **Company code & name** | Nav bar button: "Current company is XXXX, activate to choose a company" |
| **User name & email** | Click the initials button (top-right) → shows name and email |
| **Available workspaces** | Dashboard shows workspace buttons (e.g., "Expense management") |

To change company: click the company button → combobox "Current company" appears.

### Step 3: Navigate to Expense Management

Either:
- Click **"Expense management"** button on the Dashboard, OR
- Use nav menu: Workspaces → Expense management, OR
- Direct URL: `https://myexpense.operations.dynamics.com/?cmp=XXXX&mi=ExpenseWorkspace`

## Discovering Expense Categories

Categories are org-specific. To discover available categories:

1. Go to the Expenses tab or click "New expense"
2. Click the **Category** combobox to open the dropdown
3. The dropdown is a **virtualized FixedDataTable grid** — only 3-6 rows render at a time
4. Use `browser_run_code` with `mouse.wheel(0, deltaY)` to scroll through all options
5. Take snapshots to read visible rows as you scroll

**Tip**: Type a partial name in the Category combobox to filter results (e.g., "Hardware", "Airfare", "Hotel").

**Special character quirk**: Some category names contain em-dashes (–) not regular hyphens. `browser_type` can't reliably type these — use the filter + dropdown selection method instead.

## Recommended Workflow

Before diving in, navigate to the Expense Management workspace, take a snapshot, and then **ask the user** what they need help with. Key questions:

1. **What do you want to do?** — Reconcile existing corporate card charges, add new out-of-pocket expenses, create a report, or all of the above?
2. **Open expenses** — Check the "Open expenses" count in the workspace. If there are unattached expenses (e.g., corporate card charges), review them with the user to identify which ones to include.
3. **Receipts / invoices** — Where are the corresponding receipts? Local files, email attachments (searchable via WorkIQ), or already uploaded?
4. **Cost center / IO overrides** — The "New expense report" dialog pre-fills CC and IO from org defaults. Ask if the user needs a different cost center or internal order for this report.
5. **Any special notes** — Pre-approval numbers, chargeback instructions, or comments for the approver?

**Tip**: Review the user's past expense reports (visible in the Reports grid) to understand their typical categories, patterns, and naming conventions.

## How to Create Expenses

### Batch Entry Pattern

Use **Save and new** (Alt+Enter) to create multiple expenses without reopening the form:

```javascript
async (page) => {
  for (const exp of expenses) {
    const catInput = page.locator('input[aria-label="Category"]');
    await catInput.click();
    await catInput.fill(exp.category);
    await page.waitForTimeout(500);
    await page.keyboard.press('ArrowDown');
    await page.keyboard.press('Enter');
    await page.waitForTimeout(300);

    await page.locator('input[aria-label="Transaction date"]').fill(exp.date);
    await page.locator('input[aria-label="Transaction amount"]').fill(exp.amount);
    await page.locator('input[aria-label="Merchant"]').fill(exp.merchant);
    await page.locator('input[aria-label="Description"]').fill(exp.description);

    await page.keyboard.press('Alt+Enter'); // Save and new
    await page.waitForTimeout(2000);
  }
}
```

## How to Create an Expense Report

1. Go to **Reports** tab → click **New expense report**
2. Fill in:
   - **Title/Purpose**: Short descriptive title
   - **Description of Business Purpose**: Detailed description
   - **Expenses**: "Add all" includes all open expenses; "Add none" lets you pick later
   - **Pre-approval number**: Optional — if pre-approval was obtained
3. Review auto-populated fields: interim/final approvers, cost center
4. Click **Create**

### Adding Specific Expenses to a Report

If you chose "Add none" at creation, or need to add more:

1. Open the report from the Reports list (click report number link)
2. Click **Unattached expenses** button
3. Check the expenses to add — identify carefully by amount, merchant, date
4. Click **OK**

**Warning**: The workspace may contain expenses from other sources (e.g., corporate card charges). Be selective.

### Corporate Card Charges Already in D365

**IMPORTANT**: Before creating a new cash expense, always check the **Expenses tab** for existing corporate card charges (CC_Amex, etc.) that match. D365 auto-imports credit card transactions from banks — if the user paid with a corporate card, the charge will already appear as an open expense with the correct amount, date, and merchant (e.g., "RAFI SYDNEY", $890.89, CC_Amex).

In this case, **do NOT create a duplicate cash expense**. Instead:
1. Use the existing CC_Amex charge as-is (it already has the correct total including tip/surcharge)
2. Create a new expense report and include the existing charge via "Add all" or "Unattached expenses"
3. The receipt and guests still need to be attached manually

**How to tell**: Corporate card amounts may differ from the receipt subtotal because they include the full payment (tip, card surcharge, etc.). For example, a receipt showing $810.99 total may have an $890.89 card charge (the extra $79.90 being a tip).

### Adding Guests / Attendees (Required for Meals & Morale)

For **Meals** and **Employee Morale** expense categories, D365 requires listing all attendees in the **Guests** section. This is a compliance requirement — approvers will reject reports with missing guest lists.

**IMPORTANT**: The Guests section is accessed via the **Actions** menu, NOT the detail panel.

1. Open the expense report and click on the meal/morale expense row
2. Click the **Actions** button (in the detail panel toolbar, below the expense card)
3. Click **Guests** from the dropdown menu
4. The Guests dialog opens with a grid and these buttons:
   - **New** — manually add a guest (type alias and name)
   - **Delete** — remove selected guest
   - **Previous guests** — select from people listed on past expense reports
   - **Coworkers** — search the company directory by alias/name
5. Click **OK** to save the guest list

**Using Previous guests**: Opens a dialog with checkboxes for people from your prior reports. Select all that apply, click OK — they're added to the grid.

**Using Coworkers**: Opens a dialog showing the first 50 coworkers. To find a specific person:
1. Type their **alias** (e.g., "sumathew") in the **Filter** combobox and press **Enter**
2. A dropdown appears with filter column options — the default **"User alias"** is usually correct, just press Enter or click it
3. The grid filters to matching results
4. **Important**: The filter dropdown overlay can block checkbox clicks — press **Escape** first to dismiss the dropdown, then click the checkbox with `force: true`
5. Click **OK** to add selected coworkers to the guest list

**Limitation**: Changing the filter deselects previously checked rows. If you need to add people with different aliases, add them one at a time (OK after each search, then re-open Coworkers for the next person).

**How to discover attendees:**
- **Ask the user** who attended the event — they may have a receipt with a headcount or remember the group
- **Check the user's calendar** for the event date — morale events are usually calendar invites with attendees listed
- **Check team chat membership** — if the user mentions a team chat (e.g., a group chat), use `ListChatMembers` to get the roster, then ask which members attended
- **Check previous morale reports** — open a past report with the same category to see who was typically listed
- **Check user-context skill** — team member aliases and details may be documented there

**Tip**: The Guests field in D365 shows as a comma-separated list of names (e.g., "Amos Robinson, Anjani Kumar Verma, Ashish Jain"). After saving, the detail panel shows all guests below the Actions toolbar.

### Setting Country/Region (Required per expense)

Each expense in the report needs a Country/region value:

1. Click on an expense row in the report
2. In the detail panel, find "Country/region" combobox
3. Type the country code (e.g., "AUS", "USA") and select
4. Click **Save and continue** to advance to the next expense

**Note**: Setting Country/region may trigger tax calculations (e.g., GST for AUS) — amounts in the grid may change to show tax-exclusive values.

## How to Attach Receipts

### Retrieving Receipts from Email

If the user says the receipt is in their email, use `MailTools-SearchMessages` to find it:

```
MailTools-SearchMessages: "receipt from [restaurant name]" or "jotnot receipt [name]"
```

The search returns message IDs and metadata. To download the attachment:

1. **Search** for the email with `MailTools-SearchMessages`
2. **URL-encode the message ID** before passing to any other Mail tool — Graph message IDs contain `/`, `+`, and `=` characters that break API calls:
   - `/` → `%2F`
   - `+` → `%2B`
   - `=` → `%3D`
   - See the **m365-mcp-tools** skill for full details on this known gotcha
3. **Get attachments** with `MailTools-GetAttachments` using the URL-encoded message ID
4. **Download** with `MailTools-DownloadAttachment` (attachment IDs use URL-safe base64 and do NOT need encoding)
5. **Save to disk** — decode the base64 content and write to a file:
   ```bash
   echo "<base64_content>" | base64 -d > ~/Downloads/receipt.pdf
   ```

**Example URL-encoding in practice:**
```
Raw message ID:    AQMkADBh...AADt/00xOu0ekm1rgcnW/5MQwcA...HwS+yVwAA...YQAAAA==
Encoded for tools: AQMkADBh...AADt%2F00xOu0ekm1rgcnW%2F5MQwcA...HwS%2ByVwAA...YQAAAA%3D%3D
```

### Programmatic Receipt Upload

D365's `/filemanagement` endpoint blocks direct browser uploads with `ERR_ACCESS_DENIED`. The working approach is a **two-phase upload**: use Playwright route interception to capture the dynamic form fields D365 generates, then use `curl` from bash to POST the actual file with those fields.

> **Why route interception alone doesn't work**: Playwright's `request.postDataBuffer()` does NOT capture binary file content in multipart form data — the file part arrives empty (0 bytes). D365 accepts the upload but stores an empty file, then rejects it on attach with "file size is empty".

#### Step 1: Open the upload dialog and select the file

Use individual Playwright tool calls (NOT `browser_run_code` — you need the file chooser modal):

1. Click the **Receipts** tab on the expense report
2. Click **Add receipts** → the Receipts dialog opens with "Add new" tab
3. Click **Browse** → triggers a file chooser modal
4. Use `browser_file_upload` with `paths: ["/path/to/receipt.pdf"]`
5. Verify the filename appears in the "File to upload" textbox and the **Upload** button is enabled

#### Step 2: Intercept form fields (browser_run_code)

Install a route interceptor, click Upload, capture the dynamic fields, then abort the request so `curl` can send the real one:

```javascript
async (page) => {
  let capturedData = null;

  await page.route('**/filemanagement', async (route) => {
    const request = route.request();
    if (request.method() === 'POST') {
      const postData = request.postData();
      const headers = request.headers();

      const fields = {};
      const fieldRegex = /name="([^"]+)"\r\n\r\n([^\r]*)/g;
      let match;
      while ((match = fieldRegex.exec(postData)) !== null) {
        if (match[1] !== 'files[]') {
          fields[match[1]] = match[2];
        }
      }

      capturedData = {
        csrfToken: headers['ms-dyn-csrftoken'],
        bsid: headers['ms-dyn-bsid'],
        cookie: headers['cookie'],
        fields: fields
      };

      await route.abort();
    } else {
      await route.continue();
    }
  });

  await page.getByRole('button', { name: 'Upload' }).click();
  await page.waitForTimeout(3000);
  await page.unroute('**/filemanagement');

  return JSON.stringify(capturedData, null, 2);
}
```

The returned JSON contains:
- `csrfToken`, `bsid`, `cookie` — authentication headers
- `fields.clientId` — session-specific upload client ID
- `fields.accesstoken` — time-limited upload authorization token
- `fields.tableid`, `fields.recid`, `fields.companyid` — record identifiers
- `fields.docuname`, `fields.docutypeid` — document metadata

#### Step 3: Upload via curl

> **CRITICAL: Cookie handling** — The captured `cookie` string is 4000+ characters (D365 auth uses chunked cookies: `DynamicsOwinAuthC1`, `DynamicsOwinAuthC2`). Shell variables will silently truncate it, causing HTTP 302 redirects. **Always write the cookie to a curl config file** using Python:

```python
# Write curl config file with the full cookie (avoids shell truncation)
python3 -c "
cookie = '''<paste full cookie string from captured JSON>'''
csrf = '<csrfToken value>'
bsid = '<bsid value>'
with open('/tmp/d365_upload.conf', 'w') as f:
    f.write(f'url = \"https://myexpense.operations.dynamics.com/filemanagement\"\n')
    f.write(f'header = \"Accept: application/json, text/javascript, */*; q=0.01\"\n')
    f.write(f'header = \"X-Requested-With: XMLHttpRequest\"\n')
    f.write(f'header = \"ms-dyn-bsid: {bsid}\"\n')
    f.write(f'header = \"ms-dyn-csrftoken: {csrf}\"\n')
    f.write(f'cookie = \"{cookie}\"\n')
"
```

Then upload:

```bash
curl -s -o /tmp/upload_response.txt -w "%{http_code}" \
  -K /tmp/d365_upload.conf \
  -X POST \
  -F "clientId=${CLIENT_ID}" \
  -F "maxChunkSize=1024000" \
  -F "tableid=${TABLE_ID}" \
  -F "recid=${REC_ID}" \
  -F "companyid=${COMPANY_ID}" \
  -F "accesstoken=${ACCESS_TOKEN}" \
  -F "notes=" \
  -F "docuname=${DOCU_NAME}" \
  -F "docutypeid=File" \
  -F "ischunked=false" \
  -F "docuRefRecId=0" \
  -F "files[]=@/path/to/receipt.pdf;type=application/pdf"
```

**Expected results:**
- ✅ HTTP **200** with JSON `[{"fileId":"<guid>"}]` — upload succeeded
- ❌ HTTP **302** — cookie was truncated; rewrite using the config file approach
- ❌ HTTP **401/403** — token expired; re-run Step 2 to get fresh tokens

#### Step 4: Close dialog and verify

1. Click **Close** on the Receipts upload dialog
2. D365 auto-detects the server-side upload — the receipt appears in the Receipts tab
3. Verify the Receipts count incremented (e.g., "Receipts: 1")

#### Step 5: Attach receipt to expense

See [Attaching Uploaded Receipts to Individual Expenses](#attaching-uploaded-receipts-to-individual-expenses) below. Alternatively, from the Receipts tab, select the receipt checkbox and click **Attach to expense**, then click the add icon on the target expense row.

**What does NOT work** (tested and failed):
- **Playwright route interception** (`page.request.fetch(request)`, `route.fetch()`) → multipart binary content is lost, file stored as 0 bytes
- **`page.evaluate` + `fetch()`** from browser context → can't access local files
- **`page.setInputFiles()`** without the Browse→file chooser flow → D365 page crash
- **JS `DataTransfer` API** → D365 ignores synthetic events (`isTrusted: false`)
- **Direct OData API** (`/data/DocumentAttachments`) → 404/403, not exposed on this instance

**Finding receipt files on disk**: Check `~/Downloads/`, `/mnt/c/Users/<username>/Downloads/`, or use `glob`/`ls` to search for `*receipt*`, `*invoice*`, `*.pdf`.

### Reading Receipt Content (Scanned PDFs)

Many receipts are scanned images (e.g., JotNot) — `pdftotext` won't extract anything useful. To read the amount, date, and merchant from an image-based receipt PDF:

1. **Convert PDF to PNG** using ImageMagick (available on this system):
   ```bash
   convert -density 200 '/path/to/receipt.pdf[0]' '/path/to/receipt.png'
   ```
2. **View the image** using the `view` tool — it renders image files and lets you read the receipt visually:
   ```
   view → /path/to/receipt.png
   ```
3. Read the total, date, merchant name, and any other details from the rendered receipt image.

**Tip**: The `[0]` suffix in the convert command selects only the first page. Use `[1]` for the second page, etc. A density of 200 DPI gives good readability without excessive file size.

### Attaching Uploaded Receipts to Individual Expenses

Once receipts are uploaded to the report, attach them to each expense:

1. Click on an expense row in the report grid
2. In the detail panel, find **Receipts** section → click **Edit** link
3. Click **Add receipts**
4. Switch to **Select existing** tab
5. Check the checkbox next to the correct receipt — **use `force: true` click** (a textbox element overlays the checkbox and intercepts pointer events)
6. Click **Add**
7. Click **Close** — **scope to the dialog**: `page.getByLabel('Edit receipts').getByRole('button', { name: 'Close' })` to avoid strict mode violations from multiple Close buttons on the page

## Changing Expense Categories

To change a category on an existing expense in a report:

1. Click the expense row, then click the **Edit** link (id contains "EditExpense")
2. In the Edit expense dialog, change the **Category** combobox
3. A confirmation dialog appears: *"By changing the expense category, some information that was previously saved will be lost. Do you want to continue?"* → Click **Yes**
4. Close the dialog: `page.getByLabel('Edit expense').getByRole('button', { name: 'Close' }).click({ force: true })`
5. **Save and continue** to persist

**Warning**: The Close button on Edit expense dialog sometimes navigates back to the workspace. Use `force: true` and verify page title afterwards.

## Travel Expenses — Airfare & Booking Fees

When submitting travel expenses with GBT (or similar travel agency) booking fees:

- **Keep all travel-related charges under "Airfare" category** — including air booking fees, hotel booking fees, and other travel agent charges
- Use the **Service class** field to distinguish fee types: set to "Other charges" for fees, "Business class"/"Economy" etc. for tickets
- **Do NOT use "Hotel" category for hotel booking fees** — Hotel requires itemisation (subcategories like Daily Room Rate, Hotel Tax, etc.) which adds complexity for simple flat fees

**Tip**: Look at the user's previous travel expense reports to see what categories and patterns they used — consistency with prior submissions avoids approval friction.

## Policy Compliance

### Blocking Errors (must fix before submit)
- **Missing Country/region**: Must be set on every expense
- **Missing receipt**: Must attach a receipt file to each expense

### Informational Warnings (non-blocking)
- **Procurement/SSPA notice**: Appears on Hardware/Lab Supplies — informational only
- **"See policy" on Hotel category**: Triggers itemisation requirement — avoid by using Airfare for simple booking fees

### Visual Indicators in the Expense Grid
- ✅ Green checkmark = "This expense does not have any policy violations"
- ⚠️ Yellow/info icon = "See policy" — click SEE POLICY button to check if blocking or informational

## Verifying Uploaded Receipts

After uploading a receipt via the two-phase method, verify it's not corrupt:

### What to Check

1. **HTTP response** — curl must return HTTP **200** with JSON `[{"fileId":"<guid>"}]`. Any other code means failure.
2. **Receipt count** — The workspace counter (e.g., "Receipts: 1") should increment after closing the upload dialog.
3. **Thumbnail image** — In the Receipts tab, the receipt card shows a PDF icon (red Acrobat logo) for valid PDFs. A broken image or generic file icon suggests corruption.
4. **Persistence after refresh** — Reload the page (`browser_navigate` back to the workspace). The receipt should still appear with the same count.
5. **Close the upload dialog first** — D365 auto-detects the server-side upload only after the dialog is closed. The receipt appears in the Receipts tab at that point.

### Download Verification (MD5 Hash Comparison)

You **can** download the uploaded file and verify its integrity via MD5 hash comparison. The key discovery is that the "Open" button uses `window.open()` with a signed URL — intercept that URL, then download with curl.

#### Step 1: Intercept the download URL

```javascript
async (page) => {
  await page.evaluate(() => {
    const origOpen = window.open;
    window.__capturedOpen = null;
    window.open = function(...args) {
      window.__capturedOpen = args[0];
      return origOpen.apply(this, args);
    };
  });
  await page.getByRole('button', { name: 'Open', exact: true }).click();
  await page.waitForTimeout(3000);
  return await page.evaluate(() => window.__capturedOpen);
}
```

This returns a relative URL like: `filemanagement/{fileId}?access_token=...&ms-dyn-caid=...&ms-dyn-bsid=...`

#### Step 2: Download via curl with cookie-only config

**Critical**: Use a **cookie-only** curl config (no `url` line). If you reuse the upload config, curl makes two requests and the output gets mixed up.

```python
# Write download-only config
with open('/tmp/d365_download.conf', 'w') as f:
    f.write(f'cookie = "{cookie}"\n')
```

```bash
curl -s -o /tmp/verified_receipt.pdf \
  -K /tmp/d365_download.conf \
  "https://myexpense.operations.dynamics.com/{captured_url}"
```

#### Step 3: Compare MD5

```python
import hashlib
def md5(path):
    with open(path, 'rb') as f:
        return hashlib.md5(f.read()).hexdigest()

assert md5('/tmp/verified_receipt.pdf') == md5('/path/to/original.pdf')
```

**Tested result**: HTTP 200, exact size match (189,246 bytes), valid PDF header, **MD5 identical** to the original file.

### Warning Banner After Upload

After the route interception aborts the browser-side upload, D365 may show a yellow warning banner: *"[filename] failed to upload. Please try to upload the document again."* This is expected — it refers to the **aborted Playwright upload**, not the curl upload that actually succeeded. The warning clears on page refresh. Verify the receipt is present in the Receipts tab to confirm success.

### Corruption Root Cause (Historical)

Previous corruption occurred when using Playwright's `postDataBuffer()` to forward multipart POST requests — the binary file content was silently dropped, resulting in 0-byte files on the server. D365 would accept these but then reject them when attaching to an expense with "file size is empty". The two-phase approach (intercept fields → curl upload) completely avoids this because curl handles the binary file directly.

## Recalling Submitted Reports

To modify a submitted report (e.g., to change approvers or add expenses), you must first recall it:

1. Go to the **Reports** tab in the Expense Management workspace
2. Click the report row to select it
3. Click the **Recall** button in the toolbar
4. A comment dialog appears — enter a reason (e.g., "Updating interim approvers")
5. Click **Confirm** / **OK**
6. Report status changes from "In review" back to "Draft"

**Note**: All approvers lose their assignment when a report is recalled. You'll need to resubmit after making changes.

## Managing Interim Approvers

Interim approvers are optional additional approvers before the final approver. You may need to add or remove them (e.g., when an approver is on leave).

### Removing an Interim Approver

The interim approvers field is **read-only** on the report summary view. To modify it:

1. **Recall** the report first (if status is "In review") — see [Recalling Submitted Reports](#recalling-submitted-reports)
2. Open the report by clicking the report number
3. Click **Actions** in the header toolbar
4. Click **"Edit expense report"** from the dropdown menu
5. In the Edit dialog, click **"Select interim approvers"** button
6. A grid shows current interim approvers — select the row to remove
7. Click **Delete** → confirm **"Yes"** in the confirmation dialog
8. Click **OK** to close the interim approvers dialog
9. **D365 quirk — double dialog**: A second "Interim approvers" dialog may appear with an empty row. Just click **OK** again to dismiss it.
10. Click **OK** on the Edit expense report dialog
11. **Resubmit** the report — it will now route without the removed approver

### Batch Processing Multiple Reports

If you need to update multiple reports (e.g., removing the same approver from all pending reports):

1. Recall all reports first (they must be in "Draft" status)
2. Process each one: open → Actions → Edit → Select interim approvers → Delete → OK → OK → OK → Submit
3. After submitting all reports, verify the workspace shows "Open reports: 0"

## Deleting Test Receipts

To clean up test uploads from the Receipts tab:

1. Go to the **Receipts** tab in the workspace
2. Click the **checkbox** next to the receipt to select it
3. Click the **Delete** button in the toolbar
4. Confirm: **"Are you sure you want to delete the selected items?"** → Click **Yes**
5. Wait for processing (may show "Please wait" overlay)
6. Verify the Receipts count returns to 0

## OData API Status

D365 Finance & Operations exposes OData entities for expense management, but they may not be accessible depending on your organization's security configuration.

### Entities That Exist (Found in `$metadata`)

| Entity | Purpose |
|--------|---------|
| `Expenses` | Main expenses |
| `TrvReceipts` | Travel receipts |
| `UploadReceipts` | Receipt uploads |
| `ExpenseCategories` | Available categories |
| `ecpTrvExps` | Expense transactions |
| `ecpTrvInterimApprovers` | Interim approvers |
| `MobileExpensesV2` | Mobile expense API |

### Why They Return 403

The OData API requires specific D365 security roles (e.g., read access to `TrvExpTransEntity`) that are separate from UI access. Error: *"User is not authorized to read view TrvExpTransEntity. Request denied."*

**To enable**: A D365 admin must assign the user the appropriate data entity security roles (e.g., `TrvExpenseTransEntity` read/write privileges).

### Bearer Token vs Cookie Auth

| Endpoint | Bearer Token (`az account get-access-token`) | Cookie Auth (Playwright) |
|----------|----------------------------------------------|--------------------------|
| `/data/*` (OData) | 403 (security role needed) | N/A |
| `/filemanagement` (uploads) | Returns login page (rejected) | ✅ Works |
| D365 UI pages | N/A | ✅ Works |

**Bottom line**: Until OData security roles are granted, Playwright + curl is the only working upload method. The bearer token approach (`az account get-access-token --resource https://myexpense.operations.dynamics.com`) gets a valid token, but the endpoints either require OData roles or don't accept bearer auth.

## Submitting the Report

1. Open the report → verify all expenses show green policy compliant icons
2. Click **Submit**
3. A **"Expense Management Workflow - Submit"** dialog appears with a **Comment** field
4. Add any relevant notes (e.g., approval context, chargeback info)
5. Click **Submit** in the dialog
6. Report moves to "In review" status and routes to interim → final approvers

## Known Limitations & Workarounds

| Issue | Workaround |
|-------|-----------|
| Email attachment download fails | URL-encode the message ID (`/`→`%2F`, `+`→`%2B`, `=`→`%3D`) — see m365-mcp-tools skill |
| File uploads lose binary content via route interception | Playwright's `postDataBuffer()` drops multipart file content. **Fix: capture form fields via route interception, then upload the actual file via `curl` from bash.** See [Programmatic Receipt Upload](#programmatic-receipt-upload) section. |
| Cannot download uploaded receipt for verification | **Fixed**: Intercept `window.open()` URL from the Open button, then `curl` with a cookie-only config (no `url` line). Enables full MD5 hash comparison. See [Download Verification](#download-verification-md5-hash-comparison). |
| OData API returns 403 | D365 OData entities exist but require security roles not granted by default. See [OData API Status](#odata-api-status). |
| Bearer token rejected by `/filemanagement` | This endpoint only accepts cookie-based auth. Must use Playwright to obtain cookies. |
| "Failed to upload" warning banner after two-phase upload | This is from the aborted Playwright upload, not the curl upload. Ignore it — verify receipt in the Receipts tab. Clears on page refresh. |
| Grid row CSS IDs are unstable | Use `aria-label` selectors, not CSS IDs |
| Virtualized grids render few rows | Scroll with `mouse.wheel()` via `browser_run_code` |
| Em-dash in category names | Filter by partial name + select from dropdown |
| Multiple "Close" buttons on page | Scope to dialog: `page.getByLabel('Dialog Name').getByRole('button', { name: 'Close' })` |
| Edit expense Close sometimes navigates away | Use `force: true`, verify page title after click |
| Hotel category requires itemisation | Use Airfare with Service class "Other charges" for simple fees |
| Interim approvers field is read-only on report view | Must use Actions → Edit expense report → Select interim approvers workflow |
| D365 pages load slowly | Add `waitForTimeout(3000-5000)` after navigation. Console SSL errors (`ERR_SSL_UNRECOGNIZED_NAME_ALERT`) are normal and don't affect functionality. |
| "Please wait" overlay blocks interactions | Wait for the overlay to disappear before clicking. Use `browser_wait_for` with appropriate time. |
| MCP server auth tokens expire | M365 MCP tools (Mail, Calendar, Teams) use OAuth tokens that expire. Check with health check scripts and re-authenticate with `/mcp` command when needed. |
| Teams MCP scope errors | `TeamsServer` may return AADSTS9010010 scope errors. Workaround: use Playwright to navigate to `teams.cloud.microsoft` and interact via browser automation. |

## Environment & Platform Notes

### Running from Linux / CDE / WSL

This skill is designed to run from a Linux environment (Cloud Development Environment, WSL, or native Linux) with:

- **Playwright MCP server** controlling a Chromium-based browser (typically Edge)
- **bash / curl / python3** available for the two-phase upload
- **az cli** authenticated for `az account get-access-token` (useful for future OData access)

### Browser Session Management

- D365 uses cookie-based authentication with chunked cookies (`DynamicsOwinAuthC1`, `DynamicsOwinAuthC2`) that can be 4000+ characters total
- Browser sessions are maintained by Playwright — if the session expires, navigate back to the D365 URL and it will re-authenticate via SSO
- The CSRF token, bsid, and access tokens are session-scoped and expire quickly — always capture fresh tokens immediately before each upload

### Multi-Company Support

D365 instances can host multiple companies (legal entities). The company code appears in the URL (`?cmp=XXXX`) and the nav bar shows the current company name. To switch:

1. Click the company button in the nav bar (e.g., "Current company is 1074")
2. Change the **"Current company"** combobox
3. The workspace refreshes with that company's data

**Important**: Expense reports, cost centers, and approvers are company-specific. Verify you're in the correct company before creating reports.

### Personal Card vs Corporate Card Expenses

- **Corporate card** charges (e.g., CC_Amex) auto-import into D365 as open expenses — reconcile these, don't duplicate them
- **Personal card** expenses require manual cash expense creation and result in reimbursement to the employee
- Keep corporate card and personal card expenses in **separate reports** — they have different payment processing workflows

## Tips

- **Save and new** (Alt+Enter) for batch expense entry is much faster than Save + click New
- **Discover before hardcoding**: Read company, worker, approvers, cost centers from the UI rather than assuming values
- **Review past reports**: Click on completed reports in the Reports grid to see how previous expenses were categorised — this reveals org-specific patterns
- **Corporate card expenses** (e.g., CC_Amex) may auto-appear as open expenses — only add relevant ones to your report
- **Country/region triggers tax recalculation** — amounts in the grid change after setting it
- **Receipt thumbnails** in the "Select existing" tab help identify the right file visually
- **Upload immediately after intercepting** — Access tokens in the captured form fields expire quickly. Don't delay between Phase 1 (interception) and Phase 2 (curl upload).
- **Verify receipt after every upload** — Close the upload dialog, check the Receipts count incremented, and confirm the PDF icon appears. Don't assume success from the curl HTTP 200 alone.
- **Keep temp files tidy** — Clean up `/tmp/d365_upload.conf` and downloaded receipts after each session to avoid stale tokens being reused
