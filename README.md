# Open Source Contribution — CodePath AI301

**Contribution Number:** 1
**Student:** Biraj Lamichhane
**Issue:** [#4742 — Bug: Internal error when syncing GoCardless after restoring backup](https://github.com/actualbudget/actual/issues/4742)
**Status:** Phase III Complete

---

## Why I Chose This Issue

GoCardless is a service that connects a real bank account to Actual Budget. When restoring a backup, the GoCardless Secret ID and Secret Key are not restored. If a user tries to sync their bank account after restoring, the app fails silently with no useful feedback. There is no indication that the credentials are missing, which makes it highly confusing for the user to diagnose and repair the problem.

I am interested in this issue because I have experience in React.js and Next.js. At my internship, I handled error states by displaying proper error messages on screen. This issue is similar because right now the app fails silently with no useful feedback when GoCardless credentials are missing after a restore. I want to fix this by detecting when the credentials are missing and showing a clear, helpful error message in the UI so the user knows exactly what went wrong and how to fix it.

---

## Understanding the Issue

### Problem Description

When a user restores a budget backup, the GoCardless Secret ID and Secret Key are not included in the export. If the user then tries to sync their bank account, the app crashes with a generic "internal error" message instead of telling the user that their credentials are missing.

### Expected Behavior

When a user tries to sync a GoCardless-linked account after restoring a backup and the credentials are missing, the app should:
- Show a clear, specific error message explaining that GoCardless credentials are missing
- Provide a button or direct path for the user to re-enter their credentials
- Show a warning during export that server configuration including GoCardless credentials is not included in the backup

### Current Behavior

The app shows a generic "An internal error occurred. Try to log in again, or get in touch for support." message with only an "Unlink account" button and no way to re-enter credentials. There is no warning during export that credentials will not be included.

### Affected Components

- `packages/sync-server/src/app-gocardless/app-gocardless.ts` — the sync server handler that processes GoCardless transaction requests. This is where the missing credentials cause a generic UNKNOWN error instead of a specific one.
- `packages/desktop-client/src/components/accounts/AccountSyncCheck.tsx` — the frontend component that displays sync error messages to the user. This is where the generic "internal error" message is shown.
- `packages/desktop-client/src/components/settings/Export.tsx` — the export settings UI where a warning about missing server configuration should be added.

---

## Reproduction Process

### Environment Setup

**Problem 1 — Node version mismatch**
The project requires Node v22 but I had Node v24 installed.
Fix: Installed nvm-windows from GitHub, then ran `nvm install 22` and `nvm use 22`.

**Problem 2 — corepack not recognized in PowerShell**
Running `corepack enable` in PowerShell threw "not recognized as a cmdlet" error.
Fix: Closed PowerShell and reopened as Administrator. corepack worked in the new session.

**Problem 3 — Disk space error during yarn install**
Got `ENOSPC: no space left on device` error during `yarn install`.
Fix: Ran Disk Cleanup on C: drive and cleared temp files to free up space, then reran `yarn install`.

**Problem 4 — Backend worker fatal error**
App loaded in browser but showed "Actual couldn't load a critical backend worker."
Fix: Used Claude Code to investigate. The loot-core backend worker needed to be built first. Claude Code started both the frontend and sync server correctly using Git Bash instead of PowerShell.

**Important note for Windows users:** Always use Git Bash, not PowerShell. The project uses Unix shell scripts that don't work in PowerShell.

### Steps to Reproduce

1. Clone the repository and install dependencies:
```bash
git clone https://github.com/actualbudget/actual.git
cd actual
corepack enable
corepack prepare yarn@4.13.0 --activate
yarn install
```

2. Start both servers in Git Bash:
```bash
# Terminal 1 — frontend
yarn start:browser

# Terminal 2 — sync server
yarn workspace @actual-app/sync-server start
```

3. Open `http://localhost:3001` in your browser and connect to the sync server at `http://localhost:5006`

4. Create a budget and open the browser console (F12)

5. Inject a GoCardless-linked account to simulate the post-restore state:
```javascript
await window.$send('gocardless-accounts-link', {
  requisitionId: 'TEST-REQ-12345',
  account: {
    account_id: 'TEST-ACC-12345',
    institution: { name: 'Test Bank (repro #4742)' },
    name: 'Repro GoCardless Account',
    mask: '1234',
    official_name: 'Repro GoCardless Account',
  },
});
```

6. Trigger a sync:
```javascript
await window.$send('accounts-bank-sync', { ids: [] });
```

7. Click "Repro GoCardless Account" in the sidebar and observe the generic error message with no way to re-enter credentials.

### Reproduction Evidence

- **Screenshots:**
  - Bank sync failure toast notification
  <img width="1912" height="911" alt="Screenshot 2026-06-19 210231" src="https://github.com/user-attachments/assets/5b41aa7e-7165-472b-ada9-f1394bee5b91" />

  - Generic internal error popup with no recovery path
  <img width="1908" height="882" alt="Screenshot 2026-06-19 210330" src="https://github.com/user-attachments/assets/99ec8af7-4327-4929-ade4-b50e358f6db3" />

- **My findings:** The secrets table on the sync server has 0 rows after setup. Syncing returns error_code: 'UNKNOWN' instead of a meaningful error. The UI shows a generic internal error with only an "Unlink account" button and no way to re-enter GoCardless credentials.

---

## Solution Approach

### Analysis

The root cause is a chain of three failures:

1. The sync server's `/transactions` handler does not check if GoCardless credentials exist before attempting to use them. The `isConfigured()` method already exists in `gocardless-service.ts` but is never called during sync.

2. When GoCardless rejects the empty credentials with a 400 error, the server's error handler maps it to a generic `UNKNOWN` error code instead of something meaningful.

3. The frontend's `AccountSyncCheck.tsx` has no case for handling a missing-credentials error, so it falls through to the default "internal error" message with no recovery path.

### Proposed Solution

Define one new specific error code on the server (`GOCARDLESS_NOT_CONFIGURED`) when credentials are missing, then thread it through the frontend so the user sees a clear message instead of the generic internal error.

### Implementation Plan

**Understand:**
When GoCardless credentials are missing after a backup restore, the app shows a generic "internal error" with no recovery path. The user has no idea their credentials are missing or how to fix it. The expected behavior is a clear error message that tells the user exactly what went wrong.

**Match:**
The `ITEM_LOGIN_REQUIRED` error code in `AccountSyncCheck.tsx` already follows the exact pattern I needed — it shows a specific message for a specific error code. I followed this same pattern for the missing credentials case. The `isConfigured()` method already existed in `gocardless-service.ts` but was never used in the `/transactions` handler — I used it as the guard.

**Plan:**
1. In `packages/sync-server/src/app-gocardless/app-gocardless.ts` — add an `isConfigured()` check at the start of the `/transactions` handler. If credentials are missing, return `error_code: 'GOCARDLESS_NOT_CONFIGURED'` immediately instead of continuing.
2. In `packages/desktop-client/src/components/accounts/AccountSyncCheck.tsx` — add a case in `getErrorMessage` for `GOCARDLESS_NOT_CONFIGURED` with a clear, user-friendly message.

**Implement:**
https://github.com/birajlamichhane75/actual/tree/fix-issue-4742

**Review:**
- Code follows existing patterns in the codebase
- No new dependencies added
- Commit message follows project convention
- Changes are small and focused on this issue only
- typecheck passed, lint passed, test added

**Evaluate:**
- Manually triggered the reproduction steps after the fix and confirmed the new specific error message appears instead of the generic internal error
- Ran `yarn typecheck` and `yarn lint:fix` — both passed
- Wrote a unit test that confirms the fix works

---

## Testing Strategy

### Unit Tests
- Test that `/transactions` endpoint returns `GOCARDLESS_NOT_CONFIGURED` when `isConfigured()` is false — added in `packages/sync-server/src/app-gocardless/tests/transactions.spec.ts`

### Manual Testing

I reproduced the bug by injecting a GoCardless-linked account into the app using the browser console with no credentials on the server. Before my fix, clicking Bank Sync showed the generic "An internal error occurred" message with only an "Unlink account" button. After my fix, the user sees a clear message explaining that GoCardless credentials are missing. I verified this manually on my local environment before committing.

---

## Implementation Notes

### Week 2 Progress

I traced the bug myself across 6 files before writing any code. The bug lives in two places — the sync server sends a generic UNKNOWN error when credentials are missing, and the frontend has no specific handler for that case so it falls back to a generic error message.

I fixed it by making changes in two files. In `app-gocardless.ts`, I added a check at the very start of the `/transactions` handler using the `isConfigured()` method that already existed in the codebase but was never called on this path. When credentials are missing, the server now immediately returns `GOCARDLESS_NOT_CONFIGURED` instead of trying to contact GoCardless and crashing with a confusing error. In `AccountSyncCheck.tsx`, I added a new case in the `getErrorMessage` function that handles `GOCARDLESS_NOT_CONFIGURED` and shows the user a clear, actionable message.

I also wrote a unit test that confirms the guard works correctly, ran typecheck and lint to make sure nothing was broken, and created a release note file as required by the project's contribution guidelines.

### Code Changes

- **Files modified:**
  - `packages/sync-server/src/app-gocardless/app-gocardless.ts`
  - `packages/desktop-client/src/components/accounts/AccountSyncCheck.tsx`
- **New files added:**
  - `packages/sync-server/src/app-gocardless/tests/transactions.spec.ts`
  - `upcoming-release-notes/gocardless-missing-credentials-error.md`
- **Branch:** https://github.com/birajlamichhane75/actual/tree/fix-issue-4742
- **Commit:** Show a clear error when GoCardless credentials are missing after restore

---

## Pull Request

*(To be filled in during Phase IV)*

---

## Learnings & Reflections

*(To be filled in at the end of the program)*

---

## Resources Used

- [Issue #4742](https://github.com/actualbudget/actual/issues/4742)
- [Actual Budget Documentation](https://actualbudget.org/docs/)
- [Actual Budget Contributing Guide](https://github.com/actualbudget/actual/blob/master/CONTRIBUTING.md)

---

## AI Usage Log

**Task:** Understanding the codebase and tracing the GoCardless sync flow
**AI used:** Claude (claude.ai) and Claude Code (VS Code)
**What I asked:** Helped me understand the issue line by line, explained GoCardless concepts, helped set up the local environment on Windows, and guided me through reading the codebase to find the root cause.
**Result:** I traced the bug myself across 6 files from the frontend error message all the way back to the sync server. Claude Code helped me understand what I was reading and reviewed the code I wrote. The test and release note files were drafted by Claude Code and reviewed by me.
**Verification:** I followed every step myself, understood each file and function before moving forward, and manually verified the fix worked on my local environment before committing.
**Reflection:** Using AI to navigate an unfamiliar codebase saved a lot of time, but the most valuable part was tracing the bug myself. When I got to code review I could explain every line because I wrote it and understood why it was there. Next time I want to trace more of the codebase myself before asking for help.
