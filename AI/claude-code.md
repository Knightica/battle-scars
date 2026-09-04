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

## Conclusions / best practices

- Browser automation is the last resort. Anything with an API or MCP goes direct. The browser is for UI-only surfaces: payment provider dashboards, chat-automation builders, ad managers, one-off sites.
- One Chrome profile for the agent, logged into the needed accounts once. Sessions persist across launches, so the agent never handles credentials.
- If browser tools time out, check the connected-browsers list first. Empty means Chrome is closed. An entry that answers nothing means the wrong browser holds the extension.
