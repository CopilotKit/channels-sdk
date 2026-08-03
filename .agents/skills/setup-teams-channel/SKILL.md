---
name: setup-teams-channel
description: Use for the PROVIDER half of getting a locally running CopilotKit Channels agent to answer in Microsoft Teams, when no Teams app exists yet — creating the Azure Bot and its Entra app registration, setting the Teams messaging endpoint, attaching the Teams adapter to a managed Intelligence Channel, building and sideloading the Teams app package, or when a Teams Channel reports setup_required, sits at "Waiting for runtime", is Online but a Teams mention gets no reply, or the bot answers mentions but never sees ordinary channel messages. If the Teams app and Channel already exist and the question is about declaring or customising the Channel in code, use the copilotkit-channels skill instead. For Slack, use setup-slack-channel.
---

# Set up a Teams Channel for a local Channels agent

Take a developer from a code checkout to a working local Microsoft Teams agent.

**Read Phase 0 before you plan anything.** Two of the seven steps need
privileges that most developers in most tenants do not have, and both fail
*late* — after you have already created an Entra app and pasted credentials.
Checking first turns a half-built app into a two-minute conversation.

| System | Who owns it | Where you work on it |
| --- | --- | --- |
| Azure subscription | Whoever holds Azure billing | portal.azure.com |
| Azure Bot + its Entra app registration | The developer | portal.azure.com, entra.microsoft.com |
| Tenant admin consent for Graph permissions | A tenant admin | entra.microsoft.com |
| Intelligence project, API key, Channel, Teams adapter | The developer | The Intelligence dashboard |
| Teams app package + install | The developer | dev.teams.microsoft.com, then Teams |
| Local Channels runtime + AG-UI agent | The developer | This repo, in the shell |

Note the column: **almost all of this is browser work.** Do it in the
developer's own signed-in session, and for each consequential change — creating
a resource, granting a tenant-wide permission, installing an app — state what
you are about to do and get an explicit yes first.

## Phase 0 — Check access before you build anything

Ask the developer to confirm these two, or check them yourself if you can drive
their browser. **If either is missing, stop and say so.** Do not start
registering an app you cannot finish.

**1. An Azure subscription in the same directory as the Teams tenant.**
`portal.azure.com` → Subscriptions. An Azure Bot is a billed Azure resource;
with zero subscriptions there is nowhere to create it. Directory or tenant
*ownership* does not satisfy this — Azure subscriptions are a separate billing
artifact from Entra roles, so a Global Administrator with no subscription is
still blocked here.

**2. The ability to grant tenant-wide admin consent.** Step 4 adds the Microsoft
Graph **application** permission `Files.ReadWrite.All` and requires **Grant admin
consent**. That needs Global Administrator, Privileged Role Administrator, or
Cloud Application Administrator. `entra.microsoft.com` → Roles & admins shows
"Your Role:" — if it is blank, the developer cannot do this step.

Entra role assignments do not appear until the session has a fresh token. If a
role was granted moments ago and still reads blank, have the developer sign out
and back in before concluding they lack it.

Two things that are commonly assumed to be blockers and usually are not:
registering an Entra app is governed by the tenant setting *Users can register
applications*, which is frequently `Yes` for all members; and sideloading a
custom Teams app needs **Upload a custom app** to be present under Teams →
Apps → Manage your apps → Upload an app, which is a per-tenant app-setup policy
independent of any admin role.

**If a Teams bot already works in this tenant, find out who built it before
building a second one.** Its Azure Bot resource lives in some subscription, and
whoever owns that is a faster unblock than provisioning new billing. Do not
reuse or modify that app — see the prohibitions — but its owner is the right
person to ask.

## How delivery actually works — two legs, two mechanisms

Getting this wrong is the most expensive mistake here, because a misconfigured
Teams app installs cleanly and answers nothing.

| Leg | Mechanism | What authenticates it |
| --- | --- | --- |
| **Teams → Intelligence** | Teams posts activities over **HTTPS** to an Intelligence-hosted messaging endpoint on the path `/api/channels/adapters/teams/messages` | The Entra app's client ID, tenant ID, and client secret, held by Intelligence |
| **Intelligence → your runtime** | Your runtime dials **out** to the realtime gateway over a websocket | `INTELLIGENCE_API_KEY` |

Two consequences:

- **No tunnel and no public URL of your own is needed.** *Intelligence* owns the
  public messaging endpoint, and the second leg is outbound from the developer's
  machine.
- **The Azure Bot's Messaging endpoint must be the Intelligence URL**, taken
  from the dashboard's Teams setup step. Copy it from there rather than
  composing it by hand — it is environment-specific, and a wrong endpoint means
  no activity ever reaches Intelligence.

The Teams adapter form in Intelligence asks for exactly three values: **Client
ID**, **Tenant ID**, and **Client secret**. All three are write-only after
setup — they cannot be read back, only replaced. Record them where the developer
keeps secrets *before* submitting.

## Done means three things, all verified

Do not report success until **all three** hold. Any one alone is a false positive.

1. **The Teams app is installed**, and the bot has been added to the team or
   personal chat you will test in.
2. **The managed Channel reports `online`** — from `controls.status()` in the
   process, or Online in the dashboard. Not "the runtime started."
3. **A real human mention got a real reply** in Teams.

Gate 2 is where agents fail. `await controls.ready()` resolves on
`setup_required` too — that state is a valid degraded state, not a failure. A
runtime with **no Teams connection at all** starts cleanly, prints its listening
line, returns HTTP 200 on `/api/copilotkit/info`, and answers nothing.
`/api/copilotkit/info` reports license and runtime info, **not** channel state,
so a 200 there is not evidence of anything Teams-related.

## The seven steps, in order

Work through the Intelligence dashboard's Teams setup, which drives this
sequence. Full detail in `references/azure-bot-and-entra.md`.

1. **Create the Azure Bot** for your Channel, with a **single-tenant** Microsoft
   Entra app and **client-secret** authentication.
2. **Configure the bot.** Set its **Messaging endpoint** to the Intelligence URL
   from the dashboard, then enable the **Microsoft Teams** channel on the bot.
3. **Paste the bot credentials** into Intelligence — Client ID, Tenant ID, and a
   client secret created under **Certificates & secrets**. The Client ID also
   becomes the App ID in the Teams package.
4. **Grant Microsoft Graph file access** — application permission
   `Files.ReadWrite.All`, then **Grant admin consent**. This is a broad
   tenant-wide permission and is required for supported Teams channel file reads
   and writes. This is the step Phase 0 gate 2 exists for.
5. **Review channel message access.** The generated package requests the Teams
   resource-specific consent permission `ChannelMessage.Read.Group`. See below —
   this one changes behaviour in a way that looks like a bug.
6. **Confirm Teams reaction support.** Microsoft's Teams bot reaction API is a
   required *preview* dependency; the tenant must allow it. If Teams rejects a
   reaction, the Channel surfaces that through delivery health rather than
   implying support.
7. **Install the app in Teams.** Download the generated app package (`.zip`),
   upload it via Teams → Apps → Manage your apps → **Upload a custom app**, then
   add the bot to a team or personal chat.

## The symptom that is not a bug

`ChannelMessage.Read.Group` is resource-specific consent, granted **when the app
is installed**, not in the Entra portal.

- **Granted:** the Channel receives ambient, unmentioned team channel messages.
- **Not granted:** the bot receives only what Teams routes directly to it —
  mentions and personal chats.

So "the bot replies when I @-mention it but ignores everything else" is the
expected shape of a working install without that consent. Check the consent
before treating it as a delivery failure.

## Scope — read before planning

**In scope:** production CopilotKit Intelligence; a managed Channel; a dedicated
Azure Bot and Entra app; a local runtime and agent.

**Out of scope. These are hard limits, not defaults to weigh:**

- **Do not reuse, reinstall, or modify an Azure Bot, Entra app, or Teams app
  that is already installed and in use.** Create a dedicated one. Rotating a
  secret on a shared app breaks whatever else uses it.
- **Do not switch to a self-hosted provider adapter** to avoid a blocked step.
  If Phase 0 fails, the answer is to get access, not to change products.
- **Never ask the developer to paste a client secret into the conversation.** It
  goes into the Intelligence dashboard and their own secret store, typed by them.
- Do not grant Graph permissions beyond `Files.ReadWrite.All`, and do not grant
  admin consent on the developer's behalf even if you can.
- Do not deploy anything, and do not target internal or dev Intelligence
  environments.

## Version provenance and what is unverified

The setup sequence, the three credential fields, the `/api/channels/adapters/teams/messages`
path, the `Files.ReadWrite.All` consent requirement, the
`ChannelMessage.Read.Group` behaviour split, and the reaction preview dependency
were all read from the shipped Teams setup wizard on CopilotKit Intelligence
`main` as of 2026-08-03. The `ready()` / `setup_required` trap is shared with the
Slack path and verified against `@copilotkit/channels@0.6.0` and
`@copilotkit/runtime@1.65.0`.

**Not yet verified by a real end-to-end run.** Unlike the Slack skill, whose
guardrails came from observed agent failures against a live setup, this skill has
not been exercised against a completed Teams install — the Azure subscription
gate blocked validation. Treat the *ordering* and *guardrails* as sound but
provisional: if what you observe contradicts this skill, trust the dashboard and
the installed packages, and do not argue with the runtime.
