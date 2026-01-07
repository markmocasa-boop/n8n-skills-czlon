# n8n Execution Data Analysis Guide

Guide to interpreting n8n execution data for effective error diagnosis. Learn how to read execution responses, trace errors to their source, and extract diagnostic evidence.

---

## Table of Contents

1. [Execution Data Structure](#execution-data-structure)
2. [Error Mode Analysis](#error-mode-analysis)
3. [Root Cause Tracing](#root-cause-tracing)
4. [Comparative Analysis](#comparative-analysis)
5. [Evidence Collection](#evidence-collection)

---

## Execution Data Structure

### Execution Response Format

When you fetch an execution with `n8n_executions`, you get a structured response containing:

```json
{
  "id": "123",
  "workflowId": "workflow-abc",
  "mode": "trigger",
  "status": "error",
  "startedAt": "2024-01-07T10:00:00.000Z",
  "stoppedAt": "2024-01-07T10:00:15.234Z",
  "finished": true,
  "data": {
    "resultData": {
      "runData": {
        "Node Name 1": [...],
        "Node Name 2": [...],
        "Failed Node": [...]
      },
      "lastNodeExecuted": "Failed Node",
      "error": {
        "message": "Error message here",
        "node": "Failed Node",
        "stack": "Error stack trace..."
      }
    },
    "executionData": {
      "contextData": {},
      "nodeExecutionStack": [],
      "metadata": {},
      "waitingExecution": {},
      "waitingExecutionSource": {}
    }
  }
}
```

### Key Fields Explained

**Top Level**:
- `id`: Unique execution identifier
- `workflowId`: The workflow this execution belongs to
- `status`: "success", "error", or "waiting"
- `startedAt` / `stoppedAt`: Execution timing
- `finished`: Whether execution completed (true/false)

**data.resultData**:
- `runData`: Object containing execution results for each node
- `lastNodeExecuted`: Name of the last node that ran
- `error`: Error object (only present if status = "error")

**runData Structure**:
```json
{
  "Node Name": [
    {
      "startTime": 1234567890,
      "executionTime": 150,  // milliseconds
      "data": {
        "main": [[...]]  // Output data items
      },
      "source": [...]  // Input data
    }
  ]
}
```

### Node Execution Order

n8n executes nodes in a specific order based on workflow connections. The `runData` object shows which nodes executed and in what order.

**Example**:
```json
{
  "runData": {
    "Webhook": [...]        // 1st
    "Clean Input": [...]    // 2nd
    "Write File": [...]     // 3rd
    "Execute Command": [...] // 4th - FAILED
  },
  "lastNodeExecuted": "Execute Command"
}
```

**Execution Path**: Webhook → Clean Input → Write File → Execute Command (failed)

---

## Error Mode Analysis

### What is Error Mode?

`mode="error"` is a specialized execution fetch mode that provides:
1. **Error node data** - Complete output from failed node
2. **Upstream node data** - Sample data from nodes before failure
3. **Execution path** - Sequence of nodes leading to error
4. **Error context** - Stack trace, error message, node details

### Using Error Mode

**Standard Fetch** (not recommended for diagnostics):
```javascript
mcp__n8n-mcp__n8n_executions({
  action: "get",
  id: "123",
  mode: "summary"  // Only shows structure, limited data
})
```

**Error Mode Fetch** (⭐ RECOMMENDED):
```javascript
mcp__n8n-mcp__n8n_executions({
  action: "get",
  id: "123",
  mode: "error",  // ⭐ Optimized for diagnostics
  includeExecutionPath: true,  // Shows execution flow
  errorItemsLimit: 2  // Sample items from upstream nodes
})
```

### Error Mode Response

**What You Get**:
```json
{
  "error": {
    "message": "Cannot read property 'email' of undefined",
    "node": "Process User",
    "timestamp": "2024-01-07T10:00:15.234Z",
    "stack": "Error: Cannot read property...\n  at ..."
  },
  "errorNode": {
    "name": "Process User",
    "type": "n8n-nodes-base.set",
    "parameters": {
      "values": {
        "string": [
          {
            "name": "email",
            "value": "={{ $json.body.user.email }}"  // ← Failing expression
          }
        ]
      }
    }
  },
  "executionPath": [
    "Webhook",
    "Clean Input",
    "Process User"  // ← Failed here
  ],
  "upstreamData": {
    "Clean Input": {
      "data": {
        "main": [[
          {"body": {"user": {"name": "John"}}},  // ❌ Missing 'email'
          {"body": {"user": {"name": "Jane", "email": "jane@example.com"}}}  // ✅ Has 'email'
        ]]
      }
    }
  }
}
```

### Critical Parameters

**includeExecutionPath**:
- `true`: Shows sequence of nodes that executed before error
- Use for: Understanding execution flow

**errorItemsLimit**:
- `0`: No sample data (just error message)
- `2`: Sample 2 items from upstream (recommended)
- `-1`: All items (can be very large, use cautiously)
- Use for: Identifying problematic data patterns

**includeStackTrace**:
- `false`: Truncated stack trace (default)
- `true`: Full stack trace
- Use for: Deep debugging of code errors

### Interpreting Error Mode Data

**Step 1: Identify Failed Node**
```json
"error": {
  "node": "Process User"  // ← This node threw the error
}
```

**Step 2: Check Execution Path**
```json
"executionPath": [
  "Webhook",      // 1st
  "Clean Input",  // 2nd
  "Process User"  // 3rd - Failed
]
```
**Insight**: Error occurred after 2 successful nodes.

**Step 3: Examine Upstream Data**
```json
"upstreamData": {
  "Clean Input": {
    "data": {"main": [[
      {"body": {"user": {"name": "John"}}},  // ❌ No 'email'
    ]]}
  }
}
```
**Insight**: Upstream node missing expected 'email' property.

**Step 4: Locate Failing Expression**
```json
"errorNode": {
  "parameters": {
    "values": {
      "string": [{
        "value": "={{ $json.body.user.email }}"  // ← Problem here
      }]
    }
  }
}
```
**Insight**: Expression assumes 'email' always exists.

---

## Root Cause Tracing

### Error vs Symptom

**Symptom**: The node that threw the error
**Root Cause**: The underlying reason for failure

**Example**:
- **Symptom**: "Process User" node fails with "Cannot read property 'email'"
- **Root Cause**: Upstream "Webhook" received data without 'email' field

### Tracing Methodology

**Step 1: Identify Symptom Node**
```javascript
const symptomNode = execution.error.node;  // "Process User"
```

**Step 2: Find Position in Execution Path**
```javascript
const pathIndex = execution.executionPath.indexOf(symptomNode);  // 2
const upstreamNodes = execution.executionPath.slice(0, pathIndex);  // ["Webhook", "Clean Input"]
```

**Step 3: Examine Upstream Data**
```javascript
// Check each upstream node's output
for (const nodeName of upstreamNodes.reverse()) {
  const nodeData = execution.upstreamData[nodeName];
  // Look for problematic data patterns
}
```

**Step 4: Identify Data Transformation Issue**
```javascript
// Where did the data become problematic?
// - Original data source (Webhook)
// - Data transformation node (Clean Input)
// - Failing node assumption (Process User)
```

### Common Root Cause Patterns

**Pattern 1: Data Source Variation**
```
Root Cause: Webhook (data source)
↓
Symptom: Process User (expression error)

Fix Location: Process User (add safe navigation)
```

**Pattern 2: Missing Transformation**
```
Root Cause: API returns strings for numbers
↓
Transform Node: Doesn't convert types
↓
Symptom: Calculate node (type mismatch)

Fix Location: Add type conversion in Transform node
```

**Pattern 3: Timing/Race Condition**
```
Root Cause: Write File node (separate SSH session)
↓
Symptom: Read File node (file doesn't exist)

Fix Location: Combine into single SSH node
```

### Finding the Originating Node

**Algorithm**:
```javascript
function findOriginatingNode(execution) {
  const failedNode = execution.error.node;
  const path = execution.executionPath;
  const failedIndex = path.indexOf(failedNode);

  // Work backward from failed node
  for (let i = failedIndex - 1; i >= 0; i--) {
    const nodeName = path[i];
    const nodeData = execution.upstreamData[nodeName];

    // Check if this node introduced the problem
    if (isProblematicelData(nodeData, execution.error)) {
      return {
        originating: nodeName,
        symptom: failedNode,
        reason: "Data from this node caused failure"
      };
    }
  }

  // If no upstream issue found, problem is in failed node itself
  return {
    originating: failedNode,
    symptom: failedNode,
    reason: "Node configuration or logic issue"
  };
}
```

---

## Comparative Analysis

### Failed vs Successful Execution

**Use Case**: Workflow works sometimes, fails other times

**Process**:
1. Fetch recent successful execution
2. Fetch recent failed execution
3. Compare data structures
4. Identify differences

**Example**:

**Successful Execution Data**:
```json
{
  "Webhook": {
    "data": {"main": [[
      {
        "body": {
          "user": {
            "name": "John",
            "email": "john@example.com"  // ✅ Has email
          }
        }
      }
    ]]}
  }
}
```

**Failed Execution Data**:
```json
{
  "Webhook": {
    "data": {"main": [[
      {
        "body": {
          "user": {
            "name": "Jane"
            // ❌ Missing email
          }
        }
      }
    ]]}
  }
}
```

**Insight**: Email field is optional in webhook payload.

### Timing Comparison

**Check Execution Duration**:
```javascript
const successful = {
  startedAt: "2024-01-07T10:00:00.000Z",
  stoppedAt: "2024-01-07T10:00:05.123Z",
  duration: 5123  // ms
};

const failed = {
  startedAt: "2024-01-07T10:05:00.000Z",
  stoppedAt: "2024-01-07T10:05:11.456Z",
  duration: 11456  // ms - took longer!
};
```

**Insight**: Failed execution took 2x longer, suggests timeout or performance issue.

### Environmental Differences

**Factors to Compare**:
- Time of day (peak hours vs off-peak)
- Day of week (weekday vs weekend)
- Data volume (small payload vs large payload)
- Concurrent executions
- External service availability

**Example**:
```javascript
const pattern = {
  successfulExecutions: [
    { time: "02:00 AM", items: 10, duration: 5000 },
    { time: "03:00 AM", items: 15, duration: 6000 }
  ],
  failedExecutions: [
    { time: "10:00 AM", items: 100, duration: 11000 },  // ❌ Peak hours
    { time: "11:00 AM", items: 120, duration: 12000 }   // ❌ Large volume
  ]
};

// Insight: Failures during peak hours with high volume
// Root Cause: Rate limiting or resource contention
```

---

## Evidence Collection

### What Evidence to Collect

**For Every Diagnosis**:
1. ✅ Error message (verbatim)
2. ✅ Failed node name and type
3. ✅ Execution path (sequence of nodes)
4. ✅ Timestamp

**For Pattern Matching**:
5. ✅ Upstream data samples (2+ items)
6. ✅ Node configuration (parameters)
7. ✅ Execution timing

**For Comparative Analysis**:
8. ✅ Recent execution history (success vs failure)
9. ✅ Environmental context (time, volume, etc.)
10. ✅ External service status

### Evidence Extraction

**Extract Error Message**:
```javascript
const errorMessage = execution.data.resultData.error.message;
```

**Extract Failed Node Config**:
```javascript
const failedNode = execution.data.resultData.error.node;
const nodeConfig = execution.data.resultData.runData[failedNode][0];
```

**Extract Upstream Sample**:
```javascript
const upstreamNode = execution.executionPath[execution.executionPath.length - 2];
const upstreamData = execution.upstreamData[upstreamNode].data.main[0];
const sample = upstreamData.slice(0, 2);  // First 2 items
```

**Extract Timing**:
```javascript
const timing = {
  started: execution.startedAt,
  stopped: execution.stoppedAt,
  duration: new Date(execution.stoppedAt) - new Date(execution.startedAt),
  nodeTimings: Object.entries(execution.data.resultData.runData).map(([name, runs]) => ({
    node: name,
    executionTime: runs[0].executionTime
  }))
};
```

### Presenting Evidence

**Format for Clarity**:
```markdown
### Evidence

**Error Message**:
```
Cannot read property 'email' of undefined
```

**Failed Node**: Process User (n8n-nodes-base.set)

**Execution Path**:
1. Webhook
2. Clean Input
3. Process User ❌ Failed here

**Upstream Data Sample** (Clean Input):
```json
[
  {"body": {"user": {"name": "John"}}},  // ❌ Missing 'email'
  {"body": {"user": {"name": "Jane", "email": "jane@example.com"}}}
]
```

**Node Configuration**:
```json
{
  "values": {
    "string": [{
      "name": "email",
      "value": "={{ $json.body.user.email }}"  // ← Assumes 'email' exists
    }]
  }
}
```

**Timing**:
- Execution Duration: 234ms
- Process User Execution Time: 45ms
- No timeout issues
```

### Evidence Quality Checklist

✅ **Good Evidence**:
- Actual error message (not paraphrased)
- Sample problematic data shown
- Configuration snippets included
- Execution path clearly laid out
- Timestamps for context

❌ **Poor Evidence**:
- Vague descriptions ("it failed")
- No data samples
- Missing node configuration
- No execution context
- Summarized instead of exact quotes

---

## Practical Examples

### Example 1: SSH Race Condition

**Execution Data**:
```json
{
  "error": {
    "message": "The file `task-abc123.json` doesn't exist",
    "node": "Execute Claude Skill"
  },
  "executionPath": ["Webhook", "Clean Input", "Write Task to VM", "Execute Claude Skill"],
  "upstreamData": {
    "Write Task to VM": {
      "exitCode": 0,
      "stdout": "",  // Successful write
      "stderr": ""
    }
  }
}
```

**Analysis**:
1. **Symptom**: Execute Claude Skill can't find file
2. **Evidence**: Write Task succeeded (exitCode 0)
3. **Execution Path**: Sequential SSH nodes
4. **Root Cause**: File written in one SSH session, not visible in next session
5. **Pattern**: SSH Race Condition (95% confidence)

### Example 2: Expression Reference Error

**Execution Data**:
```json
{
  "error": {
    "message": "Cannot read property 'email' of undefined",
    "node": "Process User"
  },
  "executionPath": ["Webhook", "Process User"],
  "upstreamData": {
    "Webhook": {
      "data": {"main": [[
        {"body": {"user": {"name": "John"}}},
        {"body": {"user": {"name": "Jane", "email": "jane@example.com"}}}
      ]]}
    }
  },
  "errorNode": {
    "parameters": {
      "values": {
        "string": [{
          "value": "={{ $json.body.user.email }}"
        }]
      }
    }
  }
}
```

**Analysis**:
1. **Symptom**: Process User expression error
2. **Evidence**: First item missing 'email' property
3. **Upstream Data**: Inconsistent structure
4. **Root Cause**: Webhook payload variation, no safe navigation
5. **Pattern**: Expression Reference Error (88% confidence)

---

## Credits

Conceived by Romuald Członkowski - https://www.aiadvisors.pl/en

Part of the n8n-mcp-skills project.
