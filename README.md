<div align="center">

# Channels SDK

<img src="./assets/hero-any-agent-any-channel.png" alt="Any agent. Any channel." width="820">

**Bring any AI agent into Slack, Microsoft Teams, and the channels where work happens — with native, interactive UI.**

[**Try Channels**](https://www.copilotkit.ai/try-channels) · [**Build with the SDK**](#build-your-first-channel) · [**Explore OpenTag**](#see-a-complete-channels-app)

[![npm](https://img.shields.io/npm/v/@copilotkit/channels.svg?label=%40copilotkit%2Fchannels)](https://www.npmjs.com/package/@copilotkit/channels)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)

</div>

https://github.com/user-attachments/assets/98300c69-d4c5-4367-b6a1-a191426d7605

<div align="center">

Your agent keeps its tools, model, and business logic. Channels gives it a native place to work with people.

</div>

## Your agent belongs where work happens

Channels connects an AG-UI-compatible agent to the communication platforms your team already uses. The agent can understand the conversation, stream a response, call tools, work with files, render interactive UI, and pause for human approval.

| Bring your agent                                                                                                       | Render native UI                                                                                                 | Keep people in control                                                                        |
| ---------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Use CopilotKit's built-in agent or connect LangGraph, CrewAI, Mastra, Pydantic AI, Google ADK, and other AG-UI agents. | Describe a message once and render it as native Slack Block Kit, Teams Adaptive Cards, and platform-specific UI. | Put buttons, choices, and approval gates directly into the conversation before an agent acts. |

### One interaction, native to every channel

| Slack                                                                                                  | Microsoft Teams                                                                                                  | Discord                                                                                                   |
| ------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| <img src="./assets/demo-slack.png" alt="An agent triages a bug report and asks for approval in Slack"> | <img src="./assets/demo-teams.png" alt="An agent analyzes a spreadsheet and returns metrics in Microsoft Teams"> | <img src="./assets/demo-discord.png" alt="An agent reads deployment logs and renders a chart in Discord"> |

Channels is built for a world where the same agent can meet users across every communication surface. Managed connections for Slack and Microsoft Teams are available through CopilotKit Intelligence, with more channels on the way.

## Try it before you build it

Experience a real Channels agent in Slack or Microsoft Teams without configuring an app, runtime, or provider credentials.

### [Try Channels →](https://www.copilotkit.ai/try-channels)

Choose a platform, join the experience, and see how an agent handles context, tool use, and native channel UI.

## Build your first Channel

Your agent and application logic run in your infrastructure. CopilotKit Intelligence manages the platform connection and delivers each turn to your long-running Channels process.

### Fastest path for Slack: install the skill and let your coding agent drive

Slack setup spans a Slack app, a workspace install, a managed Channel, and a local runtime. Install the `setup-slack-channel` skill and your coding agent walks all of it with you:

```sh
npx copilotkit@latest skills install --skill setup-slack-channel
```

Or install every CopilotKit skill at once:

```sh
npx copilotkit@latest skills install
```

Then ask your agent to set up Slack. The steps below are the same path, done by hand.

> **`Unknown option '--skill'`?** An older `copilotkit` — globally installed or left in the npx cache — is shadowing the current CLI. Keep the `@latest`; that is what forces npx to fetch the current version instead of reusing what it already has.

<details>
<summary><b>Copy-paste prompt: new project → answering in Slack</b></summary>

Run `npx copilotkit@latest skills install` in the directory where you want the project, then paste this into your coding agent. It scaffolds a new CopilotKit app and takes it all the way to a real Slack channel.

```text
Help me create a new CopilotKit app from scratch, then connect its agent to Slack so it answers in a real channel.

Use these skills as your instructions, in this order:
- `copilotkit-setup` for creating the app and wiring managed Intelligence.
- `copilotkit-channels` for the CLI-scaffolded Channel and its long-running host.
- `setup-slack-channel` for the Slack app and managed Intelligence provider setup.

Read all three before you plan anything, and follow them rather than working from memory. If any skill is not installed, stop and tell me to run `npx copilotkit@latest skills install` in the directory where I want the project, then wait for me. Do not attempt the setup without them.

Use the CopilotKit CLI through `npx copilotkit@latest`; do not assume or require a global CLI installation. Before you start, verify that Node.js 20 or newer is available. If it is not, stop and tell me what I need to install.

When you implement the plan:
- Run `npx copilotkit@latest create` to scaffold a brand-new project. Ask me for the project name and framework instead of choosing them for me.
- Choose managed Intelligence because Channels are not available in self-hosted SSE mode. Follow the CLI's current prompts rather than guessing undocumented flags.
- Treat the CLI-generated `channel-host.mts` and `channels.mts` as the code half. Do not add a second Channel declaration or move the Channel into a serverless route.
- Use `setup-slack-channel` for the provider half, including the dedicated Slack app, managed Channel, credentials, and real workspace verification. Do not replace the managed path with a direct Slack adapter.

Before your first step, tell me which mode you are in:
- You can drive a browser yourself. Then do so, and confirm with me before each consequential change in a live account — signing in, creating an Intelligence project, creating the Slack app, attaching the adapter, or issuing a key.
- You cannot. Then say so plainly and walk me through the browser steps one at a time, waiting for me to confirm each before you continue.

Do not guess which one you are. Check what tools you actually have.

Rules for the whole run:
- Never ask me to paste a token, signing secret, or API key into this conversation. They go into the Intelligence dashboard and my `.env`, typed by me.
- Do not report success until all three gates in `setup-slack-channel` hold. "The runtime started" is not one of them.

Start by reading the three skills, then tell me which are available, what you found in the current directory, which browser mode you are in, and what you plan to do.
```

Already have a project? Use the **Build it with your coding agent** prompt at the end of this section instead — it adds a Channels listener to code you already have.

</details>

### 1. Configure the connection

[Create a Channel in CopilotKit Intelligence](https://docs.copilotkit.ai/channels) and connect Slack. Keep the Channel **Code** and project-scoped Intelligence API key for the next steps.

You need Node.js 22 or later and a long-running Node process or container.

### 2. Install the SDK

```sh
npm install @copilotkit/channels @copilotkit/runtime
npm install --save-dev tsx typescript @types/node
npm pkg set type=module
```

Channels and Runtime ship together as a tested pair. Upgrade both packages together.

### 3. Create the listener

The example below uses CopilotKit's built-in agent. Replace `makeAgent` with any AG-UI-compatible agent factory without changing the Channel lifecycle.

```ts
// channel.ts
import { createServer } from "node:http";
import { createChannel } from "@copilotkit/channels";
import {
  BuiltInAgent,
  CopilotKitIntelligence,
  CopilotRuntime,
} from "@copilotkit/runtime/v2";
import { createCopilotNodeListener } from "@copilotkit/runtime/v2/node";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing required environment variable: ${name}`);
  return value;
}

function makeAgent(threadId: string) {
  const agent = new BuiltInAgent({ model: "openai:gpt-5.4-mini" });
  agent.threadId = threadId;
  return agent;
}

const channel = createChannel({
  name: required("CHANNEL_CODE"),
  identifyUser: "platform",
  agent: makeAgent,
});

channel.onMessage(async ({ thread, message }) => {
  await thread.runAgent({
    prompt: message.contentParts?.length
      ? [
          ...(message.text
            ? [{ type: "text" as const, text: message.text }]
            : []),
          ...message.contentParts,
        ]
      : message.text,
    context: [{ description: "Originating platform", value: message.platform }],
  });
});

const intelligence = new CopilotKitIntelligence({
  apiKey: required("INTELLIGENCE_API_KEY"),
});

const runtime = new CopilotRuntime({
  agents: {},
  intelligence,
  identifyUser: () => ({
    id: "channels-runtime",
    name: "Channels Runtime",
  }),
  channels: [channel],
});

const listener = createCopilotNodeListener({
  runtime,
  basePath: "/api/copilotkit",
});

const channels = listener.channels;
if (!channels) throw new Error("Channels control surface was not created.");

const server = createServer(listener);
const shutdown = async () => {
  await channels.stop();
  if (server.listening) server.close();
};
process.once("SIGINT", shutdown);
process.once("SIGTERM", shutdown);

await channels.ready({ timeoutMs: 30_000 });

const status = channels.status();
if (status.overall !== "online") {
  throw new Error(`Channel is not online: ${JSON.stringify(status)}`);
}

const port = Number(process.env.PORT ?? 3000);
server.listen(port, () => {
  console.log(`Channel online; lifecycle server listening on :${port}`);
});
```

### 4. Start it

```dotenv
# .env
OPENAI_API_KEY=<openai-api-key>
INTELLIGENCE_API_KEY=<project-api-key>
CHANNEL_CODE=<channel-code-from-intelligence>
PORT=3000
```

```sh
node --env-file=.env --import tsx channel.ts
```

When Intelligence reports **Online**, invite the app to a Slack channel and mention it. Your agent now receives the conversation and responds in the thread.

> Want Microsoft Teams, a different agent framework, interactive approvals, files, or production deployment guidance? Continue in the [Channels documentation](https://docs.copilotkit.ai/channels).

<details>
<summary><b>Build it with your coding agent</b></summary>

Copy this prompt into your coding agent:

```text
Add a managed CopilotKit Channels listener to this project.

Use @copilotkit/channels and @copilotkit/runtime with a
CopilotKitIntelligence connection. The process must create a Channel whose
name comes from CHANNEL_CODE, attach it to CopilotRuntime, create the
framework-appropriate runtime listener, call channels.ready(), verify
channels.status(), and stop Channels during graceful shutdown.

Connect my existing agent through an AG-UI-compatible agent factory. Keep the
agent, tools, and application logic in this project; CopilotKit Intelligence
owns the Slack or Microsoft Teams connection.

Use the current APIs and setup instructions from:
https://docs.copilotkit.ai/channels
```

</details>

## How it works

<img src="./assets/architecture.png" alt="Channels architecture connecting any agent through CopilotKit and AG-UI to communication platforms" width="820">

Every turn follows the same path:

1. A person messages your app in Slack or Microsoft Teams.
2. CopilotKit Intelligence receives the platform event and delivers it to your Channels process.
3. Channels runs your agent over AG-UI, executes tools, and renders the result.
4. Intelligence sends native platform UI back into the conversation.

| You run                                                  | CopilotKit Intelligence manages                |
| -------------------------------------------------------- | ---------------------------------------------- |
| Your agent, model credentials, tools, and business logic | Slack and Microsoft Teams platform credentials |
| The long-running Channels listener                       | Platform ingress and credentialed delivery     |
| Application state, deployment, and logs                  | Runtime registration, health, and reconnects   |

The SDK is open source and MIT licensed. CopilotKit Intelligence can be hosted by CopilotKit or self-hosted for enterprise deployments.

## See a complete Channels app

[**OpenTag**](https://github.com/CopilotKit/OpenTag) is an open-source, self-hosted on-call triage assistant built with Channels.

Use it to study a complete application with:

- a Python LangGraph agent connected over AG-UI
- native Slack and Microsoft Teams experiences
- file-aware prompts and generative UI
- human approval before Linear or Notion writes
- a production-shaped Node runtime and agent service

### [Explore the OpenTag source →](https://github.com/CopilotKit/OpenTag)

## Developer resources

| I want to…                        | Start here                                                                                            |
| --------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Experience Channels without setup | [Try Channels](https://www.copilotkit.ai/try-channels)                                                |
| Set up Slack with my coding agent | `npx copilotkit@latest skills install --skill setup-slack-channel`                                    |
| Build my first Channel            | [Channels documentation](https://docs.copilotkit.ai/channels)                                         |
| Inspect the SDK implementation    | [Channels source in CopilotKit](https://github.com/CopilotKit/CopilotKit/tree/main/packages/channels) |
| Install the package               | [`@copilotkit/channels` on npm](https://www.npmjs.com/package/@copilotkit/channels)                   |
| Study a complete application      | [OpenTag](https://github.com/CopilotKit/OpenTag)                                                      |
| Connect an existing agent         | [AG-UI integrations](https://docs.ag-ui.com/introduction)                                             |
| Understand the protocol           | [AG-UI](https://github.com/ag-ui-protocol/ag-ui)                                                      |

## License

[MIT](./LICENSE) © CopilotKit
