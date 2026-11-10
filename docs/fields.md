# Field dictionary

All API names below were used in the verified prototype (Zoho CRM, 2026-11-10).
Create the fields with exactly these API names before importing the functions.

## Leads — fields to add (Standard layout)

| Label | API name | Type |
|---|---|---|
| AI_Demo_Flag | `AI_Demo_Flag` | Checkbox |
| AI_Summary | `AI_Summary` | Multi-line (large) |
| AI_Summary_At | `AI_Summary_At` | DateTime |
| AI_Category | `AI_Category` | Picklist: CRM導入 / AI連携 / 業務アプリ開発 / 保守・改善 / その他 |
| AI_Category_At | `AI_Category_At` | DateTime |

The category labels are Japanese in this sample because the prototype targeted
Japanese inquiries. Rename the picklist values and the matching list inside
`aiGenClassify` together if you localize them — the classifier validates the
model's answer against that closed list.

## AI Drafts — custom module (API name `AI_Drafts`)

24 fields. Unless noted, API names are the label with spaces replaced by `_`.

- `Name` (single line, primary), `Target_Lead` (lookup to Leads)
- `Draft_Type` (picklist: Summary / Classification / Email)
- `Status` (picklist: Queued / Processing / Generated / Approved / Applying / Applied / Rejected / Failed / Stale / Superseded)
- `Generation` (number), `Attempt_ID` (single line)
- `AI_Output_Summary` (multi-line, large)
- `AI_Category` (picklist, same five values as Leads), `AI_Confidence` (picklist: High / Medium / Low), `AI_Reason` (single line)
- `Email_Subject` (single line), `Email_Body` (multi-line, large)
- `Provider`, `Model`, `Prompt_Version` (single line)
- `Input_Snapshot` (multi-line, small — 2,000 characters max; holds the sent text plus a trailing `[FP]` fingerprint line)
- `Source_Modified_Time` (DateTime), `Input_Char_Count` (number)
- `Generation_Started_At`, `Completed_At`, `Applied_At` (DateTime)
- `Provider_Request_ID` (single line), `Usage_Tokens` (number)
- `Error_Detail` (multi-line, small), `Approved_Snapshot` (multi-line, large)

Reviewed By / Reviewed At are intentionally not defined: in the single-admin
prototype org the approver is the record's modifier. Add them if you run this
with a team.
