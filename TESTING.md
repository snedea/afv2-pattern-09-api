# Pattern #9: API Integration - Testing Guide

**Pattern:** API Integration (HTTP Request Node + Condition Routing)
**Version:** 1.0
**Last Updated:** 2025-11-05
**Repository:** https://github.com/snedea/afv2-pattern-09-api

---

## 🎯 Overview

This guide provides comprehensive test cases for the **API Integration** pattern, which demonstrates:
- **HTTP Request Node** for external API calls
- **Condition Node** for HTTP status code routing (200/4xx/5xx)
- Intelligent error handling (retryable vs non-retryable errors)
- Exponential backoff retry strategy
- Triple terminal paths (Success, Error, Fatal)

---

## 🏗️ Pattern Architecture

**Flow:** Start → Parameter Extractor → HTTP Request Node → Condition Node (status check) → [SUCCESS (200) → Format Agent → Direct Reply | ERROR (5xx) → Retry Agent → loop back to HTTP | FATAL (4xx) → Error Handler → Direct Reply]

**Key Components:**
- **Parameter Extractor Agent:** Parses user request, extracts API parameters
- **HTTP Request Node:** Makes external API calls (GET/POST/PUT/DELETE)
- **Condition Node:** Routes based on HTTP status code:
  - `200-299` → SUCCESS (non-retryable, format and return)
  - `500-599` → ERROR (retryable, loop back with backoff)
  - `400-499` → FATAL (non-retryable, client error)
- **Retry Agent:** Implements exponential backoff (2s, 4s, 8s)
- **Format Agent:** Parses successful API response
- **Error Handler:** Formats error message with diagnostics
- **2x Direct Reply:** Terminal nodes for success and error paths

**Nodes:** 10 total (1 Start, 4 Agents, 1 HTTP Request Node, 1 Condition Node, 2 Direct Reply, 2 Sticky Notes)
**Edges:** 9 connections (1 animated loop-back for retries)

---

## 📋 Test Case Matrix

| Test ID | Scenario | HTTP Status | Expected Path | Retries | Duration |
|---------|----------|-------------|---------------|---------|----------|
| TC-9.1 | Valid API request | 200 OK | SUCCESS → Format | 0 | ~10-15s |
| TC-9.2 | Server error (retryable) | 503 | ERROR → RETRY → FAIL | 3 | ~30-45s |
| TC-9.3 | Client error (non-retryable) | 404 | FATAL → Error Handler | 0 | ~8-12s |

---

## 🧪 Test Case Details

### Test Case TC-9.1: Success (200 OK)

**Objective:** Validate HTTP Request Node + SUCCESS path routing for valid API calls

#### Prerequisites
- ✅ Flowise instance running with internet access
- ✅ Pattern imported: `09-api-integration.json`
- ✅ Anthropic API key configured
- ✅ Test API endpoint accessible: https://api.github.com

#### Import Steps
1. Open Flowise UI → **Agentflows** → **Add New**
2. Click **"Load Agentflow"**
3. Select `09-api-integration.json`
4. Verify 10 nodes load correctly:
   - 1 Start node
   - 4 Agent nodes: Parameter Extractor, Format, Retry, Error Handler
   - 1 HTTP Request Node (blue/gray)
   - 1 Condition Node: "Status Check" (orange)
   - 2 Direct Reply nodes: Success and Error terminals
   - 2 Sticky Notes
5. Click **"Save Agentflow"**

#### Configuration

**HTTP Request Node:**
1. Click HTTP Request Node
2. Configure:
   - **Method:** GET
   - **URL:** `https://api.github.com/users/anthropics`
   - **Headers:**
     ```json
     {
       "Accept": "application/json",
       "User-Agent": "Flowise-Test"
     }
     ```
   - **Body:** (leave empty for GET)
   - **Timeout:** 10000ms

**Condition Node (Status Check):**
- **Scenario 0 (SUCCESS):**
  - Variable: `{{ $flow.state.api.status_code }}`
  - Operator: `equal`
  - Value: `200`
  - Type: `number`

- **Scenario 1 (ERROR - retryable):**
  - Variable: `{{ $flow.state.api.status_code }}`
  - Operator: `larger` (≥)
  - Value: `500`
  - Type: `number`

- **Scenario 2 (FATAL - non-retryable):**
  - Variable: `{{ $flow.state.api.status_code }}`
  - Operator: `larger` (≥)
  - Value: `400`
  - Additional condition: `< 500`

#### Test Input
```
Fetch user information for GitHub user 'anthropics'
```

#### Expected Output

**Parameter Extractor Output:**
```json
{
  "username": "anthropics",
  "api_endpoint": "https://api.github.com/users/anthropics",
  "method": "GET"
}
```

**HTTP Request Output:**
```json
{
  "login": "anthropics",
  "id": 12345678,
  "name": "Anthropic",
  "company": "Anthropic",
  "blog": "https://anthropic.com",
  "location": "San Francisco, CA",
  "public_repos": 50,
  "followers": 10000,
  "following": 0,
  "created_at": "2020-01-01T00:00:00Z"
}
```

**HTTP Status Code:** 200 OK

**Condition Node Decision:**
- Input: `status_code = 200`
- Check: `200 == 200` → **TRUE**
- Route: Scenario 0 (SUCCESS)

**Format Agent Output:**
```
✅ GitHub User Information Retrieved

Username: anthropics
Name: Anthropic
Location: San Francisco, CA
Company: Anthropic
Website: https://anthropic.com

Statistics:
- Public Repositories: 50
- Followers: 10,000
- Following: 0
- Account Created: January 1, 2020

API Status: 200 OK
Response Time: 234ms
Retries: 0
```

**Direct Reply (Success) fires** - workflow terminates successfully

#### Validation Checklist
- [ ] Parameter Extractor parses username correctly ("anthropics")
- [ ] HTTP Request Node executes GET request
- [ ] Response status code = 200
- [ ] Response body is valid JSON
- [ ] Condition Node routes to Scenario 0 (SUCCESS)
- [ ] Format Agent parses JSON fields correctly
- [ ] Direct Reply (Success) fires with formatted data
- [ ] No retries attempted (retry count = 0)
- [ ] Execution time < 15 seconds
- [ ] No errors in execution log

#### Common Issues & Debugging

**Issue:** HTTP Request fails with "CORS policy" error

**Root Cause:** Browser-based CORS restriction (shouldn't happen in Flowise backend)

**Fix:**
1. Verify Flowise is running in server mode (not browser-only)
2. Test API with `curl` first:
   ```bash
   curl https://api.github.com/users/anthropics
   ```
3. If curl works but Flowise fails, check Flowise network settings

---

**Issue:** Condition Node always routes to ELSE

**Root Cause:** Status code not stored in flow state

**Fix:**
1. Check HTTP Request Node configuration
2. Verify it saves status code to state:
   ```json
   {
     "key": "api.status_code",
     "value": "{{ statusCode }}"
   }
   ```

---

### Test Case TC-9.2: Retry on Server Error (5xx)

**Objective:** Validate retry loop with exponential backoff for server errors

#### Configuration

**HTTP Request Node:**
- **URL:** `https://httpstat.us/503` (simulates 503 Service Unavailable)
- **Method:** GET
- **Headers:**
  ```json
  {
    "Accept": "application/json"
  }
  ```

**Note:** `httpstat.us` is a test service that returns specified HTTP status codes

#### Test Input
```
Test API error handling with a server unavailable scenario
```

#### Expected Output - Attempt 1

**HTTP Request:**
- Status Code: 503
- Response: `{"code": 503, "description": "Service Unavailable"}`

**Condition Node Decision:**
- Input: `status_code = 503`
- Check: `503 >= 500` → **TRUE**
- Route: Scenario 1 (ERROR - retryable)

**Retry Agent Output:**
```json
{
  "action": "RETRY",
  "retry_count": 1,
  "backoff_seconds": 2,
  "reasoning": "Server error (503) is transient, retry with exponential backoff"
}
```

**Loop-back edge fires:** Retry Agent → HTTP Request (wait 2 seconds, retry)

#### Expected Output - Attempt 2

**HTTP Request:** 503 (again)

**Retry Agent:**
- Retry count: 2
- Backoff: 4 seconds
- Loop back to HTTP Request

#### Expected Output - Attempt 3

**HTTP Request:** 503 (again)

**Retry Agent:**
- Retry count: 3
- Backoff: 8 seconds
- Loop back to HTTP Request

#### Expected Output - Max Retries Exhausted

**Retry Agent Decision:**
```json
{
  "action": "FAIL",
  "retry_count": 3,
  "reason": "Maximum retries (3) exhausted. Server still unavailable (503).",
  "total_wait_time": "14 seconds (2s + 4s + 8s)"
}
```

**Route to Error Handler:**

**Error Handler Output:**
```
❌ API Request Failed

Endpoint: https://httpstat.us/503
Method: GET
Status: 503 Service Unavailable

Error Type: Server Error (retryable)
Retries Attempted: 3 (max exhausted)

Retry History:
1. Attempt 1 → 503 (waited 2s)
2. Attempt 2 → 503 (waited 4s)
3. Attempt 3 → 503 (waited 8s)

Total Retry Time: 14 seconds

Recommendation: The remote server is experiencing issues. Please try again later or contact the API provider.
```

**Direct Reply (Error) fires** - workflow terminates on error path

#### Validation Checklist
- [ ] HTTP Request executes 4 times (initial + 3 retries)
- [ ] Condition Node routes to Scenario 1 (ERROR) on 503 status
- [ ] Retry Agent implements exponential backoff:
  - Retry 1: 2 second wait
  - Retry 2: 4 second wait
  - Retry 3: 8 second wait
- [ ] Loop-back edge triggers 3 times (animated edge visible)
- [ ] Retry counter increments: 0 → 1 → 2 → 3
- [ ] After 3 retries, routes to Error Handler (not retry again)
- [ ] Direct Reply (Error) fires with retry history
- [ ] Total execution time ~30-45 seconds (including backoff waits)
- [ ] **CRITICAL:** No 4th retry (max 3 enforced)

#### Exponential Backoff Validation

**Expected Timeline:**
```
00:00 - Initial request → 503
00:02 - Retry 1 (after 2s wait) → 503
00:06 - Retry 2 (after 4s wait) → 503
00:14 - Retry 3 (after 8s wait) → 503
00:14 - Max retries, route to Error Handler
```

**Check execution log timestamps to verify backoff intervals**

---

### Test Case TC-9.3: Fatal Error (4xx)

**Objective:** Validate non-retryable client errors fail immediately without retries

#### Configuration

**HTTP Request Node:**
- **URL:** `https://api.github.com/users/THIS_USER_DOES_NOT_EXIST_12345`
- **Method:** GET

#### Test Input
```
Fetch user information for GitHub user 'THIS_USER_DOES_NOT_EXIST_12345'
```

#### Expected Output

**HTTP Request:**
- Status Code: 404
- Response:
  ```json
  {
    "message": "Not Found",
    "documentation_url": "https://docs.github.com/rest/users/users#get-a-user"
  }
  ```

**Condition Node Decision:**
- Input: `status_code = 404`
- Check 1: `404 == 200` → FALSE
- Check 2: `404 >= 500` → FALSE
- Check 3: `404 >= 400` → TRUE (FATAL)
- Route: Scenario 2 (FATAL - non-retryable)

**Error Handler Output:**
```
❌ API Request Failed (Client Error)

Endpoint: https://api.github.com/users/THIS_USER_DOES_NOT_EXIST_12345
Method: GET
Status: 404 Not Found

Error Type: Client Error (non-retryable)
Retries Attempted: 0 (client errors are not retried)

Response:
{
  "message": "Not Found",
  "documentation_url": "https://docs.github.com/rest/users/users#get-a-user"
}

Diagnosis: The requested resource does not exist. This is a client-side error (4xx) which typically indicates:
- Invalid username or ID
- Resource was deleted
- Incorrect API endpoint

Recommendation: Verify the username 'THIS_USER_DOES_NOT_EXIST_12345' is correct and try again.
```

**Direct Reply (Error) fires immediately** - no retries

#### Validation Checklist
- [ ] HTTP Request executes exactly **1 time** (no retries)
- [ ] Status code = 404 (or other 4xx)
- [ ] Condition Node routes to Scenario 2 (FATAL)
- [ ] Retry Agent **DOES NOT** execute (bypassed)
- [ ] Loop-back edge **DOES NOT** trigger (no retry on 4xx)
- [ ] Error Handler executes immediately
- [ ] Direct Reply (Error) fires with client error message
- [ ] Retry count = 0 (confirmed in output)
- [ ] Execution time < 12 seconds (single request, no waits)
- [ ] **CRITICAL:** No retry attempts (4xx errors are non-retryable)

#### 4xx vs 5xx Validation

**Key Difference:**
- **4xx (Client Error):** Immediate failure, no retries (user's fault)
- **5xx (Server Error):** Retry with backoff (server's fault, may recover)

**Test both scenarios to ensure correct routing:**

| Status | Type | Retryable? | Expected Path |
|--------|------|------------|---------------|
| 200 | Success | N/A | SUCCESS → Format |
| 400 | Bad Request | ❌ No | FATAL → Error Handler |
| 401 | Unauthorized | ❌ No | FATAL → Error Handler |
| 404 | Not Found | ❌ No | FATAL → Error Handler |
| 429 | Rate Limit | ✅ Yes* | ERROR → Retry (special case) |
| 500 | Internal Server Error | ✅ Yes | ERROR → Retry |
| 502 | Bad Gateway | ✅ Yes | ERROR → Retry |
| 503 | Service Unavailable | ✅ Yes | ERROR → Retry |
| 504 | Gateway Timeout | ✅ Yes | ERROR → Retry |

*Note: 429 (Rate Limit) is technically 4xx but should be treated as retryable with longer backoff

---

## 🔍 Advanced Testing Scenarios

### Scenario A: Timeout Handling

**Configuration:**
- HTTP Request Node timeout: 5000ms (5 seconds)
- Target URL: `https://httpstat.us/200?sleep=10000` (sleeps 10 seconds)

**Expected:** Request times out after 5s, treated as 5xx error, retries with backoff

---

### Scenario B: Large Response Payload

**Configuration:**
- URL: `https://jsonplaceholder.typicode.com/photos` (5000 items)

**Expected:**
- Request succeeds (200)
- Format Agent handles large JSON array
- May take 15-20 seconds to process

---

### Scenario C: POST Request with Body

**Configuration:**
```json
{
  "method": "POST",
  "url": "https://jsonplaceholder.typicode.com/posts",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": {
    "title": "Test Post",
    "body": "This is a test",
    "userId": 1
  }
}
```

**Expected:** 201 Created, Format Agent parses created resource ID

---

### Scenario D: Authentication (API Key)

**Configuration:**
```json
{
  "headers": {
    "Authorization": "Bearer YOUR_API_KEY",
    "Content-Type": "application/json"
  }
}
```

**Expected:** 200 if valid key, 401 Unauthorized if invalid (routes to FATAL)

---

## 📊 Test Execution Report Template

```markdown
## Pattern #9 Test Execution Report

**Date:** 2025-11-05
**Tester:** [Your Name]
**Flowise Version:** [e.g., v2.1.0]

### Test Results

| Test Case | Status | HTTP Status | Retries | Duration | Path Taken |
|-----------|--------|-------------|---------|----------|------------|
| TC-9.1: Success (200) | ✅ PASS | 200 | 0 | 11s | SUCCESS → Format |
| TC-9.2: Retry (5xx) | ✅ PASS | 503 | 3 | 42s | ERROR → Retry (3x) → Fail |
| TC-9.3: Fatal (4xx) | ✅ PASS | 404 | 0 | 9s | FATAL → Error Handler |

### Critical Validation
- [ ] ✅ 4xx errors do NOT trigger retries (non-retryable)
- [ ] ✅ 5xx errors trigger retries with exponential backoff
- [ ] ✅ Max 3 retries enforced (no infinite loops)
- [ ] ✅ HTTP Request Node executes successfully
- [ ] ✅ Condition Node routes correctly based on status code

### Issues Found
- None

### Overall Assessment
✅ **PASS** - Pattern ready for production use
```

---

## 🎯 Success Criteria

Mark testing complete when:
- [ ] TC-9.1 (Success 200) passes
- [ ] TC-9.2 (Retry 5xx) passes with correct backoff
- [ ] TC-9.3 (Fatal 4xx) passes without retries
- [ ] **CRITICAL:** 4xx errors do NOT retry (client errors)
- [ ] **CRITICAL:** 5xx errors DO retry (server errors)
- [ ] Exponential backoff implemented correctly (2s, 4s, 8s)
- [ ] Max 3 retries enforced
- [ ] Both success and error Direct Reply nodes fire correctly

---

## 🐛 Known Issues

**Current Issues:**
- None

Report issues at: https://github.com/snedea/afv2-pattern-09-api/issues

---

## 📚 Additional Resources

- **Pattern README:** `README.md` in this repository
- **Integration Guide:** `INTEGRATION_GUIDE.md`
- **Full Test Suite:** [Context Foundry TESTING_GUIDE.md](https://github.com/context-foundry/context-foundry/blob/main/extensions/flowise/templates/afv2-patterns/TESTING_GUIDE.md)
- **HTTP Status Codes:** https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
- **Flowise HTTP Request Node Docs:** https://docs.flowiseai.com/using-flowise/agentflowv2#http-request-node

---

## 🔧 Useful Test APIs

**For Testing:**
- **httpstat.us** - Returns any HTTP status: `https://httpstat.us/503`
- **JSONPlaceholder** - Fake REST API: `https://jsonplaceholder.typicode.com`
- **GitHub API** - Real data: `https://api.github.com/users/anthropics`
- **ReqRes** - Mock API: `https://reqres.in/api/users`

---

**Version History:**
- **1.0** (2025-11-05): Initial test suite with 3 test cases covering 200/4xx/5xx responses
