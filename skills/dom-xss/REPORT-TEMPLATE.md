# Report Template

## Structure

Generate `domxss-report.md` in the target directory:

```markdown
# DOM XSS Analysis Report

**Target Directory**: <path>
**Analysis Date**: <date>
**Files Analyzed**: X app files (Y libraries excluded, Z uncertain included)
**Frameworks Detected**: <list>

## Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | X |
| 🟠 High | X |
| 🟡 Medium | X |
| 🔵 Low | X |
| ⚪ Info | X |

| Category | Findings |
|----------|----------|
| Location-based | X |
| Eval/Function | X |
| innerHTML/outerHTML | X |
| document.write | X |
| Framework-specific | X |
| PostMessage | X |
| iframe/embed | X |

## 🔴 Critical Findings

### DOMXSS-001: <title>
- **Category**: <category>
- **Severity**: <severity> | **Confidence**: <confidence>
- **File**: `<path>:<line>`
- **Source**: `<source expression>`
- **Sink**: `<sink expression>`
- **Flow**: <source→sink description>
- **Evidence**:
  ` ` `javascript
  // <relevant code>
  ` ` `
- **Recommendation**: <specific fix>

## 🟠 High Findings
(same format)

## 🟡 Medium Findings
(same format)

## 🔵 Low Findings
(condensed — group similar findings)

## ⚪ Info — Cross-file Review Candidates

| File | Line | Sink | Exported Via | Potential Sources |
|------|------|------|-------------|-------------------|

## Excluded Libraries
<list>

## Methodology
DOM XSS Hunter — 7 specialist agents with single-file taint tracking.
```

## Deduplication Rules

1. Same code location from multiple agents → keep the more specific finding
2. Same sink at different severities → use the highest severity
3. Same file + same data flow → group under single finding ID with sub-items
4. Confidence LOW findings → group into "Requires Manual Verification" subsection

## Severity Validation

Before finalizing:
- **CRITICAL** requires confirmed source→sink with no effective sanitization
- Downgrade to **HIGH** if sanitization might be bypassable but uncertain
- Downgrade to **MEDIUM** if path requires specific conditions
- Use **INFO** generously for sinks without confirmed in-file sources
