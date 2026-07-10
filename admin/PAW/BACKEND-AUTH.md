# Admin access key — what's actually going on (code.gs v7)

Your `code.gs` already has an access-key gate. It's simpler than a `Keys`
sheet: one shared secret, checked in `apiDispatch_`:

```javascript
var ADMIN_KEY = "<your-actual-key-is-in-code.gs-not-here>";
var ADMIN_ACTIONS = ['getAdminData','saveServiceRoster','deleteService','saveNote','deleteNote'];

function apiDispatch_(action, p) {
  if (ADMIN_ACTIONS.indexOf(action) !== -1) {
    if (!p.key || p.key !== ADMIN_KEY) {
      return { error: "Unauthorized" };
    }
  }
  ...
}
```

**That string is your current admin password.** To change it, edit that one
line and redeploy (Deploy → Manage deployments → active deployment → Version:
New version → Deploy — the `EXEC_URL` doesn't change).

## The one thing missing: `verifyKey`

The login screen on `admin/PAW/index.html` calls an action named `verifyKey`
to check what the user typed before letting them in. Your `apiDispatch_`
switch doesn't have a case for it yet — without it, the login screen will
always fail with "Unknown action: verifyKey".

Add this one case to the `switch (action)` block in `apiDispatch_`, anywhere
among the other `case` lines (order doesn't matter):

```javascript
    case 'verifyKey':           return { ok: (p.key === ADMIN_KEY) };
```

It intentionally sits outside the `ADMIN_ACTIONS` gate (it's not in that
array) — otherwise you'd have a chicken-and-egg problem: you'd need a valid
key to check whether your key is valid.

## Why `getAdminData` was probably already failing for you

`getAdminData` is in `ADMIN_ACTIONS`, which means it has required the key
since before this login screen existed. The admin page's boot sequence calls
`getAdminData` immediately on load — so until the frontend was sending a key
(which it now does, after this session's changes), that first load was very
likely already returning `{error:"Unauthorized"}` on every visit, regardless
of any login screen. This wasn't something introduced by this session's
work — it's how the deployed backend already behaved.

## Redeploy

Apps Script serves the deployed version, not the saved one. After adding the
`verifyKey` case: **Deploy → Manage deployments → edit the active deployment
→ Version: New version → Deploy.**

## Test

1. Open the admin page — you should see the lock screen (not a broken/blank
   load).
2. Enter the wrong key — should show "Kunci salah."
3. Enter the value of `ADMIN_KEY` from your code.gs — should unlock and load
   real data.
4. Refresh the tab — should skip the lock screen (key persists in
   `sessionStorage` for the tab's lifetime) and re-verify silently.

## Note on this key's exposure

You pasted the full `code.gs` — including `ADMIN_KEY` in plaintext — into
this chat, so that value now exists in this conversation's history. It was
**not** committed to this repo (an earlier draft of this file had it inline;
that draft was caught before commit and rewritten to this placeholder
version instead). Apps Script source itself is never sent to browsers, so
the key was never exposed via GitHub. Still, given it's sat in a chat log,
rotating it (edit the `ADMIN_KEY` line in code.gs, redeploy) is a reasonable
precaution if you want one.
