# n8n Diagnose Error Skill

Automated error diagnosis for failed n8n workflow executions. Get instant root cause analysis and recommended fixes when your workflows fail.

---

## Overview

The n8n-diagnose-error skill automates the manual debugging process for failed n8n workflows. Instead of manually checking execution logs, tracing data flow, and pattern matching, simply invoke this skill and get:

- **Automated Root Cause Analysis**: Identifies the underlying issue, not just the symptom
- **Pattern-Based Recognition**: Matches against 6 major error patterns with detection logic
- **Actionable Fix Recommendations**: Provides code examples and step-by-step fixes
- **Evidence-Based Diagnosis**: Shows actual execution data supporting the diagnosis
- **Cross-Skill Integration**: References other n8n skills for deeper guidance

---

## Problem Statement

**Manual Debugging is Time-Consuming**:
- Check workflow structure
- Fetch execution data
- Trace error through nodes
- Examine upstream data
- Identify root cause
- Research potential fixes
- **Total Time**: 15-30 minutes per error

**Common Errors are Recognizable**:
- SSH race conditions
- Expression reference errors
- Rate limiting issues
- Credential expiry
- Timeouts
- Type mismatches

**Solution**: Automate this process with the n8n-diagnose-error skill.

---

## Invocation Methods

### By Workflow ID
```
/diagnose-error abc123xyz
```

### By Workflow Name
```
/diagnose-error "API Integration Workflow"
```

### What Happens Automatically
1. Fetches workflow structure
2. Gets last 10 executions
3. Identifies most recent failure
4. Analyzes with error mode (upstream context)
5. Pattern matches against known errors
6. Generates diagnostic report

**Time Saved**: ~20 minutes per diagnosis

---

## Key Features

### 1. Automatic Data Fetching

No need to manually run MCP tools - the skill handles:
- `n8n_list_workflows` (if name provided)
- `n8n_get_workflow` (structure analysis)
- `n8n_executions` (list recent runs)
- `n8n_executions` with mode="error" (detailed diagnosis)

### 2. Pattern-Based Error Recognition

Matches against 6 major patterns:
- **SSH Race Condition**: File operations across sessions
- **Expression Reference**: Missing property access
- **Rate Limiting**: API throttling
- **Credential Expiry**: Expired tokens
- **Timeout**: Slow operations
- **Data Type Mismatch**: String vs number issues

Each pattern includes:
- Detection signature
- Evidence markers
- Confidence scoring
- Fix strategies

### 3. Root Cause Identification

Distinguishes **error** (symptom) from **root cause**:
- Traces execution path backward
- Examines upstream data flow
- Identifies originating node
- Reports true source of issue

### 4. Actionable Fix Recommendations

Every diagnosis includes:
- Specific code examples
- Configuration changes
- Step-by-step instructions
- Priority-ranked fixes
- Cross-references to related skills

### 5. Evidence-Based Reporting

Shows actual data supporting diagnosis:
- Error messages (verbatim)
- Execution path visualization
- Upstream data samples
- Node configuration snippets
- Timing information

### 6. Cross-Skill Integration

References other n8n skills for deeper guidance:
- **n8n-expression-syntax**: For expression fixes
- **n8n-validation-expert**: For validation errors
- **n8n-workflow-patterns**: For architectural guidance
- **n8n-node-configuration**: For setup issues
- **n8n-mcp-tools-expert**: For advanced analysis

---

## Usage Examples

### Example 1: Quick Diagnosis of Latest Failure

**Input**:
```
/diagnose-error L4e8eIb2mri8DJp3
```

**Output**:
```markdown
## 🔍 Diagnostic Report: Todoist Task Processor

### Executive Summary
SSH race condition: file written in one session not visible in next session.

### Root Cause
**Problem**: Multiple SSH nodes using separate sessions
**Originating Node**: Workflow architecture (Write Task + Execute Skill)
**Error Type**: SSH Race Condition

### Evidence
- ✅ "Write Task to VM" succeeded (exitCode 0)
- ✅ "Execute Claude Skill" fails: "file doesn't exist"
- ✅ Sequential SSH nodes detected
- ✅ File name matches between write and read

### Recommended Fix

**Priority 1**: Combine operations in single SSH node

```bash
cat > /path/file.json << 'EOF'
{{ JSON.stringify($json) }}
EOF

cd /path && command-using-file
```

**Why This Works**: Single SSH session ensures file immediately available.

### Related Documentation
- See [COMMON_ERRORS.md] → Connection/Race Conditions → SSH Session Race
- See [n8n-workflow-patterns] for SSH execution patterns
```

### Example 2: Deep Analysis of Recurring Errors

**Input**:
```
/diagnose-error "User Registration Flow" --mode deep
```

**Output**:
Analyzes last 20 executions, identifies pattern across failures, reports systematic issue.

### Example 3: Comparative Analysis

**Input**:
```
/diagnose-error xyz789 --mode comparative
```

**Output**:
Compares failed vs successful executions, identifies what changed (data, timing, environment).

---

## Diagnostic Process

### Step-by-Step Workflow

```
1. Identify Workflow
   ├─ By ID → Use directly
   └─ By Name → Search with n8n_list_workflows

2. Fetch Workflow Structure
   └─ Use n8n_get_workflow (mode: structure)

3. Fetch Recent Executions
   └─ Use n8n_executions (action: list, limit: 10)

4. Get Failed Execution Details ⭐ CRITICAL
   └─ Use n8n_executions (action: get, mode: "error")
      ├─ includeExecutionPath: true
      ├─ errorItemsLimit: 2
      └─ Provides upstream context automatically

5. Pattern Match & Root Cause Analysis
   ├─ Match error signature against 6 patterns
   ├─ Collect supporting evidence
   ├─ Calculate confidence score
   └─ Trace to originating node

6. Generate Diagnostic Report
   ├─ Executive summary
   ├─ Root cause identification
   ├─ Evidence presentation
   ├─ Recommended fixes (with code)
   └─ Cross-references to other skills
```

### Critical Tool Usage

**Always Use mode="error"**:
```javascript
n8n_executions({
  action: "get",
  id: "execution-id",
  mode: "error",  // ⭐ Optimized for diagnostics
  includeExecutionPath: true,
  errorItemsLimit: 2
})
```

**Why mode="error" is Essential**:
- Automatically includes upstream node data
- Shows execution path leading to failure
- Provides sample data causing the error
- Filters out noise from successful nodes
- Saves analysis time and tokens

---

## File Structure

```
n8n-diagnose-error/
├── SKILL.md                    # Main skill file (~600 lines)
│   ├── Diagnostic Philosophy
│   ├── Quick Start Guide
│   ├── 6-Step Process
│   ├── Error Analysis Modes
│   ├── MCP Tool Integration
│   └── Cross-Skill References
│
├── COMMON_ERRORS.md            # Error catalog (~800 lines)
│   ├── 1. Expression Errors
│   ├── 2. Connection & Race Conditions
│   ├── 3. Authentication & Credentials
│   ├── 4. API Integration Errors
│   ├── 5. Data Processing Errors
│   └── 6. Workflow Structure Errors
│
├── DIAGNOSTIC_PATTERNS.md      # Pattern recognition (~500 lines)
│   ├── SSH Race Condition Pattern
│   ├── Expression Reference Pattern
│   ├── Rate Limiting Pattern
│   ├── Credential Expiry Pattern
│   ├── Timeout Pattern
│   └── Data Type Mismatch Pattern
│
├── EXECUTION_ANALYSIS.md       # Data interpretation (~400 lines)
│   ├── Execution Data Structure
│   ├── Error Mode Analysis
│   ├── Root Cause Tracing
│   ├── Comparative Analysis
│   └── Evidence Collection
│
└── README.md                   # This file
```

---

## Cross-Skill Integration

### With n8n-expression-syntax
**Trigger**: Expression-related errors ({{ }})
**Handoff**: Provide expression fixes and common mistakes

**Example**:
```markdown
Root cause: Expression reference error

For expression syntax help, see [n8n-expression-syntax]:
- Safe navigation operators
- Fallback values
- Common expression mistakes
```

### With n8n-validation-expert
**Trigger**: Validation errors or workflow structure issues
**Handoff**: Provide validation error interpretation

**Example**:
```markdown
Root cause: Workflow validation failure

For validation guidance, see [n8n-validation-expert]:
- Error severity levels
- Validation loop process
- False positive handling
```

### With n8n-workflow-patterns
**Trigger**: Architectural issues or pattern violations
**Handoff**: Recommend proven workflow patterns

**Example**:
```markdown
Root cause: SSH race condition (workflow architecture)

For pattern guidance, see [n8n-workflow-patterns]:
- SSH execution patterns
- Webhook processing best practices
- API integration patterns
```

### With n8n-node-configuration
**Trigger**: Node configuration or property issues
**Handoff**: Provide node setup guidance

**Example**:
```markdown
Root cause: Missing required node property

For configuration help, see [n8n-node-configuration]:
- Property dependencies
- Operation-specific requirements
- Common configuration patterns
```

---

## Limitations

### Cannot Diagnose Without Execution History
If workflow has never run:
- No execution data available
- Can only validate workflow structure
- Cannot analyze runtime errors

**Workaround**: Run workflow once to generate execution data

### Limited to Known Patterns
Diagnosis accuracy depends on pattern catalog:
- 6 major patterns well-covered
- Novel errors may need manual analysis
- Catalog expands over time

**Workaround**: Use raw execution data analysis for unknown patterns

### Complex Multi-Factor Issues
Some failures involve multiple causes:
- May identify primary cause but miss secondary
- Cascading failures can be complex

**Workaround**: Use Deep Mode to analyze multiple failures

### UI-Based Fixes
Cannot automate fixes requiring n8n UI:
- Credential setup/refresh
- OAuth authorization
- Workflow structure edits in UI

**Workaround**: Provide clear UI-based instructions

---

## Success Metrics

**Based on Expected Performance**:
- ✅ 80%+ accurate root cause identification
- ✅ 90%+ diagnoses include actionable fixes
- ✅ Average diagnosis time: 30-60 seconds
- ✅ 70%+ reduction in manual debugging time
- ✅ Pattern catalog covers most common errors

---

## Contributing

Want to add new error patterns or improve diagnostics?

1. **Identify New Pattern**: Document error signature and evidence
2. **Add to COMMON_ERRORS.md**: Create new error entry
3. **Add to DIAGNOSTIC_PATTERNS.md**: Define detection logic
4. **Test**: Verify pattern matching works
5. **Submit**: Create pull request with examples

---

## Related Skills

- **n8n-expression-syntax**: Fix expression errors
- **n8n-validation-expert**: Understand validation errors
- **n8n-workflow-patterns**: Learn proven patterns
- **n8n-node-configuration**: Configure nodes correctly
- **n8n-mcp-tools-expert**: Use MCP tools effectively

---

## Credits

Conceived by Romuald Członkowski - https://www.aiadvisors.pl/en

Part of the n8n-mcp-skills project: https://github.com/czlonkowski/n8n-skills

---

## License

MIT License - See parent repository LICENSE file.
