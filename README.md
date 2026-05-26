# Edgelink Supply Order Center

A parallel build to your `edgelinkinc/orders` accessory system — same pipeline
(EmailJS → Power Automate → SharePoint, with `data.json` on GitHub Pages holding
the editable lists), but kept fully separate so supplies never mix with accessories.

## Files

| File | What it is |
|------|------------|
| `supply-order-form.html` | Employee-facing form. 6 steps (Store Info → Office → Cleaning → Store Ops → Marketing → Review). Reads `data.json` on load, submits via EmailJS + Power Automate. |
| `supply-dashboard.html` | HQ dashboard **with Admin Center built in** (PIN-gated). 7 dashboard sections + the order detail modal + 8 admin tabs. |
| `data.json` | The editable lists — categories, items, stores, announcements, users. Admin edits these and publishes; the form reads them. |

The dashboard works **right now** on sample data — open it before wiring anything.
The form will also render and walk through to a confirmation screen before the
flows are connected (it just won't save to SharePoint until you paste the URLs).

## Run it locally (to test before deploying)

Because the files use `fetch("data.json")`, open them through a local server, not
by double-clicking (browsers block `fetch` on `file://`):

```
cd supplies
python -m http.server 8080
```
Then visit `http://localhost:8080/supply-order-form.html` and `…/supply-dashboard.html`.
(The dashboard falls back to sample orders if no server / flow is set up.)

## Deploy (GitHub Pages)

1. Create a new repo `edgelinkinc/supplies` (keeps it separate from `orders`).
2. Upload all three files.
3. Settings → Pages → deploy from `main` / root.
4. URLs:
   - Form: `https://edgelinkinc.github.io/supplies/supply-order-form.html`
   - Dashboard: `https://edgelinkinc.github.io/supplies/supply-dashboard.html`
   - (Rename the form to `index.html` if you want the bare URL to open it.)

## Wiring the pipeline (same 3 flows as /orders)

Set the values in the `CONFIG` block at the top of each file's `<script>`.

### 1. SharePoint list — `Edgelink Supply Orders`
Create a list with these columns (all single-line text unless noted):
`OrderId`, `Employee`, `Store`, `OrderDate` (date), `Status`, `PO`, `Tracking`,
`Notes`, `FulfillmentNotes`, `DateFulfilled` (date), `UpdatedBy`,
`Items` (multiple lines — stores the JSON of line items).

### 2. Flow "Save Edgelink Supply Order"  → `SAVE_FLOW_URL` (form file)
Instant cloud flow → **When an HTTP request is received** → **SharePoint Create item**
into `Edgelink Supply Orders`. Map the request body fields (`orderId`, `employee`,
`store`, `orderDate`, `status`, `notes`) to columns; put `items` JSON into `Items`.
Copy the HTTP POST URL into `SAVE_FLOW_URL`.

### 3. Flow "Get Edgelink Supply Orders"  → `GET_ORDERS_FLOW_URL` (dashboard)
Instant cloud flow → HTTP trigger → **SharePoint Get items** (`Edgelink Supply Orders`)
→ **Response** (status 200, body = the items). The dashboard reads this on load /
Refresh. It tolerates `value`, `orders`, or a bare array.

### 4. Flow "Update Edgelink Supply Order" → `UPDATE_ORDER_FLOW_URL` (dashboard)
HTTP trigger → **SharePoint Get items** filtered by `OrderId eq <body OrderId>` →
**Update item** with the new `Status`, `PO`, `Tracking`, `FulfillmentNotes`,
`DateFulfilled`, `UpdatedBy`. Fires when you hit **Save changes** in the order modal.

### 5. EmailJS (optional but recommended)
Reuse your existing EmailJS account. Make a new template for supplies using these
variables: `{{order_id}} {{employee}} {{store}} {{order_date}} {{items_text}}
{{line_count}} {{unit_count}} {{notes}}`. Set `EMAILJS_PUBLIC_KEY`,
`EMAILJS_SERVICE_ID`, `EMAILJS_TEMPLATE_ID` in the form file.

### 6. Publishing config (Admin → Settings → Save & publish)
Two paths, and the dashboard supports **both**:
- **Power Automate** → set `PUBLISH_CONFIG_FLOW_URL` to a flow that commits `data.json`
  to GitHub. (This is the GitHub-API PUT that gave the accessory system "Bad
  Credentials" trouble — if it misbehaves, use the fallback below.)
- **Manual fallback (always available):** click **Download data.json** or **Copy JSON**,
  then commit it to `edgelinkinc/supplies`. Zero dependency on the flaky flow.

## Admin

- Open the dashboard → **Admin** tab → enter PIN (default `1234`, change
  `CONFIG.ADMIN_PIN` at the top of `supply-dashboard.html`).
- Add/edit/remove items, stores, users, announcements; rename categories (which
  renames the matching form step); toggle items active/inactive; set reorder notes.
- Changes are held until you **Save & publish**.

## Notes
- Built ES5 (var / function declarations, no arrow functions or template literals),
  single-page per file, light theme, Edgelink magenta `#E20074` + purple `#5C2D91`,
  no Metro branding — consistent with your other dashboards.
- The 12 starter stores are placeholders. Paste your real 34-store list into
  `data.json` (or copy the store array from your accessory `data.json` — same stores),
  or add them in Admin → Stores and publish.
- Work week assumed Mon–Sun; the "quiet store" tracker counts calendar days vs today.
