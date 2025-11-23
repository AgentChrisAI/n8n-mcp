# Railway Multi-Tenant Deployment Guide

Deploy n8n-mcp to Railway with multi-tenant support, enabling web applications to proxy MCP requests where each user's n8n credentials are passed via HTTP headers.

## Overview

Multi-tenant mode allows a single Railway deployment to serve multiple users, where each user's n8n management tools connect to their own n8n instance. This is ideal for:

- SaaS platforms where users have their own n8n instances
- Web applications that need to provide MCP tools to multiple users
- Platforms where users configure their own n8n API credentials

## Architecture

```
User's Web App
    ↓
User enters n8n credentials (API URL + API Key)
    ↓
Web app stores credentials securely
    ↓
Web app proxies MCP requests to Railway
    ↓ (with X-N8n-Url, X-N8n-Key headers)
Railway n8n-mcp (multi-tenant mode)
    ↓
Per-session MCP servers with user's n8n instance
    ↓
User 1 → Their n8n instance
User 2 → Their n8n instance
User 3 → Their n8n instance
```

## Prerequisites

- Railway account
- Web application that can proxy HTTP requests
- Understanding of MCP (Model Context Protocol)

## Step-by-Step Deployment

### 1. Deploy to Railway

1. **Click the Deploy button** on the main Railway deployment guide
2. **Sign in to Railway** (or create account)
3. **Configure your deployment**:
   - Project name (optional)
   - Environment (leave as "production")
   - Region (choose closest to your users)
4. **Click "Deploy"** and wait ~2-3 minutes

### 2. Configure Environment Variables

**CRITICAL**: Configure these environment variables in Railway dashboard:

1. **Go to Railway dashboard** → Your service → **Variables** tab
2. **Set the following variables**:

#### Required Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `AUTH_TOKEN` | `[generate secure token]` | Bearer token for Railway endpoint authentication. Generate with: `openssl rand -base64 32` |
| `USE_FIXED_HTTP` | `false` | **CRITICAL**: Must be `false` to enable multi-tenant support. This uses `SingleSessionHTTPServer` which supports instance context headers. |
| `ENABLE_MULTI_TENANT` | `true` | Enables multi-tenant mode. Shows all tools (42 total) and routes via headers. |
| `MCP_MODE` | `http` | HTTP mode for cloud deployment (already set in Dockerfile) |

#### Optional Variables

| Variable | Value | Description |
|----------|-------|-------------|
| `CORS_ORIGIN` | `https://your-webapp.com` | Your web app's domain. Use `*` for development only. |
| `LOG_LEVEL` | `info` | Logging level: `debug`, `info`, `warn`, `error` |
| `TRUST_PROXY` | `1` | Railway runs behind proxy (already set in Dockerfile) |
| `HOST` | `0.0.0.0` | Listen on all interfaces (already set in Dockerfile) |

#### Variables to NOT Set

**DO NOT SET** these variables (they should come from user headers):
- `N8N_API_URL` - Should be provided via `X-N8n-Url` header
- `N8N_API_KEY` - Should be provided via `X-N8n-Key` header

### 3. Generate Secure AUTH_TOKEN

```bash
# Generate a secure token (minimum 32 characters)
openssl rand -base64 32
```

Copy the output and set it as the `AUTH_TOKEN` environment variable in Railway.

### 4. Get Your Railway Service URL

1. In Railway dashboard, click on your service
2. Go to **Settings** tab
3. Under **Domains**, you'll see your URL:
   ```
   https://your-app-name.up.railway.app
   ```
4. Copy this URL - you'll need it for your web app integration

### 5. Verify Deployment

Test the health endpoint:

```bash
curl https://your-app-name.up.railway.app/health
```

Expected response:
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

## Multi-Tenant Configuration Details

### How It Works

1. **Tool Listing**: When `ENABLE_MULTI_TENANT=true`, all 42 tools are shown:
   - 24 documentation tools (always available)
   - 18 n8n management tools (connect to user's n8n instance via headers)

2. **Instance Context**: Each request can include HTTP headers:
   - `X-N8n-Url`: User's n8n instance URL
   - `X-N8n-Key`: User's n8n API key
   - `X-Instance-Id`: Unique identifier for the user/instance (optional)
   - `X-Session-Id`: Session identifier (optional)

3. **Session Isolation**: Each user session gets an isolated MCP server instance with their n8n credentials.

4. **Automatic Cleanup**: Railway handles session cleanup and resource management automatically.

### Header Extraction Logic

The server extracts instance context from headers (see `src/http-server-single-session.ts:1164-1201`):

1. If either `X-N8n-Url` or `X-N8n-Key` header is present, an instance context is created
2. All headers are extracted and validated
3. The server uses the instance-specific configuration instead of environment variables
4. If no headers are present, the server falls back to documentation-only mode (24 tools)

### Validation

Instance context is validated using `validateInstanceContext()` from `src/types/instance-context.ts`:

- **URL Validation**: Must be valid HTTP/HTTPS URL, no file:// or javascript: protocols
- **API Key Validation**: Non-empty string, no placeholder values
- **Error Messages**: Field-specific error messages for debugging

## Testing Multi-Tenant Mode

### Test 1: Tool Listing with Headers

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://test.n8n.cloud" \
  -H "X-N8n-Key: test-api-key" \
  -H "X-Instance-Id: test-user-1" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 1
  }'
```

**Expected**: Should return 42 tools (24 documentation + 18 management)

### Test 2: Tool Listing without Headers

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

**Expected**: Should return 24 tools (documentation only)

### Test 3: Call Management Tool

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer YOUR_RAILWAY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://your-n8n-instance.com" \
  -H "X-N8n-Key: your-actual-api-key" \
  -H "X-Instance-Id: user-123" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "n8n_list_workflows",
      "arguments": {}
    },
    "id": 2
  }'
```

**Expected**: Should connect to your n8n instance and return workflows

## Troubleshooting

### Issue: Only 24 tools returned (documentation only)

**Cause**: Headers not being sent or `ENABLE_MULTI_TENANT` not set to `true`

**Solution**:
1. Verify `ENABLE_MULTI_TENANT=true` in Railway environment variables
2. Check that `X-N8n-Url` and `X-N8n-Key` headers are being sent
3. Verify headers are lowercase: `x-n8n-url`, `x-n8n-key` (Express normalizes headers)

### Issue: 401 Unauthorized

**Cause**: Invalid or missing `AUTH_TOKEN`

**Solution**:
1. Verify `Authorization: Bearer YOUR_TOKEN` header is correct
2. Check Railway environment variables for `AUTH_TOKEN`
3. Ensure token matches exactly (no extra spaces)

### Issue: Invalid instance context errors

**Cause**: Invalid URL or API key format

**Solution**:
1. Verify URL is valid HTTP/HTTPS (not file:// or javascript:)
2. Check API key is non-empty and not a placeholder
3. Review error messages in Railway logs for specific validation failures

### Issue: Tools not connecting to user's n8n instance

**Cause**: Headers not being extracted or instance context not created

**Solution**:
1. Check Railway logs for "Instance context extracted from headers" debug messages
2. Verify both `X-N8n-Url` and `X-N8n-Key` headers are present
3. Ensure `USE_FIXED_HTTP=false` (required for multi-tenant support)

## Security Best Practices

1. **AUTH_TOKEN**: Use a strong token (minimum 32 characters), never commit to git
2. **CORS_ORIGIN**: Set to your web app's domain in production (not `*`)
3. **HTTPS**: Railway provides HTTPS by default, always use HTTPS URLs
4. **API Keys**: Never log or expose user API keys in frontend or logs
5. **Validation**: Always validate instance context before storing user credentials
6. **Rate Limiting**: Implement rate limiting in your web app to prevent abuse

## Next Steps

- See [WEB_APP_INTEGRATION.md](./WEB_APP_INTEGRATION.md) for web application integration guide
- See [API_EXAMPLES.md](./API_EXAMPLES.md) for MCP request examples
- See [FLEXIBLE_INSTANCE_CONFIGURATION.md](./FLEXIBLE_INSTANCE_CONFIGURATION.md) for advanced configuration options

## Architecture Notes

- The fixed HTTP server (`src/http-server.ts`) does NOT support multi-tenant headers
- Must use `USE_FIXED_HTTP=false` to enable `SingleSessionHTTPServer`
- `SingleSessionHTTPServer` extracts headers at lines 1164-1201 in `src/http-server-single-session.ts`
- Instance context validation happens in `src/types/instance-context.ts:validateInstanceContext()`
- Each user session gets isolated MCP server instance with their n8n credentials
- Railway automatically handles session cleanup and resource management

