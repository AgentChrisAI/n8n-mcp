# Railway Multi-Tenant Deployment Verification Guide

Step-by-step guide to verify your Railway multi-tenant deployment is working correctly.

## Prerequisites

- Railway deployment completed
- Environment variables configured (see [RAILWAY_MULTI_TENANT.md](./RAILWAY_MULTI_TENANT.md))
- `curl` or similar HTTP client installed
- Your Railway service URL
- Your Railway AUTH_TOKEN
- Test n8n credentials (optional, for management tool testing)

## Step 1: Verify Health Endpoint

Test that the Railway deployment is running and accessible.

### Command

```bash
curl https://your-app-name.up.railway.app/health
```

### Expected Response

```json
{
  "status": "ok",
  "mode": "http",
  "version": "2.7.13",
  "uptime": 123,
  "memory": {
    "used": 45,
    "total": 128,
    "unit": "MB"
  },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### Verification

- [ ] Status is "ok"
- [ ] Mode is "http"
- [ ] Version is displayed
- [ ] No errors in response

## Step 2: Verify Environment Variables

Check that required environment variables are set in Railway.

### Required Variables Checklist

- [ ] `AUTH_TOKEN` is set (not the default placeholder)
- [ ] `USE_FIXED_HTTP=false` is set
- [ ] `ENABLE_MULTI_TENANT=true` is set
- [ ] `MCP_MODE=http` is set
- [ ] `N8N_API_URL` is NOT set (should come from headers)
- [ ] `N8N_API_KEY` is NOT set (should come from headers)

### How to Check

1. Go to Railway dashboard
2. Click on your service
3. Go to "Variables" tab
4. Verify all required variables are set correctly

## Step 3: Test Tool Listing Without Headers

Verify that without multi-tenant headers, only documentation tools are returned.

### Command

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 1
  }'
```

### Expected Response

- Should return approximately **24 tools** (documentation tools only)
- Tool names should include: `search_nodes`, `list_nodes`, `get_node_essentials`, `validate_workflow`, etc.
- Should NOT include: `n8n_list_workflows`, `n8n_create_workflow`, etc.

### Verification

- [ ] Response contains ~24 tools
- [ ] All tools are documentation tools (no `n8n_` prefix)
- [ ] No errors in response

## Step 4: Test Tool Listing With Headers

Verify that with multi-tenant headers, all tools (42 total) are returned.

### Command

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://test.n8n.cloud" \
  -H "X-N8n-Key: test-api-key" \
  -H "X-Instance-Id: test-user-1" \
  -H "X-Session-Id: test-session-1" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 2
  }'
```

### Expected Response

- Should return approximately **42 tools** (24 documentation + 18 management)
- Tool names should include:
  - Documentation: `search_nodes`, `list_nodes`, `get_node_essentials`, etc.
  - Management: `n8n_list_workflows`, `n8n_create_workflow`, `n8n_get_workflow`, etc.

### Verification

- [ ] Response contains ~42 tools
- [ ] Both documentation and management tools are present
- [ ] Management tools have `n8n_` prefix
- [ ] No errors in response

## Step 5: Test Documentation Tool (No n8n Connection)

Test a documentation tool that doesn't require n8n connection.

### Command

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://test.n8n.cloud" \
  -H "X-N8n-Key: test-api-key" \
  -H "X-Instance-Id: test-user-1" \
  -H "X-Session-Id: test-session-1" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "search_nodes",
      "arguments": {
        "query": "slack",
        "limit": 5
      }
    },
    "id": 3
  }'
```

### Expected Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"totalResults\":5,\"results\":[...]}"
      }
    ]
  },
  "id": 3
}
```

### Verification

- [ ] Response contains search results
- [ ] No errors in response
- [ ] Results are relevant to the query

## Step 6: Test Management Tool (Requires n8n Connection)

Test a management tool that requires valid n8n credentials.

### Command (with valid credentials)

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://YOUR_ACTUAL_N8N_INSTANCE.com" \
  -H "X-N8n-Key: YOUR_ACTUAL_N8N_API_KEY" \
  -H "X-Instance-Id: test-user-1" \
  -H "X-Session-Id: test-session-1" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "n8n_list_workflows",
      "arguments": {}
    },
    "id": 4
  }'
```

### Expected Response (with valid credentials)

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"workflows\":[...]}"
      }
    ]
  },
  "id": 4
}
```

### Expected Response (with invalid credentials)

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Error executing tool n8n_list_workflows: Failed to connect to n8n instance"
  },
  "id": 4
}
```

### Verification

- [ ] With valid credentials: Returns workflows or success response
- [ ] With invalid credentials: Returns appropriate error message
- [ ] Error messages are clear and helpful

## Step 7: Test Instance Context Validation

Verify that invalid instance context is properly rejected.

### Test 1: Invalid URL Format

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: invalid-url" \
  -H "X-N8n-Key: test-key" \
  -H "X-Instance-Id: test-user-1" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 5
  }'
```

**Expected**: Should still return tools (validation happens at tool call time, not listing time)

### Test 2: Empty API Key

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://test.n8n.cloud" \
  -H "X-N8n-Key: " \
  -H "X-Instance-Id: test-user-1" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 6
  }'
```

**Expected**: Should still return tools (validation happens at tool call time)

### Verification

- [ ] Invalid URLs don't crash the server
- [ ] Empty API keys are handled gracefully
- [ ] Validation errors are clear when tools are called

## Step 8: Test Authentication

Verify that authentication is working correctly.

### Test 1: Missing Authorization Header

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 7
  }'
```

**Expected Response**:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Unauthorized"
  },
  "id": null
}
```

### Test 2: Invalid AUTH_TOKEN

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer invalid-token" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 8
  }'
```

**Expected Response**:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Unauthorized"
  },
  "id": null
}
```

### Verification

- [ ] Missing auth returns 401 Unauthorized
- [ ] Invalid token returns 401 Unauthorized
- [ ] Error messages are clear

## Step 9: Test Session Management

Verify that sessions are properly isolated.

### Test: Multiple Sessions

```bash
# Session 1
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user1.n8n.cloud" \
  -H "X-N8n-Key: user1-key" \
  -H "X-Instance-Id: user-1" \
  -H "X-Session-Id: session-1" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":9}'

# Session 2 (different user)
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user2.n8n.cloud" \
  -H "X-N8n-Key: user2-key" \
  -H "X-Instance-Id: user-2" \
  -H "X-Session-Id: session-2" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":10}'
```

### Verification

- [ ] Both sessions return tools successfully
- [ ] Each session is isolated (no cross-contamination)
- [ ] Different instance IDs work correctly

## Step 10: Check Railway Logs

Review Railway logs for any errors or warnings.

### How to Check

1. Go to Railway dashboard
2. Click on your service
3. Go to "Deployments" tab
4. Click on latest deployment
5. View logs

### What to Look For

- [ ] No fatal errors
- [ ] "Instance context extracted from headers" messages (when headers are present)
- [ ] "Tool listing: X tools available" messages
- [ ] No authentication failures (unless testing invalid tokens)
- [ ] No connection errors (unless testing invalid n8n credentials)

## Verification Checklist Summary

### Deployment Configuration

- [ ] Health endpoint returns "ok"
- [ ] All required environment variables are set
- [ ] `USE_FIXED_HTTP=false` is set
- [ ] `ENABLE_MULTI_TENANT=true` is set
- [ ] `N8N_API_URL` and `N8N_API_KEY` are NOT set

### Tool Listing

- [ ] Without headers: Returns ~24 tools (documentation only)
- [ ] With headers: Returns ~42 tools (documentation + management)

### Tool Execution

- [ ] Documentation tools work without n8n connection
- [ ] Management tools work with valid n8n credentials
- [ ] Management tools fail gracefully with invalid credentials

### Authentication & Security

- [ ] Missing auth returns 401
- [ ] Invalid token returns 401
- [ ] Valid token works correctly

### Session Management

- [ ] Multiple sessions work independently
- [ ] Instance context is properly isolated
- [ ] No cross-contamination between users

## Troubleshooting

### Issue: Only 24 tools returned even with headers

**Solution**: Check that `ENABLE_MULTI_TENANT=true` is set in Railway environment variables.

### Issue: 401 Unauthorized errors

**Solution**: Verify `AUTH_TOKEN` matches exactly in Railway and your requests.

### Issue: Management tools not connecting to n8n

**Solution**: 
1. Verify n8n credentials are correct
2. Check that n8n instance is accessible
3. Review Railway logs for connection errors

### Issue: Invalid instance context errors

**Solution**: 
1. Verify URL format (must be HTTP/HTTPS)
2. Check API key is not empty
3. Review validation error messages in logs

## Next Steps

After verification is complete:

1. Integrate with your web application (see [WEB_APP_INTEGRATION.md](./WEB_APP_INTEGRATION.md))
2. Test with real user credentials
3. Monitor Railway logs for production issues
4. Set up alerts for errors

## Additional Resources

- [RAILWAY_MULTI_TENANT.md](./RAILWAY_MULTI_TENANT.md) - Deployment guide
- [WEB_APP_INTEGRATION.md](./WEB_APP_INTEGRATION.md) - Web app integration
- [API_EXAMPLES.md](./API_EXAMPLES.md) - API request examples

