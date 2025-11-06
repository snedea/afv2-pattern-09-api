# Integration Guide: AFv2 Pattern #9 - API Integration

This guide explains how to import, configure, and use the API Integration pattern in your Flowise instance.

## Prerequisites

- Flowise v2.0+ installed and running
- Anthropic API key (for Claude Sonnet 4.5)
- Basic understanding of Flowise AgentFlows

## Step 1: Import Workflow

### Import JSON File

1. Open your Flowise UI
2. Navigate to **Agent Flows** section
3. Click **"Import Chatflow"** button
4. Select `09-api-integration.json` from your local filesystem
5. Click **"Import"**

### Verify Import Success

You should see 10 nodes on the canvas:
- ✅ 1 Start node (green)
- ✅ 1 Sticky Note (yellow) with PURPOSE description
- ✅ 4 Agent nodes (teal): ParameterExtractor, Format, Retry, ErrorHandler
- ✅ 1 HTTP Request node (gray)
- ✅ 1 Condition node (orange)
- ✅ 2 Direct Reply nodes (green/red)

### Check Connections

Verify all edges are connected:
- Start → ParameterExtractor
- ParameterExtractor → HTTP Request
- HTTP Request → Condition
- Condition [0] → Format Agent
- Condition [1] → Retry Agent
- **Retry Agent → HTTP Request** (loop-back, animated)
- Condition [2] → Error Handler
- Format Agent → Success Reply
- Error Handler → Error Reply

**Important**: The loop-back edge (Retry → HTTP) should appear animated (moving dots).

## Step 2: Configure Credentials

### Anthropic API Key

This pattern uses Claude Sonnet 4.5 for all agents. You need to configure your Anthropic API key:

1. Go to **Settings** → **Credentials**
2. Click **"Add New Credential"**
3. Select **"Anthropic API"**
4. Enter your API key
5. **Important**: Name the credential exactly: `"Anthropic API Key"`
   - This name must match the credential reference in the workflow
   - Do NOT use "Anthropic" or "Claude API Key" - must be exact

### Verify Credential Configuration

1. Double-click any agent node (e.g., Agent.ParameterExtractor)
2. Check the **Model** dropdown
3. Verify **"Anthropic API Key"** appears in credential selector
4. Verify **"claude-sonnet-4-5-20250929"** is selected

## Step 3: Configure Tools (Optional)

The pattern uses 2 standard tools that should be auto-included. However, if you encounter tool errors:

### currentDateTime Tool

**Purpose**: Provides current date/time for temporal context

**Setup**:
1. Go to **Tools** → **Custom Tools**
2. Click **"Add Tool"**
3. Configure:
   - Name: `currentDateTime`
   - Description: "Returns current date and time"
   - Type: JavaScript Function
   - Code:
     ```javascript
     return {
       currentDateTime: new Date().toISOString(),
       timestamp: Date.now(),
       date: new Date().toDateString(),
       time: new Date().toTimeString()
     };
     ```
4. Save tool

### searXNG Tool

**Purpose**: Federated web search for real-time information

**Setup**:
1. Go to **Tools** → **Custom Tools**
2. Click **"Add Tool"**
3. Configure:
   - Name: `searxng-search`
   - Description: "Federated web/meta search"
   - Type: HTTP API
   - Base URL: `https://s.llam.ai`
   - Method: GET
   - Endpoint: `/search`
   - Parameters:
     - `q` (string, required): Search query
     - `format` (string, optional): Response format (default: "json")
     - `language` (string, optional): Language code (default: "en")
4. Save tool

## Step 4: Test the Workflow

### Test 1: Successful GET Request

**Input**:
```
Get user data from https://jsonplaceholder.typicode.com/users/1
```

**Expected Output**:
```
✅ API Request Successful

Name: Leanne Graham
Email: Sincere@april.biz
Username: Bret
Phone: 1-770-736-8031 x56442
...
```

**What Happens**:
1. ParameterExtractor parses your request
2. Extracts URL: `https://jsonplaceholder.typicode.com/users/1`
3. HTTP Request sends GET request
4. Receives 200 status
5. Condition routes to Format Agent (output 0)
6. Format Agent parses JSON response
7. Success Reply returns formatted data

### Test 2: POST Request

**Input**:
```
Create a new post on JSONPlaceholder with title 'Test Post' and body 'This is a test'
```

**Expected Output**:
```
✅ API Request Successful

Post created successfully!

ID: 101
Title: Test Post
Body: This is a test
```

**What Happens**:
1. ParameterExtractor extracts:
   - URL: `https://jsonplaceholder.typicode.com/posts`
   - Method: POST
   - Body: `{"title": "Test Post", "body": "This is a test"}`
2. HTTP Request sends POST request
3. Receives 201 status (treated as success)
4. Format Agent confirms creation

### Test 3: Retry Logic (Simulated)

**Note**: Testing retry logic requires an API that returns 5xx errors. JSONPlaceholder doesn't simulate server errors.

**Input** (conceptual):
```
Call https://httpstat.us/503
```

**Expected Behavior**:
1. HTTP Request receives 503 status
2. Condition routes to Retry Agent (output 1)
3. Retry Agent increments retry_count to 1
4. Loop back to HTTP Request
5. (Repeat up to 3 times)
6. After 3 failures: Route to Error Handler
7. Error Reply with message:
   ```
   ❌ API Request Failed

   The API server is currently unavailable (503 Service Unavailable).
   We attempted 3 retries but the service remains down.
   ```

### Test 4: Client Error (404)

**Input**:
```
Get data from https://jsonplaceholder.typicode.com/invalid-endpoint
```

**Expected Output**:
```
❌ API Request Failed

The requested endpoint was not found (404 Not Found).

Troubleshooting Steps:
- Verify the API URL is correct
- Check that the endpoint exists in the API documentation
- Ensure you have permission to access this resource
```

**What Happens**:
1. HTTP Request receives 404 status
2. Condition routes to Error Handler (output 2, ELSE)
3. Error Handler analyzes 4xx error
4. Provides troubleshooting suggestions
5. No retry attempts (4xx = client error, not server issue)

## Step 5: Customize for Your Use Case

### Change Max Retries

To adjust the maximum retry attempts:

1. Double-click **Agent.ParameterExtractor** node
2. Find **Update State** section
3. Locate the state update for `api.max_retries`
4. Change value from `"3"` to desired number (e.g., `"5"`)
5. Save node

### Modify Backoff Strategy

To change the exponential backoff calculation:

1. Double-click **Agent.Retry** node
2. Find **Update State** section
3. Locate the state update for `api.backoff_delay`
4. Current formula: `{{ Math.pow(2, $flow.state.api.retry_count + 1) }}`
   - Exponential: 2^1=2s, 2^2=4s, 2^3=8s
5. Alternative (linear): `{{ ($flow.state.api.retry_count + 1) * 3 }}`
   - Linear: 3s, 6s, 9s
6. Save node

### Add Authentication Headers

To include API authentication:

1. Double-click **Agent.ParameterExtractor** node
2. Edit the system prompt/persona to include:
   ```
   Always include authentication headers:
   - Authorization: Bearer YOUR_TOKEN
   - Or extract from user input if they provide an API key
   ```
3. Or hardcode in state update:
   ```json
   {
     "key": "api.headers",
     "value": "{\"Authorization\": \"Bearer YOUR_TOKEN\", \"Content-Type\": \"application/json\"}"
   }
   ```
4. Save node

### Customize Error Messages

To modify error response format:

1. Double-click **Agent.ErrorHandler** node
2. Edit the system prompt/persona
3. Example:
   ```
   Format error messages as:
   - Brief error summary
   - HTTP status code
   - Troubleshooting steps (bullet list)
   - Contact support link if applicable
   ```
4. Save node

## Step 6: Integrate with Your Application

### Embed in Chatbot

1. Go to **Agent Flows** → Select your imported flow
2. Click **"Deploy"** → **"Embed"**
3. Copy embed code
4. Paste into your website HTML

### Use via API

1. Get your Flowise API endpoint:
   ```
   POST https://your-flowise-instance.com/api/v1/prediction/{chatflow-id}
   ```
2. Send request:
   ```bash
   curl -X POST https://your-flowise-instance.com/api/v1/prediction/abc123 \
     -H "Content-Type: application/json" \
     -d '{"question": "Get user data from https://jsonplaceholder.typicode.com/users/1"}'
   ```
3. Receive formatted response

## Troubleshooting

### Issue: "Credential not found" error

**Cause**: Anthropic API Key credential name mismatch

**Solution**:
1. Go to Settings → Credentials
2. Find your Anthropic credential
3. Rename to exactly: `"Anthropic API Key"`
4. Refresh workflow

### Issue: "Tool not found" error

**Cause**: currentDateTime or searXNG tools not created

**Solution**:
1. Follow Step 3 to create tools
2. Verify tool names match:
   - `currentDateTime` (exact case)
   - `searxng-search` (lowercase, hyphenated)
3. Refresh workflow

### Issue: HTTP Request node doesn't work

**Cause**: Flowise may not have built-in HTTP Request node

**Solution**:
1. Replace HTTP Request node with Agent
2. Give Agent an HTTP tool capability
3. Update agent persona to make HTTP requests via tool
4. Store response in Flow State as before

### Issue: Infinite retry loop

**Cause**: Retry count not incrementing correctly

**Solution**:
1. Double-click Agent.Retry node
2. Verify state update for `api.retry_count`:
   ```json
   {
     "key": "api.retry_count",
     "value": "{{ $flow.state.api.retry_count + 1 }}"
   }
   ```
3. Ensure value includes `+ 1` to increment
4. Save node

### Issue: Condition not routing correctly

**Cause**: Variable format incorrect (plain text instead of rich text)

**Solution**:
1. Double-click Condition node
2. Verify variable format includes HTML span:
   ```html
   <p><span class="variable" data-type="mention" data-id="$flow.state.api.response_status" data-label="$flow.state.api.response_status">{{ $flow.state.api.response_status }}</span></p>
   ```
3. Do NOT use plain text: `{{ $flow.state.api.response_status }}`
4. Use Flowise UI variable picker to ensure correct format

## Best Practices

### 1. API Key Security
- Never hardcode API keys in workflow JSON
- Always use Flowise credentials system
- Rotate keys regularly
- Use environment variables for deployment

### 2. Rate Limiting
- Be aware of API rate limits
- Adjust max_retries based on API limits
- Increase backoff delay for rate-limited APIs
- Consider implementing exponential backoff with jitter

### 3. Error Handling
- Customize error messages for your use case
- Log errors for debugging
- Provide actionable troubleshooting steps
- Include support contact info in error messages

### 4. Testing
- Test with real APIs (not mocks)
- Test all 3 paths: SUCCESS, RETRY, FATAL
- Verify retry logic with actual 5xx responses
- Monitor retry counts and backoff delays

### 5. Monitoring
- Track API response times
- Monitor retry rates
- Alert on high error rates
- Log all API interactions for audit

## Advanced Usage

### Multiple API Endpoints

To support multiple API endpoints:
1. Add endpoint selection logic to ParameterExtractor
2. Use different header configurations per endpoint
3. Store endpoint-specific settings in Flow State

### Webhook Integration

To use this pattern for webhooks:
1. Replace Start node input with webhook trigger
2. Parse webhook payload in ParameterExtractor
3. Use HTTP Request to call external API
4. Return webhook response via Direct Reply

### Parallel API Calls

To call multiple APIs simultaneously:
1. Duplicate HTTP Request + Condition branches
2. Use parallel execution pattern
3. Aggregate results from multiple APIs
4. Combine responses in Format Agent

## Support

For issues or questions:
- Check Flowise documentation: https://docs.flowiseai.com
- Review FAILURE_PATTERNS.md for common issues
- Submit issues to Context Foundry GitHub

## License

MIT License - See LICENSE file for details

---

🤖 **Generated with [Context Foundry](https://contextfoundry.dev)**
