---
name: dom-xss
description: >
  Statically analyzes JavaScript source files for DOM-based Cross-Site Scripting
  vulnerabilities using 7 parallel specialist agents. Triggers on "dom xss",
  "dom-xss scan", "find xss in javascript", "analyze js for xss", or
  "client-side xss". Handles jxscout/Burp Suite output directories, classifies
  app code vs libraries, beautifies minified files, and performs single-file
  taint tracking from user-controllable sources to dangerous sinks across all
  major DOM-XSS categories. Cross-file candidates are flagged for manual review.
---

# DOM XSS Hunter

Static analysis of client-side JavaScript for DOM-based XSS. Deploys 7 parallel
agents covering: location-based, eval/code-execution, innerHTML/outerHTML,
document.write, framework-specific, postMessage, and iframe/embed injection.

## Invocation

Ask the user for the **target directory** containing JavaScript files (typically
a jxscout output directory). Then execute the phases below in order.

## Phase 1: Preprocessing

Read `PREPROCESSING.md` in this skill directory for the full procedure. Summary:

1. **Discover** JS/TS/JSX/TSX/Vue/Svelte files (up to 2000)
2. **Classify** each as app-code (analyze) or library (skip) using path, filename,
   and header heuristics — including jxscout CDN domain folders
3. **Beautify** minified files into a `.domxss-workdir/` directory
4. **Chunk** large files (>3000 lines) at function/class boundaries

Log: `[PRE-PROCESS] App files: N | Libraries excluded: N | Uncertain: N`

## Phase 2: Candidate Discovery — Grep Pre-scan

Run all 7 grep scans **in parallel** to build per-agent candidate file lists.
Only files matching a category's sink patterns are sent to that agent.

### Grep patterns per agent

Run against `${APP_FILES}` (the classified app-code files):

| Agent | Category | Grep Pattern |
|-------|----------|-------------|
| 1 | Location-based | `location\.(assign\|replace\|href\s*=)\|window\.open\s*\(\|setAttribute\s*\(\s*['"]href\|\.action\s*=` |
| 2 | Eval/Code-exec | `eval\s*\(\|new\s+Function\s*\(\|setTimeout\s*\(\s*['"\x60]\|setInterval\s*\(\s*['"\x60]\|new\s+Worker\s*\(\|importScripts\s*\(\|serviceWorker\.register\s*\(\|setAttribute\s*\(\s*['"]on` |
| 3 | innerHTML | `\.innerHTML\s*[+]?=\|\.outerHTML\s*[+]?=\|insertAdjacentHTML\s*\(\|\.html\s*\(\|\.append\s*\(['"\x60]<` |
| 4 | document.write | `document\.write(ln)?\s*\(` |
| 5 | Framework-specific | `dangerouslySetInnerHTML\|v-html\|{@html\|bypassSecurityTrust\|\[innerHTML\]\|\$compile\|\$scope\.\$eval\|SafeString\|{\{\{\|<%-` |
| 6 | PostMessage | `addEventListener\s*\(\s*['"]message\|onmessage\s*=\|postMessage\s*\(` |
| 7 | iframe/embed | `<iframe\|createElement\s*\(\s*['"]iframe\|\.srcdoc\s*=\|<object\|<embed\|createElement\s*\(\s*['"]object\|createElement\s*\(\s*['"]embed` |

If a scan returns zero files, **skip that agent**.

## Phase 2.5: Source/Sink Mapping (Lightweight Triage)

Before deep analysis, run a fast `explore`-type agent to classify each candidate
file. For each file, produce:

```json
{"file": "<path>", "sources_found": [...], "sinks_found": [...], "has_both": true, "complexity": "low|medium|high"}
```

Use the mapping to:
- **Skip** files with sinks but no sources → automatic INFO cross-file candidates
- **Prioritise** files with `has_both: true`
- **Tag** `complexity: high` files for conservative confidence ratings

## Phase 3: Deep Analysis — 7 Parallel Agents

Launch all 7 agents **simultaneously** using `task` tool with
`agent_type: "general-purpose"`. Each agent receives:

1. The **common context** from `SOURCES-AND-SINKS.md` (include verbatim in each prompt)
2. Its **category definition** from `AGENT-CATEGORIES.md` (the relevant section only)
3. The **candidate files** filtered by Phase 2/2.5 (only `has_both: true` files,
   plus sink-only files where sources exist in neighboring files)

For worked examples to include in agent prompts, read `EXAMPLES.md` and include
only the example matching the agent's category.

**CRITICAL**: Launch all 7 agents in a single response using parallel tool calls.

## Phase 3.5: Finding Validation (Feedback Loop)

After all agents complete, run a validation pass on every CRITICAL and HIGH finding:

1. Re-read the specific code region (±30 lines around the sink)
2. Verify the source is genuinely user-controllable
3. Verify the sink actually receives tainted data (not dead code)
4. Verify no sanitization was missed
5. Downgrade or drop findings that fail validation

Use an `explore`-type agent for this — it only needs the flagged code regions.

## Phase 4: Report Generation

Read `REPORT-TEMPLATE.md` for the full report structure. Summary:

1. **Deduplicate**: Same location across agents → keep most specific finding
2. **Validate severities**: CRITICAL requires confirmed source→sink with no sanitization
3. **Generate** `domxss-report.md` in the target directory
4. **Present** summary to user, highlight CRITICAL/HIGH findings

## Execution Checklist

```
1. [ ] Ask user for target directory
2. [ ] Phase 1: Preprocess (discover, classify, beautify, chunk)
3. [ ] Phase 2: Run 7 parallel grep scans
4. [ ] Phase 2.5: Source/sink mapping triage
5. [ ] Phase 3: Launch 7 parallel deep-analysis agents
6. [ ] Phase 3.5: Validate CRITICAL/HIGH findings
7. [ ] Phase 4: Deduplicate, generate domxss-report.md
8. [ ] Present summary to user
```
