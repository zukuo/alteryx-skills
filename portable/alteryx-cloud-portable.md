# Alteryx Cloud — Portable Web Skill

A single self-contained instruction file for building, inspecting, running,
and repairing Alteryx One **cloud** workflows, answering business questions
with **Alteryx Auto Insights**, and finding existing **governed cloud
assets** — with an LLM agent that has **no local Alteryx Designer install, no
local file system access, and no PowerShell** — purely through the web-based
`alteryx` MCP server.

## How To Use This File

Paste this entire file into the system prompt, project instructions, or
first message of any Claude (or other capable LLM) session that has the
`alteryx` MCP server connected. It is derived from the cloud-mode parts of
the `alteryx` plugin's `alteryx-designer`, `alteryx-asset-discovery`, and
`alteryx-insights` skills (https://github.com/alteryx/skills), with every
local-only branch, local file path, and local-fallback instruction removed,
and the shared MCP reference material inlined so nothing needs to be fetched
separately.

This is the cloud counterpart to `alteryx-designer-portable.md` in this same
directory, which covers the opposite case — no MCP server at all, but a
local Designer install to run against. Use this file when the reverse is
true: a browser or hosted chat session with no local machine access, but a
working connection to Alteryx One.

## Requirements

- The `alteryx` MCP server connected in the active client, pointed at your
  workspace's regional endpoint:

  | Region | Endpoint |
  |---|---|
  | United States | `https://us1.alteryxcloud.com/mcp/v1` |
  | Europe / Middle East | `https://eu1.alteryxcloud.com/mcp/v1` |
  | Asia Pacific | `https://au1.alteryxcloud.com/mcp/v1` |

  There is no global endpoint — pick the one matching your workspace's
  region. If tool calls fail with routing or not-found errors, this is the
  first thing to check.
- An authenticated Alteryx One account with permissions for the workspace,
  workflows, and datasets involved. Every cloud call runs as the signed-in
  user and is subject to workspace permissions and data-access policy.

## Scope And Limitations

This file covers Alteryx One **cloud mode only**, entirely through the
`alteryx` MCP server:

- Building, inspecting, running, and repairing workflows stored in Alteryx
  One workspace storage.
- Answering business data questions with Alteryx Auto Insights.
- Finding and verifying existing governed cloud workflows, macros, and
  datasets.

It does not cover, and cannot substitute for:

- **Local Alteryx Designer workflows** — inspecting or running a `.yxmd` /
  `.yxmc` / `.yxwz` file that lives on a local disk. That needs either the
  `alteryx-local` MCP server or, with no MCP server at all, the sibling
  `alteryx-designer-portable.md` file in this directory (requires a local
  Designer install).
- **Any XML/script fallback.** Cloud workflows have no fallback path at all.
  If the `alteryx` MCP server is unavailable, or lacks a required operation,
  the correct response is to report the blocker — never attempt to edit a
  cloud workflow's underlying representation directly.

## Cloud Workflow Build, Inspect, Run, Repair

Use this section to build and validate Alteryx workflows that live in
Alteryx One, entirely through the `alteryx` MCP server. Client tool-name
prefixes vary (commonly `designer__*` and `formulas__*`); use the exact
names and schemas the active client exposes rather than inventing names.

### Workflow Identity

Address a cloud workflow by its Alteryx One workflow ID, passed as
`workflow_id`:

```json
{ "workflow_id": "3f9c2b7e-1d4a-4c58-9a02-8e6b1f0d77aa" }
```

- Never construct or guess a `workflow_id`. Get it from the user, or from
  Cloud Asset Discovery below, which returns stable identifiers for the
  assets it finds.
- A cloud workflow has no local file. Do not report a path for it or claim a
  local artifact was written — report the workflow ID and workspace instead.

### Semantic Operations

Map the active MCP tools to these operations. Require only the operations
the current task needs; not all are necessarily available in every
workspace — check what the client exposes and say so when a request can't
be satisfied.

| Semantic operation | Purpose |
|---|---|
| `get_available_tools` | List supported Designer tools |
| `get_tool_configuration_schema` | Get schema-backed tool configuration |
| `get_condensed_workflow` | Inspect workflow structure and configuration |
| `validate_formula` | Validate an Alteryx formula |
| `create_workflow` | Create a workflow |
| `add_tool` | Add and connect a schema-supported tool |
| `edit_tool` | Edit a schema-supported tool |
| `add_custom_tool_with_xml` | Add a custom or unsupported tool via a tool-level XML snippet |
| `edit_custom_tool_with_xml` | Edit a custom or unsupported tool via a tool-level XML snippet |
| `remove_tool` | Remove a tool |
| `replace_tool_connections` | Replace or update connections |
| `refresh_metadata` | Validate configuration and refresh metadata |
| `run_workflow` | Run the workflow through the Alteryx Engine |
| `get_metadata` | Inspect persisted field names and types |
| `get_data_at_anchor` | Sample data at an output anchor |
| `add_tools_to_container` | Create or populate a tool/control container |
| `remove_tools_from_container` | Move tools out of a container |
| `delete_container` | Delete a container while preserving its tools |
| `set_container_enabled` | Enable or disable a container |
| `update_tool_positions` | Update tool positions on the canvas |
| `annotate_tool` | Update a tool's name and annotation |
| `update_workflow_description` | Set or clear the workflow description |

### Helpers

Two capabilities sit outside the operations above. They are not workflow
operations and take no workflow identifier.

- **Knowledge search** (`knowledge__search_knowledge_sources`) answers what
  the product supports. Use it when the appropriate Designer tool,
  configuration approach, formula behavior, or recommended workflow pattern
  is unclear. Provide both `query` (a focused question or search terms) and
  `intent` (a short statement of the workflow outcome the user is trying to
  achieve). Prefer a few targeted searches over broad discovery, and
  summarize only the relevant guidance rather than copying documentation
  passages into context.
- **Asset discovery** — use Cloud Asset Discovery below to find reusable
  logic, locate and verify a canonical dataset, or find example
  configuration for a tool with no configuration schema.

### Build With Native Designer Tools

Compose workflows from native Designer tools (see the Designer Native Tool
Catalog reference below for names, palettes, and anchors). Assume a native
tool exists until discovery shows otherwise. Use a code tool (Python, R, or
Run Command) only when the requirement cannot be met natively: state the
specific gap, get user approval, and scope the code to that gap with native
tools on both sides of it. Confirm custom and connector tool availability
with `get_available_tools` instead of assuming a plugin or package is
installed — availability differs between a local Designer install and a
cloud workspace, and this file only ever runs against the cloud workspace.

### Ground The Build Before Building

When the request depends on assets or data that may already exist —
reusable logic, a canonical governed dataset, an example of an unsupported
tool's configuration — use Cloud Asset Discovery below to find and verify
them before designing the workflow. Do not duplicate business logic that
already exists, and do not invent a schema you could have verified.

### MCP Build Process

1. Inspect an existing artifact with `get_condensed_workflow` before
   mutation.
2. Discover available Designer tools with `get_available_tools` when the
   requested tool support is not already established.
3. Get the configuration schema with `get_tool_configuration_schema` before
   adding or editing a schema-supported tool, unless the exact current
   schema is already present in task context.
4. Use human-readable Designer tool names expected by the schema, such as
   `Filter`, `Formula`, or `Summarize`.
5. Make one small mutation at a time and retain the returned identifiers
   needed by later calls, such as an assigned `tool_id`.
6. Run the workflow with `run_workflow` after changes that require
   execution or output validation.
7. Inspect run messages, relevant metadata, and small anchor samples as
   needed.
8. Repair through MCP and rerun until validation succeeds or a blocker is
   clear.

### Configuration Rules

- Prefer schema-driven add/edit operations (`add_tool`, `edit_tool`) over
  custom XML.
- Use `add_custom_tool_with_xml` / `edit_custom_tool_with_xml` only when
  schema-driven configuration cannot express the required installed tool,
  custom macro, interface construct, or analytic app behavior. These take a
  tool-level XML snippet, not a full workflow document — there is no
  document-level XML to author or edit in cloud mode.
- Manage containers with the container operations
  (`add_tools_to_container`, `remove_tools_from_container`,
  `delete_container`, `set_container_enabled`). Use `annotate_tool`,
  `update_tool_positions`, and `update_workflow_description` for their
  respective changes.
- `delete_container` removes the container and moves its contained tools to
  the parent canvas. To delete an individual tool, use `remove_tool`.
- Gather upstream metadata before authoring field-dependent formulas or
  configurations.
- For formulas, read the Alteryx Formula Function Reference below. Use
  `validate_formula` when exposed, then confirm behavior with a workflow
  run.
- Preserve the existing engine selection unless the user requests a change.
  Use AMP/E2 for new workflows when `create_workflow` exposes that choice.

### Dataset Inputs And Outputs

A cloud workflow's Input Data and Output Data tools **must** read and write
through a governed dataset referenced by `dataset_id` — never a direct file
path or connection string.

- Never guess a `dataset_id`. For a dataset that already exists, take it
  from the user or from Cloud Asset Discovery below.
- To write to a dataset that does not exist yet, call
  `datasets__create_empty_dataset` first, then configure the Output Data
  tool with the `dataset_id` it returns.

### Run, Metadata, And Validation Rules

- Call `run_workflow` before `get_data_at_anchor`; anchor data comes from
  the latest successful run.
- Call either `run_workflow` or `refresh_metadata` before `get_metadata`.
- `refresh_metadata` validates tool configuration and persists metadata
  updates without processing records. Use `run_workflow` to validate
  record-dependent behavior and outputs.
- Batch related execution-affecting mutations and refresh once. Skip the
  refresh entirely when `run_workflow` is next.
- Annotation, position, and description changes do not require a workflow
  run.
- Assume all prior run context and samples are invalid after mutation.
- Use run messages to identify the failing tool and inspect its immediate
  upstream metadata when possible.
- Verify field names and data types with metadata rather than inferring
  them from configuration.
- Validate transformations and outputs with the smallest useful anchor
  sample.
- A successful engine run does not prove logical correctness. Sample the
  output and check it against the original business intent before reporting
  completion.
- Stop and report authentication, licensing, entitlement, permission,
  unsupported-version, or unavailable-plugin errors clearly — do not work
  around them.
- Treat a successful tool call as mutation evidence, not
  workflow-completion evidence.

### Shared Rules

- Preserve the existing AMP/E2 versus legacy E1 setting during unrelated
  edits. Use AMP/E2 for new workflows unless the user explicitly requests
  E1.
- Make small, reversible changes and validate incrementally.
- Treat workflow overwrites, tool removal, connection replacement, and
  broad rewrites as destructive operations.
- Treat run context, metadata, and sampled output as stale after every
  mutation.
- Never expose or embed credentials, connection strings, DCM payloads, or
  sensitive values unless the user explicitly provides and approves them.
- Inspect only the metadata and sampled data needed to validate the
  request. Do not load full datasets into context.

### Completion And Response

Verify all applicable evidence:

- The workflow is composed of native Designer tools. Any code tool present
  was approved by the user.
- The requested workflow changes are saved to the cloud workflow (by
  workflow ID).
- The Engine-backed run succeeded and produced substantive diagnostics, not
  only start/finish banners.
- Expected datasets, metadata, or sampled anchor results were created or
  updated.
- The inspected output evidence supports the requested business logic.

Summarize the workflow ID and workspace, what changed, the run result, the
evidence checked, and any remaining user action. Do not dump full tool
configuration, sensitive values, or large data samples unless the user
explicitly asks.

## Cloud Asset Discovery

Answer one question: **what existing governed Alteryx One workflows,
macros, or datasets could help solve this problem?** Discover and verify
candidates, then report the evidence. Never create, modify, or run an asset
as part of this process — discovery is read-only.

### Discovery Scope

Governed Alteryx One datasets, workflows, and macros returned by asset
search.

### Tools

Use these read-only tools through the `alteryx` MCP server. Client prefixes
vary; use the exact tool names the active client exposes.

| Tool | When to use it | Required handling |
|---|---|---|
| `assets__search_alteryx_assets(query)` | Search governed assets with a focused natural-language query. | Treat results as candidates, not verified facts. |
| `designer__get_condensed_workflow(workflow_id)` | Inspect a cloud workflow or macro. | This is the evidence that verifies a workflow candidate. |
| `datasets__preview_dataset(datasetId, limit)` | Verify a governed dataset's schema and inspect a small sample. | Pass the dataset ID in the current preview contract; respect returned limits and never use repeated previews to bulk-extract. |

### Cloud Asset Identifiers

Search results may identify cloud assets with URNs or other wrappers, for
example `urn:li:ayxDataset:(workspaceId,datasetId)`. Do not pass a full URN
directly to a follow-on tool. Extract the asset ID required by that tool and
pass it in the documented parameter, preserving the ID exactly. Remove only
a recognized wrapper; do not guess or otherwise transform the ID. For
example, a dataset preview passes the `datasetId` component:

```json
{"datasetId":"<datasetId>","limit":5}
```

Translate the business question into focused searches rather than broad
ones. Search relevance depends on the metadata indexed for each asset, so
an asset with thin metadata may only be findable by name or path — try
both.

### Agent Workflow

1. **Frame the need.** Extract the business objective, domain terms,
   expected inputs and outputs, time range, geography, system names, and
   other constraints worth searching on.
2. **Search broadly.** Search governed cloud assets with the tools above.
3. **Inspect candidates.** Never rely on names or search snippets alone.
   Establish a workflow's or macro's analytical purpose, inputs, outputs,
   transformations, and referenced assets, and verify a dataset's column
   names and types before presenting it as grounded context.
4. **Use workflows as discovery evidence.** Treat workflow Input and Output
   tools as pointers to canonical datasets, and referenced macros as
   reusable-logic candidates. Follow them to assets that did not match the
   original query.
5. **Refine and repeat.** Re-search using names, business concepts, owners,
   tags, and tool usage learned during inspection — but only when it could
   materially improve the candidate set.
6. **Return grounded candidates.** Report what was found, why each
   candidate is relevant, what was verified, and every gap.

### Effort

Scale depth to the request shape:

- **Locate-only** — "Which workflows relate to customer churn?" Rank
  candidates and verify enough to justify each one's relevance. Do not
  inspect every candidate in depth.
- **Grounding a build** — "I need a weekly churn dataset." Verify the
  leading candidates properly: schemas for datasets that may be consumed,
  condensed inspection for workflows that may be reused.

When the request shape is ambiguous, treat it as locate-only and either
offer to verify further or record the ambiguity as a discovery gap, rather
than spending the deeper budget uninvited.

Stop searching at whichever comes first: the most recent refinement
produced no new material candidates, the evidence is sufficient for the
request shape, or five search iterations have completed. Inspect at most
five candidates in depth per request — rank first, then inspect in rank
order. These counts are tunable defaults, not protocol: when a bound stops
the work, report which bound was hit and what was left uninspected. Silent
truncation reads to the user as "nothing else exists."

### Reporting Rules

- **Surface reuse candidacy.** When an inspected asset appears to already
  fulfill the stated need in whole or in part, say so explicitly and cite
  the evidence. Do not decide whether to reuse, extend, or replace it at
  this stage — but never return a strong candidate without flagging that it
  may make new work unnecessary.
- **Report empty results as searched, not absent.** Asset search returns
  only what the user has access to, so finding nothing does not establish
  that nothing exists. State which locations, asset types, and terms were
  searched, and offer the terms or locations most likely to change the
  outcome.
- **Keep evidence and inference separate.** Distinguish what was observed
  in metadata, workflow content, schemas, or samples from what you
  inferred.

A candidate is **verified** only when the evidence below was actually
obtained:

| Asset type | Required before labeling verified |
|---|---|
| Governed dataset | Column names and, where available, types, plus a bounded sample — or an explicit statement of why preview was unavailable or prohibited |
| Workflow or macro | Condensed inspection establishing analytical purpose, inputs, outputs, and referenced assets |

### Result Contract

Report in this structure. Always return a stable identifier —
`workflow_id` or `dataset_id` — for every candidate, so follow-up questions
can re-fetch a specific asset instead of re-running discovery.

```markdown
## Verified Candidates

### <Asset name> (<type>)
- Location / identifier:
- Relevance:
- Verified evidence:
- Evidence source:
- Already fulfills the need: yes / partially / no — with the evidence for that reading
- Related assets discovered:
- Freshness / ownership / certification:

## Unverified Candidates

### <Asset name> (<type>)
- Location / identifier:
- Why it may be relevant:
- What could not be verified, and why:

## Discovery Gaps

- <Access restriction, preview failure, unsearched location, or unresolved capability question>
- <Bound reached, and what was left uninspected>
```

Include a section even when it is empty, and say so. An omitted Discovery
Gaps section reads as "there were none."

## Alteryx Insights

Use Alteryx One Insights tools, powered by Alteryx Auto Insights, to answer
business data questions and deliver accurate, data-driven conclusions.

### Tool Call Error Handling

If a tool call fails:

1. Follow instructions from the error message if clear.
2. For network issues, retry once before asking the user.
3. If clarification is needed, request only what is necessary, then retry;
   if unresolved, pivot or explain the issue.

### Core Rules

- Do not fabricate information.
- Use Alteryx One Insights tools for all analytics tasks performed through
  this section.
- Use the exact Insights tool names the active client exposes; client
  prefixes may vary.
- Always provide arguments when using Alteryx One Insights tools.
- Before using tools, identify the information required to answer the
  user's request. Use as few tool calls as possible: retrieve only missing
  information, combine requirements into one call where possible, avoid
  overlapping or confirmation-only calls, and stop once sufficient evidence
  is available.
- Respond professionally, concisely, and conversationally.
- Use exact parameter values from their authoritative Insights tool
  results: dataset names from `list_datasets`; measure and date-field names
  from `list_measures`; segment names from `list_segments`; and filter
  values from `search_for_filter`. Never guess or normalize these values.
  Reuse prior Insights metadata values only when they belong to the
  selected dataset and are not stale.
- Ground analysis performed through this section in Alteryx One and
  Alteryx Auto Insights; do not attribute results to other analytics tools.
- Use Cloud Asset Discovery above only when the request is to locate or
  verify reusable assets outside the Insights dataset-selection flow. Use
  Cloud Workflow Build, Inspect, Run, Repair above for workflow inspection,
  building, execution, repair, and validation.

### Analytics Question Handling

Use only the steps the request needs:

1. Use the Insights `list_datasets` tool to determine the relevant dataset
   and its exact name. Reuse a dataset name only when that exact name was
   established by a prior Insights `list_datasets` result and is not stale.
   Do not use asset discovery or asset search tools to select a dataset for
   Insights analysis.
2. Retrieve measures as needed to understand and execute the request.
   Retrieve segments only when filtering, breakdown, grouping, or record
   selection requires them.
3. Retrieve time parameters, filter values, or additional measures only
   when they are missing, ambiguous, stale, or required by the selected
   analysis. Use the date field returned by `list_measures` that matches
   the user's request; if more than one is plausible, ask which one; use
   that field's returned default granularity unless the user specifies
   another supported granularity.
4. If the request is still ambiguous after using available tools, ask only
   the minimum clarification needed. Otherwise, execute the analysis
   directly and stop when the result is sufficient.

### Analytics Results Output

- For direct questions, start with a concise one-sentence answer.
- Surface useful charts and real source links returned by the tools using
  the active client's supported representation rather than fixed HTML or
  Markdown. Preserve chart URI and `sourceLink` values exactly, add
  appropriate chart descriptions where supported, and never invent links.

## Alteryx Formula Function Reference

All potential formula functions and operators are contained in the following sections.

### Conditional

Conditional functions let you perform an action or calculation using an IF statement.

- `IF c THEN t ELSE f ENDIF`: Returns t if the condition c is true, else returns f.
- `IF c THEN t ELSEIF c2 THEN t2 ELSE f ENDIF`: Returns t if the first condition c is true, else returns t2 if the second condition c2 is true, else returns f.
- `IIF(bool, x, y)`: Returns x if bool is true, else returns y.
- `Switch(Value,Default,Case1,Result1,...,CaseN,ResultN)`: Compares a value against a list of cases and returns the corresponding result.

### String

String functions perform operations on text data. String functions can cleanse data, convert data to a different format or case, compute metrics about the data, or perform other manipulations.

- `Contains(String, Target, CaseInsensitive=1)`: Checks if String contains Target.
- `CountWords(string)`: Counts words separated by space.
- `DecomposeUnicodeForMatch(String)`: Removes accents and converts to lowercase narrow form.
- `EndsWith(String, Target, CaseInsensitive=1)`: Checks if String ends with Target.
- `FindNth(Initial String, Target, Instance)`: Finds nth occurrence of Target.
- `FindString(String,Target)`: Returns position of Target in String.
- `GetLeft(String, Delimiter)`: Returns left side before Delimiter.
- `GetPart(String, Delimiter, Index)`: Returns substring at Index.
- `GetRight(String, Delimiter)`: Returns right side after Delimiter.
- `GetWord(string, n)`: Returns nth word (0-based).
- `Left(String, len)`: Returns first len characters.
- `Length(String)`: Returns length of string.
- `LowerCase(String)`: Converts to lowercase.
- `MD5_ASCII(String)`: MD5 hash of ASCII string.
- `MD5_UNICODE(String)`: MD5 hash of UTF-16 string.
- `MD5_UTF8(String)`: MD5 hash of UTF-8 string.
- `PadLeft(String, len, char)`: Pads left to len.
- `PadRight(String, len, char)`: Pads right to len.
- `REGEX_CountMatches(String,pattern,icase)`: Count regex matches.
- `REGEX_Match(String,pattern,icase)`: Tests full regex match.
- `REGEX_Replace(String, pattern, replace, icase)`: Regex find/replace.
- `Replace(String, Target, Replacement)`: Replaces Target with Replacement.
- `ReplaceChar(String, y, z)`: Replaces characters y with z.
- `ReplaceFirst(String, Target, Replacement)`: Replaces first occurrence.
- `ReverseString(String)`: Reverses string.
- `Right(String, len)`: Returns last len characters.
- `StartsWith(String, Target, CaseInsensitive=1)`: Checks if String starts with Target.
- `STRCSPN(String, y)`: Returns length until first occurrence of y chars.
- `StripQuotes(String)`: Removes surrounding quotes.
- `STRSPN(String, y)`: Returns length of initial segment of chars in y.
- `Substring(String, start, length)`: Returns substring.
- `TitleCase(String)`: Converts to title case.
- `Trim(String, y)`: Trims y chars from both ends (default whitespace).
- `TrimLeft(String, y)`: Trims from start.
- `TrimRight(String, y)`: Trims from end.
- `Uppercase(String)`: Converts to uppercase.
- `UuidCreate()`: Creates a unique identifier.

### DateTime

DateTime functions perform an action or calculation on a date and time value.

- `DateTimeAdd(dt,i,u)`: Adds a specific interval to a date-time value.
- `DateTimeDay(dt)`: Returns the numeric value for the day of the month.
- `DateTimeDiff(dt1,dt2,u)`: Returns the difference between two dates as an integer.
- `DateTimeFirstOfMonth()`: Returns the first day of the month, at midnight.
- `DateTimeFormat(dt,f,[l],[tz])`: Converts date-time data from ISO format to another specified format.
- `DateTimeHour(dt)`: Returns the hour portion of the time.
- `DateTimeLastOfMonth()`: Returns the last day of the current month (23:59:59).
- `DateTimeMinutes(dt)`: Returns the minutes portion of the time.
- `DateTimeMonth(dt)`: Returns the numeric value for the month.
- `DateTimeNow([tz])`: Returns the current date and time, including seconds.
- `DateTimeNowPrecise(digits,[tz])`: Returns the current date and time with fractional seconds.
- `DateTimeParse(string,f,[l],[tzName])`: Converts a date string to standard format.
- `DateTimeQuarter(dt, [Q1 Start])`: Returns the quarter of the year.
- `DateTimeSeconds(dt)`: Returns the seconds portion of the time.
- `DateTimeStart()`: Returns the date/time when the workflow started.
- `DateTimeToday()`: Returns today’s date.
- `DateTimeToLocal(dt,[tz])`: Converts UTC date-time to local time zone.
- `DateTimeToUTC(dt,[tz])`: Converts date-time to UTC.
- `DateTimeTrim(dt,t)`: Removes unwanted portions of a date-time.
- `DateTimeWeekNum(dt, [StartOfWeek])`: Returns the week number of the year.
- `DateTimeWorkDays(dt1, dt2, [StartofWeek])`: Returns number of working days between two dates.
- `DateTimeYear(dt)`: Returns the numeric value for the year.
- `ToDate(x)`: Converts a string, number, or date-time to a date.
- `ToDateTime(x)`: Converts a string, number, or date to a date-time.

### Operators

An operator is a character that represents an action. Arithmetic operators can perform mathematical calculations and boolean operators work with true and false values.

- `/* Comment */`: Block comment.
- `// Comment`: Single-line comment.
- `&&`: Boolean AND operator.
- `AND`: Boolean AND keyword.
- `!`: Boolean NOT operator.
- `NOT`: Boolean NOT keyword.
- `OR`: Boolean OR keyword.
- `||`: Boolean OR operator.
- `=`: Equal to.
- `==`: Equal to.
- `>`: Greater than.
- `>=`: Greater than or equal.
- `<`: Less than.
- `<=`: Less than or equal.
- `!=`: Not equal to.
- `+`: Adds numbers, concatenates strings, or unions spatial objects.
- `-`: Subtracts numbers or removes one spatial object from another.
- `*`: Multiplies numbers.
- `/`: Divides numbers; always returns a double.
- `value IN (...)`: Returns True if value is in list.
- `value NOT IN (...)`: Returns True if value not in list.

#### Order of Precedence

This table shows the established order of operator groups. Operations within a group bind left to right.

| Order | Operators |
|--------|------------|
| 1 | `*`, `/` |
| 2 | `+`, `-` |
| 3 | `<=`, `<`, `>=`, `>`, `IN`, `NOT` |
| 4 | `=`, `!=` |
| 5 | `&&`, `AND`, `||`, `OR` |

### Conversion

Conversion functions convert numbers to strings or strings to numbers.

- `BinToInt(s)`: Converts the binary string s to an integer (limited to 53 bits).
- `CharFromInt(x)`: Returns the Unicode character that matches the input number x.
- `CharToInt(s)`: Returns the number that matches the input Unicode character s.
- `ConvertFromCodePage(s, codePage)`: Translates text from a code page to Unicode.
- `ConvertToCodePage(s, codePage)`: Translates text from Unicode encoding to a specific code page.
- `HexToNumber(x)`: Converts a HEX string to a number (limited to 53 bits).
- `IntToBin(x)`: Converts x to a binary string.
- `IntToHex(x)`: Converts x to a hexadecimal string.
- `ToDegrees(x)`: Converts a numeric radian value (x) to degrees.
- `ToNumber(x, [bIgnoreErrors], [keepNulls], [decimalSeparator])`: Converts a string (x) to a number.
- `ToRadians(x)`: Converts a numeric degree value (x) to radians.
- `ToString(x, numDec, [addThousandsSeparator], [decimalSeparator])`: Converts a numeric parameter (x) to a string using numDec decimal places.
- `UnicodeNormalize(String, Form)`: Converts text data into a standardized Unicode form.

### Math

Math functions perform mathematical calculations.

- `ABS(x)`: Returns absolute value of x.
- `ACOS(x)`: Returns arccosine.
- `ASIN(x)`: Returns arcsine.
- `ATAN(x)`: Returns arctangent.
- `ATAN2(y, x)`: Returns arctangent of y/x.
- `Average(n1, ...)`: Average of a list of numbers.
- `AverageNonNull(n1, ...)`: Average excluding nulls.
- `CEIL(x, [mult])`: Rounds up to nearest multiple.
- `COS(x)`: Cosine of x.
- `COSH(x)`: Hyperbolic cosine.
- `DISTANCE(from_Lat,from_Lon,to_Lat,to_Lon)`: Distance between coordinates.
- `EXP(x)`: e^x.
- `FACTORIAL(x)`: Factorial of x.
- `FLOOR(x, [mult])`: Rounds down to nearest multiple.
- `LOG(x)`: Natural logarithm.
- `LOG10(x)`: Base-10 logarithm.
- `Median(...)`: Median of values.
- `Mod(n,d)`: Modulo operation.
- `PI()`: Constant π.
- `POW(x,e)`: x raised to e.
- `RAND()`: Random number [0,1).
- `RandInt(n)`: Random integer [0,n].
- `Round(x,mult)`: Rounds x to nearest multiple.
- `SIN(x)`: Sine.
- `SINH(x)`: Hyperbolic sine.
- `SmartRound(x)`: Dynamic rounding based on size.
- `SQRT(x)`: Square root.
- `TAN(x)`: Tangent.
- `TANH(x)`: Hyperbolic tangent.

### File

File functions build file paths, check to see if a file exists, or extract a part of a file path.

- `FileAddPaths(Path1, Path2)`: Adds two file path parts, ensuring one backslash between them.
- `FileExists(Path)`: Returns True if the file exists, else False.
- `FileGetDir(Path)`: Returns the directory portion of the path.
- `FileGetExt(Path)`: Returns the file extension.
- `FileGetFileName(Path)`: Returns the file name without extension.

### Finance

Finance functions apply financial algorithms or mathematical calculations.

- `FinanceCAGR(BeginningValue, EndingValue, NumYears)`: Calculates Compound Annual Growth Rate.
- `FinanceEffectiveRate(NominalRate, PaymentsPerYear)`: Calculates Effective Annual Interest Rate.
- `FinanceFV(Rate, NumPayments, PaymentAmount, PresentValue, PayAtPeriodBegin)`: Calculates Future Value.
- `FinanceFVSchedule(Principle, Year1Rate, Year2Rate)`: Calculates Future Value Schedule.
- `FinanceIRR(Value1, Value2)`: Calculates Internal Rate of Return.
- `FinanceMIRR(FinanceRate, ReinvestRate, Value1, Value2)`: Calculates Modified Internal Rate of Return.
- `FinanceMXIRR(FinanceRate, ReinvestRate, Value1, Date1, Value2, Date2)`: Modified IRR with dates.
- `FinanceNominalRate(EffectiveRate, PaymentsPerYear)`: Calculates Nominal Annual Interest Rate.
- `FinanceNPER(Rate, PaymentAmount, PresentValue, FutureValue, PayAtPeriodBegin)`: Number of periods.
- `FinanceNPV(Rate, Value1, Value2)`: Net Present Value.
- `FinancePMT(Rate, NumPayments, PresentValue, FutureValue, PayAtPeriodBegin)`: Loan payment amount.
- `FinancePV(Rate, NumPayments, PaymentAmount, FutureValue, PayAtPeriodBegin)`: Present Value.
- `FinanceRate(NumPayments, PaymentAmount, PresentValue, FutureValue, PayAtPeriodBegin)`: Interest rate per period.
- `FinanceXIRR(Value1, Date1, Value2, Date2)`: IRR with dates.
- `FinanceXNPV(Rate, Value1, Date1, Value2, Date2)`: NPV with dates.

### Bitwise

Bitwise functions operate on one or more bit patterns or binary numerals at the level of their individual bits.

- `BinaryAnd(n,m)`: Bitwise AND.
- `BinaryNot(n)`: Bitwise NOT.
- `BinaryOr(n,m)`: Bitwise OR.
- `BinaryXOr(n,m)`: Bitwise XOR.
- `ShiftLeft(n,b)`: Left shift by b bits.
- `ShiftRight(n,b)`: Right shift by b bits.

### Min/Max

Minimum or maximum functions find the smallest and largest value of a set of values.

- `BETWEEN(x, min, max)`: Tests if x is between min and max.
- `Bound(x, min, max)`: Clamps x to [min,max].
- `Max(v0, v1, ..., vn)`: Maximum value.
- `MaxIDX(v0, v1,..., vn)`: Index of maximum value.
- `Min(v0, v1,..., vn)`: Minimum value.
- `MinIDX(v0, v1,..., vn)`: Index of minimum value.

### Spatial

Spatial functions build spatial objects, analyze spatial data, and return metrics from spatial fields.

- `ST_Area(object, units)`: Area of spatial object.
- `ST_Boundary(object)`: Boundary of spatial object.
- `ST_BoundingRectangle(object, ...)`: Bounding rectangle.
- `ST_Centroid(object)`: Centroid of object.
- `ST_CentroidX(object)`: Longitude of centroid.
- `ST_CentroidY(object)`: Latitude of centroid.
- `ST_Combine(object1, object2,...)`: Combines spatial objects.
- `ST_Contains(object1,object2)`: True if object1 contains object2.
- `ST_ConvexHull(object1,...)`: Convex hull.
- `ST_CreateLine(point1, point2,...)`: Creates line from points.
- `ST_CreatePoint(x,y)`: Creates point.
- `ST_CreatePolygon(obj1, obj2,...)`: Creates polygon.
- `ST_Cut(object1,object2)`: Cuts object1 from object2.
- `ST_Dimension(object)`: Returns dimension (0=point, 1=line, 2=polygon).
- `ST_Distance(object1, object2, units)`: Distance between spatial objects.
- `ST_EndPoint(object)`: Last point of object.
- `ST_Intersection(object1, object2, ...)`: Intersection of spatial objects.
- `ST_Intersects(object1, object2, ...)`: True if objects intersect.
- `ST_InverseIntersection(object1, object2, ...)`: Inverse intersection.
- `ST_Length(object, units)`: Linear length.
- `ST_MD5(object)`: MD5 hash of spatial object.
- `ST_MaxX(object)`: Max longitude.
- `ST_MaxY(object)`: Max latitude.
- `ST_MinX(object)`: Min longitude.
- `ST_MinY(object)`: Min latitude.
- `ST_NumParts(object)`: Number of parts.
- `ST_NumPoints(object)`: Number of points.
- `ST_ObjectType(object)`: Type of spatial object.
- `ST_PointN(object, n)`: Nth point.
- `ST_RandomPoint(object)`: Random point.
- `ST_Relate(object1,object2,relation)`: True if objects satisfy DE-9IM relation.
- `ST_StartPoint(object)`: First point.
- `ST_Touches(object1, object2)`: True if objects touch.
- `ST_TouchesOrIntersects(object1, object2)`: True if touch or intersect.
- `ST_Within(object1, object2)`: True if object1 within object2.

### Specialized

These functions perform a variety of specialized actions and can be used with all data types.

- `Coalesce(v1,v2,v3,…,vn)`: Returns first non-null value.
- `EscapeXMLMetacharacters(String)`: Escapes XML metacharacters.
- `GetVal(index, v0,...vn)`: Returns value by 0-based index.
- `GetEnvironmentVariable(Name)`: Returns environment variable value.
- `Message(messageType, message, returnValue)`: Outputs message and value when condition met.
- `NULL()`: Returns Null.
- `RangeMedian(...)`: Median from aggregated ranges.
- `ReadRegistryString(Key, ValueName, DefaultValue="")`: Reads registry value.
- `Soundex(String)`: Returns Soundex code of string.
- `Soundex_Digits(String)`: Returns first 4 digits or Soundex code.
- `TOPNIDX(N, v0, v1, ..., vn)`: Index of Nth from max value.
- `UrlEncode(String)`: Legacy UTF-16 percent-encoding (use UrlEncodeUTF8 instead).
- `UrlEncodeUTF8(String)`: RFC 3986-compliant percent-encoding.

### Test

Test functions perform data comparisons. Test functions can identify the data type of a value or determine if a value exists.

- `CompareDictionary(a,b)`: Case-insensitive compare with numeric sort.
- `CompareDigits(a,b,nNumDigits)`: Compares numbers to given precision.
- `CompareEpsilon(a,b,epsilon)`: Compares floats within epsilon.
- `EqualStrings(a,b)`: Tests if strings are identical.
- `IsEmpty(v)`: True if v is NULL or empty string.
- `IsInteger(v)`: True if v can be converted to integer.
- `IsLowerCase(String)`: True if all alphabetic chars lowercase.
- `IsNull(v)`: True if v is NULL.
- `IsNumber(v)`: True if v is numeric type.
- `IsSpatialObj(v)`: True if v is spatial object.
- `IsString(v)`: True if v is string type.
- `IsUpperCase(String)`: True if all alphabetic chars uppercase.

### Null Handling

This table demonstrates how Alteryx handles Nulls. The same handling applies to numbers and strings.

| Data1 | Data2 | `>` | `<` | `==` | `!=` |
|:------|:------|:---:|:---:|:----:|:----:|
| 1 | Null | False | False | False | True |
| 0 | Null | False | False | False | True |
| Null | Null | False | False | True | False |

- `1 + Null() == Null()`: Adding a number to Null returns Null.
- `"ABC" + Null() == "ABC"`: Adding a string to Null returns the string.
- `Null() < Null()` and `Null() > Null()`: Always return False.
- `Null() <= Null()` and `Null() >= Null()`: Return True only when both sides are Null.
- Comparisons (`<`, `>`, `=`, `!=`) with Null generally return False, except `Null() == Null()` is True.

## Designer Native Tool Catalog

A discovery aid listing native Designer tools by palette, with each tool's
human-readable name and input/output anchor names. Use `tool_name` values
here when calling `add_tool` / `edit_tool`; `plugin_name` is informational
only, since cloud mode configures tools through the schema-driven MCP
operations rather than raw plugin XML. This catalog is a discovery aid, not
an authoritative schema — always confirm exact configuration fields with
`get_tool_configuration_schema` before adding or editing a tool.

```yaml
- tool_name: Auto Insights Uploader
  plugin_name: AutoInsightsUploader
  tool_palette: In/Out
  tool_description: Sends data directly from Designer to Alteryx Auto Insights.
    Use Auto Insights to create compelling visualizations, insights, and reports.
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Browse
  plugin_name: AlteryxBasePluginsGui.BrowseV2.BrowseV2
  tool_palette: In/Out
  tool_description: displays data from a connected tool as well as data profile information,
    maps, reporting snippets, and behavior analysis information in the data
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Date Time Now
  plugin_name: DateTimeNow
  tool_palette: In/Out
  tool_description: inputs the current date and time at workflow runtime in the format
    you choose
  connections:
    inputs: []
    outputs:
    - DTN
- tool_name: Directory
  plugin_name: AlteryxBasePluginsGui.Directory.Directory
  tool_palette: In/Out
  tool_description: returns a list of files that are contained in a directory and
    relevant file attributes
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Input Data
  plugin_name: AlteryxBasePluginsGui.DbFileInput.DbFileInput
  tool_palette: In/Out
  tool_description: reads data into your workflow by reading from a file or connecting
    to a database
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Map Input
  plugin_name: AlteryxSpatialPluginsGui.MapInput.MapInput
  tool_palette: In/Out
  tool_description: allows you to manually draw or select map objects (point, lines,
    and polygons) to be stored in the workflow
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Output Data
  plugin_name: AlteryxBasePluginsGui.DbFileOutput.DbFileOutput
  tool_palette: In/Out
  tool_description: sends the contents of a data stream to a file or database
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Text Input
  plugin_name: AlteryxBasePluginsGui.TextInput.TextInput
  tool_palette: In/Out
  tool_description: allows you to create a stream of data inside a workflow without
    a dependency on a separate file or database. The data set becomes part of the
    workflow
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Auto Field
  plugin_name: AlteryxBasePluginsGui.AutoField.AutoField
  tool_palette: Preparation
  tool_description: automatically sets the column type for each string column to the
    smallest possible size and type that will accommodate the data (excellent for
    automatically fixing the data types when reading from a csv)
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Create Samples
  plugin_name: Predictive Tools\Create_Samples.yxmc
  tool_palette: Preparation
  tool_description: splits the input records into 2 or 3 random samples
  connections:
    inputs:
    - Input
    outputs:
    - Estimation
    - Validation
    - Holdout
- tool_name: Data Cleanse Pro
  plugin_name: AlteryxBasePluginsGui.DataCleansePro.DataCleansePro
  tool_palette: Preparation
  tool_description: performs basic data cleansing operations such as replacing null
    values, removing punctuation, and modifying capitalization
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Data Cleansing
  plugin_name: Cleanse.yxmc
  tool_palette: Preparation
  tool_description: performs basic data cleansing operations such as replacing null
    values, removing punctuation, and modifying capitalization
  connections:
    inputs:
    - Input2
    outputs:
    - Output26
- tool_name: Filter
  plugin_name: AlteryxBasePluginsGui.Filter.Filter
  tool_palette: Preparation
  tool_description: splits a data stream into 2 streams based on a conditional expression
  connections:
    inputs:
    - Input
    outputs:
    - 'True'
    - 'False'
- tool_name: Formula
  plugin_name: AlteryxBasePluginsGui.Formula.Formula
  tool_palette: Preparation
  tool_description: perform a variety of calculations and operations to create new
    data columns or update existing columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Generate Rows
  plugin_name: AlteryxBasePluginsGui.GenerateRows.GenerateRows
  tool_palette: Preparation
  tool_description: creates new rows of data
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Imputation
  plugin_name: Imputation_v3.yxmc
  tool_palette: Preparation
  tool_description: replaces problematic numeric values (such as nulls) for specified
    columns with another value such as the median or a user-specified value
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Multi-Field Binning
  plugin_name: MultiFieldBinning_v2.yxmc
  tool_palette: Preparation
  tool_description: assigns data into bins based on the values in one or more columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Multi-Field Formula
  plugin_name: AlteryxBasePluginsGui.MultiFieldFormula.MultiFieldFormula
  tool_palette: Preparation
  tool_description: executes a single function on multiple columns at once
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Multi-Row Formula
  plugin_name: AlteryxBasePluginsGui.MultiRowFormula.MultiRowFormula
  tool_palette: Preparation
  tool_description: allows you to utilize row data as part of the formula creation;
    it is helpful for parsing complex data, creating running totals, and other such
    calculations
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Oversample Field
  plugin_name: Predictive Tools\Oversample_Field.yxmc
  tool_palette: Preparation
  tool_description: samples incoming data, oversampling a specific field to ensure
    equal representation of a specified value for effective use in a predictive model
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Random % Sample
  plugin_name: RandomRecords.yxmc
  tool_palette: Preparation
  tool_description: output a specified number or percent of rows obtained via a random
    sample of the input data
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Rank
  plugin_name: AlteryxBasePluginsGui.Rank.Rank
  tool_palette: Preparation
  tool_description: rank your data for further processing or output. Select one or
    more ranking types (ordinal, dense, standard, modified competition, fractional),
    sort by columns, specify order, and optionally group by columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Record ID
  plugin_name: AlteryxBasePluginsGui.RecordID.RecordID
  tool_palette: Preparation
  tool_description: assigns a unique identifier to each row in a dataset
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Sample
  plugin_name: AlteryxBasePluginsGui.Sample.Sample
  tool_palette: Preparation
  tool_description: extracts a specified portion of the rows in a data stream
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Select
  plugin_name: AlteryxBasePluginsGui.AlteryxSelect.AlteryxSelect
  tool_palette: Preparation
  tool_description: select the columns that flow downstream, rename and reorder columns,
    modify data types, and add column descriptions
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Select Records
  plugin_name: SelectRecords.yxmc
  tool_palette: Preparation
  tool_description: chooses a precise subset of input rows
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Sort
  plugin_name: AlteryxBasePluginsGui.Sort.Sort
  tool_palette: Preparation
  tool_description: arranges rows in a table in an ascending or descending alphanumeric
    order based on the values of one or more specified data columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Tile
  plugin_name: AlteryxBasePluginsGui.Tile.Tile
  tool_palette: Preparation
  tool_description: group data into sets (tiles) based on value ranges in a column
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Unique
  plugin_name: AlteryxBasePluginsGui.Unique.Unique
  tool_palette: Preparation
  tool_description: separates data into two streams, duplicate and unique rows, based
    on the specified columns
  connections:
    inputs:
    - Input
    outputs:
    - Unique
    - Duplicates
- tool_name: Append Fields
  plugin_name: AlteryxBasePluginsGui.AppendFields.AppendFields
  tool_palette: Join
  tool_description: adds the columns from a source input to every row in a target
    input
  connections:
    inputs:
    - Targets
    - Source
    outputs:
    - Output
- tool_name: Find Replace
  plugin_name: AlteryxBasePluginsGui.FindReplace.FindReplace
  tool_palette: Join
  tool_description: finds instances where a string contains a lookup list value, and
    either replaces it or appends additional columns to the table when a match is
    found
  connections:
    inputs:
    - Targets
    - Source
    outputs:
    - Output
- tool_name: Fuzzy Match
  plugin_name: AlteryxBasePluginsGui.FuzzyMatch.FuzzyMatch
  tool_palette: Join
  tool_description: identifies rows with similar string values in specified columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Join
  plugin_name: AlteryxBasePluginsGui.Join.Join
  tool_palette: Join
  tool_description: combines 2 data streams based on common fields or record position
  connections:
    inputs:
    - Left
    - Right
    outputs:
    - Left
    - Right
    - Join
- tool_name: Join Multiple
  plugin_name: AlteryxBasePluginsGui.JoinMultiple.JoinMultiple
  tool_palette: Join
  tool_description: puts together 2 or more inputs based on a shared feature (either
    the same position or a shared column)
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Make Group
  plugin_name: AlteryxBasePluginsGui.MakeGroup.MakeGroup
  tool_palette: Join
  tool_description: assembles pairs of matches into groups based on their relationships,
    commonly used in conjunction with the Fuzzy Match tool
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Union
  plugin_name: AlteryxBasePluginsGui.Union.Union
  tool_palette: Join
  tool_description: combines multiple data streams based on common column names or
    positions
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: DateTime
  plugin_name: AlteryxBasePluginsGui.DateTime.DateTime
  tool_palette: Parse
  tool_description: transforms datetime data to and from a variety of formats, including
    both expression-friendly and human readable formats
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: RegEx
  plugin_name: AlteryxBasePluginsGui.RegEx.RegEx
  tool_palette: Parse
  tool_description: leverages the powerful pattern matching abilities of regular expression
    syntax for the sake of parsing, matching, or replacing string data
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Text To Columns
  plugin_name: AlteryxBasePluginsGui.TextToColumns.TextToColumns
  tool_palette: Parse
  tool_description: splits the text from 1 column into separate rows or columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: XML Parse
  plugin_name: AlteryxBasePluginsGui.XMLParse.XMLParse
  tool_palette: Parse
  tool_description: parses out information from structured data into separate columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Arrange
  plugin_name: AlteryxBasePluginsGui.Arrange.Arrange
  tool_palette: Transform
  tool_description: Allows you to manually transpose and rearrange data columns for
    presentation purposes. Data is transformed so that each row is turned into multiple
    rows. Columns can be created by using column description data.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Count Records
  plugin_name: CountRecords.yxmc
  tool_palette: Transform
  tool_description: returns a simple count of the number of rows passing through a
    data stream
  connections:
    inputs:
    - Input8
    outputs:
    - Output9
- tool_name: Cross Tab
  plugin_name: AlteryxBasePluginsGui.CrossTab.CrossTab
  tool_palette: Transform
  tool_description: creates 1 new column for each categorical value held in a single
    existing column, pivoting the data from a vertical layout to a more horizontal
    layout
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Make Columns
  plugin_name: AlteryxBasePluginsGui.MakeColumns.MakeColumns
  tool_palette: Transform
  tool_description: Take rows of data (records) and arrange them into multiple columns.
    You can specify how many columns to create and how you want to display the result
    (horizontally or vertically). This tool is useful for reporting or display purposes
    where you want to layout records to fit nicely within a table.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Running Total
  plugin_name: AlteryxBasePluginsGui.RunningTotal.RunningTotal
  tool_palette: Transform
  tool_description: calculate a cumulative sum per row in a data stream
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Summarize
  plugin_name: AlteryxSpatialPluginsGui.Summarize.Summarize
  tool_palette: Transform
  tool_description: perform a host of summary calculations, including summing, min
    and max, grouping, counting, string concatenating, math, and spatial object processing
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Transpose
  plugin_name: AlteryxBasePluginsGui.Transpose.Transpose
  tool_palette: Transform
  tool_description: moves values held in multiple horizontal columns into a single
    column
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Weighted Average
  plugin_name: WeightedAvg.yxmc
  tool_palette: Transform
  tool_description: calculates the weighted average of an incoming data column using
    weights from a second column
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Browse In-DB
  plugin_name: LockInGui.LockInBrowse.LockInBrowse
  tool_palette: In-Database
  tool_description: View your data at any point in an In-DB workflow. Each Browse
    In-DB tool triggers a database query and can impact performance.
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Connect In-DB
  plugin_name: LockInGui.LockInInput.LockInInput
  tool_palette: In-Database
  tool_description: to create an in-database connection in a workflow
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Data Stream In
  plugin_name: LockInGui.LockInStreamIn.LockInStreamIn
  tool_palette: In-Database
  tool_description: bring data from Alteryx Designer into an In-DB workflow
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Data Stream Out
  plugin_name: LockInGui.LockInStreamOut.LockInStreamOut
  tool_palette: In-Database
  tool_description: streams data from an In-DB stream to a standard workflow, with
    an option to sort the records
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Dynamic Input In-DB
  plugin_name: LockInGui.LockInDynamicInput.LockInDynamicInput
  tool_palette: In-Database
  tool_description: take In-DB Connection Name and Query fields from a standard data
    stream and input them back into an In-DB data stream
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Dynamic Output In-DB
  plugin_name: LockInGui.LockInDynamicOutput.LockInDynamicOutput
  tool_palette: In-Database
  tool_description: output information about the In-DB workflow to fields in a standard
    workflow stream
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Filter In-DB
  plugin_name: LockInGui.LockInFilter.LockInFilter
  tool_palette: In-Database
  tool_description: filter In-DB records with a basic filter or with a custom expression
    using the database's native language like SQL
  connections:
    inputs:
    - Input
    outputs:
    - 'True'
    - 'False'
- tool_name: Formula In-DB
  plugin_name: LockInGui.LockInFormula.LockInFormula
  tool_palette: In-Database
  tool_description: create or update fields in an In-DB data stream with an expression
    using the native language of the database, like SQL
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Join In-DB
  plugin_name: LockInGui.LockInJoin.LockInJoin
  tool_palette: In-Database
  tool_description: combines two In-DB streams based on common fields by performing
    an inner or outer join
  connections:
    inputs:
    - Left
    - Right
    outputs:
    - Output
- tool_name: Macro Input In-DB
  plugin_name: LockInGui.LockInMacroInput.LockInMacroInput
  tool_palette: In-Database
  tool_description: create an In-DB input connection on a macro and populates it with
    placeholder values
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Macro Output In-DB
  plugin_name: LockInGui.LockInMacroOutput.LockInMacroOutput
  tool_palette: In-Database
  tool_description: creates an In-DB output connection on a macro that can be used
    with In-DB workflows
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Sample In-DB
  plugin_name: LockInGui.LockInSample.LockInSample
  tool_palette: In-Database
  tool_description: limits the In-DB stream to a number or percentage of records
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Select In-DB
  plugin_name: LockInGui.LockInSelect.LockInSelect
  tool_palette: In-Database
  tool_description: to select, deselect, rename, and reorder fields in an In-DB workflow
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Summarize In-DB
  plugin_name: LockInGui.LockInSummarize.LockInSummarize
  tool_palette: In-Database
  tool_description: summarize data within a database by grouping, summing, counting,
    counting distinct records, and more
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Union In-DB
  plugin_name: LockInGui.LockInUnion.LockInUnion
  tool_palette: In-Database
  tool_description: combine 2 or more In-DB data streams with similar data structures
    based on field names or field positions
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Write Data In-DB
  plugin_name: LockInGui.LockInOutput.LockInOutput
  tool_palette: In-Database
  tool_description: create or update a table directly in the database
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Email
  plugin_name: PortfolioPluginsGui.Email.Email
  tool_palette: Reporting
  tool_description: send an email for each record with attachments or email-generated
    reports
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Image
  plugin_name: PortfolioPluginsGui.ComposerImage.PortfolioComposerImage
  tool_palette: Reporting
  tool_description: Creates an image element to be output in a report via the Render
    tool or viewed with the Browse tool.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Interactive Chart
  plugin_name: PlotlyCharting
  tool_palette: Reporting
  tool_description: Creates interactive bar charts, line graphs, scatter plots, and
    pie charts for visualizing data and creating reports.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Layout
  plugin_name: PortfolioPluginsGui.ComposerLayout.PortfolioComposerLayout
  tool_palette: Reporting
  tool_description: Arrange two or more reporting snippets horizontally or vertically
    for output via the Render tool.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Map Legend Builder
  plugin_name: Legend_Builder.yxmc
  tool_palette: Reporting
  tool_description: Take the components that are output from the Map Legend Splitter
    tool and build them back into a legend table, perhaps after changing the data
    between them to create a custom legend.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Map Legend Splitter
  plugin_name: Legend_Splitter.yxmc
  tool_palette: Reporting
  tool_description: Split a legend created by the Report Map tool into individual
    component columns.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Overlay
  plugin_name: PortfolioPluginsGui.ComposerOverlay.Overlay
  tool_palette: Reporting
  tool_description: Place reporting snippets (maps, text, images, etc.) on top of
    one another.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Render
  plugin_name: PortfolioPluginsGui.ComposerRender.PortfolioComposerRender
  tool_palette: Reporting
  tool_description: creates presentation quality reports in HTML, PCXML, PDF, RTF,
    DOCX, XLSX, PPTX, MHT, PNG, and ZIP file formats
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Report Footer
  plugin_name: FooterMacro.yxmc
  tool_palette: Reporting
  tool_description: Creates a footer reporting element that can be added to a report.
  connections:
    inputs:
    - Report Layout
    outputs:
    - Output
- tool_name: Report Header
  plugin_name: ReportHeader\Supporting_Macros\RHEngine.yxmc
  tool_palette: Reporting
  tool_description: Creates a header reporting element that can be added to a report.
  connections:
    inputs:
    - Report Layout
    outputs:
    - Output13
- tool_name: Report Map
  plugin_name: AlteryxSpatialPluginsGui.ReportMap.ReportMap
  tool_palette: Reporting
  tool_description: Creates a map element to output in a report via the Render tool.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Report Text
  plugin_name: PortfolioPluginsGui.ComposerText.PortfolioComposerText
  tool_palette: Reporting
  tool_description: Creates a text element to output in a report via the Render tool.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Table
  plugin_name: PortfolioPluginsGui.ComposerTable.PortfolioComposerTable
  tool_palette: Reporting
  tool_description: Creates a table element to output in a report via the Render tool.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Comment
  plugin_name: AlteryxGuiToolkit.TextBox.TextBox
  tool_palette: Documentation
  tool_description: adds annotation to the workflow
  connections:
    inputs: []
    outputs: []
- tool_name: Explorer Box
  plugin_name: AlteryxGuiToolkit.HtmlBox.HtmlBox
  tool_palette: Documentation
  tool_description: Displays a web page, file directory, or file on the canvas.
  connections:
    inputs: []
    outputs: []
- tool_name: Tool Container
  plugin_name: AlteryxGuiToolkit.ToolContainer.ToolContainer
  tool_palette: Documentation
  tool_description: Organize and segment tools on the Designer canvas into a single
    box which can be collapsed or disabled.
  connections:
    inputs: []
    outputs: []
- tool_name: Buffer
  plugin_name: AlteryxSpatialPluginsGui.Buffer.Buffer
  tool_palette: Spatial
  tool_description: takes any polygon or polyline spatial object and expands or contracts
    its extents by a user specified value
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Create Points
  plugin_name: AlteryxSpatialPluginsGui.CreatePoints.CreatePoints
  tool_palette: Spatial
  tool_description: Create a spatial object from latitude/longitude coordinates.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Distance
  plugin_name: AlteryxSpatialPluginsGui.Distance.Distance
  tool_palette: Spatial
  tool_description: calculates the distance or drive time between a point and another
    point, line, or polygon
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Find Nearest
  plugin_name: AlteryxSpatialPluginsGui.FindNearest.FindNearest
  tool_palette: Spatial
  tool_description: identifies the shortest distance between spatial objects in one
    file and the objects in a second file
  connections:
    inputs:
    - Targets
    - Universe
    outputs:
    - Matched
    - Unmatched
- tool_name: Generalize
  plugin_name: AlteryxSpatialPluginsGui.Generalize.Generalize
  tool_palette: Spatial
  tool_description: Simplifies the boundary of a polygon or polyline by decreasing
    the number of nodes
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Heat Map
  plugin_name: HeatMap.yxmc
  tool_palette: Spatial
  tool_description: generates polygons representing different levels of heat in a
    given area
  connections:
    inputs:
    - Input
    outputs:
    - Output80
- tool_name: Make Grid
  plugin_name: AlteryxSpatialPluginsGui.MakeGrid.MakeGrid
  tool_palette: Spatial
  tool_description: Takes a spatial object (point or polygon) and creates a grid based
    on the object.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Non Overlapping Drivetime
  plugin_name: Non_Overlapping_DT.yxmc
  tool_palette: Spatial
  tool_description: Create drivetime trade areas, that do not overlap, for a point
    file.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Poly-Build
  plugin_name: AlteryxSpatialPluginsGui.PolyBuild.PolyBuild
  tool_palette: Spatial
  tool_description: used to combine points into a single polygon or polyline.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Poly-Split
  plugin_name: AlteryxSpatialPluginsGui.PolySplit.PolySplit
  tool_palette: Spatial
  tool_description: used to break a polygon into its points or regions
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Smooth
  plugin_name: AlteryxSpatialPluginsGui.Smooth.Smooth
  tool_palette: Spatial
  tool_description: Rounds off the sharp angles of polygon or polyline objects by
    adding nodes along the lines of the object
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Spatial Info
  plugin_name: AlteryxSpatialPluginsGui.SpatialInfo.SpatialInfo
  tool_palette: Spatial
  tool_description: returns information about spatial objects in the data
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Spatial Match
  plugin_name: AlteryxSpatialPluginsGui.SpatialMatch.SpatialMatch
  tool_palette: Spatial
  tool_description: Establishes the spatial relationship (contains, intersects, touches)
    between 2 sets of spatial objects.
  connections:
    inputs:
    - Targets
    - Universe
    outputs:
    - Matched
    - Unmatched
- tool_name: Spatial Process
  plugin_name: AlteryxSpatialPluginsGui.SpatialProcess.SpatialProcess
  tool_palette: Spatial
  tool_description: performs high level spatial operations to edit polygon spatial
    objects.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Trade Area
  plugin_name: AlteryxSpatialPluginsGui.TradeArea.TradeArea
  tool_palette: Spatial
  tool_description: creates regions around specified point objects in the input file
    based on a radius or a drivetime
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Assisted Modeling
  plugin_name: Modeling
  tool_palette: Machine Learning
  tool_description: create a machine-learning pipeline that selects a target, sets
    data types, cleans up missing values, selects a machine-learning algorithm, and
    trains a model
  connections:
    inputs:
    - Data
    outputs:
    - Model
- tool_name: Build Features
  plugin_name: BuildFeatures
  tool_palette: Machine Learning
  tool_description: create features and establish relationships between data in separate
    tables
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Classification
  plugin_name: Classification
  tool_palette: Machine Learning
  tool_description: Use as part of a machine-learning pipeline to identify what category
    a target belongs to. Choose from Random Forest, Decision Tree, XGBoost, and Logistic
    Regression algorithms.
  connections:
    inputs:
    - Model
    outputs:
    - Model
- tool_name: Data Health
  plugin_name: DataHealth
  tool_palette: Machine Learning
  tool_description: check on the health of your data by analyzing missing values,
    outliers, and sparsity
  connections:
    inputs:
    - Input
    outputs:
    - Scores
    - Report
    - Outliers
- tool_name: Feature Types
  plugin_name: FeatureTypes
  tool_palette: Machine Learning
  tool_description: automatically identify what types of features are in your data,
    or change the types manually
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Fit
  plugin_name: Fit
  tool_palette: Machine Learning
  tool_description: the final tool in a machine-learning pipeline that takes input
    from your dataset and the other machine learning tools, and then "fits" the model
    to the data and outputs the model
  connections:
    inputs:
    - Model
    outputs:
    - Model
- tool_name: Predict
  plugin_name: Predict
  tool_palette: Machine Learning
  tool_description: make predictions about new data using a machine learning pipeline
    you've built
  connections:
    inputs:
    - Data
    - Model
    outputs:
    - PredictedData
- tool_name: Regression
  plugin_name: Regression
  tool_palette: Machine Learning
  tool_description: Use as part of a machine-learning pipeline to identify a trend.
    Choose an algorithm from Decision Tree, Random Forest, and Linear Regression.
  connections:
    inputs:
    - Model
    outputs:
    - Model
- tool_name: Transformation
  plugin_name: Transformation
  tool_palette: Machine Learning
  tool_description: perform data-prep tasks such as setting data types, cleaning up
    missing values, selecting features, and encoding data
  connections:
    inputs:
    - Model
    outputs:
    - Model
- tool_name: Named Entity Recognition
  plugin_name: NamedEntityRecognition
  tool_palette: Text Mining
  tool_description: identifies entities, like people, places, and things, in text
  connections:
    inputs:
    - Data
    - Entities
    outputs:
    - Data
    - Model
- tool_name: Part-of-Speech Tagger
  plugin_name: PartOfSpeechTagger
  tool_palette: Text Mining
  tool_description: identifies parts of speech like nouns, verbs, and adjectives from
    text
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Sentiment Analysis
  plugin_name: Sentiment
  tool_palette: Text Mining
  tool_description: determine whether text responses or comments are more positive
    or negative
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Text Classification
  plugin_name: TextClassification
  tool_palette: Text Mining
  tool_description: trains and outputs a text classification model based on your training
    data, using either Multinomial Naive Bayes or Linear SVC algorithms, or choosing
    one of those automatically
  connections:
    inputs:
    - Training
    - Validation
    outputs:
    - Model
    - Evaluation
- tool_name: Text Pre-processing
  plugin_name: TextPreProcessing
  tool_palette: Text Mining
  tool_description: prepares your data for analysis with the Text Mining tools, normalizing
    text by converting to word roots and filtering digits, punctuation, and stop words
  connections:
    inputs:
    - Input
    - Stopwords
    outputs:
    - Output
- tool_name: Text Summary
  plugin_name: TextSummary
  tool_palette: Text Mining
  tool_description: summarizes bodies of text
  connections:
    inputs:
    - Data
    outputs:
    - Summary
- tool_name: Topic Modeling
  plugin_name: TopicModel
  tool_palette: Text Mining
  tool_description: identifies and categorizes topics in a body of text
  connections:
    inputs:
    - Input
    outputs:
    - Data
    - Report
    - Model
- tool_name: Word Cloud
  plugin_name: WordCloud
  tool_palette: Text Mining
  tool_description: creates visualizations from text data
  connections:
    inputs:
    - imageInput
    - textInput
    outputs:
    - Output
- tool_name: Zero-shot Text Classification
  plugin_name: ZeroShotTextClassification
  tool_palette: Text Mining
  tool_description: assigns scored categories to bodies of text based on a category
    list you define
  connections:
    inputs:
    - Data
    - Labels
    outputs:
    - Data
- tool_name: Barcode
  plugin_name: Barcode
  tool_palette: Computer Vision
  tool_description: decode (read) or encode (create) QR codes and barcodes
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Image Input
  plugin_name: PDFInput
  tool_palette: Computer Vision
  tool_description: bring images into Designer as a BLOB column
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Image Output
  plugin_name: ImageOutput
  tool_palette: Computer Vision
  tool_description: save image files to a location you choose
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Image Processing
  plugin_name: ImageProcessing
  tool_palette: Computer Vision
  tool_description: perform actions on images including align, threshold, scale, shift,
    crop, balance brightness, and convert to greyscale
  connections:
    inputs:
    - Input
    - Optional
    outputs:
    - Output
- tool_name: Image Profile
  plugin_name: ImageProfile
  tool_palette: Computer Vision
  tool_description: extracts helpful information from images
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Image Recognition
  plugin_name: ImageRecognition
  tool_palette: Computer Vision
  tool_description: Build a machine learning model that can classify images by group.
    You can use your own data and labels to train a new model, or you can use one
    of the pre-trained models provided
  connections:
    inputs:
    - Training
    - Validation
    outputs:
    - ModelOutput
    - EvaluationMetrics
    - ModelReport
- tool_name: Image Template
  plugin_name: ImageTemplate
  tool_palette: Computer Vision
  tool_description: Create templates for images using annotations or automatically
    detect tables in incoming image BLOBs
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Image to Text
  plugin_name: ImageToText
  tool_palette: Computer Vision
  tool_description: extract text from BLOB images using optical character recognition
    (OCR)
  connections:
    inputs:
    - dataInput
    - templateInput
    outputs:
    - Output
- tool_name: PDF to Text
  plugin_name: PDFToText
  tool_palette: Computer Vision
  tool_description: extracts text and graphic-encoded text from PDF files
  connections:
    inputs:
    - Data Input
    - Template Input
    outputs:
    - Output
- tool_name: LLM Override
  plugin_name: LLMOverride
  tool_palette: GenAI
  tool_description: configures a single LLM connection to enable AI-powered tools
    and Macros in the workflow
  connections:
    inputs: []
    outputs:
    - ModelConfig
- tool_name: Prompt
  plugin_name: Prompt
  tool_palette: GenAI
  tool_description: sends prompts to a LLM and return the generated responses, and
    adjust prompts and parameters to shape the output. When adding, inform the user
    they must configure Provider and Model themselves, as well as set up an Active
    Link if one does not already exist. An LLM Override tool is not needed.
  connections:
    inputs:
    - Model
    - Input
    outputs:
    - Output
- tool_name: Action
  plugin_name: AlteryxGuiToolkit.Action.Action
  tool_palette: Interface
  tool_description: Used in applications or macros. The Action tool updates configurations
    within a workflow based on values provided by other interface tools' questions
  connections:
    inputs:
    - Question
    outputs:
    - Action
- tool_name: Check Box
  plugin_name: AlteryxGuiToolkit.Questions.CheckBoxGroup.CheckBoxGroup
  tool_palette: Interface
  tool_description: creates a True or False value in an application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Condition
  plugin_name: AlteryxGuiToolkit.Condition.Condition
  tool_palette: Interface
  tool_description: tests values entered in other connected interface tools and returns
    either a True or False value in an application or macro
  connections:
    inputs:
    - Question
    outputs:
    - True Connection
    - False Connection
- tool_name: Control Parameter
  plugin_name: AlteryxGuiToolkit.Questions.ControlParam.ControlParam
  tool_palette: Interface
  tool_description: tells a Batch Macro what values to use for each batch
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Date
  plugin_name: AlteryxGuiToolkit.Questions.Date.Date
  tool_palette: Interface
  tool_description: displays a calendar for users to specify a date value in an application
    or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: DCM Connection
  plugin_name: AlteryxGuiToolkit.Questions.DcmConnection.QuestionDcmConnection
  tool_palette: Interface
  tool_description: display a prompt in an app or macro, and allow to pick a Connection
    from the DCM storage
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Drop Down
  plugin_name: AlteryxGuiToolkit.Questions.DropDownListBox.DropDown
  tool_palette: Interface
  tool_description: adds a dropdown menu for users to select a single value in an
    application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Error Message
  plugin_name: AlteryxGuiToolkit.Error.Error
  tool_palette: Interface
  tool_description: displays an error message for an application or macro based on
    criteria in an expression
  connections:
    inputs: []
    outputs: []
- tool_name: File Browse
  plugin_name: AlteryxGuiToolkit.Questions.FileBrowse.FileBrowse
  tool_palette: Interface
  tool_description: displays a file browse control in an application or macro; works
    with both Input and Output functionality
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Folder Browse
  plugin_name: AlteryxGuiToolkit.Questions.FolderBrowse.FolderBrowse
  tool_palette: Interface
  tool_description: Displays a folder browse control in an app or macro. The directory
    path specified by the user is passed to downstream tools.
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: List Box
  plugin_name: AlteryxGuiToolkit.Questions.DropDownListBox.ListBox
  tool_palette: Interface
  tool_description: adds a list box where users can make multiple selections in an
    application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Macro Input
  plugin_name: AlteryxBasePluginsGui.MacroInput.MacroInput
  tool_palette: Interface
  tool_description: pass data from a workflow into a macro for additional analysis
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Macro Output
  plugin_name: AlteryxBasePluginsGui.MacroOutput.MacroOutput
  tool_palette: Interface
  tool_description: return data from a macro to a workflow to be used downstream
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Map
  plugin_name: AlteryxGuiToolkit.Questions.Map.Map
  tool_palette: Interface
  tool_description: displays an interactive map to allow the user to create spatial
    objects or select from existing spatial objects in an application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Numeric Up Down
  plugin_name: AlteryxGuiToolkit.Questions.NumericUpDown.NumericUpDown
  tool_palette: Interface
  tool_description: adds a numeric control for users to specify a numeric value in
    an application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Radio Button
  plugin_name: AlteryxGuiToolkit.Questions.RadioButtonGroup.RadioButtonGroup
  tool_palette: Interface
  tool_description: creates a True value for the default or selected radio button
    in an application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Text Box
  plugin_name: AlteryxGuiToolkit.Questions.TextBox.QuestionTextBox
  tool_palette: Interface
  tool_description: adds a text input box for user input in an application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Tree
  plugin_name: AlteryxGuiToolkit.Questions.Tree.Tree
  tool_palette: Interface
  tool_description: displays an organized hierarchical data structure to allow the
    user to make 1 or more selections in an application or macro
  connections:
    inputs: []
    outputs:
    - Question
- tool_name: Association Analysis
  plugin_name: Predictive Tools\Association_Analysis.yxmc
  tool_palette: Data Investigation
  tool_description: helps to determine which columns in a dataset have a bivariate
    association with each other
  connections:
    inputs:
    - Data Input
    outputs:
    - Output
    - Correlation Matrix
- tool_name: Basic Data Profile
  plugin_name: AlteryxBasePluginsGui.BasicDataProfile.BasicDataProfile
  tool_palette: Data Investigation
  tool_description: shows an overview of your dataset and outputs the information
    for further analysis
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Contingency Table
  plugin_name: Predictive Tools\ContingencyTable.yxmc
  tool_palette: Data Investigation
  tool_description: allows you to look at up to 4 columns to see how they relate to
    each other
  connections:
    inputs:
    - Input
    outputs:
    - Data
    - Report
    - Interactive Pivot
- tool_name: Distribution Analysis
  plugin_name: Predictive Tools\Distribution_Analysis.yxmc
  tool_palette: Data Investigation
  tool_description: helps you understand the overall nature of your data as well as
    make decisions about how to analyze it
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Field Summary
  plugin_name: Predictive Tools\Field_Summary_Report.yxmc
  tool_palette: Data Investigation
  tool_description: produces a concise summary report of descriptive statistics for
    the selected data columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Reports
    - Interactive
- tool_name: Frequency Table
  plugin_name: Predictive Tools\Frequency.yxmc
  tool_palette: Data Investigation
  tool_description: produces a frequency table for each column selected
  connections:
    inputs:
    - Input
    outputs:
    - Data
    - Report
    - Interactive
- tool_name: Heat Plot
  plugin_name: Predictive Tools\Heat_Plot.yxmc
  tool_palette: Data Investigation
  tool_description: shows the joint distribution of 2 variables that are either continuous
    numeric variables or ordered categories
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Histogram
  plugin_name: Predictive Tools\Histogram.yxmc
  tool_palette: Data Investigation
  tool_description: shows a histogram for the empirical cumulative distribution of
    a single numeric column
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Pearson Correlation
  plugin_name: AlteryxBasePluginsGui.PearsonCorrelation.PearsonCorrelation
  tool_palette: Data Investigation
  tool_description: measures the correlation or covariance between 2 or more variables
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Plot of Means
  plugin_name: Predictive Tools\Plot_of_Means.yxmc
  tool_palette: Data Investigation
  tool_description: displays the mean of the response column for each of the categories
    of the selected categorical column
  connections:
    inputs:
    - Data Input
    outputs:
    - Graph Output
- tool_name: Scatterplot
  plugin_name: Predictive Tools\Scatterplot.yxmc
  tool_palette: Data Investigation
  tool_description: allows you to see possible relationships between any 2 model variables
    selected
  connections:
    inputs:
    - Data Input
    outputs:
    - Scater
- tool_name: Spearman Correlation
  plugin_name: SpearmanCorrCoeff.yxmc
  tool_palette: Data Investigation
  tool_description: determines how well an arbitrary monotonic function can describe
    the relationship between 2 variables
  connections:
    inputs:
    - Field Selection
    outputs:
    - Output16
- tool_name: Violin Plot
  plugin_name: Predictive Tools\Violin_Plot.yxmc
  tool_palette: Data Investigation
  tool_description: shows the distribution of a single numeric variable and conveys
    the density of the distribution
  connections:
    inputs:
    - Data Input
    outputs:
    - Graph Output
- tool_name: Boosted Model
  plugin_name: Predictive Tools\Boosted_Model.yxmc
  tool_palette: Predictive
  tool_description: creates generalized boosted regression models based on gradient
    boosting methods
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
- tool_name: Count Regression
  plugin_name: Predictive Tools\Count_Regression.yxmc
  tool_palette: Predictive
  tool_description: creates a regression model that relates a non-negative integer
    value (0,1,2,3, etc.) target to 1 or more predictor variables
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Reports
- tool_name: Decision Tree
  plugin_name: Decision_Tree/Supporting_Macros/Decision_Tree.yxmc
  tool_palette: Predictive
  tool_description: creates a set of if-then split rules to optimize model creation
    criteria based on Decision Tree Learning methods
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Reports
    - Interactive
- tool_name: Forest Model
  plugin_name: Predictive Tools\Forest_Model.yxmc
  tool_palette: Predictive
  tool_description: creates a model that constructs a set of decision tree models
    to predict a target based on 1 or more predictors
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Reports
- tool_name: Gamma Regression
  plugin_name: Predictive Tools\Gamma_Regression.yxmc
  tool_palette: Predictive
  tool_description: relates gamma distributed, strictly positive target to 1 or more
    predictors
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Reports
- tool_name: Lift Chart
  plugin_name: Predictive Tools\Lift_Chart.yxmc
  tool_palette: Predictive
  tool_description: produces a cumulative captured response chart and an incremental
    response rate chart that are used to visually assess the comparative accuracy
    of different binary (yes/no) classification models
  connections:
    inputs:
    - Left Input
    - Right Input
    outputs:
    - Output
- tool_name: Linear Regression
  plugin_name: Predictive Tools\Linear_Regression.yxmc
  tool_palette: Predictive
  tool_description: creates a simple model to estimate values, or evaluate relationships
    between variables based on a linear relationship
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Reports
    - Interactive Report
- tool_name: Logistic Regression
  plugin_name: Logistic_Regression/Supporting_Macros/Logistic_Regression.yxmc
  tool_palette: Predictive
  tool_description: creates a model that relates a binary target to 1 or more predictors
    to obtain the estimated probability for each of 2 possible responses for the target
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Reports
    - Interactive Report
- tool_name: Model Comparison
  plugin_name: Predictive Tools\Model Comparison.yxmc
  tool_palette: Predictive
  tool_description: compares the performance of one or more different predictive models
    based on the use of a validation, or test dataset
  connections:
    inputs:
    - Data
    - Models
    outputs:
    - error_measures
    - predictions
    - Report
- tool_name: Naive Bayes Classifier
  plugin_name: Predictive Tools\Naive_Bayes.yxmc
  tool_palette: Predictive
  tool_description: creates a binomial or multinomial probabilistic classification
    model of the relationship between a set of predictors and a categorical target
  connections:
    inputs:
    - Input
    outputs:
    - Object
    - Report
- tool_name: Nested Test
  plugin_name: Predictive Tools\Nested_Test.yxmc
  tool_palette: Predictive
  tool_description: examines whether 2 models, 1 of which contains a subset of the
    variables contained in the other, are statistically equivalent in terms of their
    predictive capability
  connections:
    inputs:
    - Left Input
    - Center Input
    - Right Input
    outputs:
    - Output
- tool_name: Network Analysis
  plugin_name: Predictive Tools\NetworkViz.yxmc
  tool_palette: Predictive
  tool_description: generates an interactive dashboard of a network, to explore relationships
    between the various nodes
  connections:
    inputs:
    - Nodes
    - Edges
    outputs:
    - Data
    - Interactive
- tool_name: Neural Network
  plugin_name: Predictive Tools\Neural_Network.yxmc
  tool_palette: Predictive
  tool_description: creates a feedforward perceptron neural network model with a single
    hidden layer
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
- tool_name: Score
  plugin_name: Score
  tool_palette: Predictive
  tool_description: creates an estimate of a target by applying an R model from another
    Predictive tool to a set of predictors
  connections:
    inputs:
    - Data
    - Model
    outputs:
    - Output
- tool_name: Spline Model
  plugin_name: Predictive Tools\Spline_Model.yxmc
  tool_palette: Predictive
  tool_description: provides the multivariate adaptive regression splines algorithm
    of Friedman
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
- tool_name: Stepwise
  plugin_name: Predictive Tools\Stepwise.yxmc
  tool_palette: Predictive
  tool_description: determines the best predictors to include in a model out of a
    larger set of potential predictors for linear, logistic, and other traditional
    regression models
  connections:
    inputs:
    - Left Input
    - Right Input
    outputs:
    - Output
    - Reports
- tool_name: Support Vector Machine
  plugin_name: Predictive Tools\SVM_v2.yxmc
  tool_palette: Predictive
  tool_description: popular set of supervised learning algorithms originally developed
    for classification problems
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
- tool_name: Test of Means
  plugin_name: Predictive Tools\Test_of_Means.yxmc
  tool_palette: Predictive
  tool_description: performs a Welch's two sample t-test of the difference in mean
    values for a numeric response column between a control group and 1 or more treatment
    groups
  connections:
    inputs:
    - Input
    outputs:
    - Data
    - Report
- tool_name: AB Analysis
  plugin_name: Predictive Tools\AB_Analysis.yxmc
  tool_palette: AB Testing
  tool_description: compares the percentage change in a performance measure to the
    same measure either over the same time period one year earlier, or a user specified
    time period, for two different groups
  connections:
    inputs:
    - Controls
    - Treatments
    - Performance Data
    outputs:
    - Output
    - External
    - Grouped Data
    - Interactive Report
- tool_name: AB Controls
  plugin_name: Predictive Tools\AB_Controls.yxmc
  tool_palette: AB Testing
  tool_description: matches 1 to 10 control units to each member of a set of previously
    selected test units, based on criteria such as seasonal patterns and growth trends
    for a key performance indicator along with other user provided criteria
  connections:
    inputs:
    - Data
    - Treatments
    outputs:
    - Controls
    - Assignment
- tool_name: AB Treatments
  plugin_name: Predictive Tools\AB_Treatments.yxmc
  tool_palette: AB Testing
  tool_description: helps in determining which group is the best fit for AB testing
  connections:
    inputs:
    - Data Input
    outputs:
    - Data Output
    - Report Output
- tool_name: AB Trend
  plugin_name: Predictive Tools\AB_Trend.yxmc
  tool_palette: AB Testing
  tool_description: creates measures of trend and seasonal patterns that can be used
    in helping to match treatment to control units for A/B testing
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: ARIMA
  plugin_name: Predictive Tools\ARIMA.yxmc
  tool_palette: Time Series
  tool_description: estimates a time series forecasting model, either as a univariate
    model or one with covariates (predictors), using an autoregressive integrated
    moving average method
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
    - Interactive
- tool_name: ETS
  plugin_name: Predictive Tools\ETS.yxmc
  tool_palette: Time Series
  tool_description: estimates a univariate time series forecasting model using an
    exponential smoothing method
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
    - Interactive
- tool_name: TS Compare
  plugin_name: Predictive Tools\TS_Compare.yxmc
  tool_palette: Time Series
  tool_description: compares 1 or more time series models created with either the
    ARIMA tool or ETS tool, including ARIMA models that use covariates
  connections:
    inputs:
    - Left Input
    - Right Input
    outputs:
    - Output
    - Report
    - Interactive
- tool_name: TS Covariate Forecast
  plugin_name: Predictive Tools\TS_Covariate_Forecast.yxmc
  tool_palette: Time Series
  tool_description: provides forecasts from an ARIMA model estimated using covariates
    for a user-specified number of future periods
  connections:
    inputs:
    - Left
    - Right
    outputs:
    - Output
    - Report
    - Interactive
- tool_name: TS Filler
  plugin_name: Predictive Tools\TimeSeriesFiller.yxmc
  tool_palette: Time Series
  tool_description: takes a data stream of time series data and fills in any gaps
    in the series
  connections:
    inputs:
    - Unfilled Input
    outputs:
    - Filled Output
- tool_name: TS Forecast
  plugin_name: Predictive Tools\TS_Forecast.yxmc
  tool_palette: Time Series
  tool_description: provides forecasts from a model created with either the ARIMA
    tool or ETS tool for a user-specified number of future periods
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
    - Interactive Viz
- tool_name: TS Plot
  plugin_name: Predictive Tools\TS_Plot.yxmc
  tool_palette: Time Series
  tool_description: provides a number of different univariate time series plots that
    can be used to better understand the time series data and determine how to develop
    a forecasting model
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Interactive
- tool_name: Append Cluster
  plugin_name: Predictive Tools\Append_Cluster.yxmc
  tool_palette: Predictive Grouping
  tool_description: appends the cluster assignments from a K-Centroids Cluster Analysis
    Tool to a data stream
  connections:
    inputs:
    - First Input
    - Second Input
    outputs:
    - Output
- tool_name: Find Nearest Neighbors
  plugin_name: Predictive Tools\Find_Nearest_Neighbors.yxmc
  tool_palette: Predictive Grouping
  tool_description: finds the selected number of nearest neighbors in the data stream
  connections:
    inputs:
    - Data
    - Query
    outputs:
    - Neighbors
    - Measures
- tool_name: K-Centroids Cluster Analysis
  plugin_name: Predictive Tools\K-Centroids_Cluster_Analysis.yxmc
  tool_palette: Predictive Grouping
  tool_description: divides records into the "best" K groups based on the proximity
    of each record to one of K points in the data
  connections:
    inputs:
    - Data Input
    outputs:
    - Output
    - Reports
- tool_name: K-Centroids Diagnostics
  plugin_name: Predictive Tools\K-Centroids_Diagnostics.yxmc
  tool_palette: Predictive Grouping
  tool_description: allows the user to make an assessment of the appropriate number
    of clusters to specify given the data and the selected clustering algorithm (K-Means,
    K-Medians, or Neural Gas)
  connections:
    inputs:
    - Data Input
    outputs:
    - Reports
- tool_name: MB Affinity
  plugin_name: Predictive Tools\MB_Affinity.yxmc
  tool_palette: Predictive Grouping
  tool_description: constructs a matrix of affinity measures between different items
    with respect to their likelihood of being in the same transaction
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: MB Inspect
  plugin_name: Predictive Tools\MB_Inspect.yxmc
  tool_palette: Predictive Grouping
  tool_description: takes rules or itemsets output from the MB Rules tool analysis
    of those rules
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
- tool_name: MB Rules
  plugin_name: Predictive Tools\MB_Rules.yxmc
  tool_palette: Predictive Grouping
  tool_description: takes transaction data, transforms it, and creates either a set
    of association rules using the Apriori algorithm or frequent itemsets using either
    the Apriori or Eclat algorithms
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Report
- tool_name: Principal Components
  plugin_name: Predictive Tools\Principal_Components.yxmc
  tool_palette: Predictive Grouping
  tool_description: reduces the dimensions (the number of numeric fields) in a data
    set by transforming the original set of fields into a smaller set that accounts
    for most of the variance (i.e., information) in the data
  connections:
    inputs:
    - Data Input
    outputs:
    - Output
    - Reports
- tool_name: Optimization
  plugin_name: Optimization
  tool_palette: Prescriptive
  tool_description: solves linear programming (LP), mixed-integer linear programming
    (MILP), and quadratic programming (QP) optimization problems
  connections:
    inputs:
    - InputO
    - InputA
    - InputB
    - InputQ
    outputs:
    - Simple
    - Data
    - Interactive
- tool_name: Simulation Sampling
  plugin_name: SimSampling
  tool_palette: Prescriptive
  tool_description: samples data parametrically from distribution, from input data,
    or a best fitting to a distribution
  connections:
    inputs:
    - SampleData
    - SimulationData
    outputs:
    - Data
- tool_name: Simulation Scoring
  plugin_name: SunScoring
  tool_palette: Prescriptive
  tool_description: takes a sample from an approximation of a model object error distribution
  connections:
    inputs:
    - Model
    - Validation
    - Simulation
    outputs:
    - Data
- tool_name: Simulation Summary
  plugin_name: SimSummary
  tool_palette: Prescriptive
  tool_description: visualize simulated distributions and results from operations
    on those distributions
  connections:
    inputs:
    - Field Input
    outputs:
    - Data
    - Report
    - Interactive
- tool_name: Adobe Analytics
  plugin_name: AdobeAnalytics
  tool_palette: Connectors
  tool_description: generate ad hoc reports from the Adobe Analytics report suites
  connections:
    inputs: []
    outputs:
    - Report
    - API Response
- tool_name: Amazon S3 Download
  plugin_name: AlteryxConnectorGui.AmazonS3Download.AmazonS3Download
  tool_palette: Connectors
  tool_description: retrieve data stored in the cloud hosted by Amazon Simple Storage
    Service
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Amazon S3 Upload
  plugin_name: AlteryxConnectorGui.AmazonS3Upload.AmazonS3Upload
  tool_palette: Connectors
  tool_description: transfer data from Alteryx to the cloud hosted by Amazon Simple
    Storage Service
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Anaplan Input
  plugin_name: AnaplanInput
  tool_palette: Connectors
  tool_description: read data stored in Anaplan into Designer
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Anaplan Output
  plugin_name: AnaplanOutput
  tool_palette: Connectors
  tool_description: write data from Designer to Anaplan
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Azure Data Lake File Input
  plugin_name: azure_data_lake_input
  tool_palette: Connectors
  tool_description: Read data from files located in an Azure Data Lake Store (ADLS)
    to your Alteryx workflow. The supported file formats are CSV, XLSX, JSON, or Avro.
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Azure Data Lake File Output
  plugin_name: azure_data_lake_output
  tool_palette: Connectors
  tool_description: write data from your Alteryx workflow to a file located in an
    Azure Data Lake Store
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Box Input
  plugin_name: BoxInput
  tool_palette: Connectors
  tool_description: read the data stored in your Box workspace
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Box Output
  plugin_name: BoxOutput
  tool_palette: Connectors
  tool_description: write data to your Box workspace
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Cognitive Services Text Analytics
  plugin_name: CognitiveServicesTextAnalytics
  tool_palette: Connectors
  tool_description: uses the Cognitive Services Text Analytics API to perform sentiment
    analysis, key phrase extraction, and language detection
  connections:
    inputs:
    - Input
    outputs:
    - StandardOutput
- tool_name: Dataverse Input
  plugin_name: DataverseInput
  tool_palette: Connectors
  tool_description: read data from Dataverse tables to Designer
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Dataverse Output
  plugin_name: DataverseOutput
  tool_palette: Connectors
  tool_description: write data to Dataverse tables
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Google Analytics 4 Input
  plugin_name: GoogleAnalytics4Input
  tool_palette: Connectors
  tool_description: read data from Google Analytics
  connections:
    inputs: []
    outputs:
    - Output1
    - Output2
- tool_name: Google Drive Input
  plugin_name: GoogleDriveInput
  tool_palette: Connectors
  tool_description: read files from Google Drive to Designer
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Google Drive Output
  plugin_name: GoogleDriveOutput
  tool_palette: Connectors
  tool_description: write files from Designer to Google Drive
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Marketo Append
  plugin_name: marketo-append
  tool_palette: Connectors
  tool_description: uses an incoming data stream that you specify to retrieve corresponding
    records from your Marketo instance
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Marketo Input
  plugin_name: marketo-input
  tool_palette: Connectors
  tool_description: reads Marketo records based on specific parameters
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Marketo Output
  plugin_name: marketo-output
  tool_palette: Connectors
  tool_description: 'make a call to the Marketo REST API endpoint: Create/Update Leads'
  connections:
    inputs:
    - Input1
    outputs: []
- tool_name: Microsoft Power BI Output
  plugin_name: PowerBIOutput
  tool_palette: Connectors
  tool_description: uses the Power BI REST API to upload a dataset from your Alteryx
    workflow to the Power BI web application
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: OneDrive Input
  plugin_name: OneDriveInput
  tool_palette: Connectors
  tool_description: reads files from OneDrive to Designer
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: OneDrive Output
  plugin_name: OneDriveOutput
  tool_palette: Connectors
  tool_description: writes files from Designer to OneDrive
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Outlook 365 Input
  plugin_name: Outlook_input
  tool_palette: Connectors
  tool_description: reads emails and calendar events from Outlook 365 as a table of
    values to Designer; can also download email attachments for later processing
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Power Automate
  plugin_name: PowerAutomate
  tool_palette: Connectors
  tool_description: run flows on incoming data and pass the processed data to other
    tools in Designer
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Salesforce Input
  plugin_name: SalesforceInput
  tool_palette: Connectors
  tool_description: query your tables from Salesforce.com and read them into Designer
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Salesforce Output
  plugin_name: SalesforceOutput
  tool_palette: Connectors
  tool_description: write data from Designer to Salesforce
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: ServiceNow Input
  plugin_name: ServiceNowInput
  tool_palette: Connectors
  tool_description: run, schedule, and publish workflows from Designer that read from
    ServiceNow Data tables
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: SharePoint Input
  plugin_name: SharePointInput
  tool_palette: Connectors
  tool_description: read data from your CSV, XLSX, and YXDB files as well as lists
    from your SharePoint site
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: SharePoint Output
  plugin_name: SharepointOutput
  tool_palette: Connectors
  tool_description: write data to your CSV, XLSX, and YXDB files as well as lists
    in your SharePoint site
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Tableau Input
  plugin_name: TableauInput
  tool_palette: Connectors
  tool_description: reads data from Tableau to Designer
  connections:
    inputs: []
    outputs:
    - Output1
    - Output2
    - Output3
- tool_name: Tableau Output
  plugin_name: TableauOutput
  tool_palette: Connectors
  tool_description: writes data from Designer to Tableau
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Calgary Cross Count
  plugin_name: CalgaryPluginsGui.CalgaryCrossCount.CalgaryCrossCount
  tool_palette: Calgary
  tool_description: returns the count of rows that match a given query within a Calgary
    database
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Calgary Cross Count Append
  plugin_name: CalgaryPluginsGui.CalgaryCrossCountAppend.CalgaryCrossCountAppend
  tool_palette: Calgary
  tool_description: takes an input file and appends counts to rows that join to a
    Calgary database
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Calgary Input
  plugin_name: CalgaryPluginsGui.CalgaryInput.CalgaryInput
  tool_palette: Calgary
  tool_description: Input tool allows the user to input data from a Calgary database
    file with a query
  connections:
    inputs: []
    outputs:
    - Output
- tool_name: Calgary Join
  plugin_name: CalgaryPluginsGui.CalgaryJoin.CalgaryJoin
  tool_palette: Calgary
  tool_description: takes an input file and performs joins against a Calgary database
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Calgary Loader
  plugin_name: CalgaryLoadersGui.CalgaryLoader.CalgaryLoader
  tool_palette: Calgary
  tool_description: creates a highly indexed and compressed Calgary database (CYDB)
    that allows for fast queries
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: API Output
  plugin_name: AlteryxBasePluginsGui.APIOutput.APIOutput
  tool_palette: Developer
  tool_description: returns the results of a data stream directly to an API callback
    function
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Base64 Encoder
  plugin_name: Base64_Encoder.yxmc
  tool_palette: Developer
  tool_description: issues a base 64 encoded string for a selected string column
  connections:
    inputs:
    - Macro Input
    outputs:
    - Output
- tool_name: Blob Convert
  plugin_name: AlteryxBasePluginsGui.BlobConvert.BlobConvert
  tool_palette: Developer
  tool_description: converts incoming Blob data into another data type, or it converts
    another type of incoming data into a Blob data type
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Blob Input
  plugin_name: AlteryxBasePluginsGui.BlobInput.BlobInput
  tool_palette: Developer
  tool_description: reads images or other media files as Binary Large Objects (Blob)
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Blob Output
  plugin_name: AlteryxBasePluginsGui.BlobOutput.BlobOutput
  tool_palette: Developer
  tool_description: reads Binary Large Object (Blob) columns and outputs a file from
    a Blob data type
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Block Until Done
  plugin_name: AlteryxBasePluginsGui.BlockUntilDone.BlockUntilDone
  tool_palette: Developer
  tool_description: stops datasets from moving downstream until the last record in
    the set has been processed by all previous tools
  connections:
    inputs:
    - Input
    outputs:
    - Output
    - Output2
    - Output3
- tool_name: Control Container
  plugin_name: AlteryxGuiToolkit.ControlContainer.ControlContainer
  tool_palette: Developer
  tool_description: manage the sequence in which tools run in your workflow and ensure
    that the steps in your process are executed in the correct order
  connections:
    inputs:
    - Control
    outputs:
    - Log
- tool_name: Detour
  plugin_name: AlteryxBasePluginsGui.Detour.Detour
  tool_palette: Developer
  tool_description: alter the path of data through a workflow
  connections:
    inputs:
    - Input
    outputs:
    - Left
    - Right
- tool_name: Detour End
  plugin_name: AlteryxBasePluginsGui.DetourEnd.DetourEnd
  tool_palette: Developer
  tool_description: combines the 2 possible paths from a Detour tool and returns processing
    to a single data stream
  connections:
    inputs:
    - Left
    - Right
    outputs:
    - Output
- tool_name: Download
  plugin_name: AlteryxConnectorGui.Download.Download
  tool_palette: Developer
  tool_description: retrieve data from a URL to use in downstream processing or to
    save to a file, it can also download or upload data via FTP and SFTP
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Dynamic Input
  plugin_name: AlteryxBasePluginsGui.DynamicInput.DynamicInput
  tool_palette: Developer
  tool_description: reads and processes from input files or databases at runtime based
    on the incoming data connection
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Dynamic Rename
  plugin_name: AlteryxBasePluginsGui.DynamicRename.DynamicRename
  tool_palette: Developer
  tool_description: rename any or all columns in a data stream
  connections:
    inputs:
    - Targets
    - Source
    outputs:
    - Output
- tool_name: Dynamic Replace
  plugin_name: AlteryxBasePluginsGui.DynamicReplace.DynamicReplace
  tool_palette: Developer
  tool_description: replaces data in multiple columns based on an expression or value
  connections:
    inputs:
    - Input
    - Expressions
    outputs:
    - Output
    - Counts
- tool_name: Dynamic Select
  plugin_name: AlteryxBasePluginsGui.DynamicSelect.DynamicSelect
  tool_palette: Developer
  tool_description: allows for columns to be selected and de-selected based on field
    type or a formula
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Expect Equal
  plugin_name: AlteryxBasePluginsGui.ExpectEqual.ExpectEqual
  tool_palette: Developer
  tool_description: test if two data streams are identical and report errors if not
  connections:
    inputs:
    - Expected
    - Actual
    outputs: []
- tool_name: Field Info
  plugin_name: AlteryxBasePluginsGui.FieldInfo.FieldInfo
  tool_palette: Developer
  tool_description: displays information about the incoming data's columns
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: JSON Parse
  plugin_name: AlteryxBasePluginsGui.JSONParse.JSONParse
  tool_palette: Developer
  tool_description: parses out key and value pairs from a column into a table
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Message
  plugin_name: AlteryxBasePluginsGui.Message.Message
  tool_palette: Developer
  tool_description: displays messages or errors in the Results windows
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Python
  plugin_name: JupyterCode
  tool_palette: Developer
  tool_description: runs Python code within a workflow. Use only when the
    requirement cannot be met with native Designer tools.
  connections:
    inputs:
    - Input
    outputs:
    - Output1
    - Output2
    - Output3
    - Output4
    - Output5
- tool_name: R
  plugin_name: AlteryxRPluginGui.R
  tool_palette: Developer
  tool_description: runs R code within a workflow. Use only when the requirement
    cannot be met with native Designer tools.
  connections:
    inputs:
    - Input
    outputs:
    - Output1
    - Output2
    - Output3
    - Output4
    - Output5
- tool_name: Run Command
  plugin_name: AlteryxBasePluginsGui.RunCommand.RunCommand
  tool_palette: Developer
  tool_description: allows a user to run external programs from within an Alteryx
    workflow. Use only when the requirement cannot be met with native Designer
    tools.
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Test
  plugin_name: AlteryxBasePluginsGui.Test.Test
  tool_palette: Developer
  tool_description: verifies data or processes in a workflow by comparing rows, row
    counts, or using a custom expression
  connections:
    inputs:
    - Input
    outputs: []
- tool_name: Throttle
  plugin_name: AlteryxBasePluginsGui.Throttle.Throttle
  tool_palette: Developer
  tool_description: define the number of rows per minute that can pass to downstream
    tools
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: JSON Build
  plugin_name: AlteryxBasePluginsGui.JSONBuild.JSONBuild
  tool_palette: Laboratory
  tool_description: take the table schema of the JSON Parse tool and build it back
    into properly formatted JSON
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Key-Value Pair Extraction
  plugin_name: KeyValuePairExtraction
  tool_palette: Laboratory
  tool_description: identifies key-value pair structures in your documents
  connections:
    inputs:
    - Data
    - Keys
    outputs:
    - Output
- tool_name: Transpose In-DB
  plugin_name: LockInGui.LockInTranspose.LockInTranspose
  tool_palette: Laboratory
  tool_description: pivot the orientation of a data table in an In-DB workflow so
    you can view horizontal data fields on a vertical axis
  connections:
    inputs:
    - Input
    outputs:
    - Output
- tool_name: Visual Layout
  plugin_name: VisualLayout
  tool_palette: Laboratory
  tool_description: create a single report by combining and arranging various elements
  connections:
    inputs:
    - Input
    outputs:
    - Output
```
