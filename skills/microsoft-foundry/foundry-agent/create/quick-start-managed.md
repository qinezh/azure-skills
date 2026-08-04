# Quick Start: Managed Agent PoC

Create, deploy, and remotely invoke a Microsoft Foundry Managed Agent with declarative instructions and local skills.

> **Private preview:** This workflow is a focused proof of concept. It uses a private-preview `azure.ai.agents` extension source, requires `westus2`, and may change without compatibility guarantees. Do not use this workflow for production automation.

This file captures the supplied private-preview manual and is authoritative for this PoC. Public Prompt Agent and Hosted Agent documentation does not replace this contract. Do not infer alternate MCP, SDK, provisioning, deployment, or invocation paths from public agent guidance.

## When to Use

Use this workflow only when the user explicitly asks for a **Managed Agent**, the **managed harness**, or the private-preview prompt-agent option that runs with the managed harness.

Do not use it for:

- Ordinary Prompt Agents created through Foundry MCP.
- Hosted Agents that package Python, .NET, or container code.
- Evaluation, monitoring, CI/CD, or production-hardening work.

This is a self-contained create/deploy/invoke path. Do not continue into the general [deploy](../deploy/deploy.md) or [invoke](../invoke/invoke.md) workflows.

## Fixed Preview Contract

| Property | Required value |
|----------|----------------|
| Region | `westus2` only |
| Agent kind selected in `azd ai agent init` | `Prompt agent - model + instructions only (Foundry runs Brain+Hand; Harness: GHCP)` |
| Authoring files | `instructions.md` and optional `skills/<skill-name>/SKILL.md` |
| Initial create and deploy | `azd up` |
| Redeploy after edits | `azd deploy` |
| Remote smoke test | `azd ai agent invoke "<message>"` |

Always follow [azd guidance](../azd-guidance/azd-guidance.md). Every `azd` command below uses the required one-command telemetry scope:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd <command>
```

The examples use POSIX inline environment syntax. Use the shell-equivalent one-command scope on other platforms; never persist `AZURE_DEV_USER_AGENT` in the azd environment, project files, or user profile.

## Workflow

### Step 1 - Confirm preview intent and authentication

Tell the user that this path installs an extension from a private-preview registry in their user-level azd configuration. Do not add or replace an extension source until the user has confirmed they want the preview setup.

Check authentication without opening a login flow:

```bash
az account show --output json
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd auth login --check-status
```

If either check reports that authentication is required, stop and ask the user to run `az login` or `azd auth login` themselves. Never start an interactive login for them.

### Step 2 - Set up the private-preview extension idempotently

Use this registry exactly:

```text
Name: MHA-dev
Type: url
Location: https://raw.githubusercontent.com/kshitij-microsoft/azure-dev/refs/heads/kchawla/azd-managed-harness/cli/azd/extensions/registry.json
```

First inspect installed extensions and registered sources:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd version --output json
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd extension list --installed --output json
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd extension source list --output json
```

The preview registry snapshot verified for this PoC published `azure.ai.agents` `0.1.46-preview` and required azd greater than `1.25.2`. Treat the registry metadata as authoritative because the branch can advance. If the installed azd version does not satisfy the registry requirement, stop and ask the user to update azd before changing extensions; do not pin or force an incompatible build.

Apply these conditions in order:

1. If `microsoft.azd.extensions` is absent, install it:

   ```bash
   AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd extension install microsoft.azd.extensions
   ```

2. Inspect the `MHA-dev` source entry by its JSON `name`, `type`, and `location`. Compare the name case-insensitively; azd normalizes it to `mha-dev` when saving user configuration.
   - If it is absent, add it:

     ```bash
     AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd extension source add \
       --name MHA-dev \
       --type url \
       --location https://raw.githubusercontent.com/kshitij-microsoft/azure-dev/refs/heads/kchawla/azd-managed-harness/cli/azd/extensions/registry.json
     ```

   - If it already has the exact type and location above, reuse it. Do not add it again.
   - If the name exists with a different type or location, stop and show the conflict. Do not remove or overwrite a user-configured source automatically.

3. Inspect the installed `azure.ai.agents` entry by its JSON `id` and `source`. Compare the source name case-insensitively because installed metadata can report the normalized `mha-dev` value.
   - If it is absent, install the preview build:

     ```bash
     AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd extension install azure.ai.agents --source MHA-dev
     ```

   - If it is already installed from `MHA-dev`, leave it unchanged.
   - If it is installed from another source, explain that continuing replaces the current global extension build and ask for confirmation. After approval, use:

     ```bash
     AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd extension install azure.ai.agents --source MHA-dev --force
     ```

Re-run both list commands and confirm the expected source before continuing. This makes repeated setup runs no-ops while protecting unrelated global azd configuration.

### Step 3 - Select an existing `westus2` Foundry project

Prefer the Foundry project already selected or named by the user. Do not ask again when the project is already known.

Managed Agents in this preview require `westus2`:

- If the selected project's location is known and is not `westus2`, stop. Do not deploy to it and do not silently create a replacement.
- If the location is unknown, verify it from the project metadata or the location shown by the `azd ai agent init` project picker before accepting the selection.
- If no project was supplied, ask the user to choose an existing `westus2` Foundry project or explicitly approve creating a new one. Prefer the existing-project option.

Only when the user explicitly chooses a new project, create it in `westus2`:

```bash
RG="my-foundry-rg"
LOCATION="westus2"
ACCOUNT="my-globally-unique-foundry-account"
PROJECT="my-foundry-project"

az group create --name "$RG" --location "$LOCATION"
az cognitiveservices account create \
  --name "$ACCOUNT" \
  --resource-group "$RG" \
  --location "$LOCATION" \
  --kind AIServices \
  --sku S0 \
  --custom-domain "$ACCOUNT"
az cognitiveservices account project create \
  --resource-group "$RG" \
  --name "$ACCOUNT" \
  --project-name "$PROJECT" \
  --location "$LOCATION"
```

The account name must be globally unique. The project name must be unique within the account.

### Step 4 - Initialize the Managed Agent

Run init from the directory that should contain the generated agent folder:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd ai agent init
```

The interactive sequence from the preview manual is canonical. The preview also exposes `--kind managed` as a non-interactive alias with `--no-prompt`, `--agent-name`, and `--model`. Use that path only when `azd ai agent init --help` confirms the current complete flag set and the existing `westus2` project binding is explicit; never let non-interactive init fall back to creating an unapproved project.

Select:

1. **Agent kind:** `Prompt agent - model + instructions only (Foundry runs Brain+Hand; Harness: GHCP)`.
2. **Agent name:** the user's requested name.
3. **Description:** the user's description, or blank.
4. **Subscription:** the user's selected subscription.
5. **Foundry project:** the user-selected existing project in `westus2`; select a newly created project only when Step 3 explicitly created one.
6. **Model:** a capable model available to that project.
7. **System instructions:** the user's initial instructions, or the CLI default for an initial smoke test.

If the exact Managed Agent option is absent, stop. The installed `azure.ai.agents` extension is not the expected private-preview build.

Initialization succeeds only when the CLI reports `Initialized prompt agent` and lists the generated `agent.yaml`, workspace, and model endpoint. Change into the generated agent directory before continuing.

Confirm the generated preview fingerprint:

- `agent.yaml` has `kind: prompt`.
- The Managed Agent service in `azure.yaml` has `host: azure.ai.agent` and `config.promptAgent`.

Deployment maps this definition to API `kind: prompt` with the `ghcp` harness. That mapping is expected; it does not make this an ordinary Prompt Agent or a Hosted Agent for skill routing.

### Step 5 - Verify the authoring layout

The generated project must contain:

```text
<agent-root>/
  agent.yaml
  instructions.md
  skills/
    <skill-name>/
      SKILL.md
```

`instructions.md` is the agent's system instruction source. Preserve user-provided instructions and edit this file directly for subsequent changes.

Each immediate subdirectory under `skills/` represents one local skill. Each `SKILL.md` must have YAML front matter with a stable `name` and a specific `description`, followed by the skill instructions:

```markdown
---
name: incident-triage
description: Triage service incidents, assign a severity, and recommend escalation and communication cadence.
---

# Incident triage

Follow the user's incident-triage procedure here.
```

Before deploy:

- Confirm `instructions.md` is non-empty.
- Confirm every `skills/*/` directory contains a non-empty `SKILL.md`.
- Confirm each skill's front matter has `name` and `description`.
- Do not add an empty `skills/` entry or unrelated hosted-agent source code.

During deployment, the preview CLI creates the skills and links them to the agent through a Toolbox.

### Step 6 - Initial deploy and baseline remote invoke

From the generated agent directory:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd up
```

Use `azd up` for this first Managed Agent deployment even when the workflow selected an existing Foundry project. Do not replace it with `azd deploy`, `azd provision`, or a deploy-only public-agent workflow. The preview contract uses `azd up` to provision the required managed-agent resources and deploy the initial agent while reusing the selected project.

When `azd up` asks for a location, select **West US 2 (`westus2`)** even when the existing Foundry project is already in that region. Cancel the operation if another region is selected or inferred.

After deploy, establish a baseline with a remote invocation:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd ai agent invoke "hello, tell me what tools are available to you."
```

Confirm that the command returns a response. Remote invocation can incur model usage charges, so do not run additional exploratory prompts unless requested.

### Step 7 - Apply instructions and local skills

Update `instructions.md` with the user's requested agent behavior. Add or update only the relevant `skills/<skill-name>/SKILL.md` files, then repeat the Step 5 content checks.

Do not manually create production orchestration, evaluation, monitoring, or CI/CD assets for this PoC.

### Step 8 - Redeploy and run the final smoke test

Redeploy the updated instructions and local skills:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd deploy
```

Run one representative remote invocation that should activate the new instructions or one local skill:

```bash
AZURE_DEV_USER_AGENT=microsoft_foundry_skill azd ai agent invoke "<representative skill-triggering request>"
```

Verify that:

- The command completed and returned an agent response.
- The response follows `instructions.md`.
- When the prompt matches a local skill description, the response follows that skill's required behavior and output shape.

If the baseline invoke succeeded but the final invoke ignores a skill, first improve the skill front-matter `description` so it clearly matches the request, then redeploy once and repeat the same smoke prompt.

## Error Handling

| Symptom | Resolution |
|---------|------------|
| `MHA-dev` already exists with another URL or type | Stop and show the existing source. Do not overwrite or remove it automatically. |
| `azure.ai.agents` is installed from another source | Ask before replacing the global extension; after approval, install from `MHA-dev` with `--force`. |
| azd does not satisfy the preview registry's required version | Stop and ask the user to update azd; do not force-install an incompatible extension. |
| Managed Agent option is missing during init | Recheck that `azure.ai.agents` is installed from `MHA-dev`; otherwise the preview build is unavailable. |
| Generated files lack `agent.yaml` `kind: prompt` or `azure.yaml` `config.promptAgent` | Treat the workspace as non-managed or incomplete; do not route it through this PoC based on `host: azure.ai.agent` alone. |
| Selected project or prompted deployment location is not `westus2` | Stop or cancel. This preview does not support another region. |
| `instructions.md` or `agent.yaml` is missing after init | Treat initialization as incomplete; do not invent the preview file format. Re-run init after resolving the reported error. |
| Local skill is ignored | Make the `description` specific to the triggering requests, verify `skills/<name>/SKILL.md`, then redeploy. |
| Remote invoke fails | Surface the exact CLI error. Do not switch to MCP, SDK, hosted sessions, or raw REST as a workaround in this PoC. |

## PoC Limitations

- Private preview and `westus2` only.
- Depends on a branch-hosted custom azd extension registry that can change or disappear.
- The extension source and installed extension are user-level azd state, not project-local dependencies.
- The flow is interactive and is not a supported unattended CI/CD contract.
- Portal support is limited and may require a feature flag.
- No production lifecycle guarantees, rollback design, multi-region support, evaluation, monitoring, or operational hardening are included.
