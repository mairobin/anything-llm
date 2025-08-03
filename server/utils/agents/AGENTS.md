# Tool Calling Implementation Guide

This document explains how the agent system implements tool calling and how to adapt it for regular chat mode.

## Tool Calling Architecture Overview

The agents system uses the **AIbitat** framework to orchestrate tool calls between LLMs and external tools/services. This same architecture can be leveraged in chat mode to provide tool calling capabilities without full agent complexity.

## Tool Calling Flow

### 1. Tool Registration (`index.js:381-507`)

Tools are registered in the `#attachPlugins()` method:

```javascript
// MCP tools are loaded with @@mcp_ prefix
if (name.startsWith("@@mcp_")) {
  const plugins = await new MCPCompatibilityLayer().convertServerToolsToPlugins(mcpPluginName, this.aibitat);
  plugins.forEach((plugin) => {
    this.aibitat.use(plugin.plugin()); // Register tool with AIbitat
  });
}
```

### 2. Tool Definition Structure

Each tool follows this pattern from `MCPCompatibilityLayer.convertServerToolsToPlugins()`:

```javascript
{
  name: `${serverName}-${toolName}`,
  description: tool.description,
  parameters: {
    $schema: "http://json-schema.org/draft-07/schema#",
    ...tool.inputSchema
  },
  handler: async function (args = {}) {
    // Tool execution logic
    const result = await mcp.callTool({ name: tool.name, arguments: args });
    return typeof result === "object" ? JSON.stringify(result) : String(result);
  }
}
```

### 3. LLM Provider Integration

The AIbitat framework handles tool calling through provider-specific implementations:

- **Function Registration**: Tools are registered as available functions
- **Provider Communication**: LLM receives tool definitions in its system prompt
- **Call Detection**: Provider detects when LLM wants to call a function
- **Execution**: Handler executes the tool and returns results
- **Response Integration**: Results are fed back to LLM for final response

## Frontend Rendering (`aibitat/plugins/websocket.js`)

The websocket plugin handles real-time communication and tool call visualization:

### Tool Call Display Format

```javascript
// Tool execution start
{
  type: "statusUpdate", 
  content: "Executing MCP server: filesystem with {...}",
  from: "@agent"
}

// Tool execution result  
{
  type: "statusUpdate",
  content: "MCP server: filesystem:read_text_file completed successfully", 
  from: "@agent"
}

// Final response with results
{
  type: "textResponse",
  content: "Here's the content of the file...",
  from: "@agent"
}
```

### Introspection System

The `aibitat.introspect()` method provides real-time tool execution feedback:

```javascript
aibitat.introspect(`Executing MCP server: ${name} with ${JSON.stringify(args, null, 2)}`);
// Renders as: "Executing MCP server: filesystem with { "path": "/home/user/file.txt" }"

aibitat.introspect(`MCP server: ${name}:${tool.name} completed successfully`);
// Renders as: "MCP server: filesystem:read_text_file completed successfully"
```

## Adapting for Chat Mode

### Option 1: Route Through Agent System

```javascript
// In streamChatWithWorkspace()
async function streamChatWithWorkspace(response, workspace, message, chatMode, user, thread, attachments) {
  const mcpLayer = new MCPCompatibilityLayer();
  const hasTools = (await mcpLayer.servers()).some(s => s.running && s.tools.length > 0);
  
  if (hasTools && workspace.enableToolCalling) {
    // Use agent system for tool-enabled chat
    return await routeThroughAgentSystem(response, workspace, message, user, thread, attachments);
  }
  
  // Continue with regular chat flow
}

async function routeThroughAgentSystem(response, workspace, message, user, thread, attachments) {
  const uuid = uuidv4();
  const handler = new AgentHandler({ uuid });
  
  // Create lightweight agent invocation record
  const invocation = await WorkspaceAgentInvocation.create({
    uuid,
    workspace_id: workspace.id,
    user_id: user?.id,
    thread_id: thread?.id,
    prompt: message
  });
  
  handler.invocation = invocation;
  handler.provider = workspace.chatProvider;
  handler.model = workspace.chatModel;
  
  await handler.createAIbitat({ socket: mockSocketForResponse(response) });
  return await handler.startAgentCluster();
}
```

### Option 2: Mock Socket Adapter

Create an adapter to convert HTTP response streaming to websocket-like interface:

```javascript
function mockSocketForResponse(httpResponse) {
  return {
    emit: (event, data) => {
      if (event === 'message') {
        writeResponseChunk(httpResponse, {
          type: data.type || 'textResponse',
          content: data.content,
          from: data.from,
          uuid: data.uuid,
          sources: data.sources || [],
          close: data.close || false
        });
      }
    }
  };
}
```

### Frontend Reuse Strategy

The existing agent frontend components can be reused by:

1. **Same Message Types**: Use identical message format (`statusUpdate`, `textResponse`)
2. **Same WebSocket Events**: Route through same event handlers
3. **Same UI Components**: Reuse tool call rendering components
4. **Conditional Routing**: Add workspace setting to enable tool calling in chat

### Key Integration Points

1. **Tool Detection** (`utils/chats/stream.js:42-50`): Check for MCP servers before routing to agents
2. **Response Streaming**: Use existing `writeResponseChunk` format  
3. **Frontend Components**: Reuse agent message rendering for tool calls
4. **Configuration**: Add `workspace.enableToolCalling` setting

### Benefits of This Approach

- ✅ **Zero code duplication** for tool calling logic
- ✅ **Same frontend rendering** for consistent UX  
- ✅ **All MCP tools work** immediately in chat mode
- ✅ **Same error handling** and retry logic
- ✅ **Same streaming performance** 
- ✅ **Minimal implementation effort**

## Implementation Steps

### 1. Add Workspace Configuration

Add tool calling toggle to workspace settings:

```sql
ALTER TABLE workspaces ADD COLUMN enableToolCalling BOOLEAN DEFAULT FALSE;
```

### 2. Modify Chat Stream Handler

```javascript
// In server/utils/chats/stream.js
async function streamChatWithWorkspace(response, workspace, message, chatMode, user, thread, attachments) {
  // Check if workspace has tool calling enabled and MCP servers available
  if (workspace.enableToolCalling) {
    const mcpLayer = new MCPCompatibilityLayer();
    const activeServers = await mcpLayer.servers();
    const hasActiveTools = activeServers.some(s => s.running && s.tools.length > 0);
    
    if (hasActiveTools) {
      return await routeThroughAgentSystem(response, workspace, message, user, thread, attachments);
    }
  }
  
  // Continue with existing chat flow...
}
```

### 3. Frontend Settings Integration

Add tool calling toggle in workspace settings:

```javascript
// In frontend workspace settings
<div className="tool-calling-setting">
  <label>
    <input 
      type="checkbox" 
      checked={workspace.enableToolCalling}
      onChange={(e) => updateWorkspace({enableToolCalling: e.target.checked})}
    />
    Enable Tool Calling in Chat
  </label>
</div>
```

### 4. Message Type Compatibility

Ensure frontend handles both chat and agent message types:

```javascript
// In chat message renderer
const renderMessage = (message) => {
  switch (message.type) {
    case 'textResponse':
      return <TextMessage content={message.content} />;
    case 'statusUpdate': 
      return <ToolCallStatus content={message.content} />;
    // ... other agent message types
  }
};
```

## Result: Unified Tool Calling

With this approach:

- **Chat mode with tools** = Agent system without multi-agent complexity
- **Same UI components** render tool calls in both modes  
- **Same MCP integration** works across chat and agent modes
- **Progressive enhancement** - chat works with/without tools
- **Zero duplication** of tool calling logic

The user gets the exact same tool calling experience and rendering whether they're in chat mode or agent mode, just with different conversation patterns.
