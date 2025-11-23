# MCP API Examples with Multi-Tenant Headers

Complete examples of MCP (Model Context Protocol) requests using multi-tenant headers to connect to Railway-deployed n8n-mcp.

## Overview

All requests to Railway n8n-mcp must include:
- **Authorization header**: Bearer token for Railway authentication
- **Multi-tenant headers**: User's n8n credentials (X-N8n-Url, X-N8n-Key)
- **JSON-RPC 2.0 format**: Standard MCP protocol format

## Base Configuration

```bash
# Your Railway deployment URL
RAILWAY_URL="https://your-app-name.up.railway.app"

# Your Railway AUTH_TOKEN (from Railway environment variables)
AUTH_TOKEN="your-secure-token-here"

# User's n8n credentials (from your web app)
N8N_URL="https://user-n8n-instance.com"
N8N_API_KEY="user-api-key-here"
USER_ID="user-123"
SESSION_ID="session-456"
```

## Example 1: Initialize MCP Connection

Initialize the MCP protocol connection.

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {
        "name": "your-web-app",
        "version": "1.0.0"
      }
    },
    "id": 1
  }'
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {},
      "resources": {}
    },
    "serverInfo": {
      "name": "n8n-documentation-mcp",
      "version": "2.7.13"
    }
  },
  "id": 1
}
```

### JavaScript Example

```javascript
const response = await fetch('https://your-app-name.up.railway.app/mcp', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer your-secure-token-here',
    'Content-Type': 'application/json',
    'X-N8n-Url': 'https://user-n8n-instance.com',
    'X-N8n-Key': 'user-api-key-here',
    'X-Instance-Id': 'user-123',
    'X-Session-Id': 'session-456'
  },
  body: JSON.stringify({
    jsonrpc: '2.0',
    method: 'initialize',
    params: {
      protocolVersion: '2024-11-05',
      capabilities: {},
      clientInfo: {
        name: 'your-web-app',
        version: '1.0.0'
      }
    },
    id: 1
  })
});

const result = await response.json();
console.log(result);
```

## Example 2: List Available Tools

Get all available MCP tools. With multi-tenant headers, returns 42 tools (24 documentation + 18 management).

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 2
  }'
```

### Response (Excerpt)

```json
{
  "jsonrpc": "2.0",
  "result": {
    "tools": [
      {
        "name": "search_nodes",
        "description": "Search n8n nodes by keyword...",
        "inputSchema": { ... }
      },
      {
        "name": "list_nodes",
        "description": "List n8n nodes with filtering...",
        "inputSchema": { ... }
      },
      ...
      {
        "name": "n8n_list_workflows",
        "description": "List workflows from n8n instance...",
        "inputSchema": { ... }
      },
      {
        "name": "n8n_create_workflow",
        "description": "Create workflow in n8n...",
        "inputSchema": { ... }
      },
      ...
    ]
  },
  "id": 2
}
```

**Note**: Without multi-tenant headers, only 24 documentation tools are returned.

## Example 3: Search Nodes (Documentation Tool)

Search for n8n nodes by keyword. This is a documentation tool and doesn't require n8n connection.

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "search_nodes",
      "arguments": {
        "query": "slack",
        "limit": 10
      }
    },
    "id": 3
  }'
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"totalResults\":5,\"results\":[{\"name\":\"Slack\",\"type\":\"nodes-base.slack\",\"description\":\"Interact with Slack\"},...]}"
      }
    ]
  },
  "id": 3
}
```

## Example 4: Get Node Essentials (Documentation Tool)

Get essential properties for a specific node. Documentation tool, no n8n connection needed.

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "get_node_essentials",
      "arguments": {
        "nodeType": "nodes-base.httpRequest"
      }
    },
    "id": 4
  }'
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"nodeType\":\"nodes-base.httpRequest\",\"essentialProperties\":[{\"name\":\"method\",\"required\":true,\"type\":\"string\"},...]}"
      }
    ]
  },
  "id": 4
}
```

## Example 5: List Workflows (Management Tool)

List workflows from user's n8n instance. This is a management tool and requires valid n8n credentials.

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "n8n_list_workflows",
      "arguments": {}
    },
    "id": 5
  }'
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"workflows\":[{\"id\":1,\"name\":\"My Workflow\",\"active\":true},...]}"
      }
    ]
  },
  "id": 5
}
```

### Error Response (Invalid Credentials)

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Error executing tool n8n_list_workflows: Failed to connect to n8n instance"
  },
  "id": 5
}
```

## Example 6: Create Workflow (Management Tool)

Create a new workflow in user's n8n instance.

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "n8n_create_workflow",
      "arguments": {
        "name": "My New Workflow",
        "nodes": [
          {
            "id": "node-1",
            "name": "Start",
            "type": "n8n-nodes-base.start",
            "typeVersion": 1,
            "position": [250, 300],
            "parameters": {}
          }
        ],
        "connections": {}
      }
    },
    "id": 6
  }'
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"id\":123,\"name\":\"My New Workflow\",\"active\":false,\"nodes\":[...]}"
      }
    ]
  },
  "id": 6
}
```

## Example 7: Validate Workflow (Documentation Tool)

Validate a workflow structure. Documentation tool, validates locally without n8n connection.

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "validate_workflow",
      "arguments": {
        "workflow": {
          "name": "Test Workflow",
          "nodes": [
            {
              "id": "node-1",
              "name": "Start",
              "type": "n8n-nodes-base.start",
              "typeVersion": 1,
              "position": [250, 300],
              "parameters": {}
            }
          ],
          "connections": {}
        }
      }
    },
    "id": 7
  }'
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"isValid\":true,\"errors\":[]}"
      }
    ],
    "structuredContent": {
      "isValid": true,
      "errors": []
    }
  },
  "id": 7
}
```

## Example 8: Search Templates (Documentation Tool)

Search for workflow templates by keyword.

### Request

```bash
curl -X POST https://your-app-name.up.railway.app/mcp \
  -H "Authorization: Bearer your-secure-token-here" \
  -H "Content-Type: application/json" \
  -H "X-N8n-Url: https://user-n8n-instance.com" \
  -H "X-N8n-Key: user-api-key-here" \
  -H "X-Instance-Id: user-123" \
  -H "X-Session-Id: session-456" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/call",
    "params": {
      "name": "search_templates",
      "arguments": {
        "query": "slack notification",
        "limit": 5
      }
    },
    "id": 8
  }'
```

### Response

```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"templates\":[{\"id\":\"template-1\",\"name\":\"Slack Notification Workflow\",...}]}"
      }
    ]
  },
  "id": 8
}
```

## Error Handling Examples

### Error: Missing Authorization Header

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

### Error: Invalid AUTH_TOKEN

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

### Error: Invalid Instance Context

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Error executing tool n8n_list_workflows: Invalid instance context: Invalid n8nApiUrl: URL format is malformed or incomplete"
  },
  "id": 5
}
```

### Error: n8n Connection Failed

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32603,
    "message": "Error executing tool n8n_list_workflows: Failed to connect to n8n instance: 401 Unauthorized"
  },
  "id": 5
}
```

## Complete JavaScript Client Example

```javascript
class MCPClient {
  constructor(railwayUrl, authToken) {
    this.railwayUrl = railwayUrl;
    this.authToken = authToken;
    this.sessionId = this.generateSessionId();
  }

  generateSessionId() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
      const r = Math.random() * 16 | 0;
      const v = c === 'x' ? r : (r & 0x3 | 0x8);
      return v.toString(16);
    });
  }

  async request(method, params, userCredentials, userId) {
    const response = await fetch(`${this.railwayUrl}/mcp`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.authToken}`,
        'Content-Type': 'application/json',
        'X-N8n-Url': userCredentials.n8nApiUrl,
        'X-N8n-Key': userCredentials.n8nApiKey,
        'X-Instance-Id': userId,
        'X-Session-Id': this.sessionId
      },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method,
        params,
        id: Date.now()
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const result = await response.json();

    if (result.error) {
      throw new Error(`MCP Error ${result.error.code}: ${result.error.message}`);
    }

    return result.result;
  }

  async initialize(userCredentials, userId) {
    return this.request('initialize', {
      protocolVersion: '2024-11-05',
      capabilities: {},
      clientInfo: { name: 'web-app', version: '1.0.0' }
    }, userCredentials, userId);
  }

  async listTools(userCredentials, userId) {
    return this.request('tools/list', {}, userCredentials, userId);
  }

  async callTool(toolName, arguments, userCredentials, userId) {
    return this.request('tools/call', {
      name: toolName,
      arguments
    }, userCredentials, userId);
  }
}

// Usage
const client = new MCPClient(
  'https://your-app-name.up.railway.app',
  'your-secure-token-here'
);

const userCredentials = {
  n8nApiUrl: 'https://user-n8n-instance.com',
  n8nApiKey: 'user-api-key-here'
};

// Initialize
await client.initialize(userCredentials, 'user-123');

// List tools
const tools = await client.listTools(userCredentials, 'user-123');
console.log(`Available tools: ${tools.tools.length}`);

// Call a tool
const result = await client.callTool(
  'search_nodes',
  { query: 'slack', limit: 10 },
  userCredentials,
  'user-123'
);
console.log(result);
```

## Python Example

```python
import requests
import json
import uuid

class MCPClient:
    def __init__(self, railway_url, auth_token):
        self.railway_url = railway_url
        self.auth_token = auth_token
        self.session_id = str(uuid.uuid4())

    def request(self, method, params, user_credentials, user_id):
        headers = {
            'Authorization': f'Bearer {self.auth_token}',
            'Content-Type': 'application/json',
            'X-N8n-Url': user_credentials['n8nApiUrl'],
            'X-N8n-Key': user_credentials['n8nApiKey'],
            'X-Instance-Id': user_id,
            'X-Session-Id': self.session_id
        }

        payload = {
            'jsonrpc': '2.0',
            'method': method,
            'params': params,
            'id': 1
        }

        response = requests.post(
            f'{self.railway_url}/mcp',
            headers=headers,
            json=payload
        )

        response.raise_for_status()
        result = response.json()

        if 'error' in result:
            raise Exception(f"MCP Error {result['error']['code']}: {result['error']['message']}")

        return result['result']

    def initialize(self, user_credentials, user_id):
        return self.request('initialize', {
            'protocolVersion': '2024-11-05',
            'capabilities': {},
            'clientInfo': {'name': 'python-client', 'version': '1.0.0'}
        }, user_credentials, user_id)

    def list_tools(self, user_credentials, user_id):
        return self.request('tools/list', {}, user_credentials, user_id)

    def call_tool(self, tool_name, arguments, user_credentials, user_id):
        return self.request('tools/call', {
            'name': tool_name,
            'arguments': arguments
        }, user_credentials, user_id)

# Usage
client = MCPClient(
    'https://your-app-name.up.railway.app',
    'your-secure-token-here'
)

user_credentials = {
    'n8nApiUrl': 'https://user-n8n-instance.com',
    'n8nApiKey': 'user-api-key-here'
}

# Initialize
client.initialize(user_credentials, 'user-123')

# List tools
tools = client.list_tools(user_credentials, 'user-123')
print(f"Available tools: {len(tools['tools'])}")

# Call a tool
result = client.call_tool(
    'search_nodes',
    {'query': 'slack', 'limit': 10},
    user_credentials,
    'user-123'
)
print(result)
```

## Notes

- All requests must use JSON-RPC 2.0 format
- The `id` field should be unique for each request (use timestamp or incrementing counter)
- Session ID should persist across requests for the same user session
- Instance ID should be unique per user
- Always include multi-tenant headers for management tools to work
- Documentation tools work without headers, but headers are recommended for consistency

## Next Steps

- See [WEB_APP_INTEGRATION.md](./WEB_APP_INTEGRATION.md) for complete web app integration
- See [RAILWAY_MULTI_TENANT.md](./RAILWAY_MULTI_TENANT.md) for Railway deployment details

