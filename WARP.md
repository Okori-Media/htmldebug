# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

Project overview
- This is a static HTML/JS sandbox for debugging and demonstrating browser behaviors, notably an exit-intent detection library (exit-intent.js) and two demo pages that integrate it (one with Customer.io analytics).

Common commands
- Serve locally (Python, built-in on macOS):
  - python3 -m http.server 8080
  - Open http://localhost:8080 and use index.html to navigate to test pages.
- Alternative server (Node, if installed):
  - npx http-server -p 8080
- Open a specific page directly:
  - open exit_intent_test.html
  - open cio_exit_test.html
- Enable verbose logs (for debugging exit-intent):
  - In the page script where observeExitIntent is called, pass debug: true (already enabled in the demo pages).

Notes on build/lint/tests
- There is no build step or package manager configuration (no package.json). Files are served as-is.
- There is no configured linter or formatter.
- There is no automated test runner; behavior is verified interactively in the browser via the test pages.

Running and debugging
- Primary entry for exploration: index.html (links to the three demo pages).
- VS Code Chrome debug configuration is present at .vscode/launch.json and expects http://localhost:8080. To use it, start a local server on port 8080, then launch the “Launch Chrome against localhost” config in VS Code.
- For manual testing, use the browser devtools console to observe emitted events and logs.

High-level architecture
- exit-intent.js
  - Provides observeExitIntent(options) which attaches multiple detectors for “exit intent” and dispatches a CustomEvent on window (default name: exit-intent, the demos override with my-exit-event).
  - Detectors include: time on page, idle time (manual activity reset), mouse leave with delay, tab visibility change, window blur (with child-iframe focus guard), and fast upward scroll (threshold can differ for mobile vs. desktop).
  - Page-view counter helpers (localStorage-based) allow immediate triggering after N visits; incrementPageViews() runs on load and is also exposed as observeExitIntent.incrementPageViews for SPA-style navigation.
  - Exposes a destroy() handle from observeExitIntent for cleanup of timers/listeners. Also sets module.exports when CommonJS is detected for reuse outside the browser-global pattern.
- Test/demo pages
  - index.html: simple navigation hub to the demos.
  - exit_intent_test.html: integrates exit-intent.js, listens for my-exit-event, shows a modal once per session with the reason detail, and demonstrates multiple triggers. Uses debug: true for console logging.
  - cio_test.html: minimal page that loads the Customer.io analytics snippet and calls analytics.page().
  - cio_exit_test.html: combines both—loads Customer.io and exit-intent.js; on my-exit-event, tracks exit_intent_detected via window.cioanalytics.track with payload (trigger, timestamp, page_url, page_title) and shows the modal.

Operational behavior (big picture)
- Data flow
  - User interactions in the page are monitored by observeExitIntent via native browser events. On a qualifying pattern, observeExitIntent.trigger(reason) dispatches a CustomEvent to window.
  - Consumers (the demo pages) add a single window.addEventListener for the custom event name and branch on e.detail (e.g., 'timeOnPage', 'idleTime', 'mouseLeave', 'tabChange', 'windowBlur', 'scrollUp', 'pageViews').
  - In cio_exit_test.html, the listener also forwards a tracking event to Customer.io (if the snippet is available), otherwise it warns to console.
- Tuning
  - Sensitivity and which detectors are active are controlled through options passed to observeExitIntent (e.g., timeOnPage, idleTime, mouseLeaveDelay, tabChange, windowBlur, scrollUpThreshold with separate values for mobile/desktop, mobileBreakpoint, scrollUpInterval, pageViewsToTrigger, eventName, debug).
  - Scroll thresholds are selected responsively at init based on window.innerWidth relative to mobileBreakpoint.

How to reproduce and validate core behaviors
- Start a local server and open exit_intent_test.html (or cio_exit_test.html).
- Trigger paths:
  - Mouse leave: move the cursor outside the browser window; the handler uses a short delay and ignores element-level transitions.
  - Tab change: switch tabs to make the document hidden.
  - Window blur: click outside the window; the handler ignores focus change into a child iframe on the same page.
  - Idle time: avoid input (mouse/keyboard/touch) long enough for the idle timer.
  - Time on page: remain on the page until the timeout fires.
  - Scroll up fast: scroll down, then quickly scroll up at least the configured threshold.
- Observe: a modal appears (once per session per page reload), and the console logs the reason; in cio_exit_test.html, a track call is attempted to Customer.io.

Conventions and gotchas specific to this repo
- Because this is a static project without bundling, ensure script tags load exit-intent.js before your integration code that calls observeExitIntent.
- The page-views counter uses localStorage; if testing in privacy modes with restricted storage, counters default to 0 and may affect pageViewsToTrigger behavior.
- window blur handling contains a guard for focus moving into iframes; if you embed third-party widgets via iframes, exit intent should not trigger from intra-page focus shifts.
