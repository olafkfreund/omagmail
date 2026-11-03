---
status: approved
issue: 1
author: olafkfreund
---

# Intent: Google Workspace in one Omarchy window

## Problem

There is no Omarchy window here for using Gmail, Google Calendar, Google Contacts, and Google Meet together. Omaoutlook provides a useful reference for desktop integration, but its Outlook-specific behavior does not establish that Google's services will work in the same host.

## Proposed outcome

The user can open one Omarchy window, sign in to a Google account, use mail, calendar, contacts, and meetings, and follow links without losing their place or their session. The application reports unsupported Google sign-in or meeting features clearly if they cannot be made to work in QtWebEngine.

## Affected users and systems

Omarchy desktop users; the new omagmail application and plugin; Google account sign-in, Gmail, Calendar, Contacts, and Meet; camera, microphone, screen sharing, notifications, downloads, and local browser profile data.

## Constraints

- Use QtWebEngine for the application window, as requested.
- Keep account credentials and session data in the web engine's local profile; do not implement a separate Google API login or copy cookies from another browser.
- Preserve user drafts and active meetings when switching views, hiding the window, or managing memory.
- Do not depend on disguising the embedded browser to bypass Google's sign-in restrictions. Verify actual sign-in behavior before claiming support.
- Reuse the relevant omaoutlook design and respect its MIT license and attribution.
- Follow the issue, branch, intent, spec, and plan approval gates before implementation. The later plan should propose agy and Claude coding and review roles for approval.

## Open questions

- Which Google account types and sign-in methods must pass acceptance: personal, Workspace, two-step verification, passkeys, or managed SSO?
- If Google blocks sign-in in QtWebEngine, should work stop at a documented blocker, or should the project proceed for already signed-in sessions that still function?
- Does "one window" require full Calendar and Contacts views and an active Meet call inside that window, or are Gmail's side panels and a meeting popout acceptable?
