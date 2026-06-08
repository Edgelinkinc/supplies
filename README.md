# Edgelink Supply Order Center + Store Audits

Two-app system built on the same `EmailJS -> Power Automate -> SharePoint` pipeline as `/orders`.

## Files

| File | Purpose |
|------|---------|
| `supply-order-form.html` | Employee-facing supply order form. Untouched by audits. |
| `supply-dashboard.html` | HQ ops center: supply orders + store audits (PIN-gated). |
| `data.json` | Editable config (categories, items, stores, announcements, users). |

## Live URLs

- Form: `https://edgelinkinc.github.io/supplies/supply-order-form.html`
- Dashboard: `https://edgelinkinc.github.io/supplies/supply-dashboard.html`

---

# SUPPLIES -- outstanding setup

## 1. Save flow Items mapping (fixes empty line items in the order modal)
1. Save Edgelink Supply Order flow -> Edit -> Create item step.
2. Items field -> remove the current `itemsText` token.
3. Click the field -> Expression (fx) -> paste `string(triggerBody()?['items'])` -> Add.
4. Save.
5. Submit a fresh order; the dashboard modal will now show line items.

## 2. Cost column + Update flow Cost mapping
SharePoint: Edgelink Supply Orders list -> + Add column -> Single line of text -> name it `Cost`.

Update flow: Update Edgelink Supply Order -> Update item step -> Cost field -> map to the `cost` token -> Save.

---

# AUDITS -- full setup

## 1. Two SharePoint lists (on the Edgelink Orders site)

### `Edgelink Audit Logs` -- one row per submitted audit
| Column | Type |
|--------|------|
| AuditId | Single line of text |
| Auditor | Single line of text |
| Store | Single line of text |
| AuditDate | Date |
| TotalCount | Number |
| FlaggedCount | Number |
| Keyholders | Multiple lines of text |
| Answers | Multiple lines of text (JSON of all Q/A) |

### `Edgelink Audit Issues` -- one row per flagged item (the punch list)
| Column | Type |
|--------|------|
| IssueId | Single line of text |
| AuditId | Single line of text |
| Store | Single line of text |
| Auditor | Single line of text |
| AuditDate | Date |
| Section | Single line of text |
| Question | Multiple lines of text |
| Note | Multiple lines of text |
| Status | Single line of text |
| FixedDate | Single line of text  *(NOT Date -- avoids the empty-string trap)* |
| FixedBy | Single line of text |
| FixNote | Multiple lines of text  *(how the issue was fixed)* |
| FixCost | Single line of text  *(dollar amount as text -- NOT Currency, avoids empty-string trap)* |
| PhotoUrl | Single line of text |

Enable attachments on `Edgelink Audit Issues` (List settings -> Advanced settings -> Allow attachments).

**Auditor name:** Hardcoded to "Ambulai Sheku" by default (set on `auditSettings.auditor` in data.json). Change via Admin -> Settings -> Audit defaults. Coaches don't type the auditor on the audit form anymore -- it pulls from this setting.

## 2. Three Power Automate flows

### Flow A -- "Save Edgelink Audit"
Receives the audit, writes one Audit Log row, then creates one Issue row per flagged item with photo attached.

1. Trigger: HTTP request received. Who can trigger = Anyone. Sample payload:
```
{
  "auditId":"AU-20260526-1234",
  "auditor":"Rose Perez",
  "store":"TN-01",
  "storeName":"Memphis -- Poplar",
  "auditDate":"2026-05-26",
  "totalCount":14,
  "flaggedCount":1,
  "keyholders":"Maria, John",
  "answers":{"hw1":{"answer":"yes","note":""}},
  "flaggedItems":[
    {"issueId":"IS-20260526-1234-0","auditId":"AU-20260526-1234","section":"Hardware","questionId":"hw3","question":"Are all scanners working?","note":"Scanner at register 2 not reading","photoBase64":"...","photoName":"hw3.jpg","status":"Open"}
  ]
}
```

2. SharePoint -> Create item in Edgelink Audit Logs. Map fields from trigger.
   - Answers field -> Expression: `string(triggerBody()?['answers'])`

3. Apply to each over `flaggedItems`. Inside:
   - SharePoint Create item in Edgelink Audit Issues -- map each field from `item()`. Pull Store/Auditor/AuditDate from `triggerBody()`. Leave FixedDate/FixedBy/PhotoUrl blank.
   - Condition: if `length(item()?['photoBase64']) > 0`
     - SharePoint Add attachment to the just-created issue row. Id from Create item 2. File Name = `item()?['photoName']`. File Content = Expression `base64ToBinary(item()?['photoBase64'])`.
     - SharePoint Update item on that issue row, PhotoUrl = path to the attachment (e.g. `/sites/<your-site>/Lists/Edgelink Audit Issues/Attachments/<itemId>/<filename>`).

4. Response 200.

5. Save -> Anyone + classic view -> copy signed URL.

### Flow B -- "Get Edgelink Audit Issues"
1. HTTP trigger (Anyone), no schema needed.
2. SharePoint Get items on Edgelink Audit Issues.
3. Response status 200, Body = the `value` of Get items.
4. Save -> copy signed URL.

### Flow C -- "Update Edgelink Audit Issue"
1. HTTP trigger (Anyone). Sample payload:
   `{ "issueId":"IS-20260526-1234-0", "status":"Fixed", "fixedDate":"2026-05-28", "fixedBy":"Ambulai Sheku", "fixNote":"Replaced scanner cable", "fixCost":"24.99" }`
2. SharePoint Get items filtered: `IssueId eq '[issueId token]'`.
3. Apply to each:
   - SharePoint Update item -> Id from Get items -> map Status, FixedDate, FixedBy, FixNote, FixCost.
4. Save -> copy signed URL.

### (Optional) Flow D -- "Get Edgelink Audit Logs"
Same shape as Flow B but on the Logs list. Powers the History view.

## 3. Paste the URLs into the dashboard CONFIG

```
SAVE_AUDIT_FLOW_URL:         "<Flow A URL>",
GET_AUDIT_ISSUES_FLOW_URL:   "<Flow B URL>",
UPDATE_AUDIT_ISSUE_FLOW_URL: "<Flow C URL>",
GET_AUDIT_LOGS_FLOW_URL:     "<Flow D URL>"
```

## 4. Test
1. Dashboard -> Audits tab -> enter PIN.
2. + New audit -> pick a store -> answer questions -> tap No on one to add note + photo.
3. Submit audit -> alert: "N issue(s) opened."
4. Open issues -> see flagged items with thumbnails, mark Open/In Progress/Fixed.
5. History -> see audit by store with score.

---

# Style standards

ES5 only (var / function declarations, no arrow functions or template literals).
Single-file HTML, light theme, Edgelink magenta `#E20074` + purple `#5C2D91`.
No Metro branding.
