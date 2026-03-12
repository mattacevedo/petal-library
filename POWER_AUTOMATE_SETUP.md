# Power Automate Setup Guide
## PETAL Library Kiosk

You will create **5 flows** in Power Automate. Each uses a "When an HTTP request is received"
trigger. The kiosk app calls these flows directly — no login required on the tablet.

---

## Your SharePoint Details

| Setting | Value |
|---|---|
| **Site Address** | `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu` |
| **List Name** | `PETAL Library Inventory` |

---

## Your Field Names (copy-paste ready)

> ⚠️ Note: your barcode column's internal name is **`Title`** — this is normal for
> SharePoint, which always names the first column Title internally.
> The book's actual title lives in **`field_2`**.

| What it holds | Internal field name | Type |
|---|---|---|
| Barcode Number | `Title` | Single line of text |
| ISBN | `field_1` | Number |
| Book Title | `field_2` | Single line of text |
| Subtitle | `field_3` | Single line of text |
| Authors | `field_4` | Single line of text |
| Year Published | `PublicationYear` | Date and time |
| Publisher | `field_7` | Single line of text |
| Page Count | `field_8` | Number |
| Synopsis | `field_9` | Multiple lines of text |
| Thumbnail URL | `ImageURL` | Hyperlink or picture |
| Status | `Status` | Choice |
| Borrower | `Borrower` | Person or group |
| Checked Out Date | `CheckedOut` | Date and time |
| Due Back Date | `DueBack` | Date and time |

---

## General Instructions (applies to all 5 flows)

1. Go to **make.powerautomate.com** → **Create** → **Instant cloud flow**
2. Name the flow (suggested names given below)
3. Choose trigger: **"When an HTTP request is received"**
4. Click **Create**
5. In the trigger, set **Method** to `POST`
6. Paste the **Request Body JSON Schema** shown for each flow
7. Build the remaining steps as described
8. **Save** the flow — the trigger URL appears after first save
9. Copy that URL into `CONFIG.flows` in `index.html`

### ⚠️ Two ways to enter expressions — important to get right

Power Automate has two different places you can type an expression, and they work differently:

| Where to type | When to use it | How it looks |
|---|---|---|
| **The main Inputs field** (click the white text area directly) | When the value is JSON with `@{...}` tokens mixed in | You paste the whole block; PA converts the tokens to colored chips |
| **The fx expression editor** (click the `fx` button) | When the value is a single bare expression like `length(...)` or `utcNow()` | You type only the expression itself, no quotes around it, no `@{}` wrapping |

**The most common mistake:** opening the `fx` editor and pasting a JSON block into it.
The `fx` editor only accepts a single expression — it will reject anything that looks like JSON.
Any time these instructions say *"Inputs (expression)"* and the value starts with `{`, paste it into the **white text area directly**, not the `fx` editor.

---

## Flow 1 — Get Book (`GetBook`)

**Purpose:** Takes a barcode, finds the book in your list, returns all book data as JSON.

### Trigger — "When an HTTP request is received"
- Method: `POST`
- Request Body JSON Schema:
```json
{
  "type": "object",
  "properties": {
    "barcode": { "type": "string" }
  },
  "required": ["barcode"]
}
```

### Step 1 — Get items (SharePoint)
- Site Address: `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu`
- List Name: `PETAL Library Inventory`
- Filter Query:
```
Title eq '@{triggerBody()?['barcode']}'
```
- Top Count: `1`

### Step 2 — Condition
- Left side (expression): `length(body('Get_items')?['value'])`
- Operator: `is greater than`
- Right side: `0`

### Step 3a — IF YES: Compose response
- Action: **Data Operation → Compose**
- In the Inputs field: click the **white text area directly** (do NOT click the `fx` button — that editor won't accept JSON).
  Once you're in the text area, paste this entire block:

```
{
  "barcode": "@{first(body('Get_items')?['value'])?['Title']}",
  "isbn": "@{first(body('Get_items')?['value'])?['field_1']}",
  "title": "@{first(body('Get_items')?['value'])?['field_2']}",
  "subtitle": "@{first(body('Get_items')?['value'])?['field_3']}",
  "authors": "@{first(body('Get_items')?['value'])?['field_4']}",
  "yearPublished": "@{formatDateTime(first(body('Get_items')?['value'])?['PublicationYear'], 'yyyy')}",
  "publisher": "@{first(body('Get_items')?['value'])?['field_7']}",
  "pageCount": "@{first(body('Get_items')?['value'])?['field_8']}",
  "synopsis": "@{first(body('Get_items')?['value'])?['field_9']}",
  "thumbnailUrl": "@{first(body('Get_items')?['value'])?['ImageURL']}",
  "status": "@{first(body('Get_items')?['value'])?['Status']?['Value']}",
  "borrowerName": "@{first(body('Get_items')?['value'])?['Borrower']?['DisplayName']}",
  "checkedOutDate": "@{first(body('Get_items')?['value'])?['CheckedOut']}",
  "dueBackDate": "@{first(body('Get_items')?['value'])?['DueBack']}",
  "listItemId": "@{first(body('Get_items')?['value'])?['ID']}"
}
```

> After pasting, Power Automate will convert each `@{...}` into a colored token/chip — that's expected and correct.

> **Note on ImageURL:** Your Thumbnail URL column stores the URL as plain text, so it can be used directly.

> **Note on PublicationYear:** Because Year Published is a date type, `formatDateTime(..., 'yyyy')`
> extracts just the 4-digit year for display.

### Step 3b — IF YES: Respond to request (200)
- Action: **Request → Response**
- Status Code: `200`
- Headers: add key `Content-Type` = `application/json`
- Body: click the dynamic content picker → select **Outputs** from your Compose step

### Step 4 — IF NO: Respond to request (404)
- Action: **Request → Response**
- Status Code: `404`
- Headers: add key `Content-Type` = `application/json`
- Body:
```json
{"error": "Book not found"}
```

---

## Flow 2 — Search Users (`SearchUsers`)

**Purpose:** Takes a search string, queries the O365 directory, returns matching users
with name, job title, department, and email. (User photos are shown as colored initials
in the kiosk — the app handles this gracefully without needing O365 photos.)

### Trigger — "When an HTTP request is received"
- Method: `POST`
- Request Body JSON Schema:
```json
{
  "type": "object",
  "properties": {
    "query": { "type": "string" }
  },
  "required": ["query"]
}
```

### Step 1 — Search for users (Office 365 Users)
- Action: **Office 365 Users → Search for users (V2)**
- Search Term: click dynamic content → select **query** from the trigger

### Step 2 — Initialize variable
- Action: **Variables → Initialize variable**
- Name: `userResults`
- Type: `Array`
- Value: *(leave blank)*

### Step 3 — Apply to each
- Action: **Control → Apply to each**
- Select output: click dynamic content → select **value** from the Search step

  **Inside the loop, add these two sub-steps in order:**

  #### Step 3a — Compose user
  - Action: **Data Operation → Compose**
  - In the Inputs field: click the **white text area directly** (not the `fx` button), then paste:
  ```
  {
    "id": "@{items('Apply_to_each')?['id']}",
    "displayName": "@{items('Apply_to_each')?['displayName']}",
    "jobTitle": "@{items('Apply_to_each')?['jobTitle']}",
    "department": "@{items('Apply_to_each')?['department']}",
    "mail": "@{items('Apply_to_each')?['mail']}",
    "photoBase64": ""
  }
  ```

  #### Step 3b — Append to array (still inside the loop)
  - Action: **Variables → Append to array variable**
  - Name: `userResults`
  - Value: click **fx** and enter:
  ```
  outputs('Compose')
  ```
  > If Power Automate renamed your Compose step (e.g. `Compose_2`), use that name instead.
  > Check the name shown on your Compose action card and match it exactly.

### Step 4 — Respond to request (200)
- Action: **Request → Response**
- Status Code: `200`
- Headers: add key `Content-Type` = `application/json`
- Body (expression):
```
@{variables('userResults')}
```

---

## Flow 3 — Check Out (`Checkout`)

**Purpose:** Marks a book as Checked Out, records the borrower, and sets the dates.

### Trigger — "When an HTTP request is received"
- Method: `POST`
- Request Body JSON Schema:
```json
{
  "type": "object",
  "properties": {
    "barcode":         { "type": "string" },
    "userEmail":       { "type": "string" },
    "userDisplayName": { "type": "string" },
    "userId":          { "type": "string" }
  },
  "required": ["barcode", "userEmail"]
}
```

### Step 1 — Get items (SharePoint)
- Site Address: `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu`
- List Name: `PETAL Library Inventory`
- Filter Query:
```
Title eq '@{triggerBody()?['barcode']}'
```
- Top Count: `1`

### Step 2 — Condition: book found?
- `length(body('Get_items')?['value'])` is greater than `0`

### Step 3a — IF YES: Update item (SharePoint)
- Site Address: `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu`
- List Name: `PETAL Library Inventory`
- Id (expression): `first(body('Get_items')?['value'])?['ID']`

  Set these fields in the Update item form:

  **Status** — type directly: `Checked Out`

  **Borrower Claims** — click directly in the field (do NOT use `fx`), then:
  1. Type: `i:0#.f|membership|` (use regular pipe characters `|`, no backslashes)
  2. Then open the dynamic content picker (lightning bolt icon) and select **userEmail** from the trigger

  The finished field should look like: `i:0#.f|membership|` + a blue chip for userEmail

  > This is how SharePoint identifies an Office 365 user in a Person column. Your Person
  > column "Borrower" appears here as **"Borrower Claims"** — that's normal.

  **CheckedOut** — click `fx` and enter: `utcNow()`

  **DueBack** — click `fx` and enter: `addDays(utcNow(), 14)`

  > **14 days** matches `CONFIG.checkoutDays` in `index.html`. Change both if you want
  > a different loan period.

### Step 3b — IF YES: Respond (200)
- Action: **Request → Response**
- Status Code: `200`
- Headers: Key `Content-Type` = `application/json`
- Body: `{"success": true}`

### Step 4 — IF NO: Respond (404)
- Action: **Request → Response**
- Status Code: `404`
- Headers: Key `Content-Type` = `application/json`
- Body: `{"error": "Book not found"}`

---

## Flow 4 — Check In (`Checkin`)

**Purpose:** Clears the borrower and dates, marks the book Available.

### Trigger — "When an HTTP request is received"
- Method: `POST`
- Request Body JSON Schema:
```json
{
  "type": "object",
  "properties": {
    "barcode": { "type": "string" }
  },
  "required": ["barcode"]
}
```

### Step 1 — Get items (SharePoint)
- Site Address: `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu`
- List Name: `PETAL Library Inventory`
- Filter Query:
```
Title eq '@{triggerBody()?['barcode']}'
```
- Top Count: `1`

### Step 2 — Condition: book found?
- `length(body('Get_items')?['value'])` is greater than `0`

### Step 3a — IF YES: Send an HTTP request to SharePoint
- Action: **SharePoint → Send an HTTP request to SharePoint**

  > ⚠️ We use this instead of "Update item" because the regular Update item action
  > cannot reliably clear a Person field — leaving it blank just skips the field entirely.
  > This action gives us direct control to set fields to null.

- **Site Address**: `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu`
- **Method**: `POST`
- **Uri** — click in the text field directly and type:
```
_api/web/lists/getbytitle('PETAL Library Inventory')/items(@{first(body('Get_items')?['value'])?['ID']})
```
- **Headers** — add each row as a separate key/value pair (click **Add new item** or `+` for each):

  | Key | Value |
  |---|---|
  | `Accept` | `application/json;odata=nometadata` |
  | `Content-Type` | `application/json;odata=nometadata` |
  | `If-Match` | `*` |
  | `X-HTTP-Method` | `MERGE` |
- **Body** — click in the text field directly and type:
```
{
  "BorrowerId": null,
  "CheckedOut": null,
  "DueBack": null,
  "Status": "Available"
}
```

### Step 3b — IF YES: Respond (200)
- Action: **Request → Response**
- Status Code: `200`
- Headers: Key `Content-Type` = `application/json`
- Body: `{"success": true}`

### Step 4 — IF NO: Respond (404)
- Action: **Request → Response**
- Status Code: `404`
- Headers: Key `Content-Type` = `application/json`
- Body: `{"error": "Book not found"}`

---

## Flow 5 — Renew (`Renew`)

**Purpose:** Extends the due date by 7 days from today (or from the current due date,
whichever is later). Returns the new due date to display to the patron.

### Trigger — "When an HTTP request is received"
- Method: `POST`
- Request Body JSON Schema:
```json
{
  "type": "object",
  "properties": {
    "barcode": { "type": "string" }
  },
  "required": ["barcode"]
}
```

### Step 1 — Get items (SharePoint)
- Site Address: `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu`
- List Name: `PETAL Library Inventory`
- Filter Query:
```
Title eq '@{triggerBody()?['barcode']}'
```
- Top Count: `1`

### Step 2 — Condition: book found?
- `length(body('Get_items')?['value'])` is greater than `0`

### Step 3a — IF YES: Compose new due date
- Action: **Data Operation → Compose**
- Inputs (expression — adds 7 days from whichever is later: current due date or today):
```
addDays(
  if(
    greater(
      ticks(first(body('Get_items')?['value'])?['DueBack']),
      ticks(utcNow())
    ),
    first(body('Get_items')?['value'])?['DueBack'],
    utcNow()
  ),
  7
)
```
> This adds 7 days from the existing due date if it hasn't passed yet,
> or 7 days from today if the book is overdue.
> **7** matches `CONFIG.renewalDays` in `index.html`.

### Step 3b — IF YES: Update item (SharePoint)
- Site Address: `https://miamiedu-my.sharepoint.com/personal/mma259_miami_edu`
- List Name: `PETAL Library Inventory`
- Id (expression): `first(body('Get_items')?['value'])?['ID']`

  | Field in Power Automate | Value to enter |
  |---|---|
  | `DueBack` | dynamic content → **Outputs** from the Compose step above |

### Step 3c — IF YES: Respond (200)
- Action: **Request → Response**
- Status Code: `200`
- Headers: Key `Content-Type` = `application/json`
- Body — click in the text field directly and type:
```
{"success": true, "newDueDate": "@{outputs('Compose')}"}
```
> If Power Automate renamed your Compose step, replace `Compose` with its actual name.

### Step 4 — IF NO: Respond (404)
- Action: **Request → Response**
- Status Code: `404`
- Headers: Key `Content-Type` = `application/json`
- Body: `{"error": "Book not found"}`

---

## Connecting Flows to the Kiosk App

After saving each flow, the HTTP POST URL appears at the top of the trigger step. Copy it.

Open `index.html` and find the `CONFIG` block near the top of the `<script>` section.
Fill in each URL:

```javascript
flows: {
  getBook:    "https://prod-XX.eastus.logic.azure.com/workflows/...",
  searchUsers:"https://prod-XX.eastus.logic.azure.com/workflows/...",
  checkout:   "https://prod-XX.eastus.logic.azure.com/workflows/...",
  checkin:    "https://prod-XX.eastus.logic.azure.com/workflows/...",
  renew:      "https://prod-XX.eastus.logic.azure.com/workflows/...",
},
```

---

## Hosting on GitHub Pages (Free)

1. Create a free account at **github.com** if you don't have one
2. Create a new repository (e.g. `petal-library-kiosk`)
3. Upload `index.html` to the repository (no other files needed — the logo loads from its URL)
4. Go to **Settings → Pages → Source → Deploy from a branch → main → / (root)**
5. GitHub will publish your app at: `https://yourusername.github.io/petal-library-kiosk/`
6. Open that URL in Chrome on your tablet

> ⚠️ **Keep your flow URLs private.** They contain a security token. Use a **private**
> GitHub repository — GitHub Pages works with private repos on free accounts.

---

## Adjusting Checkout & Renewal Periods

If you ever want a different loan length, update these two places together:

| File | Setting |
|---|---|
| `index.html` → CONFIG | `checkoutDays: 14` |
| `index.html` → CONFIG | `renewalDays: 7` |
| Flow 3 (Checkout) — Update item | `addDays(utcNow(), 14)` |
| Flow 5 (Renew) — Compose | last argument in `addDays(...)`, currently `7` |

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| "Could not find a book" on a valid barcode | Filter query field name wrong — your barcode column is `Title`, not `BarcodeNumber` |
| Book title shows blank | Confirm your title column is `field_2` in Power Automate dynamic content |
| Thumbnail image doesn't load | Use `?['ImageURL']` directly — the column stores a plain URL string, no `?['Url']` needed |
| Year shows as a full date instead of just the year | Make sure you used `formatDateTime(..., 'yyyy')` in the Compose expression |
| Borrower not saving | Check that "Borrower Claims" (not "Borrower") is the field you're setting, with the `i:0#.f\|membership\|email` format |
| User search returns nothing | Re-authorize the Office 365 Users connection inside the flow |
| Photos not showing (initials appear instead) | This is expected when a user has no O365 photo — colored initials are the intentional fallback |
| Flow URL stopped working | The flow may have been turned off — re-enable it in Power Automate and the URL stays the same |
