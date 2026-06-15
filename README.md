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

# Contribution [#]: [Issue Title]

**Contribution Number:** #4742  
**Student:** Biraj Lamichhane
**Issue:** https://github.com/actualbudget/actual/issues/4742
**Status:**  Phase In progress

---

## Why I Chose This Issue

GoCardless is a service that connects a real bank account to Actual Budget. When restoring a backup, the GoCardless Secret ID and Secret Key are not restored. If a user tries to sync their bank account after restoring, the app fails silently with no useful feedback. There is no indication that the credentials are missing, which makes it highly confusing for the user to diagnose and repair the problem. 
I am interested in this issue because I have experience in React.js and Next.js. At my internship, I handled error states by displaying proper error messages on screen. This issue is similar because right now the app fails silently with no useful feedback when GoCardless credentials are missing after a restore. I want to fix this by detecting when the credentials are missing and showing a clear, helpful error message in the UI so the user knows exactly what went wrong and how to fix it.

---

## Understanding the Issue

### Problem Description


### Expected Behavior


### Current Behavior


### Affected Components


---

## Reproduction Process

### Environment Setup



### Steps to Reproduce


### Reproduction Evidence

- **Commit showing reproduction:** 
- **Screenshots/logs:** 
- **My findings:** 

---

## Solution Approach

### Analysis


### Proposed Solution


### Implementation Plan


**Understand:** 

**Match:**

**Plan:** 

**Implement:** 

**Review:**

**Evaluate:** 

---

## Testing Strategy

### Unit Tests



### Integration Tests



### Manual Testing


---

## Implementation Notes

### Week [X] Progress



### Week [Y] Progress



### Code Changes

- **Files modified:** 
- **Key commits:** 
- **Approach decisions:** 

---

## Pull Request

**PR Link:**

**PR Description:** 

**Maintainer Feedback:**


**Status:** 

---

## Learnings & Reflections

### Technical Skills Gained



### Challenges Overcome



### What I'd Do Differently Next Time



---

## Resources Used

- https://github.com/actualbudget/actual/issues/4742
- https://actualbudget.org/docs/

