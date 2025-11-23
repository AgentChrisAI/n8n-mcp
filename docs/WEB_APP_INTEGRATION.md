# Web Application Integration Guide

Integration guide for web applications (including Lovable) to connect to Railway-deployed n8n-mcp with multi-tenant support.

## Overview

This guide shows how to integrate n8n-mcp into your web application, allowing users to:
1. Enter their n8n API credentials (URL + API key)
2. Access all 42 MCP tools (24 documentation + 18 management)
3. Have management tools connect to their own n8n instance

## Architecture

```
User's Browser
    ↓
Your Web App (Lovable/React/Next.js/etc.)
    ↓
User enters n8n credentials → Stored securely
    ↓
User makes MCP tool call → Your app proxies to Railway
    ↓ (with X-N8n-Url, X-N8n-Key headers)
Railway n8n-mcp (multi-tenant mode)
    ↓
User's n8n instance
```

## Prerequisites

- Railway deployment configured (see [RAILWAY_MULTI_TENANT.md](./RAILWAY_MULTI_TENANT.md))
- Web application framework (React, Next.js, Vue, etc.)
- Backend API or serverless functions to proxy requests
- Secure credential storage solution

## Step 1: User Credential Management

### 1.1 Create Credential Input Form

Create a settings/profile page where users can enter their n8n credentials:

```typescript
// Example: UserSettings.tsx (React/Next.js)
import { useState } from 'react';

interface N8nCredentials {
  n8nApiUrl: string;
  n8nApiKey: string;
}

export function UserSettings() {
  const [credentials, setCredentials] = useState<N8nCredentials>({
    n8nApiUrl: '',
    n8nApiKey: ''
  });
  const [saving, setSaving] = useState(false);

  const validateUrl = (url: string): boolean => {
    try {
      const parsed = new URL(url);
      return parsed.protocol === 'http:' || parsed.protocol === 'https:';
    } catch {
      return false;
    }
  };

  const handleSave = async () => {
    // Validate URL format
    if (!validateUrl(credentials.n8nApiUrl)) {
      alert('Invalid URL. Must be a valid HTTP or HTTPS URL.');
      return;
    }

    // Validate API key is not empty
    if (!credentials.n8nApiKey.trim()) {
      alert('API key is required.');
      return;
    }

    setSaving(true);
    try {
      // Save credentials securely (encrypted in your backend)
      const response = await fetch('/api/user/credentials', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(credentials)
      });

      if (!response.ok) {
        throw new Error('Failed to save credentials');
      }

      alert('Credentials saved successfully!');
    } catch (error) {
      alert('Error saving credentials: ' + error.message);
    } finally {
      setSaving(false);
    }
  };

  return (
    <div className="settings-form">
      <h2>n8n Integration Settings</h2>
      
      <div className="form-group">
        <label htmlFor="n8n-url">n8n Instance URL</label>
        <input
          id="n8n-url"
          type="url"
          placeholder="https://your-n8n-instance.com"
          value={credentials.n8nApiUrl}
          onChange={(e) => setCredentials({ ...credentials, n8nApiUrl: e.target.value })}
        />
        <small>Your n8n instance URL (must be HTTP or HTTPS)</small>
      </div>

      <div className="form-group">
        <label htmlFor="n8n-key">n8n API Key</label>
        <input
          id="n8n-key"
          type="password"
          placeholder="Enter your n8n API key"
          value={credentials.n8nApiKey}
          onChange={(e) => setCredentials({ ...credentials, n8nApiKey: e.target.value })}
        />
        <small>Get this from n8n Settings → API</small>
      </div>

      <button onClick={handleSave} disabled={saving}>
        {saving ? 'Saving...' : 'Save Credentials'}
      </button>
    </div>
  );
}
```

### 1.2 Backend: Store Credentials Securely

**IMPORTANT**: Never store API keys in plain text. Always encrypt at rest.

```typescript
// Example: Backend API route (Next.js API route, Express, etc.)
import { encrypt, decrypt } from './encryption'; // Your encryption utility
import { db } from './database'; // Your database

// POST /api/user/credentials
export async function saveCredentials(req, res) {
  const userId = req.user.id; // From authentication middleware
  const { n8nApiUrl, n8nApiKey } = req.body;

  // Validate URL format
  try {
    const url = new URL(n8nApiUrl);
    if (url.protocol !== 'http:' && url.protocol !== 'https:') {
      return res.status(400).json({ error: 'Invalid URL protocol' });
    }
  } catch {
    return res.status(400).json({ error: 'Invalid URL format' });
  }

  // Validate API key
  if (!n8nApiKey || !n8nApiKey.trim()) {
    return res.status(400).json({ error: 'API key is required' });
  }

  // Encrypt API key before storing
  const encryptedApiKey = encrypt(n8nApiKey);

  // Store in database
  await db.userCredentials.upsert({
    where: { userId },
    update: {
      n8nApiUrl,
      n8nApiKey: encryptedApiKey,
      updatedAt: new Date()
    },
    create: {
      userId,
      n8nApiUrl,
      n8nApiKey: encryptedApiKey
    }
  });

  res.json({ success: true });
}

// GET /api/user/credentials
export async function getCredentials(req, res) {
  const userId = req.user.id;

  const credentials = await db.userCredentials.findUnique({
    where: { userId }
  });

  if (!credentials) {
    return res.json({ n8nApiUrl: '', n8nApiKey: '' });
  }

  // Decrypt API key
  const decryptedApiKey = decrypt(credentials.n8nApiKey);

  // Return URL and decrypted key (only to authenticated user)
  res.json({
    n8nApiUrl: credentials.n8nApiUrl,
    n8nApiKey: decryptedApiKey
  });
}
```

## Step 2: MCP Request Proxy Service

### 2.1 Create MCP Client Service

Create a service to handle MCP protocol communication:

```typescript
// Example: mcpClient.ts
interface JSONRPCRequest {
  jsonrpc: '2.0';
  method: string;
  params?: any;
  id: number | string;
}

interface JSONRPCResponse {
  jsonrpc: '2.0';
  result?: any;
  error?: {
    code: number;
    message: string;
    data?: any;
  };
  id: number | string | null;
}

class MCPClient {
  private railwayUrl: string;
  private railwayAuthToken: string;
  private sessionId: string;

  constructor(railwayUrl: string, railwayAuthToken: string) {
    this.railwayUrl = railwayUrl;
    this.railwayAuthToken = railwayAuthToken;
    this.sessionId = this.generateSessionId();
  }

  private generateSessionId(): string {
    // Generate UUID v4
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
      const r = Math.random() * 16 | 0;
      const v = c === 'x' ? r : (r & 0x3 | 0x8);
      return v.toString(16);
    });
  }

  async request(
    method: string,
    params: any,
    userCredentials: { n8nApiUrl: string; n8nApiKey: string },
    userId: string
  ): Promise<JSONRPCResponse> {
    const request: JSONRPCRequest = {
      jsonrpc: '2.0',
      method,
      params,
      id: Date.now()
    };

    const response = await fetch(`${this.railwayUrl}/mcp`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.railwayAuthToken}`,
        'Content-Type': 'application/json',
        'X-N8n-Url': userCredentials.n8nApiUrl,
        'X-N8n-Key': userCredentials.n8nApiKey,
        'X-Instance-Id': userId,
        'X-Session-Id': this.sessionId
      },
      body: JSON.stringify(request)
    });

    if (!response.ok) {
      if (response.status === 401) {
        throw new Error('Unauthorized: Check Railway AUTH_TOKEN');
      }
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const result: JSONRPCResponse = await response.json();

    if (result.error) {
      throw new Error(`MCP Error ${result.error.code}: ${result.error.message}`);
    }

    return result;
  }

  // Convenience methods
  async initialize(userCredentials: { n8nApiUrl: string; n8nApiKey: string }, userId: string) {
    return this.request(
      'initialize',
      {
        protocolVersion: '2024-11-05',
        capabilities: {},
        clientInfo: {
          name: 'your-web-app',
          version: '1.0.0'
        }
      },
      userCredentials,
      userId
    );
  }

  async listTools(userCredentials: { n8nApiUrl: string; n8nApiKey: string }, userId: string) {
    return this.request('tools/list', {}, userCredentials, userId);
  }

  async callTool(
    toolName: string,
    arguments: any,
    userCredentials: { n8nApiUrl: string; n8nApiKey: string },
    userId: string
  ) {
    return this.request(
      'tools/call',
      {
        name: toolName,
        arguments
      },
      userCredentials,
      userId
    );
  }
}

export const mcpClient = new MCPClient(
  process.env.NEXT_PUBLIC_RAILWAY_MCP_URL || '',
  process.env.RAILWAY_AUTH_TOKEN || ''
);
```

### 2.2 Backend: Proxy MCP Requests

For security, proxy MCP requests through your backend (don't expose Railway AUTH_TOKEN to frontend):

```typescript
// Example: Backend API route (Next.js API route)
// POST /api/mcp/proxy
import { getCredentials } from './credentials'; // Your credential retrieval function

export async function proxyMCPRequest(req, res) {
  const userId = req.user.id;
  const { method, params } = req.body;

  // Get user's n8n credentials
  const credentials = await getCredentials(userId);
  if (!credentials.n8nApiUrl || !credentials.n8nApiKey) {
    return res.status(400).json({
      error: 'n8n credentials not configured',
      message: 'Please configure your n8n API credentials in settings'
    });
  }

  // Get Railway configuration from environment
  const railwayUrl = process.env.RAILWAY_MCP_URL;
  const railwayAuthToken = process.env.RAILWAY_AUTH_TOKEN;

  if (!railwayUrl || !railwayAuthToken) {
    return res.status(500).json({
      error: 'Railway configuration missing'
    });
  }

  // Generate session ID (or retrieve from session)
  const sessionId = req.session.mcpSessionId || generateSessionId();
  req.session.mcpSessionId = sessionId;

  try {
    // Proxy request to Railway
    const response = await fetch(`${railwayUrl}/mcp`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${railwayAuthToken}`,
        'Content-Type': 'application/json',
        'X-N8n-Url': credentials.n8nApiUrl,
        'X-N8n-Key': credentials.n8nApiKey,
        'X-Instance-Id': userId,
        'X-Session-Id': sessionId
      },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method,
        params,
        id: Date.now()
      })
    });

    if (!response.ok) {
      const errorText = await response.text();
      return res.status(response.status).json({
        error: 'Railway request failed',
        details: errorText
      });
    }

    const result = await response.json();
    res.json(result);
  } catch (error) {
    console.error('MCP proxy error:', error);
    res.status(500).json({
      error: 'Internal server error',
      message: error.message
    });
  }
}
```

## Step 3: Frontend Integration

### 3.1 React Hook for MCP Tools

```typescript
// Example: useMCPTools.ts (React hook)
import { useState, useEffect } from 'react';

interface Tool {
  name: string;
  description: string;
  inputSchema: any;
}

export function useMCPTools(userId: string) {
  const [tools, setTools] = useState<Tool[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [initialized, setInitialized] = useState(false);

  // Initialize MCP connection
  useEffect(() => {
    async function initialize() {
      setLoading(true);
      setError(null);

      try {
        // Initialize MCP connection
        const initResponse = await fetch('/api/mcp/proxy', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            method: 'initialize',
            params: {
              protocolVersion: '2024-11-05',
              capabilities: {},
              clientInfo: {
                name: 'your-web-app',
                version: '1.0.0'
              }
            }
          })
        });

        if (!initResponse.ok) {
          throw new Error('Failed to initialize MCP connection');
        }

        // List available tools
        const toolsResponse = await fetch('/api/mcp/proxy', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            method: 'tools/list',
            params: {}
          })
        });

        if (!toolsResponse.ok) {
          throw new Error('Failed to list tools');
        }

        const toolsData = await toolsResponse.json();
        setTools(toolsData.result?.tools || []);
        setInitialized(true);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    }

    if (userId) {
      initialize();
    }
  }, [userId]);

  const callTool = async (toolName: string, arguments: any) => {
    setLoading(true);
    setError(null);

    try {
      const response = await fetch('/api/mcp/proxy', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          method: 'tools/call',
          params: {
            name: toolName,
            arguments
          }
        })
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(errorData.error?.message || 'Tool call failed');
      }

      const result = await response.json();
      return result.result;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  return {
    tools,
    loading,
    error,
    initialized,
    callTool
  };
}
```

### 3.2 Example: Tool Usage Component

```typescript
// Example: MCPToolsComponent.tsx
import { useMCPTools } from './useMCPTools';
import { useState } from 'react';

export function MCPToolsComponent({ userId }: { userId: string }) {
  const { tools, loading, error, initialized, callTool } = useMCPTools(userId);
  const [selectedTool, setSelectedTool] = useState<string>('');
  const [toolArgs, setToolArgs] = useState<string>('{}');
  const [result, setResult] = useState<any>(null);

  const handleCallTool = async () => {
    try {
      const args = JSON.parse(toolArgs);
      const toolResult = await callTool(selectedTool, args);
      setResult(toolResult);
    } catch (err) {
      alert('Error: ' + err.message);
    }
  };

  if (!initialized) {
    return <div>Initializing MCP connection...</div>;
  }

  if (error) {
    return <div className="error">Error: {error}</div>;
  }

  return (
    <div className="mcp-tools">
      <h2>Available Tools ({tools.length})</h2>
      
      <div className="tool-selector">
        <label>Select Tool:</label>
        <select value={selectedTool} onChange={(e) => setSelectedTool(e.target.value)}>
          <option value="">-- Select a tool --</option>
          {tools.map(tool => (
            <option key={tool.name} value={tool.name}>
              {tool.name} - {tool.description}
            </option>
          ))}
        </select>
      </div>

      {selectedTool && (
        <div className="tool-caller">
          <label>Tool Arguments (JSON):</label>
          <textarea
            value={toolArgs}
            onChange={(e) => setToolArgs(e.target.value)}
            rows={5}
            placeholder='{"query": "slack"}'
          />
          <button onClick={handleCallTool} disabled={loading}>
            {loading ? 'Calling...' : 'Call Tool'}
          </button>
        </div>
      )}

      {result && (
        <div className="tool-result">
          <h3>Result:</h3>
          <pre>{JSON.stringify(result, null, 2)}</pre>
        </div>
      )}
    </div>
  );
}
```

## Step 4: Error Handling

### 4.1 Common Errors and Solutions

```typescript
// Error handling utility
export function handleMCPError(error: any): string {
  if (error.message?.includes('Unauthorized')) {
    return 'Authentication failed. Please check Railway configuration.';
  }
  
  if (error.message?.includes('credentials not configured')) {
    return 'Please configure your n8n API credentials in settings.';
  }
  
  if (error.message?.includes('Invalid instance context')) {
    return 'Invalid n8n credentials. Please check your URL and API key.';
  }
  
  if (error.message?.includes('n8n API')) {
    return 'Error connecting to your n8n instance. Please verify your credentials.';
  }
  
  return error.message || 'An unexpected error occurred';
}
```

## Step 5: Environment Variables

### 5.1 Backend Environment Variables

```bash
# .env (backend)
RAILWAY_MCP_URL=https://your-app-name.up.railway.app
RAILWAY_AUTH_TOKEN=your-secure-token-here
DATABASE_URL=your-database-url
ENCRYPTION_KEY=your-encryption-key
```

### 5.2 Frontend Environment Variables (if needed)

```bash
# .env.local (frontend - only if exposing Railway URL)
NEXT_PUBLIC_RAILWAY_MCP_URL=https://your-app-name.up.railway.app
# NOTE: Never expose AUTH_TOKEN to frontend - always proxy through backend
```

## Security Checklist

- [ ] Encrypt API keys at rest in database
- [ ] Never expose Railway AUTH_TOKEN to frontend
- [ ] Always proxy MCP requests through backend
- [ ] Validate URL format before storing
- [ ] Use HTTPS for all requests
- [ ] Implement rate limiting
- [ ] Log errors without exposing sensitive data
- [ ] Set CORS_ORIGIN to your domain in production

## Next Steps

- See [API_EXAMPLES.md](./API_EXAMPLES.md) for detailed MCP request examples
- See [RAILWAY_MULTI_TENANT.md](./RAILWAY_MULTI_TENANT.md) for Railway deployment details
- Test your integration with the examples provided

