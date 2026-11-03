---
status: draft
issue: 1
intent: intent/2026-11-03-1-qt-google-workspace.md
---

# Spec: QtWebEngine Google Workspace window

## Design

Build `omagmail` as one Qt Quick window with Mail, Calendar, Contacts, and Meet sections. Each section owns a retained `WebEngineView`; all four use one persistent `WebEngineProfile`. Create each view when first opened, then keep it alive when switching sections so a draft, calendar edit, or meeting is not discarded. The selected section fills the window. An active Meet call must continue when another section is selected or the window is hidden. Google sign-in redirects and required popups use the same profile.

Use the structure of [omaoutlook's Qt host](https://github.com/Divanyx/omaoutlook/blob/main/host/qml/Main.qml) and [Omarchy plugin](https://github.com/Divanyx/omaoutlook/tree/main/ui) as references, with attribution under its MIT license. Reuse only generic behavior that still applies: single-instance commands, a persistent local profile, downloads, printing, notification forwarding, window show/hide, and the bar service. Make the application identity and profile separate from omaoutlook so neither app touches the other's data.

Navigation recognizes the exact Gmail, Calendar, Contacts, and Meet hosts and preserves each service's own URLs and account index. Navigation among the four services selects the corresponding retained view. Google account sign-in redirects stay in the application. User-clicked non-service links open in the default browser; `mailto:` opens Gmail compose for the active account. Unknown schemes go to their desktop handler. Popups required by a Google service use the shared profile; an external link must not silently replace an open draft or call. Replace [Outlook's URL classifier and in-page router](https://github.com/Divanyx/omaoutlook/blob/main/host/qml/Links.js), rather than translating Outlook routes into Google routes.

The native window follows Omarchy colors and reacts to theme changes. Google page content initially uses Google's own supported theme controls; any page CSS must be scoped to the four service origins, avoid sign-in pages and message bodies, and pass visual checks across the four services before it is enabled by default. Replace [Outlook's Fluent UI mapping](https://github.com/Divanyx/omaoutlook/blob/main/host/qml/Theme.js), which does not apply to Google pages.

Use QtWebEngine's site permission requests for notifications, clipboard, camera, microphone, and screen capture. Ask before granting camera, microphone, or screen capture to Meet. Deny unsupported permissions and requests from unrelated origins. Forward permitted web notifications through the desktop notification service. Show a Gmail unread count in the bar only when it can be read reliably from the page title or a stable page signal; otherwise show no number. Do not intercept Gmail network traffic to infer a count. Replace [Outlook's unread script](https://github.com/Divanyx/omaoutlook/blob/main/host/qml/Page.js). Do not recycle a view with a draft, calendar edit, active download, or meeting; start without automatic page recycling until those protections are verified.

The first implementation checkpoint is a QtWebEngine sign-in and Meet feasibility test on the target Omarchy desktop. Use a normal QtWebEngine user agent and profile. If Google blocks sign-in or Meet rejects the embedded browser, record the exact failure and stop implementation for review; do not mask the browser identity, import another browser's cookies, or claim the feature works. No Google API OAuth client or credentials are required for this web wrapper. Google explicitly [restricts embedded user agents for OAuth](https://developers.google.com/identity/protocols/oauth2/policies), so successful sign-in on one account must not be presented as a guarantee for all accounts.

## Alternatives rejected

- Chrome or a Chrome app window: rejected because the user explicitly requires QtWebEngine.
- One `WebEngineView` that reloads across all services: switching could discard drafts or a live Meet call, and the four services are separate web apps.
- Copying omaoutlook's Microsoft-specific routes, unread data hooks, or Fluent CSS: those target Outlook internals.
- Native Gmail, Calendar, Contacts, and Meet APIs: they add OAuth scopes and a new client implementation without providing the requested web interfaces.
- User-agent disguise or browser-cookie transfer to work around sign-in: unreliable and contrary to the sign-in constraint in the approved intent.

## Risks

- **All accounts:** Google may reject QtWebEngine sign-in under its embedded-browser policy. The project cannot promise usable Gmail without a successful target-system test.
- **Workspace accounts:** administrator rules, SSO, two-step verification, and passkeys may behave differently from a personal account. Test methods available to the user; do not claim untested methods.
- **Meet on Wayland:** camera, microphone, screen sharing, and calls while the view is hidden may fail or stop. An active call must be tested before memory management or window-hiding behavior is accepted.
- **Page changes:** Google may change title formats, link flows, or CSS. A missing unread count is preferable to an incorrect one; native window theming must remain usable when page styling fails.
- **Memory and privacy:** four retained web apps use more memory than one Outlook view. The shared profile stores Google sessions locally; permissions and external-link routing must remain origin-specific.

## Verification

1. Build the Qt host and load the QML without errors on the target desktop. Verify the four sections can open, switch, and return without reloading a draft or edit.
2. Exercise personal Google sign-in in a fresh profile, including the second factor available to the tester. Exercise a Workspace account and its sign-in method when one is available. Restart the app and confirm the session persists. Capture `disallowed_useragent` or other rejection as a blocking result, not a passing test.
3. Join a Meet call in the window; check camera, microphone, screen sharing, section switching, and hiding/restoring the window. Confirm the call stays connected. Test a scheduled meeting link from Calendar and one from Gmail.
4. Test exact-host link classification, redirects, `mailto:`, popups, external links, and malicious lookalike hosts. Check that links do not replace an unsent draft or active call.
5. Check desktop notifications, attachment downloads, printing, Omarchy theme changes, bar controls, and unread display when a stable signal is present. Verify no stale or fabricated unread count is shown.
6. Run focused automated tests for URL routing and any nontrivial page signal parsing. Repeat the interactive checks after a QtWebEngine update before declaring compatibility with that version.
