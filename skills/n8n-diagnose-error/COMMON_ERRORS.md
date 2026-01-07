# Common n8n Workflow Errors

Comprehensive catalog of common n8n workflow errors, organized by category with diagnostic steps and fixes.

---

## Table of Contents

1. [Expression Errors](#1-expression-errors)
2. [Connection & Race Conditions](#2-connection--race-conditions)
3. [Authentication & Credentials](#3-authentication--credentials)
4. [API Integration Errors](#4-api-integration-errors)
5. [Data Processing Errors](#5-data-processing-errors)
6. [Workflow Structure Errors](#6-workflow-structure-errors)

---

## 1. Expression Errors

Errors related to n8n expression syntax ({{ }}) and data references.

### 1.1 Missing Property Access

**Error Signature**:
```
Cannot read property 'X' of undefined
Cannot read properties of undefined (reading 'X')
```

**Common Causes**:
- Referenced property doesn't exist in all data items
- Upstream node returns varying data structures
- Optional fields not always present
- Typo in property name

**Example Execution Data**:
```json
{
  "error": {
    "message": "Cannot read property 'email' of undefined",
    "node": "Process User Data"
  },
  "upstream": {
    "node": "HTTP Request",
    "data": [
      {"name": "John", "email": "john@example.com"},
      {"name": "Jane"}  // ❌ Missing 'email'
    ]
  }
}
```

**Diagnostic Steps**:
1. Get execution with `mode="error"` and `errorItemsLimit=2`
2. Check upstream node data samples
3. Verify property exists in ALL items
4. Check for typos in property name

**Recommended Fixes**:

**Option 1: Safe Navigation Operator**
```javascript
// Before (fails on missing property)
{{ $json.body.email }}

// After (returns undefined safely)
{{ $json.body?.email }}
```

**Option 2: Fallback Value**
```javascript
// Provide default value
{{ $json.body?.email ?? 'no-email@example.com' }}

// Conditional logic
{{ $json.body?.email ? $json.body.email : 'N/A' }}
```

**Option 3: Check Before Access**
```javascript
{{ $json.body && $json.body.email ? $json.body.email : 'N/A' }}
```

**Cross-Reference**: See n8n-expression-syntax → Common Mistakes → Safe Navigation

---

### 1.2 Node Reference Errors

**Error Signature**:
```
Referenced node "Node Name" not found
$node["Missing Node"] is undefined
```

**Common Causes**:
- Node was renamed or deleted
- Typo in node name reference
- Node hasn't executed yet in workflow path
- Branch condition didn't execute node

**Example Execution Data**:
```json
{
  "error": {
    "message": "Referenced node \"Get User Data\" not found",
    "node": "Send Email",
    "expression": "{{ $node[\"Get User Data\"].json.email }}"
  }
}
```

**Diagnostic Steps**:
1. Check workflow structure for node name
2. Verify node executed in execution path
3. Check for case sensitivity issues
4. Review workflow branches

**Recommended Fixes**:

**Option 1: Fix Node Name**
```javascript
// Check actual node name in workflow
{{ $node["Get User Data"].json.email }}  // ❌ Wrong name

{{ $node["Fetch User Data"].json.email }}  // ✅ Correct name
```

**Option 2: Use Node Index**
```javascript
// More stable - doesn't break on rename
{{ $('Get User Data').item.json.email }}
```

**Option 3: Check Node Executed**
```javascript
// Conditional access
{{ $node["Get User Data"] ? $node["Get User Data"].json.email : 'N/A' }}
```

**Cross-Reference**: See n8n-expression-syntax → Node References

---

### 1.3 Webhook Data Structure Errors

**Error Signature**:
```
Cannot read property 'body' of undefined
$json.body.field is not defined
```

**Common Causes**:
- Webhook payload structure varies
- Different content types (JSON vs form-data)
- Nested data structure misunderstood
- POST body vs query parameters confused

**Example Execution Data**:
```json
{
  "error": {
    "message": "Cannot read property 'task_id' of undefined",
    "node": "Clean Input",
    "expression": "{{ $json.body.task_id }}"
  },
  "webhook_data": {
    "body": {},
    "query": {
      "task_id": "abc123"  // ❌ In query, not body
    }
  }
}
```

**Diagnostic Steps**:
1. Check webhook node output structure
2. Verify data is in expected location (body vs query vs headers)
3. Test webhook with sample payload
4. Check Content-Type header

**Recommended Fixes**:

**Option 1: Correct Data Location**
```javascript
// Wrong location
{{ $json.body.task_id }}  // ❌ Not in body

// Correct location
{{ $json.query.task_id }}  // ✅ In query params
```

**Option 2: Flatten Webhook Data**
Add a Code node to normalize structure:
```javascript
// Normalize webhook data
return {
  task_id: $input.item.json.body?.task_id || $input.item.json.query?.task_id,
  // ... other fields
};
```

**Option 3: Safe Access All Locations**
```javascript
{{ $json.body?.task_id ?? $json.query?.task_id ?? 'not-found' }}
```

**Cross-Reference**: See n8n-workflow-patterns → Webhook Processing

---

### 1.4 Type Coercion Failures

**Error Signature**:
```
Cannot convert undefined or null to object
Expected number but got string
```

**Common Causes**:
- Attempting math on string values
- Concatenating incompatible types
- Null/undefined in calculations
- JSON parsing failures

**Example Execution Data**:
```json
{
  "error": {
    "message": "Cannot convert undefined to object",
    "node": "Calculate Total",
    "expression": "{{ $json.price * $json.quantity }}"
  },
  "data": {
    "price": "19.99",  // ❌ String, not number
    "quantity": null   // ❌ Null
  }
}
```

**Diagnostic Steps**:
1. Check data types of referenced fields
2. Verify upstream node returns expected types
3. Test expression with sample data
4. Check for null/undefined values

**Recommended Fixes**:

**Option 1: Explicit Type Conversion**
```javascript
// Before (type mismatch)
{{ $json.price * $json.quantity }}

// After (explicit conversion)
{{ parseFloat($json.price) * parseInt($json.quantity) }}
```

**Option 2: Safe Defaults**
```javascript
{{ (parseFloat($json.price) || 0) * (parseInt($json.quantity) || 1) }}
```

**Option 3: Validation Before Calculation**
```javascript
{{
  ($json.price && $json.quantity)
    ? parseFloat($json.price) * parseInt($json.quantity)
    : 0
}}
```

**Cross-Reference**: See n8n-expression-syntax → Type Conversion

---

## 2. Connection & Race Conditions

Errors related to timing, concurrency, and connection management.

### 2.1 SSH Session Race Condition ⭐

**Error Signature**:
```
The file `filename` doesn't exist
No such file or directory
Connection closed
```

**Common Causes**:
- Multiple SSH nodes using separate sessions
- File written in one session, read in another
- Filesystem sync delay
- Session not maintained between nodes

**Example Execution Data** (From User's Case):
```json
{
  "error": {
    "message": "The file `task-6fhfPW2FMpQ5Qmgp.json` doesn't exist",
    "node": "Execute Claude Skill",
    "stdout": "The file doesn't exist in the directory. The most recent task files are: task-6fhP4frM9X8GwVJp.json..."
  },
  "execution_path": [
    "Write Task to VM",  // ✅ Succeeded - wrote file
    "Execute Claude Skill"  // ❌ Failed - can't find file
  ]
}
```

**Diagnostic Steps**:
1. Check if there are multiple SSH nodes in sequence
2. Verify each SSH node creates new connection
3. Check file write succeeded in previous node
4. Review timing between nodes

**Recommended Fixes**:

**Option 1: Combine Operations in Single SSH Node** (BEST)
```bash
# Combine write + execute in one SSH command
cat > /home/azureuser/todoist-tasks/task-{{ $('Clean Input').item.json.task_id }}.json << 'JSON_EOF'
{{ JSON.stringify($('Clean Input').item.json, null, 2) }}
JSON_EOF

cd /home/azureuser/todoist-tasks && \
timeout 120 /home/azureuser/.local/bin/claude \
  --dangerously-skip-permissions \
  -p "Follow the instructions in /home/azureuser/todoist-tasks/task-{{ $('Clean Input').item.json.task_id }}.json"
```

**Option 2: Add Explicit Wait**
```bash
# Write file
cat > file.json << 'EOF'
{{ $json }}
EOF

# Wait for filesystem sync
sleep 2

# Then use file
command-using-file
```

**Option 3: Use Same SSH Connection**
Configure SSH node to reuse connection (if supported by SSH credentials).

**Why This Happens**:
- Each SSH node creates a separate connection/session
- File written in Session A may not immediately appear in Session B
- Network filesystems (NFS, etc.) can have sync delays
- Different working directories between sessions

**Cross-Reference**: See n8n-workflow-patterns → SSH Execution Patterns

---

### 2.2 Database Connection Pool Exhaustion

**Error Signature**:
```
Too many connections
Connection pool exhausted
ETIMEDOUT connecting to database
```

**Common Causes**:
- Too many parallel database operations
- Connections not being released
- Connection pool size too small
- Long-running queries blocking pool

**Example Execution Data**:
```json
{
  "error": {
    "message": "Connection pool exhausted. Max connections: 10",
    "node": "Query Database",
    "code": "ER_CON_COUNT_ERROR"
  }
}
```

**Diagnostic Steps**:
1. Check how many DB nodes run in parallel
2. Review connection pool configuration
3. Check for long-running queries
4. Verify connections are being closed

**Recommended Fixes**:

**Option 1: Increase Pool Size**
```javascript
// In database credentials
{
  "host": "db.example.com",
  "connectionLimit": 20  // Increase from default 10
}
```

**Option 2: Serialize Database Operations**
Use SplitInBatches node to limit concurrency:
```
SplitInBatches (batch size: 5)
  ↓
Database Query
  ↓
Loop back until done
```

**Option 3: Add Connection Timeout**
```javascript
{
  "connectionTimeout": 5000,  // 5 seconds
  "acquireTimeout": 5000
}
```

**Cross-Reference**: See n8n-node-configuration → Database Nodes

---

### 2.3 File Handle Limits

**Error Signature**:
```
EMFILE: too many open files
ENFILE: file table overflow
```

**Common Causes**:
- Reading/writing many files in parallel
- Files not being closed
- System file descriptor limit reached
- Batch processing too large

**Example Execution Data**:
```json
{
  "error": {
    "message": "EMFILE: too many open files",
    "node": "Read Binary Files",
    "code": "EMFILE"
  }
}
```

**Diagnostic Steps**:
1. Check number of files being processed
2. Review batch size settings
3. Verify file operations complete properly
4. Check system ulimit settings

**Recommended Fixes**:

**Option 1: Process Files in Batches**
```
SplitInBatches (batch size: 10)
  ↓
Read Binary File
  ↓
Process
  ↓
Loop back
```

**Option 2: Add Delays Between Operations**
```javascript
// In Code node between file operations
await new Promise(resolve => setTimeout(resolve, 100));  // 100ms delay
```

**Option 3: Increase System Limits** (Server-side)
```bash
# Check current limit
ulimit -n

# Increase limit
ulimit -n 4096
```

---

## 3. Authentication & Credentials

Errors related to authentication, authorization, and credential management.

### 3.1 Missing Credentials

**Error Signature**:
```
No credentials found
Credentials not set
Authentication required
```

**Common Causes**:
- Credentials not configured for node
- Wrong credential type selected
- Credential name changed or deleted
- Node copied without credentials

**Example Execution Data**:
```json
{
  "error": {
    "message": "No credentials found for this node",
    "node": "Slack",
    "details": "Please configure credentials"
  }
}
```

**Diagnostic Steps**:
1. Check if node has credentials set
2. Verify credential type matches node requirements
3. Check credential still exists in n8n
4. Review credential permissions

**Recommended Fixes**:

**Option 1: Configure Credentials**
- Open node configuration
- Click "Credentials" dropdown
- Select or create appropriate credential
- Test connection

**Option 2: Check Credential Scope**
- Verify credential is available in current environment
- Check if credential is project-scoped
- Ensure user has access to credential

**Note**: This issue requires n8n UI interaction - cannot be fixed with code.

**Cross-Reference**: See n8n-node-configuration → Credential Setup

---

### 3.2 Expired Tokens

**Error Signature**:
```
HTTP 401 Unauthorized
Token expired
Invalid authentication credentials
```

**Common Causes**:
- OAuth token expired
- API key rotated
- Session timeout
- Token refresh failed

**Example Execution Data**:
```json
{
  "error": {
    "httpCode": 401,
    "message": "Unauthorized: Token expired",
    "node": "Google Sheets"
  },
  "note": "Workflow worked yesterday, fails today"
}
```

**Diagnostic Steps**:
1. Check when workflow last succeeded
2. Review token expiration settings
3. Test credential connection manually
4. Check if API keys were rotated

**Recommended Fixes**:

**Option 1: Refresh OAuth Token**
- Go to Credentials in n8n UI
- Find the credential
- Click "Refresh Token" or reconnect
- Test connection

**Option 2: Update API Key**
- Get new API key from service
- Update credential in n8n
- Test connection

**Option 3: Enable Auto-Refresh**
- Use OAuth2 credentials (auto-refresh supported)
- Ensure refresh token is valid
- Check token expiration settings

**Note**: Requires n8n UI interaction - cannot be automated.

---

### 3.3 Insufficient Permissions

**Error Signature**:
```
HTTP 403 Forbidden
Access denied
Insufficient permissions
```

**Common Causes**:
- API key lacks required scopes
- User doesn't have permission
- Resource is restricted
- Rate limit or quota exceeded

**Example Execution Data**:
```json
{
  "error": {
    "httpCode": 403,
    "message": "Forbidden: Insufficient permissions to access resource",
    "node": "Google Drive"
  }
}
```

**Diagnostic Steps**:
1. Check API key/OAuth scopes
2. Verify user permissions in service
3. Review resource access controls
4. Check quota/rate limits

**Recommended Fixes**:

**Option 1: Increase OAuth Scopes**
- Reconnect OAuth credential
- Grant additional permissions
- Test with required scopes

**Option 2: Use Different Credential**
- Use account with higher permissions
- Create service account with needed access
- Update workflow to use new credential

**Option 3: Change Resource Access**
- Make resource publicly accessible (if appropriate)
- Grant access to service account
- Use different API endpoint with lower restrictions

---

## 4. API Integration Errors

Errors related to HTTP requests and external API integrations.

### 4.1 Rate Limiting (HTTP 429)

**Error Signature**:
```
HTTP 429 Too Many Requests
Rate limit exceeded
Retry after X seconds
```

**Common Causes**:
- Too many requests in time window
- No rate limiting implemented
- Parallel requests exceed limit
- Free tier limits reached

**Example Execution Data**:
```json
{
  "error": {
    "httpCode": 429,
    "message": "Rate limit exceeded. Retry after 60 seconds",
    "node": "HTTP Request",
    "headers": {
      "X-RateLimit-Remaining": "0",
      "Retry-After": "60"
    }
  }
}
```

**Diagnostic Steps**:
1. Check API rate limit documentation
2. Count requests per minute in workflow
3. Review if requests are parallelized
4. Check API tier/quota

**Recommended Fixes**:

**Option 1: Add Retry Logic**
```javascript
// In HTTP Request node settings
{
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000  // 5 seconds
}
```

**Option 2: Serialize Requests**
Use SplitInBatches to limit concurrency:
```
SplitInBatches (batch size: 5)
  ↓
Wait (1 second)
  ↓
HTTP Request
  ↓
Loop back
```

**Option 3: Implement Exponential Backoff**
Add Code node before HTTP request:
```javascript
const retryCount = $input.item.json.retryCount || 0;
const delay = Math.pow(2, retryCount) * 1000;  // 1s, 2s, 4s, 8s...

await new Promise(resolve => setTimeout(resolve, delay));

return {
  ...$ input.item.json,
  retryCount: retryCount + 1
};
```

**Cross-Reference**: See n8n-workflow-patterns → API Integration Patterns

---

### 4.2 Timeout Errors (504, ETIMEDOUT)

**Error Signature**:
```
ETIMEDOUT
ESOCKETTIMEDOUT
HTTP 504 Gateway Timeout
Connection timeout
```

**Common Causes**:
- API endpoint is slow
- Timeout setting too low
- Network issues
- Large data transfer
- Server processing delay

**Example Execution Data**:
```json
{
  "error": {
    "code": "ETIMEDOUT",
    "message": "Connection timed out after 10000ms",
    "node": "HTTP Request"
  }
}
```

**Diagnostic Steps**:
1. Check timeout setting in HTTP node
2. Test API endpoint response time
3. Review data size being transferred
4. Check network connectivity

**Recommended Fixes**:

**Option 1: Increase Timeout**
```javascript
// In HTTP Request node
{
  "timeout": 30000  // Increase to 30 seconds (from default 10s)
}
```

**Option 2: Add Retry Logic**
```javascript
{
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000
}
```

**Option 3: Optimize Request**
- Reduce data size with pagination
- Use query filters to limit response
- Request only needed fields
- Use POST instead of GET for large data

**Option 4: Use Webhook Response** (For very slow APIs)
```
1. Make async API request
2. API calls webhook when done
3. Workflow continues from webhook
```

---

### 4.3 Invalid Request Format (400, 422)

**Error Signature**:
```
HTTP 400 Bad Request
HTTP 422 Unprocessable Entity
Invalid request body
Validation failed
```

**Common Causes**:
- Wrong Content-Type header
- Malformed JSON
- Missing required fields
- Invalid field values
- Schema validation failure

**Example Execution Data**:
```json
{
  "error": {
    "httpCode": 422,
    "message": "Validation failed: 'email' is required",
    "node": "Create User",
    "body": {
      "name": "John Doe"
      // ❌ Missing 'email' field
    }
  }
}
```

**Diagnostic Steps**:
1. Review API documentation for required fields
2. Check request body structure
3. Verify Content-Type header
4. Test with API's example payload

**Recommended Fixes**:

**Option 1: Fix Request Body**
```javascript
// Before (missing required field)
{
  "name": "{{ $json.name }}"
}

// After (include required field)
{
  "name": "{{ $json.name }}",
  "email": "{{ $json.email }}"
}
```

**Option 2: Set Correct Content-Type**
```javascript
// In HTTP Request node headers
{
  "Content-Type": "application/json"  // or application/x-www-form-urlencoded
}
```

**Option 3: Validate Data Before Request**
Add IF node before HTTP Request:
```javascript
// Condition
{{ $json.email && $json.name }}

// Only proceed if both fields exist
```

**Cross-Reference**: See n8n-validation-expert → Request Validation

---

## 5. Data Processing Errors

Errors related to data transformation and processing.

### 5.1 Type Mismatches

**Error Signature**:
```
Expected number but got string
Type mismatch: string vs number
Cannot perform operation on incompatible types
```

**Common Causes**:
- API returns strings for numbers
- CSV/Excel data imported as text
- Type not converted after parsing
- Mixed types in same field

**Example Execution Data**:
```json
{
  "error": {
    "message": "Expected number but got string",
    "node": "Set",
    "field": "age",
    "value": "25"  // ❌ String instead of number
  }
}
```

**Diagnostic Steps**:
1. Check upstream data types
2. Verify field values and types
3. Review data source format
4. Test type conversion

**Recommended Fixes**:

**Option 1: Explicit Type Conversion**
```javascript
// In Set node or expression
{
  "age": {{ parseInt($json.age) }},
  "price": {{ parseFloat($json.price) }},
  "active": {{ $json.active === 'true' }}  // String to boolean
}
```

**Option 2: Use Code Node for Complex Conversion**
```javascript
// Convert all numeric fields
const item = $input.item.json;

return {
  ...item,
  age: parseInt(item.age),
  price: parseFloat(item.price),
  quantity: parseInt(item.quantity)
};
```

**Option 3: Configure Data Source**
If possible, configure the data source to return correct types.

---

### 5.2 Null/Undefined Handling

**Error Signature**:
```
Cannot read property of null
undefined is not an object
null reference error
```

**Common Causes**:
- Optional fields not present
- API returns null for missing data
- No default value handling
- Conditional logic missing

**Example Execution Data**:
```json
{
  "error": {
    "message": "Cannot read property 'length' of null",
    "node": "Process Items",
    "data": {
      "items": null  // ❌ Expected array, got null
    }
  }
}
```

**Recommended Fixes**:

**Option 1: Provide Defaults**
```javascript
// Before
{{ $json.items.length }}

// After (with default)
{{ ($json.items || []).length }}
```

**Option 2: Conditional Checks**
```javascript
{{ $json.items ? $json.items.length : 0 }}
```

**Option 3: Nullish Coalescing**
```javascript
{{ $json.value ?? 'default-value' }}
```

---

### 5.3 JSON Parsing Failures

**Error Signature**:
```
Unexpected token in JSON
JSON.parse error
Invalid JSON format
```

**Common Causes**:
- Response is not valid JSON
- HTML error page returned instead of JSON
- Malformed JSON string
- Single quotes instead of double quotes

**Example Execution Data**:
```json
{
  "error": {
    "message": "Unexpected token < in JSON at position 0",
    "node": "Parse JSON",
    "input": "<!DOCTYPE html><html>..."  // ❌ HTML instead of JSON
  }
}
```

**Diagnostic Steps**:
1. Check actual API response
2. Verify Content-Type header
3. Check for error responses
4. Test JSON validity

**Recommended Fixes**:

**Option 1: Check Response Before Parsing**
```javascript
// In Code node
try {
  const data = JSON.parse($input.item.json.response);
  return { data };
} catch (e) {
  return { error: 'Invalid JSON', raw: $input.item.json.response };
}
```

**Option 2: Validate Content-Type**
```javascript
// Check response headers
if ($json.headers['content-type'].includes('application/json')) {
  // Parse JSON
} else {
  // Handle non-JSON response
}
```

---

## 6. Workflow Structure Errors

Errors related to workflow configuration and topology.

### 6.1 Broken Connections

**Error Signature**:
```
Connection references non-existent node
Invalid node reference in connection
Disconnected node in workflow
```

**Common Causes**:
- Node deleted but connections remain
- Node renamed breaking references
- Copy/paste creating stale references
- Manual workflow JSON editing

**Example Execution Data**:
```json
{
  "validation": {
    "errors": [
      {
        "type": "invalid_reference",
        "message": "Connection references deleted node 'Old Node Name'",
        "connection": {
          "source": "Start",
          "target": "Old Node Name"
        }
      }
    ]
  }
}
```

**Diagnostic Steps**:
1. Use `n8n_validate_workflow` with `validateConnections: true`
2. Check for orphaned connections
3. Review recent node deletions/renames
4. Inspect workflow JSON if needed

**Recommended Fixes**:

**Option 1: Clean Stale Connections**
```javascript
// Use n8n_update_partial_workflow
{
  "operations": [
    {
      "type": "cleanStaleConnections"
    }
  ]
}
```

**Option 2: Manually Fix Connections**
- Open workflow in n8n UI
- Delete broken connections
- Reconnect nodes properly
- Save workflow

**Option 3: Rebuild Connection**
```javascript
// Use n8n_update_partial_workflow
{
  "operations": [
    {
      "type": "removeConnection",
      "source": "Start",
      "target": "Old Node Name"
    },
    {
      "type": "addConnection",
      "source": "Start",
      "target": "New Node Name"
    }
  ]
}
```

**Cross-Reference**: See n8n-validation-expert → Workflow Structure Validation

---

### 6.2 Circular Dependencies

**Error Signature**:
```
Circular reference detected
Workflow contains loop
Infinite execution path
```

**Common Causes**:
- Node A depends on Node B, Node B depends on Node A
- Accidental loop in workflow logic
- Recursive references
- Improper use of loop nodes

**Example Execution Data**:
```json
{
  "validation": {
    "errors": [
      {
        "type": "circular_dependency",
        "message": "Circular reference detected",
        "nodes": ["Node A", "Node B", "Node A"]
      }
    ]
  }
}
```

**Diagnostic Steps**:
1. Use `n8n_validate_workflow`
2. Trace workflow execution path
3. Identify loop in connections
4. Review intended workflow logic

**Recommended Fixes**:

**Option 1: Remove Circular Connection**
- Identify the connection creating the loop
- Remove or redirect connection
- Use proper loop nodes if iteration needed

**Option 2: Use Loop Nodes Properly**
```
Split In Batches
  ↓
Process Item
  ↓
Loop (back to Split In Batches) ✅ Proper loop
```

**Not This** (circular):
```
Process A
  ↓
Process B
  ↓
Process A ❌ Infinite loop
```

---

### 6.3 Missing Trigger Nodes

**Error Signature**:
```
Workflow has no trigger
Cannot activate workflow without trigger
No starting node found
```

**Common Causes**:
- Trigger node deleted
- Workflow created without trigger
- Trigger node not connected
- Using Start node instead of trigger

**Diagnostic Steps**:
1. Check if workflow has trigger node
2. Verify trigger is connected to workflow
3. Check workflow activation status
4. Review trigger configuration

**Recommended Fixes**:

**Option 1: Add Trigger Node**
- Add Webhook, Schedule, or Manual trigger
- Configure trigger settings
- Connect to first workflow node

**Option 2: Use Manual Trigger for Testing**
- Add "Manual Trigger" node
- Test workflow manually
- Replace with real trigger when ready

**Note**: Workflows can run manually without triggers, but cannot be activated.

---

## Error Catalog Summary

### By Category:
- **Expression Errors**: 4 patterns
- **Connection/Race Conditions**: 3 patterns
- **Authentication/Credentials**: 3 patterns
- **API Integration Errors**: 3 patterns
- **Data Processing Errors**: 3 patterns
- **Workflow Structure Errors**: 3 patterns

### Most Common Errors (Top 5):
1. SSH Race Condition (file doesn't exist)
2. Missing Property Access (undefined)
3. Rate Limiting (HTTP 429)
4. Expired Tokens (HTTP 401)
5. Type Mismatches (string vs number)

### By Severity:
- **Critical** (blocks execution): Broken connections, missing triggers, circular dependencies
- **High** (causes failures): Authentication, API errors, race conditions
- **Medium** (intermittent): Type mismatches, null handling, parsing failures
- **Low** (edge cases): File handle limits, connection pool exhaustion

---

## Credits

Conceived by Romuald Członkowski - https://www.aiadvisors.pl/en

Part of the n8n-mcp-skills project.
