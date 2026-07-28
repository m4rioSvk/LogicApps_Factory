# 🏭 LogicApps Factory

An AI pipeline that turns a **plain-English requirement** into a **deploy-ready
Azure Logic App (Standard) skeleton**: the workflow logic is written for you, and
every environment-specific value (connection strings, container names, queue
names, paths, secrets) is left as a **neutral placeholder** for a human to fill in
before running.

---

> You describe *what the integration should do*; the pipeline produces *the workflow
plus a checklist of what you still need to configure*.


## 🎯 Quick links
- Repo: `m4rioSvk/LogicApps_Factory`
- Topics: Azure · Logic Apps · IaC


## 📚 Table of contents
- [About](#about)
- [What it produces](#what-it-produces)
- [How it works (quick)](#how-it-works-quick)
- [Getting started](#getting-started)
- [Files & structure](#files--structure)
- [Common fixes](#common-fixes)
- [Contributing](#contributing)
- [License & contact](#license--contact)


## About
This repo is the **generator**. It is *not* the Logic App itself.

- **This project — `LogicAppsAgents/`** — the Python tool that *writes* workflows.
  You run it from a terminal. It never needs the Azure runtime.
- **The target project — a Logic App Standard project** (e.g.
  `.../Practice_Labs/LogicApps_factory/`) — a real, already-scaffolded Standard
  project (has `host.json`, `local.settings.json`, `connections.json`). This is
  where generated workflows land and where you *run* them. Its path is given
  per-run as `target_project` in the flow file.


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


## How it works 
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


## 🚀 Getting started
1. Clone the repo

```bash
git clone https://github.com/m4rioSvk/LogicApps_Factory.git
cd LogicApps_Factory
```

2. Install requirements

```bash
python -m pip install -r requirements.txt
# or: pip install claude-agent-sdk
```

3. Prepare a flow file under `flows/` (example below) and run

```bash
python main.py flows/example_flow.json
```

Example minimal flow (flows/example_flow.json)

```json
{
  "name": "invoice-automation",
  "target_project": "C:\\path\\to\\LogicApp_Project",
  "workflow_folder": "InvoiceProcessor",
  "requirement": "When an invoice arrives by email, extract invoice data, store PDF in blob, push metadata to ERP, and notify finance."
}
```


## Files & structure
LogicAppsAgents (this repo) is the generator. Its output belongs in a separate Logic App Standard project.

Repository overview (quick):

```
LogicAppsAgents/                    # generator
├── main.py                         # run: python main.py flows/<x>.json
├── flows/                          # your per-workflow parameter JSON files
├── PARAMETERS/                     # canonical parameter-name library
└── ...

target_project/                     # your Logic App STANDARD project (where output lands)
├── host.json
├── local.settings.json             # you fill values here
├── connections.json                # you create connections here
└── <workflow_folder>/
    ├── workflow.json               # generated (references, not literals)
    ├── SETUP.md                    # fill-in checklist
    └── <workflow_folder>.mmd       # diagram
```


## Common "it won't open" fixes
- Missing "Open in Designer" → open the project root (where `host.json` is) and ensure func host is running.
- Designer opens blank → the workflow is not Standard-shaped or JSON is invalid.
- Connection won't resolve → add the connector entry to `connections.json` and authenticate via Designer.


## Contributing
Contributions welcome — open an issue to discuss features or submit a PR with clear description and small, focused changes. Keep examples minimal and documented.


## License & contact
Add a LICENSE file to declare terms. For questions or feedback, open an issue or contact the maintainer.

---

Made the README more scannable and visually appealing — I can add badges, screenshots, or an architecture diagram if you'd like.
