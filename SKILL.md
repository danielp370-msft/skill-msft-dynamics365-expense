---
name: dynamics365-expense
description: Create and submit expense reports in Microsoft Dynamics 365 Finance & Operations via Playwright browser automation. Use this skill when asked to add expenses, create expense reports, attach receipts, or submit for reimbursement.
license: MIT
---

# Dynamics 365 Expense Management

This skill drives the D365 F&O Expense Management web UI via Playwright. The base URL pattern is `https://<tenant>expense.operations.dynamics.com` (Microsoft tenants use `myexpense`, others use their own subdomain). Most of this guidance is tenant-agnostic; tenant-specific items (like the receipt-by-email address) are called out where they appear.

## Getting Started

### Step 1: Navigate to the base URL

Ask the user for their D365 expense URL if you don't already know it (or check their browser bookmarks / past sessions). Then:

```
browser_navigate → https://<tenant>expense.operations.dynamics.com
```

The system auto-redirects to `?cmp=XXXX&mi=DefaultDashboard` using the logged-in user's default company. No need to hardcode company codes.

### 💡 Shortcut: Email receipts directly

D365 supports forwarding receipts by email — they auto-arrive on the user's Receipts tab without any upload work. **Look for an info banner on the workspace** that gives the org-specific address (e.g. `ExpenseReceipts@<tenant>.com` for Microsoft, or similar for other tenants).

When this banner is present, **suggest this path first** before doing any programmatic upload:

> "I see your D365 supports emailing receipts to `<address>`. If you forward your PDFs there from your work account, they'll show up automatically — much faster than me uploading them via the API. Want to try that, or should I upload directly?"

Caveat: this feature has to be enabled by the tenant admin and requires emails to come from the user's authenticated work account. If it's not configured or doesn't work, fall back to the [programmatic upload](#programmatic-receipt-upload) below.

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

> Before doing this, check whether the workspace shows an info banner about emailing receipts to a specific address. If yes, recommend that path first (see [Email receipts directly](#-shortcut-email-receipts-directly)) — it's far simpler.

D365's `/filemanagement` endpoint blocks direct browser uploads with `ERR_ACCESS_DENIED` from Playwright. The working approach is **capture tokens → curl-upload binary**.

Three things to get right (in priority order):

1. **Fresh BSID** — `ms-dyn-bsid` (Browser Session ID) changes on every `page.goto()`. Capture tokens from the *current* page session immediately before uploading. Stale BSID = HTTP 200 returned but file stored as 0 bytes (silent corruption).
2. **Cookie filtered to operations.dynamics.com only** — Including cookies from other dynamics.com subdomains (e.g. `CrmOwinAuth` from `.crm.dynamics.com`) causes HTTP 302 redirect to login. Only `ms-dyn-affinity`, `backend-affinity`, `DynamicsOwinAuth`, `ms-dyn-csrftoken`, `DynamicsOwinAuthC1`, `DynamicsOwinAuthC2` should be sent.
3. **Browse → file chooser → `browser_file_upload`** — Never use `page.locator('input[type="file"]').setInputFiles()` directly; it crashes the D365 page.

#### Step 1: Open the upload dialog and select the file

1. Click the **Receipts** tab on the expense report
2. Click **Add receipts** → the Receipts dialog opens with "Add new" tab
3. Set up the file chooser handler **before** clicking Browse, then select the file:

```javascript
async (page) => {
  const filePromise = page.waitForEvent('filechooser', { timeout: 10000 });
  await page.getByRole('button', { name: 'Browse' }).click({ force: true });
  const fc = await filePromise;
  await fc.setFiles(['/path/to/receipt.pdf']);
  await page.waitForTimeout(1500);
  // Verify
  const fileTb = await page.getByRole('textbox', { name: 'File to upload' }).inputValue();
  return fileTb;  // Should be 'receipt.pdf'
}
```

> **Common gotcha**: a stale `page.once('filechooser', ...)` handler from an earlier failed attempt can intercept the chooser and clear it. Reset with `page.removeAllListeners('filechooser')` if Browse stops opening a chooser.

#### Step 2: Intercept tokens and abort the browser-side upload

```javascript
async (page) => {
  let captured = null;
  await page.route('**/filemanagement', async (route) => {
    if (route.request().method() === 'POST') {
      const headers = route.request().headers();
      const fields = {};
      const re = /name="([^"]+)"\r\n\r\n([^\r]*)/g;
      let m;
      while ((m = re.exec(route.request().postData())) !== null) {
        if (m[1] !== 'files[]') fields[m[1]] = m[2];
      }
      captured = {
        bsid: headers['ms-dyn-bsid'],
        csrfToken: headers['ms-dyn-csrftoken'],
        fields
      };
      await page.evaluate((d) => { window.__capturedUpload = d; }, captured);
      await route.abort();
    } else {
      await route.continue();
    }
  });
  page.once('filechooser', async (fc) => { await fc.setFiles([]); });
  await page.getByRole('button', { name: 'Upload' }).click();
  await page.waitForTimeout(3000);
  await page.unroute('**/filemanagement');
  return captured ? { bsid: captured.bsid, clientId: captured.fields.clientId } : 'NO CAPTURE';
}
```

The captured `fields` object contains `clientId`, `tableid`, `recid`, `companyid`, `accesstoken`, `docuname`, `docutypeid` — needed for curl.

#### Step 3: Dump cookies via storageState (cleanest method)

Don't try to parse cookies out of the captured request — use Playwright's `storageState`, which writes them to disk in JSON form:

```javascript
async (page) => {
  await page.context().storageState({ path: '/tmp/d365_storage.json' });
  return 'wrote storage state';
}
```

#### Step 4: Build curl config (filter to operations.dynamics.com)

```python
import json
with open('/tmp/d365_storage.json') as f:
    state = json.load(f)
with open('/tmp/captured_tokens.json') as f:  # write this file via browser_evaluate from window.__capturedUpload
    tk = json.load(f)

# CRITICAL: filter cookies to operations.dynamics.com only.
# Including .crm.dynamics.com cookies causes HTTP 302 redirect.
ops_cookies = [c for c in state['cookies'] if 'operations.dynamics.com' in c.get('domain', '')]
cookie_str = '; '.join(f"{c['name']}={c['value']}" for c in ops_cookies)

with open('/tmp/d365_upload.conf', 'w') as f:
    f.write('header = "Accept: application/json, text/javascript, */*; q=0.01"\n')
    f.write('header = "X-Requested-With: XMLHttpRequest"\n')
    f.write(f'header = "ms-dyn-bsid: {tk["bsid"]}"\n')
    f.write(f'header = "ms-dyn-csrftoken: {tk["csrfToken"]}"\n')  # keep URL-encoded
    f.write(f'cookie = "{cookie_str}"\n')

# Cookie-only config for downloads (verification)
with open('/tmp/d365_download.conf', 'w') as f:
    f.write(f'cookie = "{cookie_str}"\n')
```

#### Step 5: Upload via curl (batch any number of files)

```bash
curl -s -o /tmp/upload_response.txt -w "%{http_code}" \
  -K /tmp/d365_upload.conf \
  -X POST \
  "https://<tenant>expense.operations.dynamics.com/filemanagement" \
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
- ✅ HTTP **200** with JSON `[{"fileId":"<guid>"}]` — file uploaded successfully
- ❌ HTTP **302** → cookies aren't filtered correctly, or wrong domain cookies included
- ❌ HTTP **401/403** → access token expired (re-capture)
- ⚠️ HTTP **200** but file is 0-byte when attaching → **stale BSID** (re-capture from current session)

Same tokens can upload many files — you don't need to re-capture between files in a batch.

#### Step 6: Close the upload dialog

After all curl uploads succeed, close the Receipts upload dialog. D365's Receipts tab will populate automatically with the uploaded files (this is the BSID-fresh case — no `route.fulfill` dance needed).

```javascript
await page.locator('button[id*="CloseButtonAddNewTabPage"]').first().click();
await page.waitForTimeout(2000);
```

Verify: the Receipts count should reflect the new uploads, and each file appears in the receipts list.

#### When `route.fulfill` IS needed

Historically, file size metadata was recorded as 0 bytes when the client-side upload callback didn't fire. The fix was to re-do the upload flow with `route.fulfill()` returning the curl's fileId. **In practice, with a fresh BSID matching the page session, this isn't needed** — D365 auto-detects the server-side files correctly.

If you do see 0-byte file errors after attaching, redo the upload for the affected file(s) using `route.fulfill`:

```javascript
async (page, fileId, filePath) => {
  // Re-open Receipts dialog → Browse → setFiles(filePath)
  // ... then:
  await page.route('**/filemanagement', async (route) => {
    if (route.request().method() === 'POST') {
      await route.fulfill({
        status: 200,
        contentType: 'application/json',
        body: JSON.stringify([{ fileId }])
      });
    } else { await route.continue(); }
  });
  page.once('filechooser', async (fc) => { await fc.setFiles([]); });
  await page.getByRole('button', { name: 'Upload' }).click();
  await page.waitForTimeout(3500);
  await page.unroute('**/filemanagement');
}
```

#### Optional: Verify upload integrity (MD5)

To confirm the file binary is intact, intercept the receipt's download URL via `window.open`, curl it back with the cookie-only config, and compare MD5 hashes:

```javascript
async (page) => {
  await page.evaluate(() => {
    const orig = window.open;
    window.__capturedOpen = null;
    window.open = function(...args) { window.__capturedOpen = args[0]; return orig.apply(this, args); };
  });
  await page.getByRole('button', { name: 'Open', exact: true }).click();
  await page.waitForTimeout(2000);
  return await page.evaluate(() => window.__capturedOpen);
}
```

```bash
curl -s -o /tmp/verified.pdf -K /tmp/d365_download.conf \
  "https://<tenant>expense.operations.dynamics.com/<captured_url>"
md5sum /path/to/original.pdf /tmp/verified.pdf
```

#### What does NOT work

- **Playwright route fetch + forward** — `postDataBuffer()` drops multipart binary content; file ends up 0-byte
- **Browser-context `fetch()`** — can't access local files
- **`page.setInputFiles()`** without the Browse → chooser flow — crashes the D365 page
- **JS `DataTransfer` API for file injection** — D365 ignores synthetic events (`isTrusted: false`)
- **Bearer token (`az account get-access-token`) on `/filemanagement`** — endpoint only accepts cookie-based auth

### Attaching Receipts to Expenses

From the **Receipts tab**, select a receipt checkbox and click **Attach to expense**, then click the add icon on the matching expense row in the dialog.

> **CRITICAL: Checkbox selection requires disabling overlay.** D365's receipt card layout has a `File name` textbox that overlaps the checkbox. Playwright's click lands on the textbox or refuses entirely. Disable pointer-events on the overlapping elements, click, then restore:

```javascript
async (page, receiptName) => {
  // Disable overlapping elements
  await page.evaluate(() => {
    document.querySelectorAll('input[aria-label="File name"], .dyn-container._1fifp8f').forEach(el => {
      el.style.pointerEvents = 'none';
    });
  });
  // Click checkbox — this now triggers D365's Knockout bindings properly
  await page.getByRole('checkbox', { name: new RegExp('Select or unselect ' + receiptName) }).click({ timeout: 5000 });
  // Restore
  await page.evaluate(() => {
    document.querySelectorAll('input[aria-label="File name"], .dyn-container._1fifp8f').forEach(el => {
      el.style.pointerEvents = '';
    });
  });
}
```

Then click **Attach to expense** → in the dialog, click the `addButtonImageProvider` icon on the matching expense row:

```javascript
await page.getByRole('button', { name: ' Attach to expense' }).click();
await page.waitForTimeout(2000);
await page.getByRole('row', { name: /addButtonImageProvider 3\/12\// }).getByLabel('addButtonImageProvider').click();
await page.waitForTimeout(2500);
await page.keyboard.press('Escape');  // close the dialog
```

> **The dialog grid is virtualized** — only ~6 rows visible at a time. For expenses further down, press `PageDown` to scroll before clicking the add icon.

> **WARNING: Don't "attach to first available" in a loop.** Always match receipt to the correct expense by amount/date. If the matching expense isn't in the dialog grid, scroll first.

> **Multi-checkbox state**: Selecting a new receipt doesn't automatically deselect a previously checked one. To attach receipts one-by-one, uncheck previously checked rows before each attach. Easy pattern:
> ```javascript
> const checked = await page.locator('input[type="checkbox"][role="checkbox"][aria-checked="true"]').all();
> for (const cb of checked) {
>   const lbl = await cb.getAttribute('aria-label') || '';
>   if (lbl.startsWith('Select or unselect ') && lbl.includes('Document value')) {
>     await cb.click({ force: true }).catch(() => {});
>   }
> }
> ```

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
5. Check the checkbox next to the correct receipt — **disable overlapping elements first** (a textbox element overlays the checkbox and intercepts pointer events):
   ```javascript
   await page.evaluate(() => {
     document.querySelectorAll('input[aria-label="File name"], .dyn-container._1fifp8f')
       .forEach(el => { el.style.pointerEvents = 'none'; });
   });
   await page.getByRole('checkbox', { name: /receipt_name/ }).click({ timeout: 5000 });
   await page.evaluate(() => {
     document.querySelectorAll('input[aria-label="File name"], .dyn-container._1fifp8f')
       .forEach(el => { el.style.pointerEvents = ''; });
   });
   ```
6. Click **Add** — this is the critical step! Without clicking Add, the selection is lost on Close
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

After uploading via curl, quick checks:

1. **HTTP response** — curl must return HTTP **200** with JSON `[{"fileId":"<guid>"}]`. Any other code means failure (see [error map](#step-5-upload-via-curl-batch-any-number-of-files)).
2. **Receipt count** — The workspace counter increments after closing the upload dialog.
3. **Thumbnail image** — In the Receipts tab, the receipt card shows a PDF icon. A broken image suggests corruption.
4. **Persistence after refresh** — Reload the page; receipt should still appear.

Optional: full MD5 hash verification — see [Optional: Verify upload integrity (MD5)](#optional-verify-upload-integrity-md5) above.

### Warning Banner After Upload

After route interception aborts the browser-side upload, D365 may show a yellow warning banner: *"[filename] failed to upload. Please try to upload the document again."* This is expected — it refers to the **aborted Playwright upload**, not the curl upload. The warning clears on page refresh. Verify the receipt is present in the Receipts tab to confirm success.

### Corruption Root Causes (for reference)

If you hit upload corruption, the historical root causes are:

1. **Playwright binary loss** — `postDataBuffer()` drops multipart binary content. Don't use route forwarding; use curl.
2. **Missing client-side callback** — Curl upload returned HTTP 200 but D365 stored 0-byte file size metadata. Workaround: `route.fulfill()` with the curl's fileId so D365's JS finalizes file size. *In practice, fresh BSID makes this unnecessary.*
3. **Stale BSID** — `ms-dyn-bsid` changes on every `page.goto()`. Stale BSID = D365 accepts upload (200 + fileId) but stores 0 bytes. Always re-capture before uploading.
4. **`setInputFiles()` crash** — Don't call `setInputFiles()` directly on the file input; use the Browse button → file chooser flow.
5. **Cookie filtering** (May 2026) — Including cookies from `.crm.dynamics.com` or other dynamics.com subdomains causes HTTP 302 redirect to login. Filter to `operations.dynamics.com` cookies only.

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

## OData / JSON Service API Status

> **TL;DR**: Both APIs exist, but require admin-granted security roles that aren't enabled by default in most tenants. Until that changes, Playwright + curl on `/filemanagement` is the only working programmatic upload path.

D365 F&O exposes:
- **OData entities** at `/data/*` — `Expenses`, `TrvReceipts`, `UploadReceipts`, `ExpenseCategories`, etc. Most return 403 without `TrvExpTransEntity` read/write privileges.
- **JSON service API** at `/api/services` — discoverable with bearer token (e.g. `ExpenseReceiptService.UploadReceipt`, `ExpenseReportsService.CreateExpenseReport`, `ecpAutomateExpenseReportService.CreateExpenseReport`), but operations return 401 without Service Operation entry point privileges.

Bearer token from `az account get-access-token --resource https://<tenant>expense.operations.dynamics.com` works for discovery (`GET /api/services`), but is rejected by `/filemanagement` (which only accepts cookie auth).

If your tenant has these privileges granted, the entire expense workflow could be done via curl + bearer token without Playwright. Worth checking with your D365 admin if you're going to do this regularly.

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
| File uploads lose binary content via route interception | Use three-phase upload: intercept form fields → curl binary upload → route.fulfill() with fileId. See [Programmatic Receipt Upload](#programmatic-receipt-upload-three-phase-intercept--curl--fulfill) section. |
| Curl upload returns 200 but file is 0-byte | **BSID is stale**. The `ms-dyn-bsid` changes on every `page.goto()`. Re-capture tokens from the current session before uploading. |
| `setInputFiles()` crashes D365 page | **Never use `setInputFiles()`** on D365 upload forms. Always use Browse → file chooser → `browser_file_upload`. |
| Cannot download uploaded receipt for verification | **Fixed**: Intercept `window.open()` URL from the Open button, then `curl` with a cookie-only config (no `url` line). Enables full MD5 hash comparison. See [Download Verification](#download-verification-md5-hash-comparison). |
| OData API returns 403 | D365 OData entities exist but require security roles not granted by default. See [OData API Status](#odata-api-status). |
| Bearer token rejected by `/filemanagement` | This endpoint only accepts cookie-based auth. Must use Playwright to obtain cookies. |
| "Failed to upload" warning banner after three-phase upload | This is from the **aborted** Playwright upload in Step 2, not the curl upload. Ignore it — the fulfill in Step 4 tells D365's JS the upload succeeded. Verify receipt in the Receipts tab. Clears on page refresh. |
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
