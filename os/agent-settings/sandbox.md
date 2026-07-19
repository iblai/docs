# Agent Settings: Sandbox

![Sandbox panel in the Edit Agent modal with the sandbox enable toggle, an instance search box, an Add Instance button, and a table of sandbox instances showing name, URL, type, status, health, version, last check, and Connect buttons](/images/docs/os/agent-settings/agent_settings_sandbox.webp)

## Overview

The Sandbox panel gives an agent its own secure execution workspace on independent infrastructure. The panel header reads: "Configure sandbox connection and deployment settings," and the helper banner explains: "Give this agent its own secure workspace where it can run code, tools, and skills on your behalf. When on, connect it to an instance below to unlock skills and agent prompts."

A sandbox instance is a separately hosted runtime the agent connects to. The panel lists the registered instances with their live status, health, and version, and lets you add, edit, delete, and connect to them. Connecting an agent to an instance unlocks additional capabilities in the Edit Agent modal: the **Skills** tab and the Agent Configuration prompt fields (Identity, Soul, User Context, Tools, Agents, Bootstrap, Heartbeat, Memory) in the Prompts tab.

To reach this screen, open the **Edit Agent** modal, switch to the **Integrations** tab group (its sidebar lists Sandbox, Access, Tools, MCP, Datasets, API, LTI, and Embed), and select **Sandbox**. The Sandbox tab itself only appears after the dedicated-sandbox capability ("Enable dedicated sandbox") is turned on and saved in the agent's Settings.

## Target Audience

**Administrator** | **Agent Builder**

## Panel Reference

#### Sandbox toggle
The switch in the helper banner enables or disables the sandbox for this agent. When on, the instance list below becomes the place to wire the agent to a runtime.

#### Search instances
A search box that filters the instance table by name or URL.

#### Add Instance
Opens the **New Instance** dialog for registering a sandbox runtime. The form takes a display **name**, a fully qualified https **server URL**, the instance **type** (`openclaw`, the default, or `ironclaw`), and an optional **gateway token** for authentication.

#### Instance table
Each registered instance is a row with:

- **NAME** — the instance's display name (for example `carl_ibl_ai`).
- **URL** — the instance's server URL (for example `https://carl.ibl.ai`).
- **TYPE** — the runtime type, such as `openclaw`.
- **STATUS** — **Active** or **Error** (an em-dash when unknown).
- **HEALTH** — **Healthy** or **Unhealthy** (an em-dash when unknown).
- **VERSION** — the software version reported by the instance (for example `2026.6.8`).
- **LAST CHECK** — when the instance was last health-checked (for example "1 minute ago").
- **Connect** — connects this agent to the instance. Connect is blocked (dimmed, with an explanatory tooltip) when the instance is unhealthy.
- **"..." actions menu** — per-instance actions including **Edit** (change name, server URL, or type), **Delete** (with confirmation), **Run checks** (health check plus connectivity test), and **Connect**.

## Connected State

![Sandbox panel after connecting, showing the Connected Instance card with name, server URL, status, health and last check, Run checks and Disconnect actions, an Auto Push on Save toggle, a Push Configuration row, and a Model row with Select Model](/images/docs/os/agent-settings/agent_settings_sandbox_connected.webp)

Once the agent is connected to an instance, the instance table is replaced by the connected-instance view:

#### Connected Instance card
Shows the connected instance's **Name**, **Server URL** (linked), **STATUS** (for example Active), **HEALTH** (for example Healthy), and **LAST CHECK** time. Two actions sit on the card:

- **Run checks** — re-runs the health check and connectivity test against the instance and reports the results.
- **Disconnect** — detaches the agent from the instance, after a confirmation dialog. Disconnecting hides the sandbox-gated capabilities (Skills, agent prompts) again.

#### Auto Push on Save
A toggle that, when on, automatically pushes the agent's configuration to the connected instance whenever it is saved, so the runtime never drifts from what is configured in the UI.

#### Push Configuration
Shows when the configuration was last pushed to the instance ("Never pushed" before the first push) and a **Push** button to push the current configuration to the connected worker on demand.

#### Model
The LLM the sandboxed agent runs on. **Select Model** opens the provider picker, where you choose a provider (for example Anthropic or OpenAI) and then a model; once chosen, the current model identifier is shown in this row and the button changes to allow changing it.

## How to Use

#### Step 1: Enable the sandbox capability
In the agent's **Settings** tab, turn on the dedicated-sandbox option and save. The **Sandbox** tab then appears in the **Integrations** tab group of the Edit Agent modal.

#### Step 2: Register an instance
Open **Sandbox** and click **Add Instance**. Enter a name, the https server URL of the runtime, choose the type (`openclaw` by default), optionally provide a gateway token, and submit. The new row appears in the instance table.

#### Step 3: Verify health and connect
Use **Run checks** from the row's actions menu to confirm the instance is Active and Healthy, then click **Connect**. The Connected Instance card replaces the table, and the agent's **Skills** tab and Agent Configuration prompts become available.

#### Step 4: Keep the runtime in sync
Click **Push** in **Push Configuration** to push the agent's current configuration to the instance — or enable **Auto Push on Save** so every save is pushed automatically. Use **Select Model** to set the LLM the sandboxed agent should use.

#### Step 5: Disconnect when needed
Click **Disconnect** on the Connected Instance card (and confirm) to detach the agent from the instance — for example before pointing it at a different runtime.
