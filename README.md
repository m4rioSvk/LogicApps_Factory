# 🏭 LogicApps Factory

A generator and toolkit for producing deploy-ready Azure Logic App (Standard) skeletons from plain-English requirements.

---

> Turn requirements into a scaffolding you can open in the Logic App Designer — workflow logic is generated, environment values are left as neutral placeholders for you to fill.


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
LogicApps Factory (LogicAppsAgents) is a small, opinionated generator that converts plain-English integration requirements into a ready-to-open Logic App Standard workflow skeleton plus a checklist of everything you must configure before running.


## What it produces
Each run writes a workflow folder into your target Logic App project containing:

- `workflow.json` — generated workflow that uses `@appsetting('...')` references instead of hardcoded values.
- `local.settings.json` — skeleton app settings with empty values.
- `connections.json` — connector stubs so the Designer shows "Create Connection".
- `SETUP.md` — a fill-in manifest (step-by-step checklist).
- `<workflow>.mmd` — Mermaid diagram of the flow (visual overview).


## How it works (quick)
1. You author a small flow file (JSON) describing the requirement and the path to a target Logic App Standard project.
2. Run the generator and it streams stages that: design the workflow, wire inputs/outputs, author the JSON, add error handling, validate, and output the skeleton + diagram.
3. Open the generated workflow in the Designer, fill the placeholders, create connections, and run.


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