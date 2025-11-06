# What is "function calling" / "tool calling" in LLMs?

## Question
What is "function calling" / "tool calling" in LLMs?

# What is "function calling" / "tool calling" in LLMs?

## Question
What is "function calling" / "tool calling" in LLMs?

## Answer

Function calling (also known as tool calling) is a capability that allows LLMs to interact with external tools, APIs, and functions to perform real-world actions and access real-time information.

## What is Function Calling?

### Basic Concept
```javascript
const functionCalling = {
  definition: "LLM's ability to call external functions or tools",
  purpose: "Extend LLM capabilities beyond trained knowledge",
  mechanism: "LLM outputs structured data to invoke functions",
  execution: "System executes function and returns results to LLM"
};
```

### How It Works
```javascript
// 1. User asks a question
const userQuery = "What's the weather in New York?";

// 2. LLM decides it needs external data
// 3. LLM outputs function call request
const functionCall = {
  name: "get_weather",
  arguments: {
    location: "New York",
    unit: "celsius"
  }
};

// 4. System executes the function
const weatherData = await getWeather("New York", "celsius");

// 5. LLM receives results and generates response
const finalResponse = "The weather in New York is 22°C and sunny.";
```

## Function Calling vs Regular LLM Responses

### Regular LLM Response
```javascript
// LLM generates text response directly
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: "What's 2 + 2?" }]
});

// Response: { choices: [{ message: { content: "2 + 2 equals 4." } }] }
```

### Function Calling Response
```javascript
// LLM requests function execution
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: "What's the weather in Tokyo?" }],
  functions: [{
    name: "get_weather",
    description: "Get current weather for a location",
    parameters: {
      type: "object",
      properties: {
        location: { type: "string" },
        unit: { type: "string", enum: ["celsius", "fahrenheit"] }
      },
      required: ["location"]
    }
  }]
});

// Response might be:
// {
//   choices: [{
//     message: {
//       content: null,
//       function_call: {
//         name: "get_weather",
//         arguments: '{"location":"Tokyo","unit":"celsius"}'
//       }
//     }
//   }]
// }
```

## Implementing Function Calling

### OpenAI Function Calling
```javascript
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});

// Define available functions
const functions = [
  {
    name: "get_weather",
    description: "Get current weather for a location",
    parameters: {
      type: "object",
      properties: {
        location: { type: "string", description: "City name" },
        unit: { type: "string", enum: ["celsius", "fahrenheit"], default: "celsius" }
      },
      required: ["location"]
    }
  },
  {
    name: "calculate",
    description: "Perform mathematical calculations",
    parameters: {
      type: "object",
      properties: {
        expression: { type: "string", description: "Math expression" }
      },
      required: ["expression"]
    }
  }
];

async function handleUserQuery(userMessage) {
  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: userMessage }],
    functions: functions,
    function_call: "auto" // Let LLM decide when to call functions
  });

  const message = response.choices[0].message;

  // Check if LLM wants to call a function
  if (message.function_call) {
    const { name, arguments: args } = message.function_call;
    
    // Execute the function
    const result = await executeFunction(name, JSON.parse(args));
    
    // Call LLM again with function result
    const finalResponse = await openai.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "user", content: userMessage },
        message, // Function call request
        {
          role: "function",
          name: name,
          content: JSON.stringify(result)
        }
      ]
    });
    
    return finalResponse.choices[0].message.content;
  }

  // Regular response
  return message.content;
}

async function executeFunction(name, args) {
  switch (name) {
    case "get_weather":
      return await getWeatherData(args.location, args.unit);
    case "calculate":
      return calculateExpression(args.expression);
    default:
      throw new Error(`Unknown function: ${name}`);
  }
}
```

### Anthropic Tool Calling
```python
import anthropic

client = anthropic.Anthropic()

# Define tools
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }
]

response = client.messages.create(
    model="claude-3-sonnet-20240229",
    max_tokens=1000,
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather in Paris?"}]
)

# Check for tool use
if response.stop_reason == "tool_use":
    tool_use = response.content[-1]
    tool_name = tool_use.name
    tool_args = tool_use.input
    
    # Execute tool
    result = execute_tool(tool_name, tool_args)
    
    # Continue conversation with tool result
    response2 = client.messages.create(
        model="claude-3-sonnet-20240229",
        max_tokens=1000,
        tools=tools,
        messages=[
            {"role": "user", "content": "What's the weather in Paris?"},
            {"role": "assistant", "content": response.content},
            {
                "role": "user", 
                "content": [
                    {
                        "type": "tool_result",
                        "tool_use_id": tool_use.id,
                        "content": str(result)
                    }
                ]
            }
        ]
    )
```

## Common Use Cases

### 1. Real-Time Data Access
```javascript
const realTimeTools = [
  {
    name: "get_current_weather",
    description: "Get live weather data"
  },
  {
    name: "get_stock_price",
    description: "Get current stock prices"
  },
  {
    name: "search_news",
    description: "Search recent news articles"
  }
];
```

### 2. External API Integration
```javascript
const apiTools = [
  {
    name: "send_email",
    description: "Send an email"
  },
  {
    name: "create_calendar_event",
    description: "Add event to calendar"
  },
  {
    name: "query_database",
    description: "Query company database"
  }
];
```

### 3. Calculations and Processing
```javascript
const processingTools = [
  {
    name: "calculate",
    description: "Perform mathematical calculations"
  },
  {
    name: "convert_units",
    description: "Convert between units"
  },
  {
    name: "analyze_data",
    description: "Analyze datasets"
  }
];
```

## Advanced Patterns

### Multi-Tool Calling
```javascript
// LLM can call multiple tools in sequence
async function handleComplexQuery(query) {
  const tools = [weatherTool, calculatorTool, searchTool];
  
  let messages = [{ role: "user", content: query }];
  let toolCalls = [];
  
  // Allow multiple rounds of tool calling
  for (let i = 0; i < 3; i++) { // Max 3 rounds
    const response = await openai.chat.completions.create({
      model: "gpt-4",
      messages,
      tools,
      tool_choice: "auto"
    });
    
    const message = response.choices[0].message;
    messages.push(message);
    
    // Check for tool calls
    if (message.tool_calls) {
      for (const toolCall of message.tool_calls) {
        const result = await executeTool(toolCall.function);
        messages.push({
          role: "tool",
          tool_call_id: toolCall.id,
          content: JSON.stringify(result)
        });
      }
    } else {
      // No more tool calls, return final response
      return message.content;
    }
  }
}
```

### Tool Selection Strategy
```javascript
const toolSelection = {
  automatic: "LLM decides when to use tools",
  forced: "Force tool use for specific queries",
  none: "Prevent tool use for certain contexts",
  examples: {
    auto: "Let LLM choose based on query",
    forced: "Always check weather for location queries",
    none: "No tools for simple conversations"
  }
};
```

## Tool Definition Best Practices

### Clear Descriptions
```javascript
// Good: Clear, specific description
{
  name: "get_weather",
  description: "Retrieve current weather conditions and forecast for a specific location",
  parameters: {
    // Well-defined parameters
  }
}

// Bad: Vague description
{
  name: "api_call",
  description: "Call an API",
  parameters: {
    // Poorly defined
  }
}
```

### Parameter Validation
```javascript
function validateToolParameters(toolName, args) {
  const validators = {
    get_weather: (args) => {
      if (!args.location) throw new Error("Location required");
      if (!["celsius", "fahrenheit"].includes(args.unit)) {
        args.unit = "celsius"; // Default
      }
      return args;
    },
    calculate: (args) => {
      // Validate mathematical expression
      if (!/^[\d\s\+\-\*\/\(\)\.]+$/.test(args.expression)) {
        throw new Error("Invalid math expression");
      }
      return args;
    }
  };
  
  return validators[toolName]?.(args) || args;
}
```

## Error Handling

### Tool Execution Errors
```javascript
async function safeExecuteTool(toolCall) {
  try {
    const args = JSON.parse(toolCall.arguments);
    const validatedArgs = validateToolParameters(toolCall.name, args);
    
    const result = await executeTool(toolCall.name, validatedArgs);
    return result;
  } catch (error) {
    console.error(`Tool execution failed: ${toolCall.name}`, error);
    
    // Return error information to LLM
    return {
      error: true,
      message: error.message,
      tool: toolCall.name
    };
  }
}
```

### LLM Response Handling
```javascript
function handleLLMResponse(response) {
  if (response.choices[0].finish_reason === "tool_calls") {
    // Process tool calls
    return processToolCalls(response.choices[0].message.tool_calls);
  } else if (response.choices[0].finish_reason === "stop") {
    // Regular response
    return response.choices[0].message.content;
  } else {
    // Handle other finish reasons (length, etc.)
    return handleIncompleteResponse(response);
  }
}
```

## Security Considerations

### Input Validation
```javascript
const securityChecks = [
  "Validate all tool parameters",
  "Sanitize user inputs",
  "Limit tool execution time",
  "Restrict available tools per user",
  "Log all tool executions",
  "Rate limit tool usage"
];
```

### Safe Tool Execution
```javascript
// Sandbox tool execution
async function executeToolSafely(toolName, args) {
  // Check user permissions
  if (!userCanAccessTool(userId, toolName)) {
    throw new Error("Access denied");
  }
  
  // Rate limiting
  if (!checkRateLimit(userId, toolName)) {
    throw new Error("Rate limit exceeded");
  }
  
  // Execute with timeout
  const result = await Promise.race([
    executeTool(toolName, args),
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error("Tool timeout")), 30000)
    )
  ]);
  
  return result;
}
```

## Frameworks and Libraries

### LangChain Tools
```python
from langchain.tools import Tool
from langchain.utilities import SerpAPIWrapper

# Create custom tools
search_tool = Tool(
    name="Search",
    description="Search for information online",
    func=search_function
)

calculator_tool = Tool(
    name="Calculator", 
    description="Perform mathematical calculations",
    func=calculate_function
)

# Use with agents
from langchain.agents import initialize_agent
agent = initialize_agent([search_tool, calculator_tool], llm, agent="zero-shot-react-description")
```

### Vercel AI SDK
```javascript
import { openai } from '@ai-sdk/openai';
import { generateText, tool } from 'ai';

const result = await generateText({
  model: openai('gpt-4'),
  prompt: 'What is the weather in San Francisco?',
  tools: {
    getWeather: tool({
      description: 'Get the weather for a location',
      parameters: z.object({
        location: z.string().describe('The location to get weather for'),
      }),
      execute: async ({ location }) => {
        // Tool implementation
        return getWeatherFromAPI(location);
      },
    }),
  },
});
```

## Real-World Examples

### Customer Support Agent
```javascript
const supportTools = [
  {
    name: "check_order_status",
    description: "Check the status of a customer order"
  },
  {
    name: "create_support_ticket",
    description: "Create a new support ticket"
  },
  {
    name: "search_knowledge_base",
    description: "Search the knowledge base for solutions"
  }
];
```

### Code Assistant
```javascript
const codingTools = [
  {
    name: "run_tests",
    description: "Run the test suite"
  },
  {
    name: "lint_code",
    description: "Check code for linting errors"
  },
  {
    name: "search_documentation",
    description: "Search programming documentation"
  }
];
```

## Interview Tips
- **Function calling**: LLM's ability to invoke external functions/tools
- **Purpose**: Access real-time data, perform actions, extend capabilities
- **Mechanism**: LLM outputs structured function call, system executes, results fed back
- **Use cases**: Weather APIs, calculations, database queries, external integrations
- **Implementation**: Define tool schemas, handle function calls, return results
- **Security**: Validate inputs, rate limiting, access control
- **Frameworks**: LangChain, Vercel AI SDK for implementation
- **Best practices**: Clear descriptions, parameter validation, error handling