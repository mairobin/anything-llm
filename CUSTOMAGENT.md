# Custom Agent Framework - Simplified Implementation

You're absolutely right! We can leverage the existing chat infrastructure much more directly.

## The Existing Flow Already Works

Looking at `server/utils/chats/stream.js`, the flow is:

1. **Message comes in** → `streamChatWithWorkspace()`
2. **Check for agents** → `grepAgents()` in line 42-50
3. **If agent found** → Route to agent system (websocket handoff)
4. **Otherwise** → Continue with regular chat

## The Simple Solution

Instead of building complex wrappers, we can **modify the existing agent detection** to automatically route when tools are enabled:

### Current Agent Detection (`server/utils/chats/agents.js`)

```javascript
// Current logic - only detects @agent syntax
const agentHandles = WorkspaceAgentInvocation.parseAgents(message);
if (agentHandles.length > 0) {
  // Route to agent system...
}
```

### Enhanced Agent Detection (Proposed)

```javascript
async function grepAgents({ uuid, response, message, workspace, user = null, thread = null }) {
  // Existing @agent detection
  const agentHandles = WorkspaceAgentInvocation.parseAgents(message);
  
  // NEW: Auto-route to agents if workspace has tools enabled
  const shouldUseAgents = agentHandles.length > 0 || 
    (workspace.customAgentEnabled && await hasAvailableTools(workspace));
  
  if (shouldUseAgents) {
    // Use existing agent invocation flow - it already works perfectly!
    const { invocation: newInvocation } = await WorkspaceAgentInvocation.new({
      prompt: message,
      workspace: workspace,
      user: user,
      thread: thread,
    });
    
    // Existing websocket handoff logic continues unchanged...
    writeResponseChunk(response, {
      id: uuid,
      type: "agentInitWebsocketConnection",
      websocketUUID: newInvocation.uuid,
      // ... rest of existing logic
    });
    
    return true;
  }
  
  return false;
}

async function hasAvailableTools(workspace) {
  const mcpLayer = new MCPCompatibilityLayer();
  const servers = await mcpLayer.servers();
  return servers.some(s => s.running && s.tools.length > 0);
}
```

## Why This Approach is Brilliant

### ✅ **Zero Infrastructure Changes**
- Agent system works as-is
- WebSocket handoff works as-is  
- Frontend components work as-is
- Tool calling works as-is

### ✅ **Minimal Code Changes**
- Modify 1 function: `grepAgents()`
- Add 1 helper: `hasAvailableTools()`
- Add 1 database field: `workspace.customAgentEnabled`

### ✅ **Perfect User Experience**
- No `@agent` syntax required when tools enabled
- Same chat interface
- Same tool visualization
- Seamless tool calling

### ✅ **Backward Compatibility**
- `@agent` syntax still works
- Regular chat still works
- Progressive enhancement

## Implementation Steps

### 1. Database Migration
```sql
ALTER TABLE workspaces ADD COLUMN customAgentEnabled BOOLEAN DEFAULT FALSE;
```

### 2. Modify Agent Detection
```javascript
// In server/utils/chats/agents.js
const MCPCompatibilityLayer = require('../MCP');

async function grepAgents({ uuid, response, message, workspace, user = null, thread = null }) {
  const agentHandles = WorkspaceAgentInvocation.parseAgents(message);
  
  // Check if we should auto-route to agents
  const hasToolsEnabled = workspace.customAgentEnabled && await hasAvailableTools(workspace);
  const shouldUseAgents = agentHandles.length > 0 || hasToolsEnabled;
  
  if (shouldUseAgents) {
    // Rest of existing logic unchanged...
  }
  
  return shouldUseAgents;
}

async function hasAvailableTools(workspace) {
  const mcpLayer = new MCPCompatibilityLayer();
  const servers = await mcpLayer.servers();
  return servers.some(s => s.running && s.tools.length > 0);
}
```

### 3. Frontend Toggle
```javascript
// In workspace settings
<div className="custom-agent-setting">
  <label>
    <input 
      type="checkbox" 
      checked={workspace.customAgentEnabled}
      onChange={(e) => updateWorkspace({customAgentEnabled: e.target.checked})}
    />
    Enable Automatic Tool Calling
  </label>
  <p className="help-text">
    When enabled, chat messages will automatically access available tools without needing @agent syntax.
  </p>
</div>
```

## The Result

With just these small changes:

- **User types normal message** → "Read the file at /home/user/file.txt"
- **System detects tools enabled** → Auto-routes to agent system
- **Tools execute seamlessly** → MCP filesystem tool runs
- **User sees tool execution** → Same beautiful UI as agent mode
- **Final response delivered** → "Here's the content of the file..."

**No `@agent` required. No complex wrappers. Just intelligent routing.**

## Estimated Implementation Time

| Task | Duration | 
|------|----------|
| Modify `grepAgents()` | 2 hours |
| Add `hasAvailableTools()` helper | 1 hour |
| Database migration | 30 minutes |
| Frontend toggle | 2-3 hours |
| Testing | 2-3 hours |

**Total: 1 day of work**

## Why This is the Right Approach

1. **Leverages existing architecture** instead of fighting it
2. **Reuses 100% of agent infrastructure** 
3. **Requires minimal changes** to proven code
4. **Provides seamless UX** without syntax requirements
5. **Maintains all existing functionality**

This is exactly the kind of elegant solution that makes the most of what's already built while providing the enhanced functionality you want.