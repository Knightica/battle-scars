# Hermes Agent (Nous Research)

**Use for:** Self-hosted always-on autonomous agent daemon: multi-channel gateway (Telegram/Discord/Slack/etc.), persistent memory, NL cron, self-written skills, MCP tool use. Runs Claude (or any provider) as the brain.
**Status:** Active

## Setup & access

- Open-source (MIT), `NousResearch/hermes-agent` on GitHub, with hosted docs. Python 3.11 + Node 22, runs as a gateway on a local port (polls outbound, no inbound exposure needed).
- Files live under a dedicated OS user, e.g. `~/.hermes/config.yaml` (config), `~/.hermes/.env` (secrets), `~/.hermes/SOUL.md` (persona), `~/.hermes/memories/` (MEMORY.md, USER.md), `~/.hermes/state.db` (SQLite+FTS5 history).
- Some VPS providers (DigitalOcean among them) offer a 1-click Hermes Agent marketplace image, preconfigured for a dedicated user; first SSH login runs the setup wizard.
- CLI: `hermes` (chat), `hermes setup`, `hermes model`, `hermes gateway setup/start/status`, `hermes doctor`.

## Scars & gotchas

- **Anthropic OAuth does NOT ride the Claude Max subscription.** Hermes routes "Claude Pro/Max OAuth" to the standard `api.anthropic.com` endpoint and bills metered "extra usage" at API token rates regardless. The OAuth login flow is also flaky (stale token endpoints, 404s). Use a direct Anthropic API key instead, and set the spend cap provider-side in the Anthropic console: Hermes has no internal dollar/budget cap.

- **Approval gating covers dangerous SHELL COMMANDS ONLY, there is NO gate on MCP tools.** If the agent has an MCP tool like a mail-send or messaging-send tool, it fires immediately with no confirmation. There is no PreToolUse-style equivalent and no per-tool policy. Any human-in-the-loop gate for MCP/outbound actions must be built yourself (e.g. a gating-proxy MCP that holds the real credentials).

- **Per-user auth is binary allow/deny at the gateway, NO RBAC.** The allowed-users setting gates who may talk to the agent, but once allowed everyone gets identical capabilities. No admin-vs-guest, no per-user tool scoping. For tiered access, run separate instances or a custom identity-bound proxy, and never trust a model-supplied "user" argument (prompt injection defeats it): use the gateway-authenticated sender ID.

- **The shell-command approval gate is SILENTLY SKIPPED on Docker/Modal/Singularity backends** (only runs on `local` and `ssh`). Choosing a container backend for "isolation" turns off the one built-in safety check. Use the `local` backend to keep it active. YOLO mode (`--yolo`, `/yolo`, an env-var flag) disables all approval except a hardline blocklist: never enable it on an exposed box.

- **One bot token = one poller (Telegram 409 Conflict).** Two processes long-polling the same bot fight over messages / throw 409. You cannot share a bot between an always-on Hermes instance and another consumer: give the always-on agent its own dedicated bot. Sending is unlimited; only receiving/`getUpdates` is exclusive.

- **A VPS marketplace image does NOT install your SSH key.** Terminal `ssh <user>@<ip>` returns publickey-denied on a fresh box. Bootstrap once via the provider's web console: append your ed25519 pubkey to the agent user's `~/.ssh/authorized_keys` (700 dir / 600 file / correct ownership). After that, terminal SSH works.

- **`su - <agent user>` prompts for a password that was never set.** The dedicated agent user has no password on the marketplace image. Don't `su`; SSH in directly as that user, or from root use `sudo -iu <user>`.

- **No hard dollar budget cap and no real audit trail** in Hermes. Only max-turns, reasoning-effort caps, retry limits, and cheap-model routing exist. Enforce spend limits upstream at the provider; build your own action log if you need auditability.

- **The WhatsApp connector is an UNOFFICIAL Baileys bridge, not the Business API.** Hermes' own docs: "This works by emulating a WhatsApp Web session, not through the official WhatsApp Business API." QR-paired like WhatsApp Web. The vendor acknowledges the risk themselves: "WhatsApp does not officially support third-party bots outside the Business API. Using a third-party bridge carries a small risk of account restrictions." Their stated mitigations include not automating outbound messages to non-initiators, which directly conflicts with any proactive agent sitting in a customer-facing group. Group support is not documented anywhere: Baileys itself speaks the full protocol so groups very likely work, but Hermes does not claim it, so test before selling it. Always give the bot a dedicated number, never a business' primary line, and back up the Baileys auth state (losing it means a human physically re-scans a QR code).

- **Memory files are HARD-CAPPED and do NOT auto-compact.** `MEMORY.md` caps at roughly 2,200 characters (~800 tokens) and `USER.md` at roughly 1,375 (~500 tokens). They are injected into the system prompt as a frozen snapshot at session start, not queried. When a write would exceed the cap the tool returns an error and the agent must consolidate entries itself before retrying. Consequence: a domain knowledge base cannot live in memory. Anything larger than a few facts belongs in the Skills System (on-demand knowledge documents, progressive disclosure, agentskills.io standard), which is also attachable to cron jobs. Budget ongoing human curation of the memory files or the agent eventually cannot learn anything new.

- **Never point two agent processes at the same Hermes home directory.** Memory is per-agent, not global, and concurrent writes corrupt state. Docs say so explicitly. Multi-agent setups need a separate home per agent plus an external memory provider (Honcho, Mem0, etc.) or a shared external store if they must share knowledge. Relevant when splitting an internal-facing agent from a customer-facing one for risk reasons.

- **Built-in cron is more capable than it looks, so don't reach for an external scheduler by reflex.** "Schedule tasks to run automatically with natural language or cron expressions. Jobs can attach skills, deliver results to any platform, and support pause/resume/edit operations." That covers agent-shaped recurring work (briefings, check-ins) with no extra infrastructure. But do not run deterministic, money-critical pipelines through it: waking an LLM to execute code with a known right answer burns tokens for nothing and gives you no per-item retries, idempotency, or audit trail. Correct split: deterministic code in a real orchestrator (e.g. Trigger.dev), which filters and only escalates genuine exceptions to the agent.

## Conclusions / best practices

- **Model routing:** run a cheap main model for the always-on loop; escalate to a stronger model only for hard subtasks via a task-delegation override or per-skill frontmatter. Pin auxiliary (vision/compression) work to cheap models.
- **Least privilege on exposed boxes:** use a blank-slate setup, skip browser/image/TTS tools you don't need, manual approval mode, `local` backend, YOLO off, firewall inbound to SSH-only.
- **Persistence:** the gateway runs as a `systemd --user` service with linger enabled, so it survives logout and reboot. Check with `hermes gateway status`.
- **Shared brain across machines:** SOUL.md + memories/*.md are plain markdown, so they are git-syncable. Do NOT git the live `state.db` (WAL SQLite): use `hermes backup` for it, and keep session DBs local per host.
- **Secrets:** API keys/tokens go in `~/.hermes/.env` (mode 0600), referenced from `config.yaml` via `${VAR}`. Never in startup scripts (cloud providers often store those in readable instance metadata) and never pasted into chat.
- **Instance sizing:** 2GB RAM is the bare floor; 4GB is comfortable (Python + Node + SQLite together). Browser automation needs 2GB or more on its own.
