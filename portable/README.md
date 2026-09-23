# Portable Alteryx Skills

Three single, self-contained markdown files, each generated from the
`alteryx` plugin's skills (see [`../alteryx/skills`](../alteryx/skills)) for
use without installing the plugin — paste the whole file into an agent
session instead. They cover three different dependency footprints:

| File | Environment | Needs | Can it run a workflow? |
|---|---|---|---|
| [`alteryx-designer-portable.md`](alteryx-designer-portable.md) | Local machine, no MCP server, no internet | A local Alteryx Designer install | Yes, locally |
| [`alteryx-cloud-portable.md`](alteryx-cloud-portable.md) | Browser or hosted chat, no local machine access | The `alteryx` MCP server connected, and an Alteryx One account | Yes, in Alteryx One |
| [`alteryx-xml-authoring-portable.md`](alteryx-xml-authoring-portable.md) | Any chat, no tools at all | Nothing | No — text output only |

The first two cover everything the full plugin does between them, each
against a different execution engine (local Designer Engine vs. Alteryx
One's cloud Engine) — a workflow has to run *somewhere*, so neither can be
made fully dependency-free without losing the ability to run anything. The
third file accepts that trade explicitly: it authors workflow XML as text,
with zero dependencies, and never runs or validates it.

## `alteryx-designer-portable.md` — Local, Offline

For building, inspecting, running, and repairing local Alteryx Designer
workflows (`.yxmd`, `.yxmc`, `.yxwz`) with an LLM agent that has **no plugin
install, no MCP server, and no internet access** — just the file itself and
a local Alteryx Designer installation.

Generated from the `alteryx-designer` and `alteryx-asset-discovery` skills,
with the cloud-only parts removed and every reference file the local build
path depends on inlined: the build/repair loop, the local asset discovery
process, the XML/script fallback process, the formula function reference,
the workflow XML structure reference, the full native tool catalog, and the
four PowerShell scripts needed to run a workflow through
`AlteryxEngineCmd.exe`.

**Usage:**

1. Copy the file's contents into the system prompt, project instructions, or
   first message of the agent session.
2. Extract the four PowerShell scripts from its **Appendix** section into a
   local directory — each is in its own fenced code block, named in its
   heading (`AlteryxDiscoveryUtils.ps1`, `Get-DesignerVersion.ps1`,
   `Find-DesignerSampleWorkflows.ps1`, `Invoke-AlteryxWorkflow.ps1`). Keep
   all four together; the last three dot-source the first by relative path.
3. Point the agent at a local `.yxmd` / `.yxmc` / `.yxwz` file and work as
   normal.

**Requirements:** Alteryx Designer installed on the machine (so
`AlteryxEngineCmd.exe` exists locally), and a terminal with PowerShell to
run the Appendix scripts.

**Out of scope**, with no offline substitute — these require the `alteryx`
HTTP MCP server instead: Alteryx One cloud workflows, Alteryx Auto Insights
analytics, and governed cloud asset search (discovery here is limited to
files on disk).

## `alteryx-cloud-portable.md` — Cloud, Web-Only

For building, inspecting, running, and repairing Alteryx One **cloud**
workflows, answering business questions with **Alteryx Auto Insights**, and
finding existing **governed cloud assets** — with an LLM agent that has **no
local Alteryx Designer install, no local file system access, and no
PowerShell** — purely through the web-based `alteryx` MCP server.

Generated from the cloud-mode parts of the `alteryx-designer`,
`alteryx-asset-discovery`, and `alteryx-insights` skills, with every
local-only branch, local file path, and local-fallback instruction removed.
Cloud workflows have no XML/script fallback in the source skills either, so
unlike the local file there's nothing to inline beyond the shared MCP
reference material, the formula function reference, and the native tool
catalog.

**Usage:**

1. Connect the `alteryx` MCP server in the client, pointed at your
   workspace's regional endpoint (`us1`, `eu1`, or `au1` —
   `https://<region>1.alteryxcloud.com/mcp/v1`; there is no global endpoint)
   and signed in to Alteryx One.
2. Copy the file's contents into the system prompt, project instructions, or
   first message of the agent session.
3. Work as normal — the file names the exact operations, run/validation
   rules, and reference material the agent needs for cloud workflows,
   Insights analytics, and cloud asset discovery.

**Requirements:** the `alteryx` MCP server connected and reachable, and an
authenticated Alteryx One account with permissions for the workspace,
workflows, and datasets involved.

**Out of scope**, with no substitute in this file: local Alteryx Designer
files on disk (use `alteryx-designer-portable.md`, or the `alteryx-local`
MCP server, instead) and any form of direct XML editing — cloud workflows
have no fallback path, so an unavailable required operation is reported as
a blocker rather than worked around.

## `alteryx-xml-authoring-portable.md` — Zero Dependencies, Authoring Only

For **authoring** Alteryx workflow XML with an LLM agent that has **zero
dependencies of any kind** — no plugin, no MCP server, no local Designer
install, no PowerShell, no internet, no file system access. It works as
pure text generation in any chat interface, including one with no tool
access at all.

This is the trade-off version: it can never run, validate, or inspect a
live workflow, because every execution path in the source skills requires
either a local Designer install or the `alteryx` MCP server — both
explicitly excluded here. What's left is authoring: given a request (or
pasted-in existing XML), it produces `.yxmd` / `.yxmc` / `.yxwz` XML text
using the workflow XML structure reference, the formula function reference,
and the native tool catalog, and says plainly what in that output is
sourced versus inferred. You save the result yourself and run it in
Designer to confirm it before trusting it.

**Usage:**

1. Copy the file's contents into the system prompt, project instructions, or
   first message of any agent session — no setup needed beforehand.
2. Describe the workflow you want, or paste in existing workflow XML to
   edit.
3. Save the returned XML as a `.yxmd` / `.yxmc` / `.yxwz` file yourself and
   open it in Alteryx Designer to run and verify it.

**Requirements:** none.

**Out of scope**, with no substitute in this file: running a workflow,
validating a formula through execution, sampling real data, or verifying
tool configuration against an actual schema. For any of that, use
`alteryx-designer-portable.md` or `alteryx-cloud-portable.md` instead —
this file's whole purpose is covering the case where neither is available.

## Regenerating These Files

All three files are derived artifacts, not hand-maintained independently.
If the source skills in [`../alteryx/skills`](../alteryx/skills) change,
regenerate the affected file(s) from the current skill content (each
skill's `SKILL.md` plus `alteryx-designer/references/*` and, for the local
file, `alteryx-designer/scripts/*.ps1`), carrying forward the same
adaptations:

- For `alteryx-designer-portable.md`: drop every cloud-mode branch, MCP tool
  name, and cloud-only section; keep all four PowerShell scripts verbatim,
  including the shared `AlteryxDiscoveryUtils.ps1` helper the other three
  depend on.
- For `alteryx-cloud-portable.md`: drop every local-mode branch, local file
  path, and local-fallback instruction; there are no scripts to carry
  forward.
- For `alteryx-xml-authoring-portable.md`: drop every MCP tool name, cloud
  reference, local-execution instruction, and PowerShell/engine reference;
  keep only the workflow XML structure reference, the formula function
  reference, and the native tool catalog, plus authoring guidance that
  never claims execution or validation happened.
- In all three: inline each reference file in place of the path the skill
  text pointed to, rewriting those pointers to name the corresponding
  section in the combined document instead of a separate file.
