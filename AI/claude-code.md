# Claude Code (Anthropic) - agent tooling

**Use for:** Anthropic's terminal agent, and the tooling around it: browser automation via the Claude in Chrome extension, MCP servers. This file is about the tooling, not the Claude API itself (see `claude-api.md`).
**Status:** Active

## Setup & access

- Never pass secrets to MCP servers as literal env values on the command line. Launch MCPs through a small wrapper that reads a local secret store (macOS Keychain or equivalent) at runtime.
- Browser automation is the Claude in Chrome extension. It connects to the same Claude account the terminal is signed into. Aim for exactly one connected browser; two or more forces a "which browser" prompt on every task.

## Scars & gotchas

- **Claude in Chrome does not work in Arc.** Arc is Chromium-based, so the extension installs, signs in, and registers a connection. It even appears as a normal macOS browser in the connected-browsers list. But every command times out, including the lightest one (getting tab context). Arc does not implement the Chrome side panel and tab group APIs the extension drives. The error text is misleading: "extension is connected but the page may be loading, unresponsive, or waiting on a permission prompt." There is no prompt to find. Fix: install the extension in Google Chrome and remove it from Arc. Expect the same from other Chromium forks that lack side panel support.

- **Dedicated agent browser pattern.** Keep your daily browser for yourself and give the agent its own Google Chrome. Chrome stays closed until a task needs it. The agent launches it from the terminal (`open -a "Google Chrome"` on macOS) and the extension reconnects in about six seconds. Verified cold-start behavior: zero connected browsers while Chrome is closed, reconnect plus a fresh tab group right after launch. Do not quit Chrome while the agent is mid-task.

- **Uninstall, do not disable.** A disabled extension in the wrong browser keeps registering and can be picked as the target. Only after uninstalling it did the stale device entry disappear and Chrome register cleanly under a new device id.

- **The "list connected browsers" tool wants you to prompt the user even with a single browser connected.** The tool description says to ask the user to pick a browser before any action. With exactly one local browser connected that prompt is dead weight and blocks an autonomous run: skip it and go straight to getting tab context (with create-if-empty). Only prompt when two or more browsers are listed.

- **The agent Chrome caches `localhost` preview pages across navigations.** Re-rendering a file and navigating to the same URL can show the stale page. Bump a query param (`?v=N`) on every re-render; it does not matter whether the dev server itself busts the cache.

- **A screenshot immediately after a navigate can fail with a script-injection timeout.** The page is still loading. Put a short wait (about two seconds) between navigate and screenshot in the same batch call; a batch stops on the first error, so the screenshot never runs otherwise.

- **A headless `--screenshot` browser invocation can hang forever while the agent's own Chrome is already running**, even with a separate `--user-data-dir`. It completes in seconds when that Chrome is closed. When the agent Chrome is up, take screenshots through the extension (screenshot, scroll, screenshot) instead of shelling out to a second headless instance.

- **When the browser extension is not connected, fall back to a real browser-automation library rather than guessing a fix from CSS alone.** A globally installed Playwright package ships a bundled WebKit build: launch it with `webkit.launch()` and a mobile device profile (e.g. `iPhone 13`) to get a real Safari-engine mobile render, not just Chromium. This mattered concretely on a mobile CSS bug reported via phone screenshots: a first fix was written from reading the CSS alone (a plausible-sounding Safari-only theory) and shipped wrong; only rendering the actual page in WebKit, against both a local dev server and the live preview URL, surfaced the real cause and proved the fix. If running from a scratchpad script with no local `node_modules`, import the library via its absolute install path.

- **A page's own load-time deep-link/scroll logic can fight a script's manual `scrollIntoView`.** A homepage intro loader that restores scroll position itself once its animation finishes (honoring `location.hash` if present, else scrolling to top), asynchronously after the page's `load` event, will silently override a script's `element.scrollIntoView()` called right after `domcontentloaded`. Fix: navigate directly to the URL with the target `#hash` already in it, so the site's own loader does the scrolling correctly instead of a script fighting it from outside.

## Conclusions / best practices

- Browser automation is the last resort. Anything with an API or MCP goes direct. The browser is for UI-only surfaces: payment provider dashboards, chat-automation builders, ad managers, one-off sites.
- One Chrome profile for the agent, logged into the needed accounts once. Sessions persist across launches, so the agent never handles credentials.
- If browser tools time out, check the connected-browsers list first. Empty means Chrome is closed. An entry that answers nothing means the wrong browser holds the extension.
