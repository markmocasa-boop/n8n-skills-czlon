---
name: n8n-diagnose-error
description: Automate error diagnosis for failed n8n workflow executions. Use when a workflow execution fails, you need to identify the root cause, or you want to analyze execution errors and get recommended fixes. Can be invoked with workflow ID or name.
---

# n8n Error Diagnosis Expert

Automated root cause analysis and error diagnosis for failed n8n workflow executions.

---

## Diagnostic Philosophy

**Error vs Symptom Distinction**

When a workflow fails, the error message you see is often a *symptom* of a deeper *root cause*. This skill helps you trace execution failures back to their origin.

**Key Principle**: Always trace back to the originating node, not just the node that threw the error.

**Data-Driven Diagnosis**

This skill uses actual execution data to diagnose issues, not just theory:
- Analyze real error messages and stack traces
- Examine upstream node data flow
- Compare failed vs successful executions
- Pattern-match against known error signatures

**Automated Root Cause Analysis**

The diagnostic process follows a systematic approach:
1. Fetch workflow structure and execution history
2. Identify failed execution with error mode analysis
3. Pattern-match against known error types
4. Trace error to originating node
5. Generate actionable recommendations

---

## Quick Start

### Invocation Methods

**By Workflow ID**:
```
/diagnose-error abc123xyz
```

**By Workflow Name**:
```
/diagnose-error "API Integration Workflow"
```

**Auto-fetch Latest Failure**:
When invoked, the skill automatically:
- Fetches the workflow structure
- Gets the last 10 executions
- Identifies the most recent failure
- Analyzes error with upstream context
- Generates diagnostic report

---

## The 6-Step Diagnostic Process

### Step 1: Identify Workflow

**If Workflow ID Provided**:
Use the ID directly to fetch workflow data.

**If Workflow Name Provided**:
```javascript
// Use n8n_list_workflows to search
mcp__n8n-mcp__n8n_list_workflows({
  // Name matching will happen in the response filtering
})
```

**MCP Tool**: `mcp__n8n-mcp__n8n_list_workflows`
- **Parameters**: `limit: 100` (search through recent workflows)
- **Filter Response**: Match workflow name against `workflow.name`

### Step 2: Fetch Workflow Structure

Get the workflow structure to understand node relationships and data flow.

```javascript
mcp__n8n-mcp__n8n_get_workflow({
  id: "workflow-id",
  mode: "structure"  // Nodes + connections, no full config
})
```

**MCP Tool**: `mcp__n8n-mcp__n8n_get_workflow`
- **Parameters**:
  - `id`: Workflow ID
  - `mode: "structure"` (faster, includes nodes and connections)
- **Purpose**: Understand workflow topology for error tracing

### Step 3: Fetch Recent Executions

Get the last 10 executions to identify failures and patterns.

```javascript
mcp__n8n-mcp__n8n_executions({
  action: "list",
  workflowId: "workflow-id",
  limit: 10,
  includeData: false  // Metadata only for listing
})
```

**MCP Tool**: `mcp__n8n-mcp__n8n_executions`
- **Parameters**:
  - `action: "list"`
  - `workflowId`: The workflow ID
  - `limit: 10` (recent executions)
  - `includeData: false` (faster listing)
- **Purpose**: Identify failed executions (status: "error")

### Step 4: Get Failed Execution Details (ERROR MODE)

**This is the most important step** - use mode="error" for focused diagnostics.

```javascript
mcp__n8n-mcp__n8n_executions({
  action: "get",
  id: "execution-id",
  mode: "error",  // ⭐ Optimized for error debugging
  includeExecutionPath: true,  // Shows path leading to error
  errorItemsLimit: 2  // Sample upstream data
})
```

**MCP Tool**: `mcp__n8n-mcp__n8n_executions`
- **Parameters**:
  - `action: "get"`
  - `id`: Execution ID of failed run
  - `mode: "error"` ⭐ **CRITICAL** - provides:
    - Error node data
    - Upstream node data (for tracing)
    - Execution path to error
    - Sample problematic items
  - `includeExecutionPath: true` - shows execution flow
  - `errorItemsLimit: 2` - sample items from upstream
- **Purpose**: Get focused error context for diagnosis

**Why mode="error" is Essential**:
- Automatically includes upstream node data
- Shows execution path leading to failure
- Provides sample data that caused the error
- Filters out noise from successful nodes
- Saves tokens and analysis time

### Step 5: Pattern Match & Root Cause Analysis

Analyze the error data against known patterns (see DIAGNOSTIC_PATTERNS.md):

**Pattern Signatures to Check**:
1. **SSH Race Condition**: Multiple SSH nodes, "file doesn't exist", intermittent failures
2. **Expression Reference Error**: "Cannot read property X of undefined", missing data
3. **Rate Limiting**: HTTP 429, "Too Many Requests", API nodes
4. **Credential Expiry**: HTTP 401/403, previously working workflow
5. **Timeout**: "ETIMEDOUT", "ESOCKETTIMEDOUT", 504 errors
6. **Data Type Mismatch**: "Expected number got string", validation failures

**Root Cause Tracing**:
- Find the **originating node** (not just the node that threw the error)
- Trace data flow backward through execution path
- Identify where data became problematic
- Distinguish error from symptom

**Evidence Collection**:
- Extract error message and stack trace
- Capture sample data from upstream nodes
- Record node configuration if relevant
- Note execution timing if timeout-related

### Step 6: Generate Diagnostic Report

**Output Structure**:

```markdown
## 🔍 Diagnostic Report: [Workflow Name]

### Executive Summary
[1-2 sentence root cause identification]

### Root Cause
**Problem**: [Clear identification of the issue]
**Originating Node**: [Node name where issue originates]
**Error Type**: [Pattern name from catalog]

### Error Context
- **Failed Node**: [Node that threw error]
- **Execution Path**: [Sequence of nodes leading to error]
- **Upstream Data**: [Relevant data from previous nodes]

### Evidence
```
[Error message, stack trace, or sample data]
```

### Recommended Fixes

**Priority 1** (Must Fix):
[Actionable fix with code example]

**Priority 2** (Should Fix):
[Additional improvements]

**Priority 3** (Optional):
[Best practice suggestions]

### Code Examples

```javascript
// Before (problematic)
[Original configuration]

// After (fixed)
[Fixed configuration]
```

### Related Documentation
- See [n8n-expression-syntax] for expression fixes
- See [n8n-validation-expert] for validation issues
- See [n8n-workflow-patterns] for pattern best practices
```

---

## Error Analysis Modes

### Quick Mode (Default)
Analyzes the most recent failed execution only.

**Use When**:
- Single failure needs diagnosis
- Fast root cause needed
- Error is obvious

**Process**:
1. Get last 10 executions
2. Find first failure (most recent)
3. Analyze with mode="error"
4. Report findings

### Deep Mode
Analyzes multiple failures to identify patterns.

**Use When**:
- Intermittent failures occur
- Root cause is unclear
- Need to compare multiple failures

**Process**:
1. Get last 20-50 executions
2. Filter for failures only
3. Analyze each with mode="error"
4. Identify common patterns across failures
5. Report systematic issues

### Comparative Mode
Compares failed vs successful executions.

**Use When**:
- Workflow sometimes succeeds, sometimes fails
- Environmental factors suspected
- Data variation may be the cause

**Process**:
1. Get recent executions (both failed and successful)
2. Compare execution data between successes and failures
3. Identify differences (timing, data values, environment)
4. Report what changed

### Root Cause Mode
Deep trace from error back to origination.

**Use When**:
- Error occurs far from root cause
- Complex workflow with many nodes
- Cascading failures suspected

**Process**:
1. Get failed execution with mode="error"
2. Examine execution path
3. Trace backward through upstream nodes
4. Identify where data became problematic
5. Report true root cause (not just symptom)

---

## Integration with n8n MCP Tools

### Tool: n8n_list_workflows
**Purpose**: Find workflow by name when ID not provided

**Usage**:
```javascript
mcp__n8n-mcp__n8n_list_workflows({
  limit: 100  // Get recent workflows
})
// Then filter by name in the response
```

**When to Use**:
- User provides workflow name instead of ID
- Need to search for workflow
- Fuzzy matching needed

### Tool: n8n_get_workflow
**Purpose**: Get workflow structure for topology analysis

**Usage**:
```javascript
mcp__n8n-mcp__n8n_get_workflow({
  id: "workflow-id",
  mode: "structure"  // Faster than "full"
})
```

**When to Use**:
- Need to understand node relationships
- Want to see workflow topology
- Analyzing connection issues

**Modes**:
- `"structure"`: Nodes + connections (recommended for diagnosis)
- `"full"`: Complete workflow JSON (use if need full config)
- `"minimal"`: Just metadata (not useful for diagnosis)

### Tool: n8n_executions
**Purpose**: Fetch and analyze execution data

**Action: "list"** - Get recent executions:
```javascript
mcp__n8n-mcp__n8n_executions({
  action: "list",
  workflowId: "workflow-id",
  limit: 10,
  status: "error",  // Optional: filter for failures only
  includeData: false  // Faster listing
})
```

**Action: "get"** - Get execution details:
```javascript
mcp__n8n-mcp__n8n_executions({
  action: "get",
  id: "execution-id",
  mode: "error",  // ⭐ ALWAYS use for diagnostics
  includeExecutionPath: true,
  errorItemsLimit: 2
})
```

**Critical Parameters**:
- `mode: "error"` - Optimized for error analysis
- `includeExecutionPath: true` - Shows execution flow
- `errorItemsLimit: 2` - Sample data from upstream

### Tool: n8n_validate_workflow
**Purpose**: Check for structural issues

**Usage**:
```javascript
mcp__n8n-mcp__n8n_validate_workflow({
  id: "workflow-id",
  options: {
    validateConnections: true,
    validateExpressions: true,
    validateNodes: true,
    profile: "runtime"
  }
})
```

**When to Use**:
- Suspect broken connections
- Expression syntax issues
- Workflow structure problems
- After seeing "invalid reference" errors

---

## Cross-Skill Integration

### With n8n-expression-syntax
**Use When**: Error involves expressions ({{ }})

**Handoff Pattern**:
```markdown
Root cause identified: Expression syntax error

See [n8n-expression-syntax] for:
- Common expression mistakes
- Safe navigation operators
- Fallback value patterns
```

### With n8n-validation-expert
**Use When**: Validation errors or structural issues

**Handoff Pattern**:
```markdown
Root cause identified: Validation failure

See [n8n-validation-expert] for:
- Understanding validation errors
- Error severity levels
- Validation loop process
```

### With n8n-workflow-patterns
**Use When**: Architectural issues or pattern violations

**Handoff Pattern**:
```markdown
Root cause identified: Workflow pattern issue

See [n8n-workflow-patterns] for:
- Webhook processing patterns
- SSH execution best practices
- API integration patterns
```

### With n8n-node-configuration
**Use When**: Node configuration or property issues

**Handoff Pattern**:
```markdown
Root cause identified: Node misconfiguration

See [n8n-node-configuration] for:
- Property dependencies
- Operation-specific requirements
- Common configuration patterns
```

### With n8n-mcp-tools-expert
**Use When**: Need help using MCP tools for further analysis

**Handoff Pattern**:
```markdown
For deeper analysis, use these MCP tools:

See [n8n-mcp-tools-expert] for:
- Advanced search techniques
- Node introspection methods
- Template discovery
```

---

## Common Diagnostic Scenarios

### Scenario 1: "File Doesn't Exist" SSH Error

**Symptoms**:
- SSH node fails with "file doesn't exist"
- File was just created in previous node
- Intermittent failures

**Diagnostic Steps**:
1. Check execution path - are there multiple SSH nodes?
2. Examine upstream Write/Execute node sequence
3. Identify if SSH sessions are separate

**Root Cause**: SSH race condition (separate sessions)

**Fix**: Combine operations in single SSH node:
```bash
cat > /path/file.json << 'EOF'
{{ JSON.stringify($json, null, 2) }}
EOF

cd /path && command-using-file
```

**Reference**: See COMMON_ERRORS.md → Connection/Race Conditions → SSH Session Race

### Scenario 2: "Cannot Read Property" Expression Error

**Symptoms**:
- Error: "Cannot read property 'field' of undefined"
- Expression references upstream node data
- Works sometimes, fails other times

**Diagnostic Steps**:
1. Get execution with mode="error" and errorItemsLimit=2
2. Examine upstream node sample data
3. Check if property exists in all cases

**Root Cause**: Data structure variation or missing property

**Fix**: Use safe navigation or fallback:
```javascript
// Before
{{ $json.body.field }}

// After
{{ $json.body?.field ?? 'default' }}
```

**Reference**: See COMMON_ERRORS.md → Expression Errors → Missing Property Access

### Scenario 3: HTTP 429 Rate Limiting

**Symptoms**:
- HTTP 429 "Too Many Requests"
- API node failing after N requests
- Works initially, then fails

**Diagnostic Steps**:
1. Check execution timing in failed runs
2. Count number of API calls per minute
3. Check if requests are parallelized

**Root Cause**: Exceeding API rate limits

**Fix**: Add retry logic and delays:
```javascript
// In node settings
{
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000  // 5 second delay
}
```

**Reference**: See COMMON_ERRORS.md → API Integration Errors → Rate Limiting

---

## Best Practices

### Always Use mode="error"
When fetching failed execution details, **always** use mode="error":
- Provides upstream context automatically
- Includes execution path
- Samples problematic data
- Filters noise

### Trace to Root Cause
Don't stop at the node that threw the error:
- Follow execution path backward
- Check upstream data
- Identify where data became problematic
- Report the originating issue

### Provide Actionable Fixes
Every diagnosis should include:
- Specific code examples
- Configuration changes
- Step-by-step fix instructions
- Cross-references to related skills

### Cross-Reference Other Skills
When diagnosis reveals related issues:
- Reference expression syntax skill for {{ }} fixes
- Reference validation expert for structural issues
- Reference workflow patterns for architectural guidance
- Reference node configuration for setup problems

### Collect Evidence
Support your diagnosis with:
- Actual error messages
- Sample data from execution
- Execution timing information
- Node configuration snippets

---

## Limitations

### Cannot Diagnose Without Execution History
If workflow has never run or executions are deleted, diagnosis is limited to:
- Workflow structure validation
- Configuration analysis
- Theoretical issues

**Workaround**: Run the workflow once to generate execution data.

### Limited to Known Patterns
Diagnosis accuracy depends on pattern catalog coverage:
- Common errors are well-covered
- Novel or unique errors may need manual analysis
- Pattern catalog expands over time

**Workaround**: Use raw execution data analysis when pattern matching fails.

### Complex Multi-Factor Issues
Some failures involve multiple contributing factors:
- May identify primary cause but miss secondary issues
- Cascading failures can be complex to trace

**Workaround**: Use Deep Mode to analyze multiple failures.

### UI-Based Issues
Cannot fix issues requiring Notion UI interaction:
- Credential setup/configuration
- OAuth authorization flows
- Manual workflow adjustments

**Workaround**: Provide instructions for UI-based fixes.

### Depends on Error Message Quality
Diagnosis quality depends on n8n's error messages:
- Vague errors are harder to diagnose
- Some errors don't include enough context
- Stack traces may be truncated

**Workaround**: Use comparative analysis to gather more context.

---

## Edge Cases

### No Failed Executions Found
**Response**:
```markdown
✅ No recent failures found for this workflow.

Last 10 executions all succeeded.

If you're experiencing issues:
1. Check if workflow is active
2. Verify trigger configuration
3. Run a test execution manually
```

### Multiple Possible Root Causes
**Response**:
```markdown
⚠️ Multiple potential root causes identified:

**Most Likely** (70% confidence):
[Primary root cause with evidence]

**Also Possible** (30% confidence):
[Secondary root cause with evidence]

Recommend investigating primary cause first.
```

### Unknown Error Pattern
**Response**:
```markdown
🔍 Error pattern not in catalog.

**Raw Analysis**:
- Error Node: [node name]
- Error Message: [message]
- Upstream Data: [data sample]

**Recommendation**:
Analyze the execution data manually:
[Provide execution data and interpretation guidance]

Consider this a new pattern to document.
```

### Workflow Not Found
**Response**:
```markdown
❌ Workflow not found: "workflow-name-or-id"

**Similar workflows** (by name):
- [List of similar workflow names]

Did you mean one of these?
```

### All Recent Executions Failed
**Response**:
```markdown
🚨 Critical: All recent executions failed

**Pattern Analysis**:
[Analyze if same error or different errors]

**Possible Systematic Issues**:
- Configuration change
- Credential expiry
- External service outage
- Workflow structural issue

Recommend checking for recent workflow modifications.
```

---

## Reference Files

- **COMMON_ERRORS.md**: Comprehensive error catalog with 6 categories
- **DIAGNOSTIC_PATTERNS.md**: Pattern recognition guide with detection logic
- **EXECUTION_ANALYSIS.md**: Guide to interpreting n8n execution data
- **README.md**: Skill overview and usage examples

---

## Credits

Conceived by Romuald Członkowski - https://www.aiadvisors.pl/en

Part of the n8n-mcp-skills project.
