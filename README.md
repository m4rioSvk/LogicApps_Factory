# LogicApps_Factory

# LogicAppsAgents — Logic App Skeleton Generator

An AI pipeline that turns a **plain-English requirement** into a **deploy-ready
Azure Logic App (Standard) skeleton**: the workflow logic is written for you, and
every environment-specific value (connection strings, container names, queue
names, paths, secrets) is left as a **neutral placeholder** for a human to fill in
before running.

You describe *what the integration should do*; the pipeline produces *the workflow
plus a checklist of what you still need to configure*.

---

## What it produces

For each run, into your Logic App project's workflow folder:

| File | What it is | Who fills it |
|------|------------|--------------|
| `workflow.json` | The complete workflow. Every environment value is an `@appsetting('Name')` reference, **never a hardcoded value**. | Generated — don't edit the logic |
| `local.settings.json` | App settings skeleton — the referenced names, with **empty values** (secrets point at Key Vault). | **You** fill the values |
| `connections.json` | Connector stubs, so the Designer shows "Create Connection". | **You** authenticate |
| `SETUP.md` | The **fill-in manifest** — a checklist of exactly what to configure. | **You** follow it |
| `<folder>.mmd` | A Mermaid diagram of the flow (visual overview). | Just view it |

The guiding rule: **the workflow holds *references*, the config files hold the
(empty) *values*.** That is what makes the skeleton open cleanly in the Designer
and prompt you to connect your own accounts — rather than shipping someone else's
credentials.

---

## The two projects (don't mix them up)

This repo is the **generator**. It is *not* the Logic App itself.

- **This project — `LogicAppsAgents/`** — the Python tool that *writes* workflows.
  You run it from a terminal. It never needs the Azure runtime.
- **The target project — a Logic App Standard project** (e.g.
  `.../Practice_Labs/LogicApps_factory/`) — a real, already-scaffolded Standard
  project (has `host.json`, `local.settings.json`, `connections.json`). This is
  where generated workflows land and where you *run* them. Its path is given
  per-run as `target_project` in the flow file.

---

## How to run

### 1. Prerequisites (one-time)

- Python 3.12+.
- `pip install claude-agent-sdk`
- An Anthropic API key available in your environment (the pipeline calls the
  model). e.g. `set ANTHROPIC_API_KEY=...` (Windows) / `export ANTHROPIC_API_KEY=...`.
- A real **Logic App Standard** project on disk to receive output (the
  `target_project`).

### 2. Write a flow file

A flow file is a small JSON "parameters file" under `flows/`. Minimum:

```json
{
  "name": "logic-app-tester",
  "target_project": "C:\\path\\to\\LogicApps_factory",
  "workflow_folder": "MyFirstLogicApp-rolling",
  "requirement": "(example) We currently receive supplier invoices by email. 
    Employees manually download PDF invoices, check the supplier information, enter invoice details into our ERP system, 
    and notify the finance team. We want to automate this process using Azure."
}
```

Required keys: `name`, `requirement`, `target_project`, `workflow_folder`.
Optional keys:

- `trigger` / `actions` — pin the design yourself. **Omit both** and the AI
  designs them (the `solution-architect` stage runs). Supply both and that stage
  is skipped.
- `reference_project` — absolute path to another Standard project the agents may
  read for schema examples.
- `groups` — explicitly list which parameter groups to pull (e.g.
  `["storage","servicebus"]`). Omit it and the AI infers the groups from your
  requirement.

### 3. Run it

```powershell
cd C:\...\LogicAppsAgents
python main.py flows\your_flow.json
```

You'll see the pipeline stream its stages, then write the four artifacts (plus the
`.mmd`) into `target_project\workflow_folder\`.

---

## What happens inside (the pipeline)

A coordinator runs five subagents **in order**, each feeding the next:

1. **solution-architect** *(runs only if the design isn't pinned in the flow
   file)* — turns the requirement into a trigger + ordered actions, and names the
   **parameter groups** the workflow needs.
2. **wiring-specialist** — maps which source field feeds which target input.
3. **workflow-author** — writes the real `workflow.json`, using
   `@appsetting('...')` references for every environment value (never literals).
4. **error-handler-designer** — adds Try/Catch scopes, retries, fail-loud
   termination.
5. **validator** — checks for gaps, and flags any value hardcoded that should
   have been a reference.

Then two deterministic (no-AI) steps run:

- **Mermaid render** — the `.mmd` diagram.
- **Skeleton assembly** — pulls the named parameter groups from `PARAMETERS/`,
  writes `local.settings.json` + `connections.json` + `SETUP.md`.

### The PARAMETERS library

`PARAMETERS/` holds the **canonical parameter names**, grouped (sql, storage,
servicebus, http, …). It guarantees every generated skeleton uses the *same* names
for the same things. Each name carries a tier hint:

- `appsetting` → goes in `local.settings.json`.
- `appsetting` + `secret:true` → Key Vault reference.
- `connection` → goes in `connections.json`.

**Fail-loud, self-improving:** if the workflow references a setting name that no
parameter group defines, the run **fails with a specific log** naming exactly what
to add — e.g.:

```
SKELETON_GENERATION_FAILED: missing_library_parameters
  referenced via @appsetting(...) but not defined in any pulled group:
    - CosmosThroughput
  action: add these keys to the relevant PARAMETERS/*.json and re-run
```

You add the missing key to the right `PARAMETERS/*.json`, re-run, done. The
library grows as you meet new connectors.

---

## After generation — configure the Logic App to actually run

The generated workflow will **not run until you fill in the placeholders.** Open
`SETUP.md` in the workflow folder — it lists every step. In general:

1. **Fill app settings.** Open `local.settings.json` and set the empty values
   (storage account, container, queue name, endpoints, etc.).
2. **Add secrets to Key Vault.** Any setting shown as a
   `@Microsoft.KeyVault(...)` reference needs the real secret created in your Key
   Vault; leave the reference in place.
3. **Create connections.** For each entry in `connections.json` (and each
   connector the Designer flags "⚠ Create Connection"), open the workflow in the
   Designer and authenticate with *your* account.
4. **Open the project in VS Code** (the `target_project` root, the folder with
   `host.json` — **not** the workflow subfolder).
5. **Start the local runtime.** Start **Azurite** (local storage emulator), then
   run the **`func: host start`** task (or press **F5**). The built-in blob
   operations use Azurite via `AzureWebJobsStorage`.
6. **Open the Designer.** Right-click the workflow's `workflow.json` →
   **Open in Designer**. Fire a run and inspect it under **Overview → Run
   History**.

> **Note on Service Bus:** there is no local Service Bus emulator. Any Service Bus
> step needs a **real** namespace connection string, even when running locally
> against Azurite.

### Common "it won't open" fixes

- **No "Open in Designer" option** → you opened the wrong folder (open the project
  root with `host.json`), or the file isn't named exactly `workflow.json` in its
  own subfolder, or the func host isn't running.
- **Designer opens blank** → the file is Consumption/ARM-shaped, not Standard
  (`{ definition, kind }`), or has invalid JSON.
- **Connection won't resolve** → the connector needs an entry in `connections.json`
  and a created/authenticated connection.

---

## Quick reference

```
LogicAppsAgents/                    # THIS generator project
├── main.py                         # run: python main.py flows\<x>.json
├── flow_spec.py                    # loads/renders a flow file
├── skeleton_assembler.py           # deterministic config-skeleton writer
├── wf_to_mermaid.py                # deterministic .mmd renderer
├── flows/                          # your per-workflow "parameters" JSON files
└── PARAMETERS/                     # canonical parameter-name library (tier-hinted)

target_project/                     # the Logic App STANDARD project (runs the output)
├── host.json                       # one per project
├── local.settings.json             # you fill values here
├── connections.json                # you create connections here
└── <workflow_folder>/
    ├── workflow.json               # generated (references, not literals)
    ├── SETUP.md                    # <-- start here: your fill-in checklist
    └── <workflow_folder>.mmd       # generated diagram
```
