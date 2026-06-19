# Open Source Contribution — CodePath AI301

**Contribution Number:** 1
**Student:** Biraj Lamichhane
**Issue:** [#4742 — Bug: Internal error when syncing GoCardless after restoring backup](https://github.com/actualbudget/actual/issues/4742)
**Status:** Phase II Complete

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

- **Branch:** [fix-issue-4742](https://github.com/birajlamichhane/actual/tree/fix-issue-4742)
- **My findings:** Queried the sync server's database directly and confirmed the secrets table has 0 rows after setup, proving `secretsService.get('gocardless_secretId')` returns null. Sent a real POST request to `/gocardless/transactions` and received `error_code: "UNKNOWN"` with the underlying GoCardless 400 error `"secret_id: This field is required"` buried inside — proving the server receives a meaningful error but discards it in favor of a generic UNKNOWN response.

---

## Solution Approach

### Analysis

The root cause is a chain of three failures:

1. The sync server's `/transactions` handler does not check if GoCardless credentials exist before attempting to use them. The `isConfigured()` method already exists in `gocardless-service.ts` but is never called during sync.

2. When GoCardless rejects the empty credentials with a 400 error, the server's error handler maps it to a generic `UNKNOWN` error code instead of something meaningful.

3. The frontend's `AccountSyncCheck.tsx` has no case for handling a missing-credentials error, so it falls through to the default "internal error" message with no recovery path.

### Proposed Solution

Define one new specific error code on the server (e.g. `GOCARDLESS_NOT_CONFIGURED`) when credentials are missing, then thread it through the frontend so the user sees a clear message and a button to re-enter their credentials. Add a warning to the export UI that server configuration is not included in the backup.

### Implementation Plan

**Understand:**
When GoCardless credentials are missing after a backup restore, the app shows a generic "internal error" with no recovery path. The user has no idea their credentials are missing or how to fix it. The expected behavior is a clear error message and a direct path to re-enter credentials.

**Match:**
The `ITEM_LOGIN_REQUIRED` error code in `AccountSyncCheck.tsx` already follows the exact pattern I need — it shows a specific message and opens a recovery modal. I will follow this same pattern for the missing credentials case. The encryption warning in `Export.tsx` (lines 70-77) is the exact pattern to follow for the export warning.

**Plan:**
1. In `packages/sync-server/src/app-gocardless/app-gocardless.ts` — add an `isConfigured()` check at the start of the `/transactions` handler. If credentials are missing, return `error_type: 'ITEM_ERROR', error_code: 'GOCARDLESS_NOT_CONFIGURED'` instead of continuing.
2. In `packages/loot-core/src/server/accounts/app.ts` — add `GOCARDLESS_NOT_CONFIGURED` as a recognized status in `getBankSyncStatusFromError` so it is classified as a recoverable failure.
3. In `packages/desktop-client/src/components/accounts/AccountSyncCheck.tsx` — add a case in `getErrorMessage` for `GOCARDLESS_NOT_CONFIGURED` with a clear message, and extend `showAuth` to show a button that opens `GoCardlessInitialiseModal` so the user can re-enter their credentials.
4. In `packages/desktop-client/src/components/settings/Export.tsx` — add a `<Text>` warning after line 69 (mirroring the encryption warning at lines 70-77) explaining that server configuration including GoCardless credentials is not included in the export.

**Implement:**
[Link to branch will be added as commits are pushed during Phase III]

**Review:**
Before submitting PR I will check:
- Code follows existing patterns in the codebase
- No new dependencies added
- Commit messages follow the project's convention
- Changes are small and focused on this issue only
- CONTRIBUTING.md guidelines are followed

**Evaluate:**
- Manually trigger the bug reproduction steps above and confirm the new specific error message appears instead of "internal error"
- Confirm a button to re-enter credentials appears and opens the correct modal
- Confirm the export warning appears in the settings UI
- Run the existing test suite to confirm nothing is broken

---

## Testing Strategy

### Unit Tests
- [ ] Test that `getErrorMessage('ITEM_ERROR', 'GOCARDLESS_NOT_CONFIGURED')` returns the correct message
- [ ] Test that `showAuth` returns true for `GOCARDLESS_NOT_CONFIGURED`
- [ ] Test that the `/transactions` endpoint returns `GOCARDLESS_NOT_CONFIGURED` when `isConfigured()` is false

### Integration Tests
- [ ] Test that syncing with missing credentials shows the correct error in the UI
- [ ] Test that the recovery button opens the GoCardless credentials modal

### Manual Testing

Follow the reproduction steps above. After the fix, the user should see a clear message like "Your GoCardless credentials are missing. Please re-enter them." with a button to fix it instead of the generic internal error.

---

## Implementation Notes



---

## Pull Request



---

## Learnings & Reflections


---

## Resources Used

- [Issue #4742](https://github.com/actualbudget/actual/issues/4742)
- [Actual Budget Documentation](https://actualbudget.org/docs/)
- [Actual Budget Contributing Guide](https://github.com/actualbudget/actual/blob/master/CONTRIBUTING.md)

---

## AI Usage Log

**Task:** Understanding the codebase and tracing the GoCardless sync flow
**AI used:** Claude (claude.ai) and Claude Code (VS Code)
**What I asked:** Helped me understand the issue line by line, explained GoCardless and SharedArrayBuffer concepts, helped set up the local environment on Windows, and traced the full sync flow across the codebase to find the root cause.
**Result:** Claude Code traced the bug from the sync server's missing credential check all the way to the frontend's generic error message, identified the exact files and line numbers for each fix, and confirmed the bug by querying the database directly.
**Verification:** I followed along with every step, understood each file and function involved, and confirmed the secrets table was empty on my own running server.
**Reflection:** AI helped me navigate an unfamiliar codebase much faster than reading every file myself. However I made sure to understand every finding before moving forward so I can explain and defend my changes during code review.
