# 444 Construction — Google Sheets Review Backend Setup

This guide walks you through connecting the "Write a Review" form on your website to a Google Sheet via Google Apps Script, so that every submitted review is saved automatically and displayed live in the carousel.

---

## Overview

```
Website Visitor
    │  submits form (name, rating, review)
    ▼
Google Apps Script Web App   ◄──── also serves reviews on page load (GET)
    │
    ▼
Google Sheet (Timestamp | Name | Rating | Review | Initial)
```

---

## Step 1 — Create a Google Sheet

1. Go to [https://sheets.google.com](https://sheets.google.com) and click **Blank spreadsheet**.
2. Rename it to something memorable, e.g. **444 Construction Reviews**.
3. Leave the first sheet on the default name ("Sheet1") — the script uses this automatically.

> **Note:** You do NOT need to create column headers manually. The script will add them automatically on the very first POST submission. If you want them right away, add these headers to row 1:
> `Timestamp` | `Name` | `Rating` | `Review` | `Initial`

---

## Step 2 — Open Apps Script

1. In your Google Sheet, click the **Extensions** menu in the top toolbar.
2. Click **Apps Script**.
3. The Apps Script editor will open in a new browser tab.
4. Delete all existing placeholder code in the editor (usually a default `function myFunction() {}`).

---

## Step 3 — Paste the Script

Copy the entire block below and paste it into the Apps Script editor:

```javascript
// ============================================================
// 444 Construction — Review Backend (Google Apps Script)
// Deploy as: Web App | Execute as: Me | Access: Anyone
// Sheet columns: Timestamp | Name | Rating | Review | Initial
// ============================================================

var SHEET_NAME = 'Sheet1'; // Change if your sheet tab has a different name

/**
 * doPost(e) — Receives a new review submission from the website form.
 * Expects a JSON body with: { name, rating, reviewText, initial }
 * Appends a new row to the sheet and returns { status: "success" }.
 */
function doPost(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);

  // Ensure header row exists
  if (sheet.getLastRow() === 0) {
    sheet.appendRow(['Timestamp', 'Name', 'Rating', 'Review', 'Initial']);
  }

  var data;
  try {
    data = JSON.parse(e.postData.contents);
  } catch (err) {
    return buildResponse({ status: 'error', message: 'Invalid JSON' });
  }

  var timestamp = new Date().toISOString();
  var name      = String(data.name      || '').trim();
  var rating    = String(data.rating    || '5').trim();
  var text      = String(data.reviewText|| '').trim();
  var initial   = String(data.initial   || (name.charAt(0).toUpperCase())).trim();

  if (!name || !text) {
    return buildResponse({ status: 'error', message: 'Name and review text are required.' });
  }

  sheet.appendRow([timestamp, name, rating, text, initial]);

  return buildResponse({ status: 'success' });
}

/**
 * doGet(e) — Returns all reviews from the sheet as a JSON array.
 * Each object has: { name, rating, text, initial, date }
 * Called on page load to populate the review carousel.
 */
function doGet(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
  var lastRow = sheet.getLastRow();

  // Return empty array if there are no reviews yet (or only a header row)
  if (lastRow <= 1) {
    return buildResponse([]);
  }

  // Read all data rows (skip header row 1)
  var rows    = sheet.getRange(2, 1, lastRow - 1, 5).getValues();
  var reviews = rows
    .filter(function (row) { return row[1] && row[3]; }) // must have Name and Review
    .map(function (row) {
      return {
        date   : row[0] ? new Date(row[0]).toLocaleDateString('en-CA') : '',
        name   : row[1],
        rating : row[2] || '5',
        text   : row[3],
        initial: row[4] || String(row[1]).charAt(0).toUpperCase()
      };
    });

  return buildResponse(reviews);
}

/**
 * Helper: wraps any value in a JSON ContentService response with CORS headers.
 */
function buildResponse(data) {
  var output = ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
  return output;
}
```

> **Tip:** Click the floppy disk icon (or press Ctrl+S / Cmd+S) to save before deploying.

---

## Step 4 — Deploy as a Web App

1. In the Apps Script editor, click the blue **Deploy** button (top right).
2. Select **New deployment**.

   ![Deploy > New deployment menu item]
   *(Description: A dropdown appears with options including "New deployment" at the top.)*

3. In the dialog that opens:
   - Click the gear icon next to **Select type** and choose **Web app**.
   - **Description:** (optional) e.g. `v1 — initial deploy`
   - **Execute as:** `Me` (your Google account)
   - **Who has access:** `Anyone`

   ![Deployment settings dialog]
   *(Description: A modal with "Execute as: Me" and "Who has access: Anyone" dropdowns.)*

4. Click **Deploy**.
5. Google will ask you to **Authorise access** — click through the prompts:
   - Click **Authorise access**
   - Choose your Google account
   - Click **Advanced** → **Go to [project name] (unsafe)** (this is normal for personal scripts)
   - Click **Allow**

6. After authorisation, you'll see a success screen with the **Web app URL**.

   ![Web app URL shown after deployment]
   *(Description: A green success screen showing a URL like https://script.google.com/macros/s/AKfycby.../exec)*

7. **Copy that URL** — you'll need it in the next step.

---

## Step 5 — Update index.html

Open `/Users/kanwaljeetkaur/Websites/444-Construction/index.html` in your code editor.

Find this line near the top of the `<script>` block (around line 499):

```javascript
const APPS_SCRIPT_URL = 'YOUR_APPS_SCRIPT_URL';
```

Replace `YOUR_APPS_SCRIPT_URL` with the URL you copied. Example:

```javascript
const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycby1234abc.../exec';
```

Save the file and reload your website. The review form is now live.

---

## Step 6 — Test It

1. Open your website and scroll to the **Write a Review** section.
2. Fill in a name, select a star rating, and write a short test review.
3. Click **Submit Review**.
4. You should see:
   - The success popup: *"Thank you for your review! It has been submitted successfully."*
   - The new review card appearing immediately in the carousel (no page refresh needed).
5. Open your Google Sheet — a new row should have been added with your test data.

---

## Updating the Script (Re-deploy)

If you ever need to change the Apps Script code:

1. Make your edits in the Apps Script editor.
2. Click **Deploy** > **Manage deployments**.
3. Click the pencil (edit) icon next to your deployment.
4. Change the version to **New version**.
5. Click **Deploy**.

> **Important:** You do NOT get a new URL when you redeploy — the same URL keeps working.

---

## Removing the Placeholder Reviews

Once you have real reviews in your sheet, you can clean up the hardcoded cards in `index.html`.

Look for this comment block in the `<!-- ================= REVIEWS ================= -->` section:

```html
<!--
    PLACEHOLDER REVIEWS — These hardcoded cards serve as examples/fallback
    while you gather real reviews. Once you have real reviews in your Google
    Sheet and the Apps Script backend is set up, you can safely remove these
    hardcoded cards (Card 1 through Card 6 AND the duplicates below them).
    Dynamic reviews fetched from the sheet will appear alongside them until
    you do.
-->
```

Delete everything from `<!-- Card 1 -->` through to (and including) the last `</div>` before `</div> <!-- end .reviews-track -->`, including the duplicates section marked with `<!-- Duplicates for seamless infinite loop ... -->`.

---

## Troubleshooting

| Problem | Solution |
|---|---|
| Form submits but nothing appears in the sheet | Make sure "Who has access" is set to **Anyone** (not "Anyone with Google account") |
| CORS error in browser console | Redeploy the Apps Script with the exact settings above |
| Reviews don't load on page load | Check that `APPS_SCRIPT_URL` in index.html is correct and has no extra spaces |
| "Script function not found: doPost" error | Make sure you saved the script before deploying |
| Carousel animation breaks after adding cards | This is handled automatically — the JS duplicates new cards for the loop |
| You see "Going to unsafe site" warning | This is normal for personal Google Apps Script projects — click Advanced → Allow |

---

## Security Notes

- The Apps Script runs under your Google account but is publicly accessible for write (POST) and read (GET).
- Review data (names and text) is HTML-escaped in `index.html` before being inserted into the DOM, protecting against XSS.
- If you want to prevent spam, you can add a simple secret token check in `doPost` and set `APPS_SCRIPT_URL` with a `?token=yourtoken` suffix.
- No sensitive business data is stored — only review submissions (name, rating, text).
