# Replacing AIbitat with Direct OpenAI Tool Calling

## What You Want to Replace

Currently, AnythingLLM uses the **AIbitat framework** to handle tool calling. This framework:

1. **Manages multi-agent conversations** (complex)
2. **Orchestrates tool calling** through a plugin system
3. **Handles WebSocket communication** 
4. **Provides introspection and debugging**

**Your Goal**: Replace this with **direct OpenAI API tool calling** - simpler, more direct, easier to understand.

## Current AIbitat Tool Registration

Right now, tools are registered like this:

```javascript
// In MCP/index.js and various plugins
aibitat.function({
  name: `${name}-${tool.name}`,
  description: tool.description,
  parameters: {
    $schema: "http://json-schema.org/draft-07/schema#",
    ...tool.inputSchema,
  },
  handler: async function (args = {}) {
    const result = await mcp.callTool({ name: tool.name, arguments: args });
    return typeof result === "object" ? JSON.stringify(result) : String(result);
  }
});
```

## Direct OpenAI Tool Calling

The OpenAI API supports tool calling natively. Here's how it works:

### 1. **Define Tools for OpenAI**

```javascript
// Convert MCP tools to OpenAI format
const tools = [
  {
    type: "function",
    function: {
      name: "filesystem-read_text_file",
      description: "Read the contents of a text file",
      parameters: {
        type: "object",
        properties: {
          path: {
            type: "string",
            description: "The path to the file to read"
          }
        },
        required: ["path"]
      }
    }
  }
  // ... more tools
];
```

### 2. **Send Message with Tools**

```javascript
const completion = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [
    { role: "system", content: "You are a helpful assistant with access to tools." },
    { role: "user", content: "Read the file at /home/user/document.txt" }
  ],
  tools: tools,  // Available tools
  tool_choice: "auto"  // Let OpenAI decide when to use tools
});
```

### 3. **Handle Tool Calls**

```javascript
const response = completion.choices[0].message;

if (response.tool_calls) {
  // LLM wants to call tools
  const toolResults = [];
  
  for (const toolCall of response.tool_calls) {
    const { name, arguments: args } = toolCall.function;
    
    // Execute the tool
    const result = await executeToolByName(name, JSON.parse(args));
    
    toolResults.push({
      tool_call_id: toolCall.id,
      role: "tool",
      content: result
    });
  }
  
  // Send tool results back to OpenAI for final response
  const finalCompletion = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [
      ...previousMessages,
      response,  // Assistant's message with tool calls
      ...toolResults  // Tool execution results
    ]
  });
  
  return finalCompletion.choices[0].message.content;
}
```

## What This Replaces in AnythingLLM

### Current Complex Flow
```
User Message → AIbitat → Agent Orchestration → Tool Registry → MCP Execution → Multi-turn Conversation → WebSocket Response
```

### Your Simple Flow
```
User Message → OpenAI API (with tools) → Tool Execution → OpenAI API (with results) → Direct Response
```

## Implementation Strategy

### Step 1: Create Tool Registry Adapter

```javascript
// New file: server/utils/chats/openaiToolCalling.js
const MCPCompatibilityLayer = require('../MCP');

class OpenAIToolCaller {
  constructor() {
    this.mcpLayer = new MCPCompatibilityLayer();
  }

  // Convert MCP tools to OpenAI format
  async getAvailableTools() {
    const servers = await this.mcpLayer.servers();
    const tools = [];
    
    for (const server of servers) {
      if (!server.running) continue;
      
      for (const tool of server.tools) {
        tools.push({
          type: "function",
          function: {
            name: `${server.name}-${tool.name}`,
            description: tool.description,
            parameters: tool.inputSchema
          }
        });
      }
    }
    
    return tools;
  }

  // Execute a tool by name
  async executeTool(toolName, args) {
    const [serverName, toolName_] = toolName.split('-');
    const mcp = this.mcpLayer.mcps[serverName];
    
    if (!mcp) throw new Error(`MCP server ${serverName} not found`);
    
    const result = await mcp.callTool({
      name: toolName_,
      arguments: args
    });
    
    return typeof result === "object" ? JSON.stringify(result) : String(result);
  }

  // Main tool calling flow
  async processWithTools(messages, openaiClient) {
    const tools = await this.getAvailableTools();
    
    if (tools.length === 0) {
      // No tools available, regular chat
      return await openaiClient.chat.completions.create({
        model: "gpt-4",
        messages
      });
    }

    // First call with tools
    const completion = await openaiClient.chat.completions.create({
      model: "gpt-4", 
      messages,
      tools,
      tool_choice: "auto"
    });

    const response = completion.choices[0].message;
    
    if (!response.tool_calls) {
      // No tools called, return response
      return completion;
    }

    // Execute tools
    const toolResults = [];
    for (const toolCall of response.tool_calls) {
      try {
        const result = await this.executeTool(
          toolCall.function.name,
          JSON.parse(toolCall.function.arguments)
        );
        
        toolResults.push({
          tool_call_id: toolCall.id,
          role: "tool", 
          content: result
        });
      } catch (error) {
        toolResults.push({
          tool_call_id: toolCall.id,
          role: "tool",
          content: `Error: ${error.message}`
        });
      }
    }

    // Final call with tool results
    return await openaiClient.chat.completions.create({
      model: "gpt-4",
      messages: [
        ...messages,
        response,
        ...toolResults
      ]
    });
  }
}

module.exports = { OpenAIToolCaller };
```

### Step 2: Integrate with Chat Stream

```javascript
// Modify server/utils/chats/stream.js
const { OpenAIToolCaller } = require('./openaiToolCalling');

async function streamChatWithWorkspace(response, workspace, message, chatMode, user, thread, attachments) {
  // ... existing setup ...

  // Check if workspace has tool calling enabled
  if (workspace.customAgentEnabled) {
    const toolCaller = new OpenAIToolCaller();
    const openaiClient = new OpenAI({ apiKey: process.env.OPEN_AI_KEY });
    
    const messages = [
      { role: "system", content: await chatPrompt(workspace, user) },
      ...chatHistory,
      { role: "user", content: message }
    ];

    const completion = await toolCaller.processWithTools(messages, openaiClient);
    
    writeResponseChunk(response, {
      uuid,
      type: "textResponse",
      textResponse: completion.choices[0].message.content,
      close: true,
      error: null
    });
    
    return;
  }

  // Continue with regular chat flow...
}
```

## Benefits of This Approach

### ✅ **Simpler Architecture**
- No AIbitat framework complexity
- Direct OpenAI API calls
- Easier to debug and understand

### ✅ **Better Performance**  
- Fewer layers of abstraction
- No WebSocket overhead for simple cases
- Direct streaming support

### ✅ **Easier Maintenance**
- Standard OpenAI API patterns
- Well-documented by OpenAI
- Less custom code to maintain

### ✅ **Full Tool Compatibility**
- Still uses MCP tools (no tool changes needed)
- Same error handling
- Compatible with existing tool ecosystem

## What You'll Need to Handle

### 1. **Streaming Tool Calls**
OpenAI supports streaming tool calls, but it's more complex:

```javascript
// For streaming with tools
const stream = await openai.chat.completions.create({
  model: "gpt-4",
  messages,
  tools,
  stream: true
});

for await (const chunk of stream) {
  if (chunk.choices[0]?.delta?.tool_calls) {
    // Handle streaming tool calls
  }
}
```

### 2. **Status Updates**
You'll need to manually send status updates during tool execution:

```javascript
// Send tool execution status
writeResponseChunk(response, {
  type: "statusUpdate",
  content: `Executing tool: ${toolName}...`
});

const result = await executeTool(toolName, args);

writeResponseChunk(response, {
  type: "statusUpdate", 
  content: `Tool completed: ${toolName}`
});
```

### 3. **Error Handling**
Handle tool execution errors gracefully:

```javascript
try {
  const result = await executeTool(toolName, args);
} catch (error) {
  // Send error back to OpenAI as tool result
  toolResults.push({
    tool_call_id: toolCall.id,
    role: "tool",
    content: `Error executing ${toolName}: ${error.message}`
  });
}
```

## Implementation Complexity

**LOW to MEDIUM** - Much simpler than the current AIbitat system:

- **Tool Registration**: Convert MCP tools to OpenAI format (few hours)
- **Basic Tool Calling**: Handle non-streaming tool calls (1-2 days)  
- **Streaming Support**: Add streaming with tool calls (2-3 days)
- **Status Updates**: Manual progress reporting (1 day)
- **Integration**: Wire into chat system (1 day)

**Total Estimated Time: 5-8 days**

## Getting Started

1. **Start Simple**: Implement basic non-streaming tool calling first
2. **Test with MCP Tools**: Use existing filesystem MCP server for testing
3. **Add Streaming**: Implement streaming tool calls once basic version works
4. **Add Status Updates**: Enhance UX with progress indicators
5. **Replace AIbitat**: Switch workspace tool calling to use new system

This approach gives you **direct control** over the tool calling flow while **reusing all existing MCP infrastructure**.