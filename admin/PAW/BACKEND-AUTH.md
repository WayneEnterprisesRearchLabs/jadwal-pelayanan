# Admin access key — backend setup (REQUIRED)

The login screen on the admin page is **only a UX wrapper**. The real
protection has to live in your Google Apps Script, because anyone can read
`EXEC_URL` from this public repo and POST write requests directly, bypassing
the browser entirely. Until you complete the steps below, the admin API is
effectively open.

## 1. Make a `Keys` sheet

In the same spreadsheet the script is bound to, add a tab named exactly `Keys`.
Put one access key per row in **column A** (column B is a free-text label for
your own reference):

| A (key)            | B (label)     |
|--------------------|---------------|
| `paw-2026-hod`     | Head of Dept  |
| `s0me-l0ng-random` | Backup key    |

Anyone who knows a value in column A can write. Treat these like passwords:
make them long, and delete a row to instantly revoke that key.

## 2. Paste this guard into the script

Add these two helpers to your Apps Script project (`Code.gs` or wherever
`doGet`/`doPost` live):

```javascript
// Returns true if `key` matches any non-empty value in the Keys sheet, col A.
function isValidKey_(key) {
  if (!key) return false;
  var sh = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Keys');
  if (!sh) return false;
  var last = sh.getLastRow();
  if (last < 1) return false;
  var vals = sh.getRange(1, 1, last, 1).getValues(); // column A
  key = String(key).trim();
  for (var i = 0; i < vals.length; i++) {
    var cell = String(vals[i][0]).trim();
    if (cell && cell === key) return true;
  }
  return false;
}

function jsonOut_(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## 3. Handle `verifyKey` in `doGet`

The login screen calls this to check a key before letting the user in. Near the
top of your existing `doGet(e)`:

```javascript
function doGet(e) {
  var action = e.parameter.action;

  if (action === 'verifyKey') {
    return jsonOut_({ ok: isValidKey_(e.parameter.key) });
  }

  // ... your existing read actions (getAdminData, getMonthStatus, getNotes) ...
}
```

## 4. Gate every write in `doPost`

This is the line that actually stops an attacker. At the very top of `doPost(e)`,
before any sheet is modified:

```javascript
function doPost(e) {
  var params = JSON.parse(e.postData.contents);

  // Reject any write that doesn't carry a valid key.
  if (!isValidKey_(params.key)) {
    return jsonOut_({ error: 'UNAUTHORIZED' });
  }

  var action = params.action;
  // ... your existing write actions (setMonthStatus, saveServiceRoster,
  //     deleteService, saveNote, deleteNote) ...
}
```

The frontend recognises `{ error: 'UNAUTHORIZED' }` and automatically re-shows
the lock screen, so a revoked key logs the user out cleanly.

## 5. Redeploy

Apps Script serves the **deployed** version, not the saved one. After editing:
**Deploy → Manage deployments → edit the active deployment → Version: New
version → Deploy.** The `EXEC_URL` stays the same. Test by entering a key on the
admin page, then by removing your key row and confirming writes start failing.

## Notes / limits

- `verifyKey` travels as a URL query param (it's a GET), so it can appear in
  server logs. It's not a password to a bank — for a church roster this is a
  reasonable trade-off — but don't reuse these keys anywhere sensitive.
- This does not encrypt anything; it gates writes. Reads stay public, which is
  fine because the volunteer page already exposes the same schedule data.
- For stronger protection later, consider Google account–based auth
  (`Session.getActiveUser()`), which requires no shared secret.
