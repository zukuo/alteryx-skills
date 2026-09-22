# Portable Alteryx Designer Skill

`alteryx-designer-portable.md` is a single, self-contained markdown file for
building, inspecting, running, and repairing local Alteryx Designer workflows
(`.yxmd`, `.yxmc`, `.yxwz`) with an LLM agent that has **no plugin install, no
MCP server, and no internet access** — just the file itself and a local
Alteryx Designer installation.

It is generated from the `alteryx` plugin's `alteryx-designer` and
`alteryx-asset-discovery` skills (see [`../alteryx/skills`](../alteryx/skills)),
with the cloud-only parts removed and every reference file the local build
path depends on inlined into one document: the build/repair loop, the local
asset discovery process, the XML/script fallback process, the formula
function reference, the workflow XML structure reference, the full native
tool catalog, and the four PowerShell scripts needed to run a workflow
through `AlteryxEngineCmd.exe`.

## When To Use This Instead Of The Plugin

Use the plugin (see the [repository README](../README.md)) whenever you can
install it — it gets you MCP-backed workflow tools, cloud mode, Alteryx
Auto Insights, and governed cloud asset search, none of which this file can
do. Reach for this file only when the plugin can't be installed: a locked-down
or offline machine, a Claude/LLM surface without plugin support, or any
session where you just need to hand an agent one file and a local Designer
install.

## Usage

1. Copy the contents of `alteryx-designer-portable.md` into the system
   prompt, project instructions, or first message of the agent session.
2. Extract the four PowerShell scripts from its **Appendix** section into a
   local directory — they're the last thing in the file, each in its own
   fenced code block, named in its heading (`AlteryxDiscoveryUtils.ps1`,
   `Get-DesignerVersion.ps1`, `Find-DesignerSampleWorkflows.ps1`,
   `Invoke-AlteryxWorkflow.ps1`). Keep all four together; the last three
   dot-source the first by relative path.
3. Point the agent at a local `.yxmd` / `.yxmc` / `.yxwz` file and work as
   normal — the file contains everything the agent needs to inspect, edit,
   and run it.

## Requirements

- Alteryx Designer installed on the machine, so `AlteryxEngineCmd.exe`
  exists locally.
- A terminal with PowerShell to run the Appendix scripts.

## Scope And Limitations

Out of scope, with no offline substitute — these require the `alteryx` HTTP
MCP server from the full plugin:

- Alteryx One cloud workflows (building or running workflows stored in an
  Alteryx One workspace).
- Alteryx Auto Insights analytics (the `alteryx-insights` skill).
- Governed cloud asset search — discovery in this file is limited to files
  on disk.

## Regenerating This File

This file is a derived artifact, not hand-maintained independently. If the
source skills in [`../alteryx/skills`](../alteryx/skills) change, regenerate
`alteryx-designer-portable.md` from the current `alteryx-designer` and
`alteryx-asset-discovery` skill content (SKILL.md plus the
`alteryx-designer/references/*` and `alteryx-designer/scripts/*.ps1` files),
carrying forward the same adaptations:

- Drop every cloud-mode branch, MCP tool name, and cloud-only section.
- Inline each reference file in place of the path the skill text pointed to,
  rewriting those pointers to name the corresponding section in this
  document instead of a separate file.
- Keep all four PowerShell scripts verbatim, including the shared
  `AlteryxDiscoveryUtils.ps1` helper the other three depend on.
