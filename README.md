# Open Source Contribution — CodePath AI301

## Issue
**Project:** Actual Budget  
**Issue:** [#4742 — Bug: Internal error when syncing GoCardless after restoring backup](https://github.com/actualbudget/actual/issues/4742)  
**Status:** In progress



## Why I Picked This Issue
GoCardless is the service that connects real bank account to actual app.
The issue is when restoring a backup the GoCardless Secret Id and Secret Key are not restored. If a user tries to sync the bank accounts after restoring a backup the message "An internal error has occured" appears. The app give no indication that the secrets are not restored so it is highlt confusing for user to repair the problem.

I am interested in this issue because I have experience in React.js and Next.js. At my internship, I handled error states by displaying proper error messages on screen. This issue is similar because right now the app fails silently with no useful feedback when GoCardless credentials are missing after a restore. I want to fix this by detecting when the credentials are missing and showing a clear, helpful error message in the UI so the user knows exactly what went wrong and how to fix it.




## My Understanding of the Problem
GoCardless is the service that connects real bank account to actual app.
The issue is when restoring a backup the GoCardless Secret Id and Secret Key are not restored. If a user tries to sync the bank accounts after restoring a backup the message "An internal error has occured" appears. The app give no indication that the secrets are not restored so it is highlt confusing for user to repair the problem. 
I need to fix this by detecting when the credentials are missing and showing a clear, helpful error message in the UI so the user knows exactly what went wrong and how to fix it.




## AI Usage Log
