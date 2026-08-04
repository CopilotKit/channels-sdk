---
name: setup-teams-channel
description: Use for the PROVIDER half of getting a locally running CopilotKit Channels agent to answer in Microsoft Teams, when no Teams app exists yet — registering the bot and its Entra app, setting the Teams messaging endpoint, attaching the Teams adapter to a managed Intelligence Channel, building and sideloading the Teams app package, or when a Teams adapter reports setup_failed, a Channel reports setup_required, sits at "Waiting for runtime", is Online but a Teams mention gets no reply, or the bot answers mentions but never sees ordinary channel messages. If the Teams app and Channel already exist and the question is about declaring or customising the Channel in code, use the copilotkit-channels skill instead. For Slack, use setup-slack-channel.
---

# Set up a Teams Channel for a local Channels agent

Take a developer from a code checkout to a working local Microsoft Teams agent.

**Read Phase 0 before you plan anything.** Two checks fail *late* — after a
Channel exists and credentials are in — and one of them cannot be cleared by the
developer at all. Checking first turns a half-built setup into a two-minute
conversation.

**Do not use `portal.azure.com` for this.** The Intelligence dashboard's Teams
step links to it, but the Azure route needs a billed Azure subscription that most
developers do not have. The **Teams Developer Portal** creates the same bot, the
same Entra app registration, the same client secret, and the same messaging
endpoint — from a single form field, with no subscription. See step 1.

| System | Who owns it | Where you work on it |
| --- | --- | --- |
| Bot + its Entra app registration | The developer | dev.teams.microsoft.com |
| Tenant admin consent for Graph permissions | **A tenant admin, not the developer** | entra.microsoft.com |
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

**1. The ability to grant tenant-wide admin consent.** Step 4 adds the Microsoft
Graph **application** permission `Files.ReadWrite.All` and requires **Grant admin
consent**. That needs Global Administrator, Privileged Role Administrator, or
Cloud Application Administrator. `entra.microsoft.com` → Roles & admins shows
"Your Role:" — if it is blank, the developer cannot do this step.

This is the gate that actually bites, and it bites **late**: every earlier step
succeeds, the Channel gets created, and only the adapter attach fails. Verified
against a real tenant — see "When the adapter reports `teams_setup_failed`".

Entra role assignments do not appear until the session has a fresh token. If a
role was granted moments ago and still reads blank, have the developer sign out
and back in before concluding they lack it.

**2. A free Channel slot in the Intelligence org.** The Channels page shows
`N of M used`. If the org is at its cap, **no project will let you create a
Channel** — the cap is org-wide, not per-project, so creating a new project does
not help. It comes from the license claim `managed_channels.max_channels`, so
raising it is an internal entitlement change rather than anything to do with
Microsoft. Caps vary by org and change over time; read the counter rather than
assuming a value.

Two traps. A capped `Create channel` button looks *identical* to a disabled one —
greyed, silently doing nothing — yet carries no `disabled` attribute, so the
accessibility tree will not tell you either. **Read the `N of M used` counter, not
the button.** And if the org already has a working Channel for another provider,
deleting it to make room **trades one working setup for another**; say that out
loud before anyone deletes anything, because the provider credentials on the
deleted side are write-only and have to be re-issued to rebuild.

Two things that are commonly assumed to be blockers and usually are not:
registering an Entra app is governed by the tenant setting *Users can register
applications*, which is frequently `Yes` for all members; and sideloading a
custom Teams app needs **Upload a custom app** to be present under Teams →
Apps → Manage your apps → Upload an app, which is a per-tenant app-setup policy
independent of any admin role.

**If a Teams bot already works in this tenant, ask whoever built it whether they
granted the Graph consent.** That single answer tells you whether gate 1 is a hard
requirement or something the dashboard asks for but delivery does not need. Do not
reuse or modify their app — see the prohibitions — but its owner is the right
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
- **The bot's Endpoint address must be the Intelligence URL**, taken from the
  dashboard's Teams setup step. Copy it from there rather than composing it by
  hand — it is environment-specific, and a wrong endpoint means no activity ever
  reaches Intelligence.

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
sequence. Full detail in `references/bot-registration-and-permissions.md`.

The dashboard's step 1 offers an **Open Azure portal** button. Ignore it — the
Developer Portal route below produces the same result without a subscription.

1. **Create the bot** at `dev.teams.microsoft.com` → **Tools → Bot management →
   New bot**. It asks for a name and nothing else: no subscription, no resource
   group, no region, no pricing tier. Creating it also creates the matching Entra
   app registration, and **the bot's ID is that app's Application (client) ID** —
   the same value step 3 wants. Microsoft Teams is enabled on the bot by default.
2. **Set the messaging endpoint.** On the bot's **Configure** page, put the
   **Teams messaging endpoint** copied from the dashboard into **Endpoint
   address** and save. Confirm **Channels → Microsoft Teams** is checked.
3. **Paste the bot credentials** into Intelligence — Client ID (the bot ID),
   Tenant ID (Entra → Overview → Tenant ID), and a client secret created under
   the bot's **Client secrets → Create your first client secret**. The Client ID
   also becomes the App ID in the Teams package.
4. **Grant Microsoft Graph file access** — application permission
   `Files.ReadWrite.All`, then **Grant admin consent**. This is a broad
   tenant-wide permission. This is the step Phase 0 gate 1 exists for, and a
   Developer-Portal-created app starts with **no** API permissions at all, so
   this is an addition, not a confirmation.
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

## When the adapter reports `teams_setup_failed`

The Channel is created but its Teams adapter shows **Setup failed**, with the
failure code `teams_setup_failed` and the message *"Reconnect the Teams app and
complete setup again to restore delivery."*

**That message points at the wrong thing.** It reads as though the Teams app or
its install is broken. In practice the attach validation failed, and the app is
usually fine. Work the causes in this order — cheapest and most-likely-wrong
first:

| Check | How | Ruled out when |
| --- | --- | --- |
| Client ID matches the app registration | Entra → App registrations → Owned applications | The listed Application (client) ID equals the bot ID you pasted |
| Teams enabled on the bot | Developer Portal → bot → **Channels** | Microsoft Teams is checked |
| Messaging endpoint saved | Developer Portal → bot → **Configure** | Endpoint address holds the dashboard URL and saved cleanly |
| **Graph permission and consent** | Entra → app → **API permissions** | `Files.ReadWrite.All` is listed **and** shows granted |
| Client secret | Cannot be read back | Only re-issuable — mint a fresh one and reconnect |

In a real tenant the first three all passed and **API permissions was completely
empty**, with **Grant admin consent** greyed out for a non-admin. So an empty
permissions list plus a blank "Your Role:" is the signature of Phase 0 gate 1
having been skipped — and the adapter is where it finally surfaces.

Note that adding the permission is something any member can do; only the consent
needs the role. Adding it without consenting does not clear the failure.

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
bot registration and Entra app; a local runtime and agent.

**Out of scope. These are hard limits, not defaults to weigh:**

- **Do not reuse, reinstall, or modify a bot registration, Entra app, or Teams app
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

**Partially verified by a real run, and honest about which parts.**

Verified by driving a real tenant end to end on 2026-08-03:

- The Developer Portal route in steps 1–3 — bot created from one field, Entra app
  registration produced, bot ID equal to the app's client ID, Teams enabled by
  default, endpoint saved, secrets available. Done in a tenant with **zero** Azure
  subscriptions, which is what disproves the Azure requirement.
- The messaging endpoint path, copied from the dashboard's own control.
- The Channel code being exactly what `createChannel({ name })` must declare,
  read from the wizard's Review step.
- The org-wide Channel cap, and that a capped button is indistinguishable from a
  disabled one.
- `teams_setup_failed` with an empty API-permissions list and consent unavailable
  to a non-admin.

**Not verified:** a completed install answering a real mention. The admin-consent
gate stopped the run short of gates 1 and 3 of "Done means three things." So the
`ready()` / `setup_required` trap, the `ChannelMessage.Read.Group` behaviour split
and the reaction preview dependency are still read from source rather than
observed, and the prohibitions have not been tested against an agent under
pressure the way `setup-slack-channel`'s were.

If what you observe contradicts this skill, trust the dashboard and the installed
packages, and do not argue with the runtime.
