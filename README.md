# IBM webMethods Integration skills extension

IBM publishes AI skills for webMethods Integration Server under the name
**webMethods Integration Flow Pilot**. This repository extends them.

- It does not replace or modify any IBM file.
- It is a set of Markdown files that you copy into the Flow Pilot skills' `extended-skill/` folder.
- The Flow Pilot skill reads that folder automatically.

---

## Ambition

IBM's skills generate FSL code, unit-test files and documentation. IBM's VS Code extension installs
the skills and lets the developer validate, deploy, run and debug services on an Integration Server.

This extension goes further:

1. **End-to-end automation with a feedback loop.** Through an MCP server, the AI model deploys its
   code, runs the tests and checks what was really compiled. It sees its own failures and fixes them
   on its own.
2. **Tests that prove something.** Expected values come from real calls, not guesses. Each suite is
   broken once on purpose to show that it can fail. Leftover pipeline variables are checked. Boundary
   cases are mandatory.
3. **Best practices for structure.** A standard package tree. Utility services in a shared `utils`
   folder, reused before new ones are created. Code split into commented sequences. The workspace as
   the source of truth, under version control.
4. **Do's and don'ts**, so the model gets it right the first time and diagnoses problems fast: silent
   compilation losses, decimal arithmetic, platform pitfalls, and corrections to the IBM content. All
   verified on a live Integration Server 12.1.
5. **The developer keeps the decisions.** The model asks instead of guessing (package, service name).
   It submits a plan before writing. It proposes factorisations and cross-package dependencies; it
   does not impose them.

---

## Prerequisites

### webMethods Integration Flow Pilot

**➡ [Download Flow Pilot from IBM](https://www.ibm.com/resources/mrs/assets/DownloadList?source=WebMtds_FlowPilot&lang=en_US)**

- This extension was written for **Flow Pilot 12.1.0.3**.
- Flow Pilot 12.1.0.3 requires Integration Server and Designer 12.1 Core Fix 3 or later.
- The extension applies to the **Flow Pilot** skills (`skills/flow/`).

### A webMethods Integration Server MCP server

The MCP server lets the AI model act on the server: deploy, reload, invoke, run tests. The skill files
describe these operations without naming any tool, so any MCP server that provides them works:

- [cpoder/wm-mcp-server](https://github.com/cpoder/wm-mcp-server), by Cyril Poder. This is the one I
  use: it is very exhaustive and modular.
- The official webMethods MCP server, on [packages.webmethods.io](https://packages.webmethods.io). It is
  a webMethods package that you install on the Integration Server.

### Also expected

- Ideally, **file access to the Integration Server `packages/` directory** (local server). Services can
  then be deployed by copying the file and reloading the package (`40-`).

---

## How it works

IBM provides the `extended-skill/` folder so that each enterprise can add its own policies,
guidelines, processes, best practices and patterns, without forking the official skill. This
repository is one such set of additions. You can add your own next to it.

Before every request, the Flow Pilot skill reads the `.md` files in `extended-skill/` and applies them on
top of its own rules.

When rules disagree:

1. The FSL grammar and the official correctness rules always win.
2. Extension files win over the official conventions, and over any official file they explicitly
   correct.
3. Between two extension files, the later one in alphabetical order wins: `95-` over `50-`, and
   `custom-instructions.md` over all numbered files.

---

## Installation

### 1. Copy the extension

Copy the content of this repository into the `skills/flow/extended-skill/` folder. It replaces the
two official files there:

- `custom-instructions.md`: this version keeps all the official rules and adds the workspace-first
  rule.
- `README.md`: the skill never reads it.

```bash
git clone https://github.com/staillanibm/wm-integration-skills-ext.git
cd wm-integration-skills-ext
cp -R *.md reference <path-to-skills>/flow/extended-skill/
```

`<path-to-skills>` is where the IBM skills are installed, for example
`<workspace>/.bob/skills/wm-integration-skills/skills`. Repeat for each AI tool (`.bob`, `.github`…).

### 2. Check

Ask the assistant for a Flow Service. Its reply must start with the list of extension files it
applied. If it says `extended-skill: none found`, the files are in the wrong folder.

### ⚠️ After reinstalling or upgrading Flow Pilot

The VS Code extension reinstalls the official skills once per version, and whenever you run
*Install Skills*. This brings back the official `README.md` and `custom-instructions.md`.
**Copy this extension again** afterwards.

---

## What each file adds

| File | What it adds or corrects |
|---|---|
| `custom-instructions.md` | The official rules, plus **workspace first**: every file is written in the workspace, then copied to the server. Never write directly into `packages/`. |
| `10-package-and-output-layout.md` | **Package layout.** Asks for the target package instead of guessing, and creates it if missing. Files go to `{package}/{rootDir}/services/…/{Service}/flow.flow`. |
| `20-inline-documentation.md` | **Inline comments** are mandatory on the service and on non-obvious steps. Logic is split into commented `SEQUENCE` blocks. |
| `30-factorisation.md` | **Factorisation.** Spots logic worth extracting into a separate service. The model proposes; the developer decides. |
| `35-main-service-name-and-utils-folder.md` | Asks for the **main service name**. Utility services go in `services/utils`. Existing ones are **reused** first. |
| `40-deployment-and-test-loop.md` | **Deploy, reload, test.** A service is done only when tested **after a package reload**: before that, the running version can differ from the saved one. Deploying by file copy **keeps the service signature**; deploying through the API loses it. |
| `50-nested-documents-and-silent-failures.md` + `reference/document-assembly-…` | **Nested documents and XML.** Build each node as a complete record, then copy whole records. Otherwise the compiler silently drops statements. Explains how to detect it. |
| `60-unit-testing.md` + `reference/unit-testing-run-and-troubleshoot.md` | **Fixes to the official unit-testing guide**: file paths, server layout, exception class. Expected values come from **calling the service**, not from guessing. |
| `75-mapping-patterns-that-survive.md` | **Mapping patterns that survive a reload**, and the ones that silently break. |
| `80-numeric-and-formats.md` | **Decimal arithmetic.** Never compute money in floating point. Round once, at the right level. Number formatting depends on the server locale. |
| `85-unit-testing-addenda.md` | **Unit testing in practice.** Test the saved version. Pick the right kind of expectation. A mandatory test verifies a clean output pipeline on the nominal case. |
| `90-delivery-workflow-and-sizing.md` | **Complete deliverable.** Plan first, then package, services bottom-up, tests, and a replay script. Size services by steps of logic, not by line count. **Schema-valid XML**: element order comes from an IS document type, not from the FSL, so create one before serialising. |
| `95-fsl-platform-pitfalls.md` | **Platform pitfalls on IS 12.1.** A reload reports success even when compilation fails. `$null` is not `""`. Some list copies are silently dropped. |

Items marked *hypothesis* in the files are observations that are not yet confirmed.

---

## Adding your own rules

- Add a `.md` file directly in `extended-skill/`.
- Its numeric prefix sets its priority: a higher number wins. `custom-instructions.md` wins over all
  numbered files.
- Put long documents in `reference/`, and point to them from a top-level file. The skill does not load
  `reference/` on its own.
