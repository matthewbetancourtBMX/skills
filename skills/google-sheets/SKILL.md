---
name: google-sheets
description: "Use this skill any time the task involves reading from or writing to a live Google Sheets spreadsheet. Trigger when the user wants to: create a new spreadsheet; add, overwrite, or delete rows; write values, formulas, or formatting into cells; rename or freeze tabs; add notes or data validation dropdowns; or read the contents of an existing sheet. Also trigger when the user references a Google Sheet by URL, spreadsheet ID, or tab name and wants something done to it. Use this skill even when the request sounds simple — like 'add a row' or 'bold the header' — because the correct action and payload structure are non-obvious. Do NOT trigger for .xlsx file operations, CSV exports, or tasks where the deliverable is a downloaded file rather than a live Google Sheet."
---

# Google Sheets Skill

## Architecture

Agents do not call the Google Sheets API directly. All operations go through `@credal/actions`:

```
runAction(name, provider, auth, params)
  → invokeAction()
  → ActionMapper["googleOauth"][name].fn()
  → Google API
```

Provider: `googleOauth`

---

## Action Selection

| Task | Action | Notes |
|------|--------|-------|
| Create a new spreadsheet | `createSpreadsheet` | Does not write initial cell values |
| Append rows below existing data | `appendRowsToSpreadsheet` | Adds at bottom only |
| Overwrite rows starting at a row number | `updateRowsInSpreadsheet` | 1-based row index; starts at column A |
| Delete a row | `deleteRowFromSpreadsheet` | Requires numeric `sheetId`; row index is **0-based** |
| Read sheet contents | `getDriveFileContentById` | Exports as .xlsx → parsed as CSV-like text; lossy |
| Write values, formulas, or formatting | `updateSpreadsheet` (batchUpdate) | Requires `sheetId`; see full rules below |
| Add / delete / rename tabs, freeze rows | `updateSpreadsheet` (batchUpdate) | Uses `addSheet`, `deleteSheet`, `updateSheetProperties` |

---

## Reading Sheets

`getDriveFileContentById` is the only read path. It exports the sheet as `.xlsx` and returns each tab as text.

**What it does NOT reliably expose:**
- Numeric `sheetId` / `gid`
- Exact used range
- Cell formatting
- Formulas (returns rendered values, not formula strings)
- Structured row/column indexes

---

## sheetId Resolution

`sheetId` is a **numeric** value (not the tab name string). It is required for:
- `deleteRowFromSpreadsheet`
- `updateSpreadsheet` batchUpdate requests targeting a specific tab

**Resolution rules — apply in order:**

| Scenario | sheetId |
|---|---|
| Sheet was created this session | Use value retained in working memory |
| Targeting the first tab of any sheet | Always `0` |
| Pre-existing sheet, non-first tab | Extract `gid` from the sheet URL (see below) |
| gid not available | **Stop and ask the user to provide it** |

**Extracting gid from URL:**
```
https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit#gid=123456789
                                                                ↑
                                                        this value = sheetId
```

**If the user has not provided a URL with a `#gid=` fragment and the target tab is not the first tab — ask before proceeding:**
> "To target that tab I need its sheet ID. Can you share the full sheet URL including the `#gid=` at the end, or open the tab and copy the URL from your browser?"

---

## updateSpreadsheet — Rules and Patterns

### Directly Exposed Request Types
Only these batchUpdate request types are in the action schema:
- `addSheet`
- `deleteSheet`
- `updateCells`
- `updateSheetProperties`
- `updateSpreadsheetProperties`

**Not available** (schema validation blocks these before reaching Google):
`repeatCell`, `updateDimensionProperties`, `autoResizeDimensions`, `insertDimension`, `deleteDimension`, `updateBorders`, `addBanding`, `setBasicFilter`, `sortRange`, `addConditionalFormatRule`, `setDataValidation`, `mergeCells`, `unmergeCells`

Do not attempt these. They fail at SDK validation, not at Google.

---

### updateCells — Fields Mask is Mandatory

Every `updateCells` request **must** include a `fields` mask. Omitting it causes Google to return:
> `Invalid requests[0].updateCells: At least one field must be listed in 'fields'.`

**Minimal working example:**
```json
{
  "updateCells": {
    "range": {
      "sheetId": 0,
      "startRowIndex": 0,
      "endRowIndex": 1,
      "startColumnIndex": 0,
      "endColumnIndex": 2
    },
    "rows": [
      {
        "values": [
          { "userEnteredValue": { "stringValue": "Name" } },
          { "userEnteredValue": { "stringValue": "Status" } }
        ]
      }
    ],
    "fields": "userEnteredValue"
  }
}
```

**Fields mask rules:**
- List only the fields you are writing: `"fields": "userEnteredValue,userEnteredFormat(backgroundColor,textFormat)"`
- Use `"fields": "*"` only as a last resort — it clears unspecified fields
- Range indices are **0-based** (`startRowIndex: 0` = row 1)

---

### Confirmed Writable Fields via updateCells

The schema uses loose validation for nested `updateCells` payloads. Fields not in the schema can pass SDK validation and reach Google. The following were confirmed working:

| Capability | Field |
|---|---|
| String values | `userEnteredValue.stringValue` |
| Numbers | `userEnteredValue.numberValue` |
| Booleans | `userEnteredValue.boolValue` |
| Formulas | `userEnteredValue.formulaValue` |
| Number / date / currency / percent format | `userEnteredFormat.numberFormat` |
| Background color | `userEnteredFormat.backgroundColor` |
| Background color style | `userEnteredFormat.backgroundColorStyle` |
| Borders | `userEnteredFormat.borders` |
| Padding | `userEnteredFormat.padding` |
| Horizontal alignment | `userEnteredFormat.horizontalAlignment` |
| Vertical alignment | `userEnteredFormat.verticalAlignment` |
| Text wrapping | `userEnteredFormat.wrapStrategy` |
| Text direction | `userEnteredFormat.textDirection` |
| Text formatting (bold, italic, font, size) | `userEnteredFormat.textFormat` |
| Hyperlink display style | `userEnteredFormat.hyperlinkDisplayType` |
| Text rotation | `userEnteredFormat.textRotation` |
| Cell notes | `note` |
| Rich text runs | `textFormatRuns` |
| Data validation / dropdowns | `dataValidation` |
| Smart chips | `chipRuns.personProperties` |

**Do not use** `effectiveValue`, `formattedValue`, `effectiveFormat`, or `hyperlink` as write targets — they are read-only fields that happen to pass API validation.

---

### Formatting Example

```json
{
  "spreadsheetId": "SPREADSHEET_ID",
  "requests": [
    {
      "updateCells": {
        "range": {
          "sheetId": 0,
          "startRowIndex": 0,
          "endRowIndex": 1,
          "startColumnIndex": 0,
          "endColumnIndex": 2
        },
        "rows": [
          {
            "values": [
              {
                "userEnteredValue": { "stringValue": "Name" },
                "userEnteredFormat": {
                  "backgroundColor": { "red": 0.8, "green": 0.9, "blue": 1 },
                  "textFormat": {
                    "bold": true,
                    "foregroundColor": { "red": 0.1, "green": 0.1, "blue": 0.1 }
                  }
                }
              },
              {
                "userEnteredValue": { "stringValue": "Status" },
                "userEnteredFormat": {
                  "backgroundColor": { "red": 0.8, "green": 0.9, "blue": 1 },
                  "textFormat": {
                    "bold": true,
                    "foregroundColor": { "red": 0.1, "green": 0.1, "blue": 0.1 }
                  }
                }
              }
            ]
          }
        ],
        "fields": "userEnteredValue,userEnteredFormat(backgroundColor,textFormat)"
      }
    }
  ]
}
```

---

### Formulas

Write formulas via `userEnteredValue.formulaValue`:
```json
{ "userEnteredValue": { "formulaValue": "=IFERROR(T2/U2, \"\")" } }
```

Row-append actions that use `USER_ENTERED` mode also interpret strings beginning with `=` as formulas.

**Formula caveats:**

| Formula Type | API Can Write? | Risk |
|---|---|---|
| `=SUM`, `=AVERAGE`, `=VLOOKUP`, `=XLOOKUP`, `=QUERY`, `=ARRAYFORMULA`, `=IFERROR` | Yes | None — evaluates normally |
| `IMPORTRANGE` | Yes | First cross-sheet connection requires UI "Allow access" click |
| `IMPORTHTML`, `IMPORTXML`, `IMPORTDATA`, `IMPORTFEED` | Yes | External fetch may be slow, blocked, or quota-limited |
| Custom Apps Script functions | Sometimes | Requires script availability and authorization |
| `NOW()`, `RAND()`, `RANDARRAY()`, `RANDBETWEEN()` in references | Sometimes | Google may reject volatile function references |

The API writes the formula string. It does not trigger UI authorization prompts.

---

### updateSheetProperties

Use for tab-level operations. Requires numeric `sheetId` and a `fields` mask.

| Operation | fields mask |
|---|---|
| Rename tab | `"title"` |
| Freeze header row | `"gridProperties.frozenRowCount"` |
| Freeze columns | `"gridProperties.frozenColumnCount"` |
| Resize grid capacity | `"gridProperties.rowCount"` or `"gridProperties.columnCount"` |

**Example — freeze header row:**
```json
{
  "updateSheetProperties": {
    "properties": {
      "sheetId": 0,
      "gridProperties": { "frozenRowCount": 1 }
    },
    "fields": "gridProperties.frozenRowCount"
  }
}
```

---

## What Is Not Available

These capabilities require batchUpdate request types not exposed by the current action schema. Do not attempt them — they will fail at SDK validation:

| Capability | Blocked By |
|---|---|
| Set column width | `updateDimensionProperties` |
| Set row height | `updateDimensionProperties` |
| Auto-resize columns/rows | `autoResizeDimensions` |
| Insert rows/columns as dimensions | `insertDimension` |
| Delete rows/columns as dimensions | `deleteDimension` |
| Basic filters | `setBasicFilter` |
| Sort range | `sortRange` |
| Alternating row colors / banding | `addBanding` |
| Conditional formatting | `addConditionalFormatRule` |
| Merge / unmerge cells | `mergeCells` / `unmergeCells` |

If a user requests one of these, tell them it is not available through the current action set and suggest a workaround (e.g., manual formatting in the UI, or achieving the visual effect through `userEnteredFormat` where possible).

---

## Memory Discipline

After creating or reading a spreadsheet, retain in working memory:
- `spreadsheetId`
- `spreadsheetUrl`
- Sheet tab names → numeric `sheetId` / `gid` map

Do not re-ask the user for these on follow-up operations within the same session.

---

## Pre-Edit Confirmation

Before any write operation, state:
- Target spreadsheet (ID or URL)
- Target tab name
- Range or row(s) affected
- What values/formatting will change

Do not proceed with destructive operations (overwrite, delete) without explicit user confirmation.

---

## Capability Summary

| Task | Status |
|---|---|
| Add tab | ✅ `addSheet` |
| Delete tab | ✅ `deleteSheet` |
| Rename tab | ✅ `updateSheetProperties` |
| Freeze rows / columns | ✅ `updateSheetProperties` |
| Write values | ✅ `updateCells` with `fields` mask |
| Write formulas | ✅ `userEnteredValue.formulaValue` |
| Cell color / text / borders / alignment / wrap | ✅ `userEnteredFormat` inside `updateCells` |
| Notes | ✅ `note` inside `updateCells` |
| Data validation / dropdowns | ✅ `dataValidation` inside `updateCells` |
| Rich text | ✅ `textFormatRuns` inside `updateCells` |
| Append rows | ✅ `appendRowsToSpreadsheet` |
| Overwrite rows | ✅ `updateRowsInSpreadsheet` |
| Delete row | ✅ `deleteRowFromSpreadsheet` (needs numeric `sheetId`) |
| Read contents | ✅ `getDriveFileContentById` (lossy) |
| Get sheet metadata / sheetIds | ❌ `getSpreadsheetMetadata` not available |
| Set column / row width or height | ❌ Not available |
| Auto-resize | ❌ Not available |
| Sort / filter / banding / conditional formatting / merge | ❌ Not available |
