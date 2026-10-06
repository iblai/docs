# Code Mode

<iframe src="https://www.youtube.com/embed/hGTsgflz8zg" title="Code Mode | Agentic OS | ibl.ai" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Overview

Code Mode is the coding agent built into the OS chat box. Its own description is the plainest one: *"an agentic coding tool that edits files, runs commands, and commits changes in the folder you choose."*

The **Code** chip sits in the message composer alongside Canvas, Prompts, and Memory (see [Chat Interface](chat.md)), so there is no separate IDE to open, no second tool to authenticate, and no context to re-explain to a different assistant. The conversation and the codebase are the same session.

What lets a single prompt go this far is the toolkit underneath rather than the model on top. Code Mode builds from the **Agentic Vibe** starter template, which already carries SSO authentication, AI chat, user profiles, notifications, and analytics, and it draws on ibl.ai skills that teach the agent how to build against the platform. The agent composes building blocks your organization already owns instead of inventing a stack from scratch, which is a different failure profile from a general-purpose model guessing at APIs.

Everything it produces is ordinary source in a folder you chose, building with your toolchain and deploying to your infrastructure.

## Target Audience

**User** | **Administrator**

## Approvals

Code Mode reads and writes files and runs commands on a real folder, so how it asks for permission is the most important setting on this page.

#### The first-run dialog
The first time you enable Code, it asks before doing anything: **"How should Code ask for approval?"** — *"Code can edit files and run commands in your workspace. Choose whether it checks with you first — you can change this later in the Code panel."* The two answers are **Ask me each time** and **Approve automatically**.

There is deliberately **no default**. The dialog exists precisely because either answer picks a security posture on your behalf, and that is a choice the product declines to make for you.

#### Ask Me
Under **Approvals → Ask Me**, Code stops before each operation and waits. The prompt is a card titled **"Code needs your permission"**, naming the kind of action it wants to take — **read**, **write**, **exec**, **delete**, **move**, **search**, **fetch**, or **think** — with **Allow** and **Deny** buttons.

This is the execution model rather than a courtesy dialog on the first call: the gate holds for every operation. Denying stops that action, not the conversation, so you can decline a step and redirect the agent.

#### Automatic
Under **Approvals → Automatic**, Code edits files and runs commands without asking. The panel states the trade-off directly: *"Code edits files and runs commands without asking. Use it only in folders you trust."*

The setting can be changed at any time from the **Approvals** control in the Code panel, and it is remembered per agent.

## Features

#### The Code chip
A composer chip, alongside Canvas, Skills, Prompts, Study Mode, Deep Research, and Memory, with the tooltip *"An agentic coding tool for your project folder."* An active tool shows as a highlighted chip with an **X** to turn it off; on a narrow composer, extra tools collapse into an overflow (`...`) menu.

#### Workspace
Code works inside a folder you nominate. The panel's **Workspace** section offers **Select Workspace** (*"Choose your workspace folder"*), **New Workspace**, and **Open Folder** — plus **Open in Finder** on macOS and **Open in Explorer** on Windows. Workspaces are held per agent, so two agents do not share a folder unless you point them at the same one.

#### Plan-before-acting
The agent states its plan before it touches the workspace — what it intends to scaffold, what it will install, and what it will run — so you see the shape of the change before any file is written.

#### Scaffolding from the Agentic Vibe starter
Rather than generating an application shell token by token, the agent materializes the vibe-starter template and installs its dependencies. The generated project inherits the starter's wiring, so the agent's effort goes into what is specific to your request.

#### Local preview
The agent starts the dev server and the application runs locally, so the end of the conversation is a working app you can click rather than a diff you have to trust.

#### Skills
Code Mode draws on ibl.ai's Agent Skills — reusable, plain-language playbooks for a specific job. If they cannot be fetched the run still proceeds, with the notice *"Skills couldn't be synced — Code will run without them."* Administrators manage which skills an agent carries in [Agent Settings: Skills](../agent-settings/skills.md).

#### Which coding agent runs the turn
Code Mode runs on one of three agents, chosen from the **Agent** control in the Code panel: **ibl.ai**, which is built into the app and is the default, or **Codex**, or **Claude Code**. The two named agents are not models we host; they are the vendors' own coding agents, run on your own subscription, so Code Mode becomes a front end to a tool you already pay for rather than a second bill.

The choice is held **per machine, not per account**, and is deliberately never synced, because an install and a CLI login are local facts about the computer you are sitting at. On another machine Code Mode is back on **ibl.ai** until you choose again there.

Each named agent reports its own state in the panel, and the states are worth knowing before picking one:

- **Not installed**, with an **Install** action (**Installing…** while it runs, then **Ready**). The desktop app installs Codex and Claude Code for you; ibl.ai needs no install.
- **Not available on this computer**, when the agent cannot run on your platform at all. No install will change that.
- **Not signed in**, which is a separate step from installing and happens outside the OS: for Codex, *"sign in to Codex in the ChatGPT app"*; for Claude Code, *"run `claude` in a terminal."* After signing in, use **Check Again** rather than restarting the app. A turn started against a signed-out agent reports *"Codex isn't signed in"* or *"Claude Code isn't signed in"*.

A **Code agent model** picker then appears in the agent's top bar for the selected agent, defaulting to **Default**, which is the agent's own choice of model. It reads the model list from the agent itself, so it shows **Loading models…** while it asks and **Couldn't load models** if the agent does not answer; **Default** stays selectable in that case. Codex's list comes from the Codex CLI's own model inventory rather than from a probe turn, so it reflects what your subscription actually grants.

Choosing **ibl.ai** keeps Code Mode on the platform's own models, which is the configuration the rest of this page describes, including the sandbox boundary in [Agent Settings: Sandbox](../agent-settings/sandbox.md).

#### On-device models
Code can run against a local model when Local Models is enabled or the desktop app is offline. It must be a **tool-capable** model — the panel suggests `qwen3`, `llama3.2`, or `phi4-mini` — and warns that a model without tool support *"isn't available for Code — turns will fail."* The first run can take a few minutes while the model loads.

## Requirements

#### Linux needs bubblewrap
On Linux, Code runs its work in a **bubblewrap** (`bwrap`) child sandbox. Without it the Code chip stays visible but disabled, with the hint *"Code needs bubblewrap (bwrap). Install it with your package manager, then reopen the app."* Install `bubblewrap` and reopen the app.

## How to Use

#### Step 1: Enable the Code chip
Activate **Code** in the message composer. On first use, answer the approval dialog — **Ask me each time** is the conservative choice and can be relaxed later.

#### Step 2: Choose a workspace
Use **Select Workspace** to point Code at an existing folder, or **New Workspace** to start a fresh one.

#### Step 3: Describe what you want built
Write the request in plain language. The walkthrough above uses four words — `build a todo list website` — and the agent replies with its plan before acting.

#### Step 4: Answer the permission prompts
Under **Ask Me**, choose **Allow** or **Deny** on each **Code needs your permission** card.

#### Step 5: Open the running app
When the agent reports the dev server is up, open the local preview and use it. In the walkthrough the result is a working task board with Home, Profile, and Account routes, priority tags, Active and Done filters, and live counters.

#### Step 6: Keep going in the same conversation
Code Mode does not end at the scaffold. Ask for the next change in the same session — the agent already holds the context of what it built.

## Administration

Code Mode's reach is a configuration decision, not a per-conversation promise. The controls in [Agent Settings: Sandbox](../agent-settings/sandbox.md) — a dedicated execution workspace on independent infrastructure, with isolated execution, allowlisting, and credential injection — determine what a coding agent may reach before anyone types a prompt. The approval mode and its per-action prompts then sit inside that boundary, so an organization sets the outer limit and the person in the conversation still governs each action within it.

## Related Resources

#### Agentic Vibe
The scaffolding layer Code Mode builds from is documented on the [Agentic Vibe product page](https://ibl.ai/product/agentic-vibe), with setup and environment variables in the [vibe skill setup guide](https://ibl.ai/developer/vibe/skill-setup).

#### Announcement
[Code Mode: From One Prompt to a Running App](https://ibl.ai/updates/code-mode) — the release walkthrough this page builds on.

#### Related pages
[Chat Interface](chat.md) · [Agent Settings: Skills](../agent-settings/skills.md) · [Agent Settings: Sandbox](../agent-settings/sandbox.md)
