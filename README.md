# skill-msft-dynamics365-expense

A [Copilot CLI skill](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) for creating and submitting expense reports in Microsoft Dynamics 365 Finance & Operations via Playwright browser automation.

## What It Does

This skill teaches Copilot CLI how to navigate the D365 Expense Management module and perform end-to-end expense workflows:

- **Create expense reports** — with proper titles, descriptions, cost centers, and approver routing
- **Reconcile corporate card charges** — detect and attach existing CC_Amex transactions instead of creating duplicates
- **Upload receipts** — two-phase upload via route interception + curl (works around D365's `ERR_ACCESS_DENIED` on direct browser uploads)
- **Retrieve receipts from email** — search Outlook via MCP mail tools, download attachments, and upload to D365
- **Manage guests / attendees** — required for Meals & Employee Morale categories; supports Previous Guests and Coworkers directory lookup
- **Handle policy compliance** — country/region, receipt attachment, and category-specific requirements
- **Batch expense entry** — Save and New (Alt+Enter) pattern for rapid multi-expense creation
- **Submit for approval** — workflow submission with comments, routed to interim and final approvers

## Prerequisites

- [GitHub Copilot CLI](https://docs.github.com/en/copilot/concepts/agents/about-copilot-cli)
- Access to a Microsoft Dynamics 365 Finance & Operations instance (e.g., `https://myexpense.operations.dynamics.com`)
- Playwright MCP server configured in Copilot CLI (for browser automation)
- Microsoft 365 MCP servers (Mail, Calendar) for receipt retrieval from email — see [skill-m365-mcp-tools](https://github.com/danielp370-msft/skill-m365-mcp-tools)

## Install

```bash
git clone https://github.com/danielp370-msft/skill-msft-dynamics365-expense.git ~/.copilot/skills/dynamics365-expense
```

Then in Copilot CLI, reload skills:

```
/skills reload
```

## Usage

Ask Copilot CLI to help with expenses:

```
Create an expense report for my team lunch at RAFI Sydney last Tuesday, 8 guests. Receipt is in my email.
```

```
Submit my October travel expenses — flights and hotel from the Sydney trip
```

```
Reconcile my corporate card charges for this month
```

The skill will:
1. Navigate to the D365 Expense Management workspace
2. Review open expenses and corporate card charges
3. Create or open an expense report
4. Add expenses, upload receipts, list guests
5. Verify policy compliance (green checkmarks)
6. Submit for approval when ready

## Example Prompts

### Team morale lunch with receipt from email

> Prepare my D365 expense submission for team morale lunch at RAFI Sydney I had during the past week. 8 guests: me, Charles, Anjani, Ashish, Amos, Susmita, Richard, Jost. Receipt is in my email (jotnot RAFI). Upload and attach it.

What happens:
1. Searches Outlook for the JotNot-scanned receipt PDF via **MailTools-SearchMessages**
2. Downloads the attachment via **MailTools-GetAttachments** → **MailTools-DownloadAttachment**
3. Opens D365, finds the existing CC_Amex charge for RAFI SYDNEY ($890.89)
4. Creates an expense report, adds the charge, verifies all 8 guests are listed
5. Uploads the receipt PDF (two-phase: Playwright interception + curl)
6. Attaches receipt to the expense, confirms policy compliance ✅
7. Submits for approval

### Travel expenses with flights and hotel

> Submit my travel expenses for the Redmond trip last week. Flights on corporate card, hotel receipt is a PDF in my Downloads folder.

### Reconcile monthly corporate card charges

> Review my open corporate card charges and create expense reports for everything from February.

### Quick out-of-pocket expense

> I paid $45 cash for an Uber to the airport on March 3rd. Add it to my open travel report.

## Suggested Tools & MCP Servers

This skill works best when paired with the following MCP servers and tools. These aren't strictly required, but they unlock the full end-to-end workflow (finding receipts in email, checking calendar for event dates, uploading to SharePoint, etc.).

### MCP Servers

| Server | Used For | Example |
|--------|----------|---------|
| **Playwright** | Browser automation for D365 UI | Navigate pages, fill forms, click buttons, upload files |
| **MailTools** (Outlook Mail) | Find and download receipt attachments | `SearchMessages` → `GetAttachments` → `DownloadAttachment` |
| **CalendarTools** (Outlook Calendar) | Look up event dates and attendees | Check meeting invite to find who attended a team lunch |
| **TeamsServer** (Teams) | Discover attendees from group chats | `ListChatMembers` to get the roster for a team event |
| **SharePointOneDrive** | Find receipts stored in OneDrive/SharePoint | `findFileOrFolder` for receipts already uploaded |
| **Word** | Read expense policies or create summaries | `GetDocumentContent` for policy docs shared by finance team |
| **M365CopilotSearch** | General workplace search | Find expense-related emails, docs, or approvals across M365 |

### CLI Tools

| Tool | Used For |
|------|----------|
| **bash / curl** | Phase 2 receipt upload (POST multipart file to D365 `/filemanagement` endpoint) |
| **python3** | Base64 decode email attachments, write curl config files (avoids shell cookie truncation) |
| **convert** (ImageMagick) | Convert scanned receipt PDFs to PNG for visual reading |

### Setup

For M365 MCP server setup, see [skill-m365-mcp-tools](https://github.com/danielp370-msft/skill-m365-mcp-tools). A typical `~/.copilot/mcp-config.json` includes:

```json
{
  "servers": {
    "Playwright": { "...": "Playwright MCP server config" },
    "MailTools": { "...": "Outlook Mail MCP" },
    "CalendarTools": { "...": "Outlook Calendar MCP" },
    "TeamsServer": { "...": "Teams MCP" },
    "SharePointOneDrive": { "...": "SharePoint/OneDrive MCP" }
  }
}
```

## Key Features

### Receipt Upload (Two-Phase Approach)

D365 blocks direct browser file uploads with `ERR_ACCESS_DENIED`. This skill uses a proven workaround:

1. **Phase 1** — Playwright route interception captures dynamic form fields (CSRF token, session ID, access token, record IDs) when the Upload button is clicked, then **aborts** the browser request
2. **Phase 2** — `curl` POSTs the actual file with captured fields from bash, bypassing the browser restriction

> **Critical**: The 4000+ character D365 auth cookie must be written to a curl config file via Python — shell variables silently truncate it, causing HTTP 302 redirects.

### Upload Verification

After upload, verify the receipt is valid (not corrupt):

- ✅ curl returned **HTTP 200** with `[{"fileId":"<guid>"}]`
- ✅ Receipt appears in the **Receipts tab** with PDF icon
- ✅ Receipt **persists after page refresh**
- ✅ **MD5 hash matches** original file (intercept `window.open()` URL → curl download → compare)
- ⚠️ "Failed to upload" warning banner is expected (from aborted Playwright request) — clears on refresh

### OData API (Not Currently Viable)

D365 exposes OData entities (`Expenses`, `TrvReceipts`, `UploadReceipts`, etc.) but they return **403 Forbidden** without specific security roles. The `/filemanagement` upload endpoint only accepts **cookie-based auth** (not bearer tokens). A D365 admin would need to grant OData data entity privileges for API-only access to work.

### Report Management

The skill also handles:
- **Recalling submitted reports** — pull "In review" reports back to "Draft" for editing
- **Managing interim approvers** — add/remove via Actions → Edit expense report → Select interim approvers
- **Batch operations** — recall and resubmit multiple reports (e.g., removing an approver who is on leave)

### Corporate Card Reconciliation

The skill checks for existing CC_Amex charges before creating cash expenses — avoiding duplicates that would be rejected by approvers.

### Guest Auto-Population

For recurring team events, D365 remembers previous guest lists. The skill leverages "Previous guests" to auto-populate attendees from prior reports.

## Known Limitations

| Issue | Workaround |
|-------|-----------|
| D365 file upload blocked in browser | Two-phase upload: route interception + curl |
| Cannot download receipts to verify | **Fixed**: Intercept `window.open()` URL → curl with cookie-only config → MD5 compare |
| OData API returns 403 | Security roles required — not granted by default |
| Bearer token rejected by `/filemanagement` | Cookie auth only — must use Playwright for auth |
| Access tokens expire quickly | Re-intercept and upload immediately |
| Cookie too long for shell variables | Write to curl config file via Python |
| Virtualized category dropdowns | Scroll with `mouse.wheel()` via `browser_run_code` |
| Multiple Close buttons on page | Scope to dialog: `page.getByLabel('Dialog Name').getByRole(...)` |
| Teams MCP may have scope errors | Fall back to Playwright browser automation on `teams.cloud.microsoft` |
| MCP auth tokens expire | Re-authenticate with `/mcp` command; check with health scripts |

## License

MIT
