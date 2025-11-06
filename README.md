# AFv2 Pattern #9: API Integration with Conditional Retry

**External API integration demonstrating HTTP requests, conditional routing, and loop-back retry logic with exponential backoff.**

## Overview

This Flowise AgentFlow v2 pattern demonstrates how to integrate with external APIs while handling various response scenarios:

- ✅ **SUCCESS (200)**: Format and return response
- 🔄 **ERROR (5xx)**: Retry with exponential backoff (up to 3 attempts)
- ❌ **FATAL (4xx or max retries)**: Handle error gracefully

## Flow Architecture

```
Start (User Request)
  ↓
Agent.ParameterExtractor (Parse API parameters)
  ↓
HTTP Request Node (Execute API call)
  ↓
Condition Node (Check status code)
  ├─→ [200] → Agent.Format → Success Reply
  ├─→ [5xx] → Agent.Retry → (loop back to HTTP)
  └─→ [ELSE] → Agent.ErrorHandler → Error Reply
```

## Key Features

### 1. **Deterministic Condition Routing**
Uses `conditionAgentflow` (If/Else) for numeric status code comparison:
- Condition 1: `status == 200` → SUCCESS path
- Condition 2: `status >= 500` → ERROR path (retry)
- Else: FATAL path (4xx or max retries exhausted)

### 2. **Loop-Back Edge for Retry**
Retry Agent connects back to HTTP Request Node, creating a retry loop:
- Visual indicator: Edge marked as `animated: true`
- Max retries: 3 attempts
- Exponential backoff: 2s, 4s, 8s

### 3. **HTTP Request Integration**
First AFv2 pattern using HTTP Request node:
- Dynamic parameters from Flow State
- Response status and body stored in state
- Supports GET, POST, PUT, DELETE methods

### 4. **Flow State Management**
Structured state tracking throughout execution:
- API parameters (url, method, headers, body)
- Response data (status, body)
- Retry tracking (retry_count, backoff_delay)
- Result formatting (formatted output or error message)

## Node Specifications

| Node | Type | Purpose |
|------|------|---------|
| **Start** | `startAgentflow` | Chat input entry point |
| **Agent.ParameterExtractor** | `agentAgentflow` | Parse user request, extract API params |
| **HTTP Request** | `httpRequestAgentflow` | Execute external API call |
| **Condition** | `conditionAgentflow` | Route based on HTTP status code |
| **Agent.Format** | `agentAgentflow` | Transform successful response |
| **Agent.Retry** | `agentAgentflow` | Implement retry logic with backoff |
| **Agent.ErrorHandler** | `agentAgentflow` | Handle errors gracefully |
| **Success Reply** | `directReplyAgentflow` | Terminal node (success) |
| **Error Reply** | `directReplyAgentflow` | Terminal node (error) |

**Total Nodes**: 10 (including sticky note)
**Total Edges**: 9 (including 1 loop-back edge)

## Flow State Schema

### Initial State (Parameter Extractor)
```json
{
  "api": {
    "url": "https://api.example.com/endpoint",
    "method": "GET",
    "headers": {"Authorization": "Bearer token"},
    "body": "",
    "retry_count": 0,
    "max_retries": 3
  }
}
```

### After HTTP Request
```json
{
  "api": {
    ...previous state,
    "response_status": 200,
    "response_body": "{...}"
  }
}
```

### Success Result
```json
{
  "result": {
    "formatted": "User-friendly response",
    "status": "success"
  }
}
```

### Error Result
```json
{
  "result": {
    "error": "Detailed error message",
    "status": "error",
    "error_code": 404,
    "retry_attempts": 3
  }
}
```

## Usage Examples

### Example 1: Successful API Call
**User Input**: "Call the JSONPlaceholder API to get user ID 1"

**Flow Execution**:
1. Parameter Extractor: Extracts URL `https://jsonplaceholder.typicode.com/users/1`
2. HTTP Request: GET request, receives 200 status
3. Condition: Routes to Format Agent (output 0)
4. Format Agent: Parses JSON, formats user info
5. Success Reply: Returns formatted data

**Output**:
```
✅ API Request Successful

Name: Leanne Graham
Email: Sincere@april.biz
Username: Bret
Phone: 1-770-736-8031 x56442
```

### Example 2: Retry on Server Error
**User Input**: "Call API endpoint that returns 503"

**Flow Execution**:
1. HTTP Request: Receives 503 status
2. Condition: Routes to Retry Agent (output 1)
3. Retry Agent: Increments retry_count to 1, waits 2s
4. Loop back to HTTP Request
5. (Repeat up to 3 times)
6. After 3 failures: Condition routes to Error Handler (ELSE path)
7. Error Reply: Returns error message

**Output**:
```
❌ API Request Failed

The API server is currently unavailable (503 Service Unavailable).
We attempted 3 retries but the service remains down.

Please try again later or contact support if the issue persists.
```

### Example 3: Client Error (No Retry)
**User Input**: "Call API with invalid endpoint (404)"

**Flow Execution**:
1. HTTP Request: Receives 404 status
2. Condition: Routes to Error Handler (output 2, ELSE)
3. Error Handler: Analyzes 4xx error
4. Error Reply: Returns error message with suggestion

**Output**:
```
❌ API Request Failed

The requested endpoint was not found (404 Not Found).

Troubleshooting Steps:
- Verify the API URL is correct
- Check that the endpoint exists in the API documentation
- Ensure you have permission to access this resource
```

## Installation

### Import into Flowise

1. Open Flowise UI
2. Click "Import Chatflow"
3. Select `09-api-integration.json`
4. Verify all nodes import correctly
5. Save the flow

### Configure Credentials

This pattern uses Claude Sonnet 4.5. Ensure you have:
- **Anthropic API Key** credential configured in Flowise
- Name must match: `"Anthropic API Key"`

### Optional: Custom Tools

The pattern uses 2 standard tools (auto-included):
- **currentDateTime**: Provides temporal context
- **searXNG**: Federated web search (if needed for API docs)

No manual tool setup required - these are built-in.

## Testing

### Test 1: Successful GET Request
```
User: "Get user data from https://jsonplaceholder.typicode.com/users/1"
Expected: Formatted user information
```

### Test 2: Successful POST Request
```
User: "Create a new post on JSONPlaceholder with title 'Test' and body 'This is a test'"
Expected: Confirmation of post creation with ID
```

### Test 3: Retry Logic
```
User: "Call an endpoint that returns 503 service unavailable"
Expected: 3 retry attempts with exponential backoff, then error message
```

### Test 4: Client Error Handling
```
User: "Call https://jsonplaceholder.typicode.com/invalid-endpoint"
Expected: Immediate error message (no retries) with troubleshooting steps
```

## Configuration

### Adjust Max Retries
In Agent.ParameterExtractor state updates:
```json
{
  "key": "api.max_retries",
  "value": "3"  // Change to 5 for more retries
}
```

### Adjust Backoff Strategy
In Agent.Retry state updates:
```json
{
  "key": "api.backoff_delay",
  "value": "{{ Math.pow(2, $flow.state.api.retry_count + 1) }}"
  // Exponential: 2^1=2s, 2^2=4s, 2^3=8s
  // Linear alternative: {{ ($flow.state.api.retry_count + 1) * 3 }}
  // Would give: 3s, 6s, 9s
}
```

### Add Authorization Header
In Agent.ParameterExtractor, modify state updates to include auth:
```json
{
  "key": "api.headers",
  "value": "{{ {\"Authorization\": \"Bearer YOUR_TOKEN\", \"Content-Type\": \"application/json\"} }}"
}
```

## Use Cases

This pattern is ideal for:

- **External API Integration**: Weather, CRM, payment gateways
- **Webhook Handling**: Process incoming webhooks with validation
- **Database API Calls**: Query external databases via REST APIs
- **Third-Party Services**: Integrate with Stripe, Twilio, SendGrid, etc.
- **Microservices Communication**: Call internal services with retry logic

## Technical Details

**Model**: Claude Sonnet 4.5 (`claude-sonnet-4-5-20250929`)
**Temperature**: 0.3 (deterministic API parameter extraction)
**Memory**: Enabled on all agents
**Streaming**: Enabled

**Pattern Compliance**:
- ✅ FLOWISE-STRUCTURE-AUTHORITY compliant
- ✅ Pattern #6 prevention (correct tool structure)
- ✅ Pattern #8 prevention (complete inputParams)
- ✅ Single file (no separate configs)
- ✅ Loop-back edge with animated indicator

## Troubleshooting

### Issue: HTTP Request Node Not Working
**Cause**: Flowise may not have built-in HTTP Request node
**Solution**: Replace with Agent using HTTP tool (custom implementation)

### Issue: Infinite Retry Loop
**Cause**: Retry Agent not checking max_retries
**Solution**: Verify Agent.Retry state updates increment retry_count correctly

### Issue: Condition Not Routing Correctly
**Cause**: Variable format incorrect
**Solution**: Use rich text HTML format: `<p><span class="variable" data-type="mention" data-id="$flow.state.api.response_status">{{ $flow.state.api.response_status }}</span></p>`

### Issue: Tools Not Available
**Cause**: currentDateTime or searXNG tools not created in Flowise
**Solution**: Go to Tools → Custom Tools, create both tools (see STANDARD_TOOLS.md)

## Related Patterns

- **Pattern #4: Iteration** - Quality improvement loops
- **Pattern #5: Looping** - Validation-driven retry
- **Pattern #8: Hierarchy** - Multi-role delegation

## Contributing

This pattern is part of the Context Foundry AFv2 pattern library. To suggest improvements:

1. Test the pattern in Flowise
2. Document any issues or enhancements
3. Submit feedback via GitHub issues

## License

MIT License - See LICENSE file for details

---

🤖 **Generated with [Context Foundry](https://contextfoundry.dev)**

**Pattern**: AFv2 Pattern #9 - API Integration
**Version**: 1.0
**Last Updated**: 2025-11-05
