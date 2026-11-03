---
status: draft
issue: 1
spec: spec/2026-11-03-1-qt-google-workspace.md
coders:
  1: claude
  2: agy
  3: claude
  4: claude
  5: claude
  6: claude
  7: agy
  8: agy
  9: agy
reviewers:
  - codex
  - agy
---

# Plan: QtWebEngine Google Workspace window

Build one Omarchy window with retained Mail, Calendar, Contacts, and Meet views sharing one persistent QtWebEngine profile. The first implementation checkpoint tests Google sign-in and Meet on the target desktop with an unmodified QtWebEngine identity. Stop and report a block if either service rejects the embedded browser. Do not substitute Chrome, import cookies, spoof the user agent, or add Google API OAuth. The nine `coders` assignments above were proposed by Claude for the user's requested Claude and agy coding help; Claude dispatches them after plan approval. Codex reviews the whole diff against this plan; agy additionally reviews Claude's routing, permission, and view logic. A separate final Claude review is read-only.

Use [omaoutlook at `ad934b6fabe3a34e33e72108e809a3612836101c`](https://github.com/Divanyx/omaoutlook/tree/ad934b6fabe3a34e33e72108e809a3612836101c) as a reference. Copy generic code only when it is shorter and safer than rewriting it, and retain its MIT license and attribution. This repo has only planning artifacts today; all application file locations below are new. Use qmake6 as omaoutlook does. Keep Qt and Node build tools in this project's Nix flake dev shell and package; do not install system packages or change the machine's NixOS configuration. The flake is one project pin for both development and packaging.

## Steps

1. `flake.nix:new`, `flake.lock:new`, `host/omagmail.pro:new`, `host/main.cpp:new`, `host/qml/Main.qml:new`, `host/qml.qrc:new`, `bin/build:new`: create the smallest buildable QtWebEngine window with one persistent profile, normal user agent, and a selectable Gmail or Meet URL. Keep the profile under the omagmail identity. Build with `nix develop -c bin/build`, then manually try fresh-profile personal Google sign-in and an actual Meet call on the target desktop. Record the Qt version, account type, sign-in method, and exact result without recording credentials. If sign-in is rejected or Meet cannot operate, stop here for review; later steps are blocked. → Verify by `nix develop -c bin/build` and the signed-in Meet checklist in **Tests**.
   Traps: Google forbids embedded OAuth user agents; no UA stripping, browser-cookie transfer, Google API OAuth client, or claim that one successful account covers all account types. No automatic page recycling.

2. `host/Bridge.h:new`, `host/Bridge.cpp:new`, `host/main.cpp:new`: add the generic single-instance local socket and CLI (`show`, `hide`, `toggle`, `open`, `quit`, `status`), atomic status-file writes, notification forwarding over D-Bus, download paths, and print-to-PDF support, adapting only needed parts of omaoutlook's bridge. Route errors to the window or CLI. → Verify by `nix develop -c bin/build` and two launches producing one host process; `status`, show/hide, a download, and print all work.
   Traps: unique `omagmail` socket, desktop identity, profile, and state paths; no Outlook or Microsoft service names; never put mail subjects or sender names on a command line; preserve safe download naming and errors.

3. `host/qml/Links.js:new`, `tests/links.test.js:new`: implement exact-host classification for `mail.google.com`, `calendar.google.com`, `contacts.google.com`, `meet.google.com`, and the necessary sign-in redirects. Preserve Google account indices and full destination URLs. Classify clicked non-service links and lookalike hosts as external, handle unknown schemes via desktop handlers, and build Gmail compose URLs from `mailto:` without losing recipients, subject, or body. → Verify by `nix develop -c node --test tests/links.test.js`.
   Traps: `google.com.evil.example` and `www.google.com/url` are not trusted service pages; do not allow a generic `*.google.com` rule; test percent encoding and multiple recipients. Do not port Outlook's `/owa` route rewriting.

4. `host/qml/Main.qml:new`, `host/qml/ServiceView.qml:new`, `host/qml/PopoutWindow.qml:new`, `host/qml.qrc:new`: replace the feasibility view with four lazy-created, retained views and a small native section switcher. All views and allowed service popups share the same profile. A switch selects a view without changing its URL or destroying it. Route service links to their view, sign-in redirects and required popups in-profile, and user-clicked external links to the browser. Keep a live Meet view loaded when switching or hiding. → Verify by drafting an unsent message, editing a calendar event, switching among all sections, and returning to both without reload; repeat during an active call.
   Traps: do not navigate the Gmail view to Calendar, Contacts, or Meet; preserve `u/N` account indices; do not force a popup into a view if it needs an opener; no auto-discard or memory watchdog before call and draft safety is proven.

5. `host/qml/Main.qml:new`, `host/qml/PermissionDialog.qml:new`, `host/main.cpp:new`, `host/qml.qrc:new`: handle site permission requests for notifications, clipboard, microphone, camera, and Wayland screen capture. Ask before capture permissions for Meet, deny unsupported and unrelated-origin requests, and enable the QtWebEngine PipeWire capture feature only if needed by the target Qt version. Forward permitted web notifications through the bridge. → Verify by joining a call with camera and microphone, sharing a screen, denying a request, and checking notification click behavior.
   Traps: match origins exactly, persist only the user's explicit decision, never auto-grant capture or a broad Google suffix, and do not run a live call through a hidden-view lifecycle state that stops media.

6. `host/qml/Main.qml:new`, `host/qml/Unread.js:new`, `tests/unread.test.js:new`: read Gmail unread from a confirmed title or stable page signal and publish `null` when unavailable; send status changes to the bar service. Do not intercept Gmail fetch, XHR, or WebSocket traffic. → Verify by `nix develop -c node --test tests/unread.test.js` and by changing unread mail in Gmail while the window is visible and hidden.
   Traps: do not display a stale or invented zero; parse the account's inbox count rather than a different folder; if no stable signal exists, keep the badge without a number and record the limitation.

7. `host/qml/Theme.js:new`, `host/qml/Main.qml:new`, `host/qml.qrc:new`: reuse only omaoutlook's generic Omarchy color parsing and file-watch approach for the native window and section switcher. Follow live theme changes. Leave Google page CSS off unless scoped visual checks show it works across all four services without touching sign-in pages or mail content. → Verify by switching a light and dark Omarchy theme while all sections are open.
   Traps: do not copy Fluent tokens or Outlook message-body color rewriting; do not edit the user's Omarchy configuration; preserve readable contrast and native focus states.

8. `manifest.json:new`, `ui/Service.qml:new`, `ui/BarWidget.qml:new`, `assets/omagmail.desktop:new`, `assets/omagmail.svg:new`, `flake.nix:new`: connect the host's status file and CLI to an Omarchy bar icon with show/hide and optional unread number. Package the application, icon, and desktop entry through the same flake; keep plugin settings limited to behavior implemented by the host. → Verify by `nix flake check`, `nix build .#omagmail`, and a local plugin load with a running host.
   Traps: package and plugin paths use `omagmail` only; no system rebuild or `home-manager switch`; do not register `mailto:` globally without an explicit user setting and a reversible handler change.

9. `README.md:new`, `LICENSE:new`, `tests/:new`, `plan/2026-11-03-1-qt-google-workspace.md:existing if needed`: document installation, tested account types, limitations, privacy of the local profile, shortcuts, and attribution. Run the automated and interactive checks below; fix failures at their source. If a required implementation choice deviates from this plan, update this plan in the same code change for review. → Verify by the complete **Tests** list and a clean diff review against the approved spec.
   Traps: do not promise passkeys, Workspace SSO, screen sharing, or unread accuracy without actual evidence; do not include real account details in screenshots, logs, tests, or issue comments.

## Tests

- `nix develop -c bin/build`: exits 0 and starts a QML window without loader errors. The project flake supplies qmake6, Qt WebEngine, and test tools.
- `nix develop -c node --test tests/links.test.js tests/unread.test.js`: all URL and unread cases pass, including malicious lookalike hosts and unknown count.
- `nix flake check` and `nix build .#omagmail`: evaluation and package build pass without machine-level installation.
- **Feasibility gate, before step 2:** from a fresh omagmail profile, sign in with a personal account using the available second factor, restart, and confirm session persistence. Join a real Meet call and check camera, microphone, and screen sharing. If the account or QtWebEngine rejects either, capture the exact error and stop for review. Test a Workspace account and its configured method when one is available; otherwise mark it unverified.
- **Acceptance after step 9:** switch among all four services without losing an unsent draft, calendar edit, or active Meet call; hide and restore during a call; open a meeting from Gmail and Calendar; test internal, external, `mailto:`, popup, and unknown-scheme links; test a notification, attachment, print, light/dark theme, bar control, and restart. State any unverified behavior plainly.

## Rollback

Stop `omagmail`, remove its plugin and package from the user's configuration, and rebuild through the user's NixOS flake if it was installed there. Revert this task branch to restore the repository. Leave the separate omagmail profile intact by default so rollback does not erase mail sessions or downloads; offer an explicit profile purge only if the user requests it. Omaoutlook's profile and settings are untouched.
