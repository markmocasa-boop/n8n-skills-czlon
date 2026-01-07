# Diagnostic Patterns for n8n Error Analysis

Pattern recognition guide for automated n8n workflow error diagnosis. Each pattern includes detection signatures, evidence markers, root cause analysis, and fix strategies.

---

## Table of Contents

1. [SSH Race Condition Pattern](#1-ssh-race-condition-pattern)
2. [Expression Reference Pattern](#2-expression-reference-pattern)
3. [Rate Limiting Pattern](#3-rate-limiting-pattern)
4. [Credential Expiry Pattern](#4-credential-expiry-pattern)
5. [Timeout Pattern](#5-timeout-pattern)
6. [Data Type Mismatch Pattern](#6-data-type-mismatch-pattern)

---

## Pattern Recognition Framework

### Detection Logic Flow

```
1. Collect error signature
   ↓
2. Match against pattern signatures
   ↓
3. Gather supporting evidence
   ↓
4. Confirm pattern match (confidence score)
   ↓
5. Generate diagnosis with recommended fix
```

### Confidence Levels

- **High (90-100%)**: Strong signature match + multiple evidence points
- **Medium (70-89%)**: Signature match + some evidence
- **Low (50-69%)**: Partial signature match or weak evidence
- **Uncertain (<50%)**: No clear pattern, requires manual analysis

---

## 1. SSH Race Condition Pattern

### Signature

**Primary Indicators**:
- Error message contains "doesn't exist" or "No such file or directory"
- File or resource was created in previous node
- Intermittent failures (works sometimes, fails other times)
- Multiple SSH nodes in workflow

**Error Patterns**:
```
The file `filename` doesn't exist
No such file or directory: filename
Connection closed
Session terminated
```

### Evidence Markers

**Strong Evidence** (each +30% confidence):
1. ✅ Previous SSH node succeeded in writing/creating resource
2. ✅ Execution path shows sequential SSH nodes
3. ✅ File/resource name is correct (not a typo)
4. ✅ Intermittent pattern (some executions succeed)

**Supporting Evidence** (each +10% confidence):
1. Different SSH nodes don't specify connection reuse
2. Filesystem operations involve network mount (NFS, etc.)
3. Timing varies between executions
4. No explicit wait/sleep between operations

### Detection Logic

```javascript
// Pattern match criteria
function detectSSHRaceCondition(execution) {
  let confidence = 0;

  // Check error message
  if (execution.error.message.match(/doesn't exist|no such file/i)) {
    confidence += 40;
  }

  // Check execution path for multiple SSH nodes
  const sshNodes = execution.executionPath.filter(n => n.type === 'n8n-nodes-base.ssh');
  if (sshNodes.length >= 2) {
    confidence += 30;
  }

  // Check if previous node succeeded
  const errorNodeIndex = execution.executionPath.findIndex(n => n.name === execution.error.node);
  const previousNode = execution.executionPath[errorNodeIndex - 1];
  if (previousNode && previousNode.status === 'success') {
    confidence += 20;
  }

  // Check for file write operations
  if (previousNode && previousNode.command.match(/cat >|echo >|tee|write/i)) {
    confidence += 10;
  }

  return {
    pattern: 'ssh_race_condition',
    confidence,
    match: confidence >= 70
  };
}
```

### Root Cause Explanation

**Why This Happens**:
1. Each n8n SSH node creates a separate SSH connection/session
2. File written in Session A may not immediately appear in Session B
3. Filesystem sync delays (especially on network filesystems)
4. Different working directories between sessions
5. Connection pooling or session management issues

**Originating Node**:
The workflow architecture (not a specific node) - the issue is splitting operations across multiple SSH connections.

### Recommended Fix Strategy

**Priority 1: Combine Operations** (BEST)
```bash
# Single SSH node with both operations
cat > /path/to/file.json << 'JSON_EOF'
{{ JSON.stringify($json, null, 2) }}
JSON_EOF

# Use file immediately in same session
cd /path && command-using-file
```

**Priority 2: Add Explicit Wait**
```bash
# In second SSH node
sleep 2  # Wait for filesystem sync
command-using-file
```

**Priority 3: Verify File Before Use**
```bash
# Check file exists before using
if [ -f /path/to/file.json ]; then
  command-using-file
else
  echo "File not found, waiting..."
  sleep 2
  command-using-file
fi
```

### Example Diagnosis Output

```markdown
## 🔍 Pattern Detected: SSH Race Condition (95% confidence)

### Root Cause
Multiple SSH nodes creating separate sessions. File written in first session
not immediately visible in second session.

### Evidence
- ✅ Previous "Write Task to VM" node succeeded
- ✅ Sequential SSH nodes detected (2 nodes)
- ✅ Error: "file doesn't exist" but file was just created
- ✅ File name matches between write and read operations

### Recommended Fix
Combine both operations in a single SSH node to ensure same session:

```bash
cat > /home/azureuser/task.json << 'EOF'
{{ JSON.stringify($json) }}
EOF

cd /home/azureuser && ./process-task.sh
```

**Why This Works**: Single SSH session ensures file is immediately available.
```

---

## 2. Expression Reference Pattern

### Signature

**Primary Indicators**:
- Error message: "Cannot read property 'X' of undefined"
- Error message: "undefined is not an object"
- Expression syntax: {{ }} involved
- Node references upstream data

**Error Patterns**:
```
Cannot read property 'field' of undefined
Cannot read properties of undefined (reading 'X')
$json.field is undefined
$node["NodeName"] is undefined
```

### Evidence Markers

**Strong Evidence** (each +25% confidence):
1. ✅ Error occurs in expression evaluation
2. ✅ Upstream node data sampled and property missing in some items
3. ✅ Inconsistent data structure from API/webhook
4. ✅ Property name is correct (no typo)

**Supporting Evidence** (each +15% confidence):
1. Webhook or API node upstream
2. Optional fields in data source
3. Works with some data items, fails with others
4. Recent API changes

### Detection Logic

```javascript
function detectExpressionReference(execution) {
  let confidence = 0;

  // Check error message for property access
  if (execution.error.message.match(/cannot read property|undefined is not an object/i)) {
    confidence += 40;
  }

  // Check for expression in node config
  if (execution.error.expression && execution.error.expression.includes('{{')) {
    confidence += 20;
  }

  // Check upstream data samples
  if (execution.upstreamData && execution.upstreamData.errorItemsLimit > 0) {
    const property = extractPropertyName(execution.error.message);
    const hasMissingProperty = execution.upstreamData.items.some(item =>
      !hasNestedProperty(item, property)
    );
    if (hasMissingProperty) {
      confidence += 30;
    }
  }

  // Check for webhook/API upstream
  const upstreamTypes = execution.executionPath
    .slice(0, -1)
    .map(n => n.type);
  if (upstreamTypes.includes('n8n-nodes-base.webhook') ||
      upstreamTypes.includes('n8n-nodes-base.httpRequest')) {
    confidence += 10;
  }

  return {
    pattern: 'expression_reference',
    confidence,
    match: confidence >= 70,
    missingProperty: extractPropertyName(execution.error.message)
  };
}
```

### Root Cause Explanation

**Why This Happens**:
1. Data structure varies between API calls
2. Optional fields not always present
3. Webhook payloads have inconsistent structure
4. No safe navigation or fallback handling
5. Assumption that property always exists

**Originating Node**:
Usually the upstream data source (webhook, API, database) that returns varying structures.

### Recommended Fix Strategy

**Priority 1: Safe Navigation Operator**
```javascript
// Before
{{ $json.body.user.email }}

// After
{{ $json.body?.user?.email }}
```

**Priority 2: Nullish Coalescing with Default**
```javascript
{{ $json.body?.user?.email ?? 'no-email@example.com' }}
```

**Priority 3: Conditional Check**
```javascript
{{ $json.body && $json.body.user && $json.body.user.email
   ? $json.body.user.email
   : 'N/A' }}
```

**Priority 4: Data Normalization**
Add Code node before failing node to normalize structure:
```javascript
return {
  email: $input.item.json.body?.user?.email || 'unknown@example.com',
  name: $input.item.json.body?.user?.name || 'Unknown'
};
```

### Example Diagnosis Output

```markdown
## 🔍 Pattern Detected: Expression Reference Error (88% confidence)

### Root Cause
Missing property 'email' in upstream data. Expression assumes property always
exists but API returns varying structures.

### Evidence
- ✅ Error: "Cannot read property 'email' of undefined"
- ✅ Upstream sample shows some items missing 'email' property
- ✅ Expression: {{ $json.body.user.email }}
- ✅ Webhook upstream with varying payload structure

### Upstream Data Sample
```json
// Item 1 (works)
{"body": {"user": {"email": "user@example.com"}}}

// Item 2 (fails)
{"body": {"user": {"name": "John"}}}  // ❌ No 'email'
```

### Recommended Fix
Use safe navigation operator with fallback:

```javascript
{{ $json.body?.user?.email ?? 'no-email-provided@placeholder.com' }}
```

**Why This Works**: Returns undefined safely if property missing, then uses fallback value.
```

---

## 3. Rate Limiting Pattern

### Signature

**Primary Indicators**:
- HTTP status code 429
- Error message contains "rate limit" or "too many requests"
- API/HTTP node failing
- Multiple requests in short time

**Error Patterns**:
```
HTTP 429 Too Many Requests
Rate limit exceeded
Too many requests
Retry after X seconds
```

### Evidence Markers

**Strong Evidence** (each +25% confidence):
1. ✅ HTTP 429 status code
2. ✅ "Retry-After" header present
3. ✅ Multiple API calls in execution (loop or parallel)
4. ✅ Previous executions succeeded, recent ones failing

**Supporting Evidence** (each +10% confidence):
1. Free tier API or low rate limits
2. No retry logic configured
3. Parallel execution branches
4. High execution frequency

### Detection Logic

```javascript
function detectRateLimiting(execution) {
  let confidence = 0;

  // Check for HTTP 429
  if (execution.error.httpCode === 429) {
    confidence += 50;  // Strong indicator
  }

  // Check error message
  if (execution.error.message.match(/rate limit|too many requests/i)) {
    confidence += 30;
  }

  // Check for Retry-After header
  if (execution.error.headers && execution.error.headers['retry-after']) {
    confidence += 10;
  }

  // Check for multiple API calls
  const apiNodes = execution.executionPath.filter(n =>
    n.type === 'n8n-nodes-base.httpRequest'
  );
  if (apiNodes.length > 10) {
    confidence += 10;
  }

  return {
    pattern: 'rate_limiting',
    confidence,
    match: confidence >= 70,
    retryAfter: execution.error.headers?.['retry-after']
  };
}
```

### Root Cause Explanation

**Why This Happens**:
1. API enforces request rate limits (e.g., 100 requests/minute)
2. Workflow exceeds limit with parallel or rapid requests
3. No backoff/retry logic implemented
4. Free tier with stricter limits
5. Shared API key across multiple workflows

**Originating Node**:
The workflow architecture or loop configuration that creates too many requests.

### Recommended Fix Strategy

**Priority 1: Add Retry Logic**
```javascript
// In HTTP Request node settings
{
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000  // 5 seconds
}
```

**Priority 2: Serialize Requests with Delays**
```
SplitInBatches (batch size: 10)
  ↓
Wait (1000ms)
  ↓
HTTP Request
  ↓
Loop back
```

**Priority 3: Exponential Backoff**
```javascript
// Code node before HTTP Request
const attempt = $input.item.json.attempt || 0;
const delay = Math.min(Math.pow(2, attempt) * 1000, 30000);  // Max 30s

await new Promise(resolve => setTimeout(resolve, delay));

return {
  ...$input.item.json,
  attempt: attempt + 1
};
```

**Priority 4: Respect Retry-After Header**
```javascript
// Parse Retry-After from previous error
const retryAfter = parseInt($input.item.json.retryAfter || '60');
await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
```

### Example Diagnosis Output

```markdown
## 🔍 Pattern Detected: API Rate Limiting (100% confidence)

### Root Cause
Exceeding API rate limit. Workflow making too many requests in short timeframe.

### Evidence
- ✅ HTTP 429 status code
- ✅ Error: "Rate limit exceeded"
- ✅ Retry-After header: 60 seconds
- ✅ 25 API calls in last minute
- ✅ No retry logic configured

### Recommended Fix
Add retry logic with exponential backoff:

```javascript
// In HTTP Request node settings
{
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000
}
```

Additionally, serialize requests to avoid parallel calls:

```
SplitInBatches (batch: 5)
  ↓
Wait (1 second)
  ↓
HTTP Request
  ↓
Loop
```

**Why This Works**: Spreads requests over time and retries with delays.
```

---

## 4. Credential Expiry Pattern

### Signature

**Primary Indicators**:
- HTTP status 401 (Unauthorized) or 403 (Forbidden)
- Workflow previously worked successfully
- OAuth or token-based authentication
- Error message about invalid/expired credentials

**Error Patterns**:
```
HTTP 401 Unauthorized
HTTP 403 Forbidden
Token expired
Invalid authentication credentials
Unauthorized access
```

### Evidence Markers

**Strong Evidence** (each +25% confidence):
1. ✅ HTTP 401 or 403 status code
2. ✅ Previous successful executions exist
3. ✅ OAuth2 credentials used
4. ✅ Last success > 24 hours ago

**Supporting Evidence** (each +15% confidence):
1. API key or token-based auth
2. Error mentions "expired" or "invalid"
3. Multiple nodes with same credentials failing
4. Recent credential rotation by service

### Detection Logic

```javascript
function detectCredentialExpiry(execution) {
  let confidence = 0;

  // Check HTTP status
  if (execution.error.httpCode === 401 || execution.error.httpCode === 403) {
    confidence += 40;
  }

  // Check error message
  if (execution.error.message.match(/expired|invalid.*credential|unauthorized/i)) {
    confidence += 20;
  }

  // Check for previous successes
  const recentExecutions = execution.recentExecutionHistory || [];
  const hadRecentSuccess = recentExecutions.some(e =>
    e.status === 'success' &&
    e.finishedAt > Date.now() - (7 * 24 * 60 * 60 * 1000)  // Last 7 days
  );
  if (hadRecentSuccess) {
    confidence += 30;
  }

  // Check credential type
  if (execution.node.credentials?.type === 'oAuth2Api') {
    confidence += 10;
  }

  return {
    pattern: 'credential_expiry',
    confidence,
    match: confidence >= 70
  };
}
```

### Root Cause Explanation

**Why This Happens**:
1. OAuth tokens expire after set time period
2. API keys manually rotated by service or admin
3. Session timeouts
4. Refresh token failed to renew access token
5. Credentials revoked by user/admin

**Originating Node**:
The credential configuration itself, not the workflow logic.

### Recommended Fix Strategy

**Priority 1: Refresh OAuth Token**
- Go to n8n Credentials UI
- Find the credential
- Click "Refresh" or "Reconnect"
- Test connection

**Priority 2: Update API Key**
- Get new API key from service
- Update in n8n Credentials
- Test connection

**Priority 3: Enable Auto-Refresh** (if not enabled)
- Use OAuth2 credentials (supports auto-refresh)
- Verify refresh token is valid
- Check token expiration settings

**Note**: This pattern requires UI interaction - cannot be fixed with code alone.

### Example Diagnosis Output

```markdown
## 🔍 Pattern Detected: Credential Expiry (95% confidence)

### Root Cause
OAuth token expired. Workflow worked previously but credentials need refresh.

### Evidence
- ✅ HTTP 401 Unauthorized
- ✅ Last successful execution: 3 days ago
- ✅ OAuth2 credentials in use (Google Sheets)
- ✅ Error: "Invalid authentication credentials"

### Recommended Fix
**Refresh OAuth Token** (Requires n8n UI):

1. Go to Credentials in n8n
2. Find "Google Sheets OAuth2" credential
3. Click "Reconnect" or "Refresh Token"
4. Authorize again if prompted
5. Test connection

**Why This Works**: Generates new access token using refresh token.

**Alternative**: If reconnect fails, revoke and recreate credential with fresh OAuth flow.
```

---

## 5. Timeout Pattern

### Signature

**Primary Indicators**:
- Error codes: ETIMEDOUT, ESOCKETTIMEDOUT, ECONNABORTED
- HTTP 504 Gateway Timeout
- Error message contains "timeout" or "timed out"
- Long-running API calls or operations

**Error Patterns**:
```
ETIMEDOUT
ESOCKETTIMEDOUT
HTTP 504 Gateway Timeout
Connection timed out
Request timeout
```

### Evidence Markers

**Strong Evidence** (each +25% confidence):
1. ✅ Timeout error code/message
2. ✅ Operation duration near timeout limit
3. ✅ Large data transfer involved
4. ✅ External API known to be slow

**Supporting Evidence** (each +15% confidence):
1. Default timeout setting (10s)
2. No retry configuration
3. Network-intensive operation
4. Peak usage time

### Detection Logic

```javascript
function detectTimeout(execution) {
  let confidence = 0;

  // Check error code
  if (execution.error.code && execution.error.code.match(/ETIMEDOUT|ESOCKETTIMEDOUT|ECONNABORTED/)) {
    confidence += 50;
  }

  // Check HTTP status
  if (execution.error.httpCode === 504) {
    confidence += 40;
  }

  // Check error message
  if (execution.error.message.match(/timeout|timed out/i)) {
    confidence += 30;
  }

  // Check operation duration
  if (execution.duration && execution.timeout) {
    if (execution.duration >= execution.timeout * 0.95) {  // Within 5% of timeout
      confidence += 20;
    }
  }

  return {
    pattern: 'timeout',
    confidence,
    match: confidence >= 70,
    duration: execution.duration,
    timeout: execution.timeout
  };
}
```

### Root Cause Explanation

**Why This Happens**:
1. API endpoint is slow to respond
2. Timeout setting too low for operation
3. Network latency or congestion
4. Large data transfers take longer than expected
5. Server-side processing delay

**Originating Node**:
Often the external service being called, not the workflow itself.

### Recommended Fix Strategy

**Priority 1: Increase Timeout**
```javascript
// In HTTP Request node
{
  "timeout": 60000  // Increase to 60 seconds
}
```

**Priority 2: Add Retry Logic**
```javascript
{
  "timeout": 30000,
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000
}
```

**Priority 3: Optimize Request**
- Use pagination for large datasets
- Filter response fields
- Use query parameters to limit data
- Request smaller chunks

**Priority 4: Async Pattern** (for very slow APIs)
```
1. Make async request (get job ID)
2. Poll status endpoint
3. Retrieve results when ready
```

### Example Diagnosis Output

```markdown
## 🔍 Pattern Detected: Timeout Error (92% confidence)

### Root Cause
API call timing out. Current timeout (10s) insufficient for operation.

### Evidence
- ✅ Error: ETIMEDOUT
- ✅ Operation duration: 9.8s (near 10s timeout)
- ✅ Large data transfer (500MB response expected)
- ✅ No retry logic configured

### Recommended Fix
Increase timeout and add retry logic:

```javascript
// In HTTP Request node
{
  "timeout": 60000,  // 60 seconds
  "retryOnFail": true,
  "maxTries": 2
}
```

**Alternative**: For very large data, use pagination:

```
Request page 1 (limit: 100)
  ↓
Process
  ↓
Request page 2
  ↓
...
```

**Why This Works**: Allows more time for slow operations and retries on failure.
```

---

## 6. Data Type Mismatch Pattern

### Signature

**Primary Indicators**:
- Error message about type incompatibility
- "Expected X but got Y"
- Validation failures
- Math operations on strings

**Error Patterns**:
```
Expected number but got string
Type mismatch: string vs number
Cannot perform operation on incompatible types
Validation failed: invalid type
```

### Evidence Markers

**Strong Evidence** (each +25% confidence):
1. ✅ Type mismatch in error message
2. ✅ Upstream data shows wrong type
3. ✅ Math operation or number comparison involved
4. ✅ Data from CSV, form, or external source

**Supporting Evidence** (each +15% confidence):
1. No type conversion in workflow
2. API returns strings for numbers
3. Form submission data (always strings)
4. CSV import without type casting

### Detection Logic

```javascript
function detectDataTypeMismatch(execution) {
  let confidence = 0;

  // Check error message
  if (execution.error.message.match(/expected.*but got|type mismatch|invalid type/i)) {
    confidence += 40;
  }

  // Check for numeric operation
  if (execution.node.operation && execution.node.operation.match(/multiply|divide|add|subtract|math/i)) {
    confidence += 20;
  }

  // Check upstream data types
  if (execution.upstreamData) {
    const hasStringNumber = execution.upstreamData.items.some(item =>
      typeof item.value === 'string' && !isNaN(item.value)
    );
    if (hasStringNumber) {
      confidence += 30;
    }
  }

  // Check data source type
  const upstreamTypes = execution.executionPath.map(n => n.type);
  if (upstreamTypes.includes('n8n-nodes-base.formTrigger') ||
      upstreamTypes.includes('n8n-nodes-base.readFile')) {
    confidence += 10;
  }

  return {
    pattern: 'data_type_mismatch',
    confidence,
    match: confidence >= 70
  };
}
```

### Root Cause Explanation

**Why This Happens**:
1. External APIs return numbers as strings (common in REST APIs)
2. CSV/Excel imports treat all data as text
3. Form submissions always send strings
4. No type conversion between nodes
5. Implicit type coercion doesn't work as expected

**Originating Node**:
The data source that returns incorrect types (API, form, file import).

### Recommended Fix Strategy

**Priority 1: Explicit Type Conversion**
```javascript
// In Set node or expression
{
  "age": {{ parseInt($json.age) }},
  "price": {{ parseFloat($json.price) }},
  "quantity": {{ Number($json.quantity) }},
  "active": {{ $json.active === 'true' }}  // String to boolean
}
```

**Priority 2: Code Node for Batch Conversion**
```javascript
// Convert all numeric fields
const item = $input.item.json;

return {
  ...item,
  age: parseInt(item.age) || 0,
  price: parseFloat(item.price) || 0.0,
  quantity: parseInt(item.quantity) || 1,
  active: item.active === 'true' || item.active === true
};
```

**Priority 3: Validation + Conversion**
```javascript
// Validate before converting
const price = $json.price;
const validPrice = typeof price === 'string' && !isNaN(price)
  ? parseFloat(price)
  : typeof price === 'number'
    ? price
    : 0;  // Default

return { price: validPrice };
```

### Example Diagnosis Output

```markdown
## 🔍 Pattern Detected: Data Type Mismatch (85% confidence)

### Root Cause
API returns numeric values as strings. Math operation expects numbers.

### Evidence
- ✅ Error: "Expected number but got string"
- ✅ Upstream data: {"price": "19.99", "quantity": "5"}  (strings)
- ✅ Operation: multiplication (price * quantity)
- ✅ Data source: HTTP Request (REST API)

### Upstream Data Sample
```json
{
  "price": "19.99",     // ❌ String
  "quantity": "5"       // ❌ String
}
```

### Recommended Fix
Convert types before calculation:

```javascript
// In Set node before calculation
{
  "price": {{ parseFloat($json.price) }},
  "quantity": {{ parseInt($json.quantity) }},
  "total": {{ parseFloat($json.price) * parseInt($json.quantity) }}
}
```

**Why This Works**: Explicitly converts strings to numbers before math operations.
```

---

## Pattern Priority Matrix

When multiple patterns match, use this priority order:

1. **SSH Race Condition** (if confidence > 90%) - Highest priority, specific fix
2. **Credential Expiry** (if confidence > 90%) - Requires immediate action
3. **Rate Limiting** (if confidence > 85%) - Time-sensitive issue
4. **Timeout** (if confidence > 80%) - May indicate systematic problem
5. **Expression Reference** (if confidence > 75%) - Common and fixable
6. **Data Type Mismatch** (if confidence > 70%) - Lower priority, edge case

## Pattern Combination Scenarios

Sometimes multiple patterns apply:

### SSH Race + Expression Reference
```markdown
Primary: SSH Race Condition (95%)
Secondary: Expression Reference (70%)

**Diagnosis**: File doesn't exist (race condition) AND expression assumes
file content structure. Fix race condition first, then verify expression logic.
```

### Rate Limiting + Timeout
```markdown
Primary: Rate Limiting (100%)
Secondary: Timeout (65%)

**Diagnosis**: Rate limiting causes retries which lead to timeouts. Fix rate
limiting first - timeout will resolve automatically.
```

---

## Credits

Conceived by Romuald Człon kowski - https://www.aiadvisors.pl/en

Part of the n8n-mcp-skills project.
