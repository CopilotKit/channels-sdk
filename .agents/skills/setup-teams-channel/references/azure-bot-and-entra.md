# The Azure Bot, its Entra app, and the permissions

This phase is **entirely browser work, in the developer's own signed-in session**.
Two of its steps require privileges the developer may not have — see Phase 0 in
`SKILL.md` and confirm both *before* creating anything, because both fail late.

## Navigate by goal, not by remembered labels

The Azure and Entra portals reorganise often, and the Intelligence Teams setup
step is not covered by a published walkthrough. So you cannot pre-load the UI's
labels, and you must not invent them.

Work by goal, and for each step: **read the page, state what you are about to
change, get an explicit yes, then act.** Creating an Azure resource, minting a
client secret, and granting a tenant-wide Graph permission are consequential
mutations in a live account and, for the third, a tenant-wide one. Never click a
control you have not read.

If a goal has no obvious control on the page, say so and ask the developer what
they see. That is faster and safer than guessing.

## Step 1 — Create the Azure Bot

At `portal.azure.com`, create an **Azure Bot** for the Channel.

Two choices are not defaults to weigh — the Intelligence Teams adapter expects
both:

- **Single-tenant** Microsoft Entra app.
- **Client-secret** authentication (not managed identity, not certificate).

Creating the Azure Bot creates or attaches an Entra app registration. That app is
the identity Teams and Intelligence both authenticate against, so keep track of
which app registration belongs to this bot — a tenant with several bots will have
several similarly named apps.

**If there is no Azure subscription, stop here.** An Azure Bot is a billed Azure
resource. Directory or tenant ownership does not supply one; Azure subscriptions
are a separate billing artifact from Entra roles. Say so plainly and let the
developer resolve access rather than looking for a way around it.

## Step 2 — Point the bot at Intelligence

In the bot's **Configuration**:

1. Set **Messaging endpoint** to the Teams messaging endpoint shown in the
   Intelligence dashboard's Teams setup step. It sits on the path
   `/api/channels/adapters/teams/messages`. **Copy it from the dashboard** rather
   than composing it by hand — the host is environment-specific, and a wrong
   endpoint means no Teams activity ever reaches Intelligence, with no error
   anywhere in your own logs.
2. Enable the **Microsoft Teams** channel on the bot.

Both are required. A bot with the right endpoint and no Teams channel enabled
looks configured and delivers nothing.

## Step 3 — Collect the three credentials

Intelligence needs exactly three values:

| Value | Where it comes from |
| --- | --- |
| **Client ID** — Application (client) ID | The Entra app registration overview |
| **Tenant ID** — Directory (tenant) ID | The same overview |
| **Client secret** | Created under **Certificates & secrets** on that app |

From the bot's **Microsoft App ID**, choose **Manage** to reach the app
registration. Create the client secret there and copy its **value**, not its ID —
the value is shown once and is unrecoverable afterwards.

The Client ID is also the **App ID embedded in the Teams app package**, so the
package must be generated after this app exists, not before.

**All three are write-only in Intelligence after setup.** They cannot be read
back, only replaced. Have the developer record them in their own secret store
*before* submitting the form.

**Never ask the developer to paste the client secret into the conversation.** It
goes into the Intelligence dashboard and their `.env`, typed by them.

## Step 4 — Microsoft Graph file access, and admin consent

On the Entra app registration, add the Microsoft Graph **application**
permission `Files.ReadWrite.All`, then select **Grant admin consent**.

This is a **broad, tenant-wide application permission**, and Intelligence
requires it for supported Teams channel file reads and writes. Two consequences
worth stating to the developer before they act:

- It is not scoped to one team or one channel. Say that out loud — a developer
  who has not been told will reasonably assume it is narrower.
- **Granting it needs an admin role**: Global Administrator, Privileged Role
  Administrator, or Cloud Application Administrator. Adding the permission
  without consenting leaves the app in a "not granted" state that produces file
  operation failures later, far from this step.

Do not grant admin consent on the developer's behalf even if their session could.
Tenant-wide consent is theirs to give.

If `Roles & admins` shows a blank "Your Role:" and the role was assigned
recently, have them sign out and back in — Entra role assignments do not appear
until the session holds a fresh token.

## Step 5 — Channel message access is granted at install, not here

The generated Teams app package requests the Teams **resource-specific consent**
permission `ChannelMessage.Read.Group`. This one is **not** granted in Entra — it
is consented when the app is installed into a team.

- **Granted:** the Channel receives ambient, unmentioned team channel messages.
- **Not granted:** the bot receives only what Teams routes directly to it —
  mentions and personal chats.

This is the most misleading state in the Teams path: a bot that answers
`@mentions` and ignores everything else is a **correctly installed bot without
that consent**, not a broken one. Check the consent before diagnosing delivery.

## Step 6 — The reaction preview dependency

Microsoft's Teams bot reaction API is a required **preview** dependency, and the
tenant must permit it. If Teams rejects a reaction, the Channel reports that
through its existing delivery health rather than pretending support exists.

Do not treat a rejected reaction as evidence that the whole adapter is
misconfigured — check the other gates first.

## What this phase does and does not prove

Finishing every step here proves the **provider half** is wired. It says nothing
about whether the code half is running: a Channel whose runtime never connects
sits at `setup_required` or "Waiting for runtime" while every Azure and Entra
screen above looks correct and green.

That is why `SKILL.md` defines done as three verified gates, and why "the runtime
started" is not one of them.
