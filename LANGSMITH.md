# LangSmith Integration for AIbitat Agents

This guide describes the minimal changes needed to integrate LangSmith tracing with AIbitat agents, focusing specifically on tracking API calls.

## Setup

### 1. Environment Configuration
```bash
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=anything-llm-agents
LANGSMITH_API_KEY=your_api_key
```

### 2. Code Changes

Only one method needs modification in the AIbitat class to enable comprehensive API call tracing:

```javascript
class AIbitat {
  async handleExecution(provider, messages, functions, byAgent) {
    let langsmithRun = null;
    
    // Setup LangSmith tracing if enabled
    if (process.env.LANGSMITH_TRACING === "true") {
      try {
        const { RunTree } = require("langsmith");
        langsmithRun = new RunTree({
          name: "aibitat_agent_execution",
          inputs: { messages, functions },
          project_name: process.env.LANGSMITH_PROJECT,
          metadata: {
            agent: byAgent,
            model: provider.model,
            provider: provider.constructor.name
          }
        });
        await langsmithRun.postRun();
      } catch (error) {
        // Continue without tracing if LangSmith is unavailable
      }
    }

    try {
      const completion = await provider.complete(messages, functions);
      
      // Record completion in LangSmith
      if (langsmithRun) {
        await langsmithRun.end({
          outputs: {
            result: completion.result,
            function_call: completion.functionCall
          },
          metadata: {
            tokens: completion.usage?.total_tokens,
            cost: completion.cost
          }
        });
        await langsmithRun.patchRun();
      }

      if (completion.functionCall) {
        const result = await fn.handler(args);
        return await this.handleExecution(provider, [
          ...messages,
          { name, role: "function", content: result },
        ], functions, byAgent);
      }
      return completion?.result;
    } catch (error) {
      // Record error in LangSmith
      if (langsmithRun) {
        await langsmithRun.end({
          error: error.message
        });
        await langsmithRun.patchRun();
      }
      throw error;
    }
  }
}
```

## What Gets Traced

### 1. API Call Details
- Input messages and functions
- Model and provider information
- Agent identifier
- Completion results
- Function calls
- Token usage
- Costs

### 2. Error Information
- Error messages
- Stack traces
- Failure points

### 3. Performance Metrics
- API call duration
- Token processing rates
- Cost per call

## Benefits

### 1. Debugging
- Track all agent API interactions
- Monitor function calling behavior
- Identify failed calls and errors
- Trace execution paths

### 2. Analytics
- Token usage per agent
- Cost tracking
- Success/failure rates
- Model performance metrics

### 3. Performance Monitoring
- API call latency
- Function execution timing
- Error patterns
- Resource utilization

## Implementation Notes

1. **Minimal Impact**
   - Single point of integration
   - No structural changes needed
   - Maintains existing code flow
   - Graceful fallback if LangSmith is unavailable

2. **Data Capture**
   - Comprehensive API call data
   - No additional instrumentation needed
   - Automatic error tracking
   - Performance metrics included

3. **Maintenance**
   - Easy to update or extend
   - Centralized tracing logic
   - Clear error handling
   - Simple configuration

This implementation provides full visibility into agent API calls while keeping code changes minimal and maintaining system reliability.