# Tool Calling and Function Calling

## Question
What are tool calling and function calling in LLMs?

## Answer
Tool calling (function calling) allows LLMs to interact with external tools, APIs, and functions to perform real-world actions and access current information beyond their training data.

## How Tool Calling Works

### Basic Flow
1. **User Query**: User asks something requiring external data
2. **LLM Analysis**: LLM determines it needs a tool
3. **Function Call**: LLM outputs structured request to call a function
4. **Execution**: System runs the function and gets results
5. **Final Response**: LLM incorporates results into natural language response

### Example: Database Query Assistant
```javascript
// Define available tools
const tools = [
  {
    type: "function",
    function: {
      name: "query_database",
      description: "Execute SQL query on company database",
      parameters: {
        type: "object",
        properties: {
          query: {
            type: "string",
            description: "SQL query to execute"
          },
          limit: {
            type: "number",
            description: "Maximum results to return"
          }
        },
        required: ["query"]
      }
    }
  }
];

// API route that handles tool calls
export async function POST(request) {
  const { messages } = await request.json();

  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages,
    tools,
    tool_choice: "auto"
  });

  const message = response.choices[0].message;

  if (message.tool_calls) {
    // Execute the tool
    const toolCall = message.tool_calls[0];
    const args = JSON.parse(toolCall.function.arguments);
    
    const result = await executeQuery(args.query, args.limit);
    
    // Get final response with tool results
    const finalResponse = await openai.chat.completions.create({
      model: "gpt-4",
      messages: [
        ...messages,
        message,
        {
          role: "tool",
          tool_call_id: toolCall.id,
          content: JSON.stringify(result)
        }
      ]
    });
    
    return Response.json({
      response: finalResponse.choices[0].message.content
    });
  }

  return Response.json({ response: message.content });
}

async function executeQuery(query, limit = 10) {
  // Simulate database query
  const results = await db.query(query).limit(limit);
  return results;
}
```

## Tool Definition Structure

### OpenAI Format
```javascript
{
  type: "function",
  function: {
    name: "tool_name",
    description: "What this tool does",
    parameters: {
      type: "object",
      properties: {
        param_name: {
          type: "string",
          description: "Parameter description"
        }
      },
      required: ["param_name"]
    }
  }
}
```

### Key Components
- **name**: Unique identifier for the tool
- **description**: What the tool does (helps LLM decide when to use it)
- **parameters**: Input schema with types and descriptions
- **required**: Which parameters are mandatory

## Common Use Cases

### Data Retrieval
- Database queries
- API calls to external services
- Search operations
- Real-time data fetching

### Actions
- Send emails
- Create calendar events
- Update records
- Trigger workflows

### Calculations
- Math operations
- Data analysis
- Unit conversions

## Interview Q&A

**Q: What's the difference between tool calling and regular LLM responses?**

**A:** Regular responses generate text directly. Tool calling allows LLMs to request external data/actions, execute them, then continue the conversation with real information.

**Q: How do you define tools for an LLM?**

**A:** Using JSON schema with name, description, and parameters object that defines input types, descriptions, and required fields.

**Q: What happens when an LLM decides to call a tool?**

**A:** LLM outputs a structured function call request instead of text. System executes the function, returns results to LLM, which then generates the final response.

**Q: When should you use tool calling?**

**A:** When you need real-time data, external API access, or actions beyond the LLM's knowledge. Examples: weather APIs, database queries, sending emails.

**Q: How do you handle tool execution errors?**

**A:** Wrap tool execution in try-catch, return error information to the LLM so it can handle gracefully in its response, and log errors for debugging.

## Tool Choice Options

### Auto (Default)
```javascript
tool_choice: "auto" // LLM decides when to use tools
```

### Forced
```javascript
tool_choice: {
  type: "function",
  function: { name: "specific_tool" } // Force specific tool
}
```

### None
```javascript
tool_choice: "none" // No tools allowed
```

### Required
```javascript
tool_choice: "required" // Must use at least one tool
```

## Best Practices
- **Clear descriptions**: Help LLM understand when to use each tool
- **Parameter validation**: Always validate inputs before execution
- **Error handling**: Graceful failure handling with user-friendly messages
- **Security**: Rate limiting, permission checks, input sanitization
- **Performance**: Caching, timeouts, resource limits

## Interview Tips
- **Explain the concept**: LLMs can call external functions to extend capabilities
- **Show examples**: Database queries, API calls, calculations
- **Discuss implementation**: Tool definitions, execution flow, error handling
- **Mention use cases**: Real-time data, external actions, complex workflows
- **Cover trade-offs**: When to use vs traditional approaches
- **Security awareness**: Input validation, rate limiting, permissions</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\11-GenAI-Agents\tool-calling-function-calling-simple.md