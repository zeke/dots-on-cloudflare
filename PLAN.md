# Plan

Planning notes for an always-on personal agent, similar to OpenAI dots, that runs on Cloudflare. Nothing is built yet. Claims about third-party products were checked in October 2026 and may be out of date.

## Goal

Give each person one long-lived agent that:

- Owns ongoing responsibilities and keeps working between conversations.
- Has its own sandboxed computer and browser.
- Connects to MCP servers and external tools like GitHub and the user's own Cloudflare account.
- Remembers preferences and learns from feedback.
- Can be reached the same way from web chat, a messaging app, or voice, with one shared memory.
- Asks before taking actions that write to the outside world.

There are two audiences, and the design should serve both from one core:

- Cloudflare employees, with access to internal systems behind Cloudflare's existing controls.
- The public, as an open source project anyone can deploy to their own Cloudflare account.

## What OpenAI dots does

From OpenAI's launch post and help center:

- One persistent agent per user, with a name and a handle like `@you-dot`.
- Runs GPT-6 Astra on its own cloud computer and browser.
- Works toward goals 24/7 and asks for input when a decision needs the user.
- Same dot, same memory across ChatGPT, Slack, Teams, and voice calls.
- Learns from ChatGPT memory and feedback.
- Connects to about 4,000 apps through plugins. The user decides what it can do alone and what needs approval.
- Included with ChatGPT Pro (from $100/month) and Business Premium. Not available on Pro in the EEA, UK, or Switzerland.

## Building blocks

| Dots feature                    | Cloudflare piece                                                     | Status                                      |
| ------------------------------- | -------------------------------------------------------------------- | ------------------------------------------- |
| Always-on agent per user        | Agent Durable Object keyed by user, sleeps when idle                 | GA                                          |
| Agent loop that survives crashes | Pi Durable + `PiHarness`, state in the object's SQLite storage      | Beta, Pi Durable marked experimental        |
| Alternative harness             | Think (memory, messengers, scheduled tasks, sub-agents)              | Agents SDK, compare before choosing         |
| Frontier models, one bill       | AI Gateway Unified Billing through the `env.AI` binding              | Beta, 5% fee on credits                     |
| Spend controls                  | AI Gateway spend limits                                              | Documented                                  |
| Its own computer                | Sandbox SDK (Linux container), or Dynamic Workers for JS-only work   | Documented                                  |
| Its own browser                 | Browser Run browser tool, keeps tabs and logins between turns        | Docs say it works with Pi Durable           |
| Keeps working 24/7              | Agent alarms and `schedule()`, plus `PiHarness` wake-up jobs         | GA / Beta                                   |
| Tool connections                | Agents SDK MCP client, MCP Server Portals for orgs                   | GA                                          |
| Who can use it                  | Cloudflare Access                                                    | GA                                          |
| Messaging channels              | Agents SDK channels: chat, Slack, email, webhooks                    | Documented                                  |
| Memory                          | Transcripts and summaries in SQLite, plus a user-editable memory file | To build                                    |
| Approvals                       | Queue writes for approval, reads go through                           | To build, or reuse Cloudflare OS Gatekeepers |

## Two paths

Path 1: standalone project on the Agents SDK with `PiHarness`.

- Small and fast to learn from. Works well as a public open source project.
- Has to rebuild a security model: scoped credentials, approvals, audit.

Path 2: a dot inside Cloudflare OS.

- Cloudflare OS is open source (Apache-2.0) and already used across Cloudflare.
- Already has scheduled tasks (`gatekeeper-scheduler`), inbound prompts into a workspace (`ExternalMessageGateway`), Gatekeepers for scoped access and write approval, MCP and MCP portal gatekeepers, GitHub and Cloudflare gatekeepers, AI Gateway billing, and push notifications.
- Missing the dot itself: one long-lived agent per user that owns responsibilities across workspaces.
- Harder to learn, but inherits Access, Gatekeepers, and an existing internal rollout.

Both are worth exploring. A reasonable order is a short path 1 spike to settle the agent loop and proactive behavior, then decide whether the real thing lives in Cloudflare OS, stays standalone, or both.

## MCP and external tools

The Agents SDK MCP client covers most of this:

- `this.addMcpServer(name, url)` connects to a remote MCP server. If the server needs OAuth, it returns an auth URL, and the connection is persisted in the agent's SQLite storage.
- Custom transport headers handle bearer tokens and Cloudflare Access service tokens.
- Code Mode for MCP adds progressive tool discovery and durable approvals, which matters once a user has connected many servers.

Specific servers:

| Target                   | Endpoint                               | Notes                                                                                                   |
| ------------------------ | -------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| GitHub                   | `https://api.githubcopilot.com/mcp/`   | Use a PAT as a bearer token. `mcp.github.com` returns Cloudflare error 1016 from Workers, even though it appears in the Agents SDK docs example. OAuth is not available from Workers. |
| User's Cloudflare account | `https://mcp.cloudflare.com/mcp`      | Cloudflare's main MCP server, OAuth                                                                      |
| Cloudflare docs          | `https://docs.mcp.cloudflare.com/mcp`  | No account access needed                                                                                 |
| Internal systems         | MCP Server Portal URL                  | Access decides who connects, Gateway logs traffic. Internal deployment only.                             |
| Anything else            | User-supplied URL                      | OAuth discovery, like Cloudflare OS `gatekeeper-mcp`                                                     |

Approval model to borrow from Cloudflare OS `gatekeeper-mcp`:

- Read-only tools run right away and are logged as observations.
- Everything else is queued for approval, and the agent collects the result afterward.
- Grants can cover a whole server or only named tools.
- Users can opt in to auto-approving specific action kinds.

Open questions:

- How does the dot tell read-only tools from write tools? MCP tool annotations like `readOnlyHint` exist, but servers may not set them reliably. The safe default is to treat unknown tools as writes.
- Where do OAuth tokens live: in the user's dot Durable Object, or in a separate credential service the agent can call but not read? Cloudflare OS keeps them in gatekeepers, outside the agent's reach.
- GitHub's PAT-only constraint from Workers is a poor experience for a public app. Is there another way, or is that just a setup step?

## Interfaces

All interfaces talk to the same dot Durable Object, so memory and responsibilities are shared.

Web chat:

- WebSocket to the Durable Object, which streams Pi session events.
- Panels for current responsibilities, a pending approvals queue, and the memory file.
- The first interface to build.

Messaging:

- Inside Cloudflare, the work chat app is Google Chat. Cloudflare OS has a Google Chat integration. The Agents SDK has Slack and email channels but no Google Chat channel.
- For the public version, Slack, email, and Discord are likely candidates.

Voice, not using Cloudflare's built-in voice features. Options from a quick survey:

| Option                  | Type                          | Notes                                                                 |
| ----------------------- | ----------------------------- | --------------------------------------------------------------------- |
| OpenAI Realtime API     | Hosted speech-to-speech model | Reported to support MCP servers and SIP phone calls in `gpt-realtime-2.1`. Not verified against OpenAI docs. |
| ElevenLabs Agents       | Hosted voice agent platform   | Tool calling, large voice library. Eleven Realtime speech-to-speech models launched October 2026. |
| Gemini Live             | Hosted speech-to-speech model | Function calling                                                      |
| Pipecat                 | Open source framework (Python, BSD-2) | Pluggable transports and providers                            |
| LiveKit Agents          | Open source framework (Apache-2.0) | Built on WebRTC, SIP for phone calls                             |

A design that keeps one brain:

- The voice layer is a thin, fast model that handles turn-taking and small talk.
- It has a few tools that call the dot: ask a question, assign a task, approve or reject a pending action, and read status.
- The dot does the real work in the background and can call or message back when it's done.
- This avoids two agents with different memories, at the cost of some latency on hard questions.

Other voice ideas to consider: a phone number for your dot over SIP, a push-to-talk button in web chat, and a daily spoken briefing.

## Open source vs internal

Keep one core and make deployment config the difference:

| Concern       | Public deployment                           | Cloudflare internal deployment                  |
| ------------- | ------------------------------------------- | ----------------------------------------------- |
| Auth          | Cloudflare Access with email OTP, or GitHub sign-in | Cloudflare Access with company SSO       |
| Models        | Unified Billing on the deployer's account, or BYOK | Approved models through the company AI Gateway |
| Tools         | User-supplied MCP servers, GitHub, Cloudflare | MCP Server Portal plus the same personal tools |
| Messaging     | Slack, email, Discord                       | Google Chat                                     |
| Policy        | User approves writes                        | Org policy on top: allowed tools, data rules    |

## Proof of concept scope

- Worker plus one `Dot` Durable Object per user, behind Access.
- Pi harness, with models through AI Gateway.
- Tools: web search, web fetch, sandbox exec, browser, and MCP servers.
- Responsibilities: the user gives a goal, the dot saves it and schedules check-ins, wakes up, works, and reports back.
- Memory: a `memory.md` stored in SQLite, read every turn and updated after feedback.
- Approvals: writes go through an approval queue shown in web chat.
- Web chat first. Voice and messaging come after.
- Out of scope for now: avatars, multiple dots per user, a large app catalog.

## Unknowns to check

- Whether internal Cloudflare accounts can use AI Gateway Unified Billing, and which frontier models are available through it.
- Pi Durable vs Think for this use case.
- Whether the Browser Run and Sandbox tools work with `PiHarness` without extra glue.
- How much work it would take to add a per-user dot to Cloudflare OS (read `overseer.ts` and the agent spawner).
- Cost per user per month for an always-on agent that mostly sleeps.

## References

- https://openai.com/index/introducing-dots/
- https://help.openai.com/en/articles/20001530-getting-started-with-your-dot
- https://developers.cloudflare.com/agents/harnesses/pi/
- https://developers.cloudflare.com/agents/tools/mcp/
- https://developers.cloudflare.com/ai-gateway/features/unified-billing/
- https://developers.cloudflare.com/cloudflare-one/access-controls/ai-controls/mcp-portals/
- https://earendil.com/posts/pi-durable-on-cloudflare/
- https://github.com/irvinebroque/pi-on-cloudflare
- https://github.com/cloudflare/cloudflare-os
- https://blog.cloudflare.com/cloudflare-os/
