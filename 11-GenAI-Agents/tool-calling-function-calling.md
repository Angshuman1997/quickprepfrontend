# What are tool calling and function calling?

## Question
What are tool calling and function calling?

# What are tool calling and function calling?

## Question
What are tool calling and function calling?

## Answer

Tool calling (also known as function calling) allows LLMs to interact with external tools, APIs, and functions to extend their capabilities beyond just text generation. This enables more sophisticated AI applications that can perform real-world actions.

## What is Function Calling?

Function calling is a feature where LLMs can:
1. **Understand** when they need external information or actions
2. **Decide** which function to call and with what parameters
3. **Execute** the function and incorporate results into their response

### Basic Concept
```javascript
// Instead of just generating text, the LLM can call functions
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: "What's the weather in Tokyo?" }],
  functions: [weatherFunctionDefinition],
  function_call: "auto" // Let the model decide when to call functions
});

// The model might respond with a function call instead of text
{
  "function_call": {
    "name": "get_weather",
    "arguments": "{\"location\":\"Tokyo\",\"unit\":\"celsius\"}"
  }
}
```

## Function Definitions

### OpenAI Function Definition
```javascript
const functions = [
  {
    name: "get_weather",
    description: "Get current weather for a location",
    parameters: {
      type: "object",
      properties: {
        location: {
          type: "string",
          description: "City name or coordinates"
        },
        unit: {
          type: "string",
          enum: ["celsius", "fahrenheit"],
          description: "Temperature unit"
        }
      },
      required: ["location"]
    }
  },
  {
    name: "search_database",
    description: "Search company database for information",
    parameters: {
      type: "object",
      properties: {
        query: {
          type: "string",
          description: "Search query"
        },
        limit: {
          type: "number",
          description: "Maximum results to return"
        }
      },
      required: ["query"]
    }
  }
];
```

### Anthropic Tool Definition
```javascript
const tools = [
  {
    name: "get_weather",
    description: "Get current weather for a location",
    input_schema: {
      type: "object",
      properties: {
        location: {
          type: "string",
          description: "City name or coordinates"
        },
        unit: {
          type: "string",
          enum: ["celsius", "fahrenheit"],
          description: "Temperature unit"
        }
      },
      required: ["location"]
    }
  }
];
```

## Complete Implementation Example

### Weather Assistant
```javascript
// server/api/chat/route.js
import OpenAI from 'openai';

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

const tools = [
  {
    type: "function",
    function: {
      name: "get_weather",
      description: "Get current weather for a location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "City name or location"
          },
          unit: {
            type: "string",
            enum: ["celsius", "fahrenheit"],
            description: "Temperature unit"
          }
        },
        required: ["location"]
      }
    }
  },
  {
    type: "function",
    function: {
      name: "get_forecast",
      description: "Get weather forecast for a location",
      parameters: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "City name or location"
          },
          days: {
            type: "number",
            description: "Number of days for forecast (1-7)"
          }
        },
        required: ["location"]
      }
    }
  }
];

export async function POST(request) {
  const { messages } = await request.json();

  try {
    const response = await openai.chat.completions.create({
      model: "gpt-4",
      messages,
      tools,
      tool_choice: "auto" // Let model decide when to use tools
    });

    const message = response.choices[0].message;

    // Check if the model wants to call a function
    if (message.tool_calls) {
      const toolCalls = message.tool_calls;
      
      // Execute the functions
      const toolResults = await Promise.all(
        toolCalls.map(async (toolCall) => {
          const result = await executeTool(toolCall.function.name, 
                                         JSON.parse(toolCall.function.arguments));
          return {
            tool_call_id: toolCall.id,
            role: "tool",
            content: JSON.stringify(result)
          };
        })
      );

      // Get final response with tool results
      const finalResponse = await openai.chat.completions.create({
        model: "gpt-4",
        messages: [
          ...messages,
          message,
          ...toolResults
        ]
      });

      return Response.json({
        response: finalResponse.choices[0].message.content
      });
    }

    // No function calls, return direct response
    return Response.json({
      response: message.content
    });

  } catch (error) {
    return Response.json(
      { error: 'Failed to process request' },
      { status: 500 }
    );
  }
}

// Execute the actual tool functions
async function executeTool(functionName, args) {
  switch (functionName) {
    case 'get_weather':
      return await getWeatherData(args.location, args.unit || 'celsius');
    case 'get_forecast':
      return await getForecastData(args.location, args.days || 3);
    default:
      throw new Error(`Unknown function: ${functionName}`);
  }
}

async function getWeatherData(location, unit) {
  // Simulate API call to weather service
  const response = await fetch(
    `https://api.weatherapi.com/v1/current.json?key=${process.env.WEATHER_API_KEY}&q=${location}`
  );
  const data = await response.json();
  
  return {
    location: data.location.name,
    temperature: unit === 'celsius' ? data.current.temp_c : data.current.temp_f,
    condition: data.current.condition.text,
    unit
  };
}

async function getForecastData(location, days) {
  const response = await fetch(
    `https://api.weatherapi.com/v1/forecast.json?key=${process.env.WEATHER_API_KEY}&q=${location}&days=${days}`
  );
  const data = await response.json();
  
  return {
    location: data.location.name,
    forecast: data.forecast.forecastday.map(day => ({
      date: day.date,
      max_temp: day.day.maxtemp_c,
      min_temp: day.day.mintemp_c,
      condition: day.day.condition.text
    }))
  };
}
```

### React Frontend
```jsx
// components/WeatherAssistant.js
import { useState } from 'react';

export default function WeatherAssistant() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const sendMessage = async () => {
    if (!input.trim()) return;

    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setIsLoading(true);

    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          messages: [...messages, userMessage]
        })
      });

      const data = await response.json();
      
      setMessages(prev => [...prev, {
        role: 'assistant',
        content: data.response
      }]);
      
    } catch (error) {
      setMessages(prev => [...prev, {
        role: 'assistant',
        content: 'Sorry, I encountered an error.'
      }]);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="weather-assistant">
      <div className="chat-messages">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message ${msg.role}`}>
            {msg.content}
          </div>
        ))}
        {isLoading && (
          <div className="message assistant loading">
            Thinking...
          </div>
        )}
      </div>
      
      <div className="input-area">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
          placeholder="Ask about weather (e.g., 'What's the weather in Tokyo?')"
          disabled={isLoading}
        />
        <button onClick={sendMessage} disabled={isLoading}>
          Send
        </button>
      </div>
    </div>
  );
}
```

## Advanced Tool Calling Patterns

### 1. Multiple Tool Calls
```javascript
// Handle multiple tool calls in parallel
if (message.tool_calls && message.tool_calls.length > 1) {
  const toolResults = await Promise.all(
    message.tool_calls.map(async (toolCall) => {
      const args = JSON.parse(toolCall.function.arguments);
      const result = await executeTool(toolCall.function.name, args);
      
      return {
        tool_call_id: toolCall.id,
        role: "tool",
        content: JSON.stringify(result)
      };
    })
  );
  
  // Continue conversation with all results
  const finalResponse = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [
      ...messages,
      message,
      ...toolResults
    ]
  });
}
```

### 2. Tool Choice Control
```javascript
// Force specific tool usage
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages,
  tools,
  tool_choice: {
    type: "function",
    function: { name: "get_weather" } // Force weather function
  }
});

// Let model decide
tool_choice: "auto"

// No tools allowed
tool_choice: "none"

// Required tool usage
tool_choice: "required"
```

### 3. Error Handling in Tools
```javascript
async function executeTool(functionName, args) {
  try {
    switch (functionName) {
      case 'get_weather':
        return await getWeatherData(args.location, args.unit);
      case 'search_database':
        return await searchDatabase(args.query, args.limit);
      default:
        return { error: `Unknown function: ${functionName}` };
    }
  } catch (error) {
    console.error(`Tool execution error for ${functionName}:`, error);
    return { 
      error: `Failed to execute ${functionName}: ${error.message}`,
      function: functionName,
      args: args
    };
  }
}
```

## Real-World Use Cases

### 1. E-commerce Assistant
```javascript
const eCommerceTools = [
  {
    name: "search_products",
    description: "Search for products in catalog",
    parameters: {
      type: "object",
      properties: {
        query: { type: "string", description: "Search query" },
        category: { type: "string", description: "Product category" },
        price_range: { 
          type: "object",
          properties: {
            min: { type: "number" },
            max: { type: "number" }
          }
        }
      }
    }
  },
  {
    name: "check_inventory",
    description: "Check product availability",
    parameters: {
      type: "object",
      properties: {
        product_id: { type: "string", description: "Product ID" },
        location: { type: "string", description: "Store location" }
      },
      required: ["product_id"]
    }
  },
  {
    name: "place_order",
    description: "Place an order for products",
    parameters: {
      type: "object",
      properties: {
        items: {
          type: "array",
          items: {
            type: "object",
            properties: {
              product_id: { type: "string" },
              quantity: { type: "number" }
            }
          }
        },
        customer_info: {
          type: "object",
          properties: {
            name: { type: "string" },
            email: { type: "string" },
            address: { type: "string" }
          }
        }
      },
      required: ["items", "customer_info"]
    }
  }
];
```

### 2. Code Assistant
```javascript
const codingTools = [
  {
    name: "run_code",
    description: "Execute code and return results",
    parameters: {
      type: "object",
      properties: {
        code: { type: "string", description: "Code to execute" },
        language: { type: "string", enum: ["python", "javascript", "java"] }
      },
      required: ["code", "language"]
    }
  },
  {
    name: "search_documentation",
    description: "Search programming documentation",
    parameters: {
      type: "object",
      properties: {
        query: { type: "string", description: "Search query" },
        language: { type: "string", description: "Programming language" }
      },
      required: ["query"]
    }
  },
  {
    name: "analyze_code",
    description: "Analyze code for bugs and improvements",
    parameters: {
      type: "object",
      properties: {
        code: { type: "string", description: "Code to analyze" },
        language: { type: "string", description: "Programming language" }
      },
      required: ["code"]
    }
  }
];
```

### 3. Database Assistant
```javascript
const databaseTools = [
  {
    name: "execute_query",
    description: "Execute SQL query on database",
    parameters: {
      type: "object",
      properties: {
        query: { type: "string", description: "SQL query to execute" },
        database: { type: "string", description: "Database name" }
      },
      required: ["query"]
    }
  },
  {
    name: "get_table_schema",
    description: "Get schema information for a table",
    parameters: {
      type: "object",
      properties: {
        table_name: { type: "string", description: "Table name" },
        database: { type: "string", description: "Database name" }
      },
      required: ["table_name"]
    }
  },
  {
    name: "analyze_performance",
    description: "Analyze query performance",
    parameters: {
      type: "object",
      properties: {
        query: { type: "string", description: "SQL query to analyze" }
      },
      required: ["query"]
    }
  }
];
```

## Best Practices

### 1. Function Design
```javascript
const bestPractices = [
  "Use clear, descriptive function names",
  "Provide detailed descriptions for parameters",
  "Include examples in descriptions when helpful",
  "Use appropriate parameter types and constraints",
  "Handle optional parameters gracefully",
  "Return structured, consistent response formats",
  "Include error handling in function implementations",
  "Document expected response schemas"
];
```

### 2. Error Handling
```javascript
// Comprehensive error handling
async function safeToolExecution(functionName, args) {
  try {
    // Validate arguments
    const validation = validateArguments(functionName, args);
    if (!validation.valid) {
      return { error: validation.message };
    }

    // Check permissions
    const permission = await checkPermissions(functionName, args);
    if (!permission.allowed) {
      return { error: "Insufficient permissions" };
    }

    // Execute with timeout
    const result = await Promise.race([
      executeTool(functionName, args),
      new Promise((_, reject) => 
        setTimeout(() => reject(new Error('Timeout')), 30000)
      )
    ]);

    return result;
    
  } catch (error) {
    console.error(`Tool execution failed: ${functionName}`, error);
    return { 
      error: `Function execution failed: ${error.message}`,
      function: functionName 
    };
  }
}
```

### 3. Security Considerations
```javascript
const securityChecks = [
  "Validate all input parameters",
  "Implement rate limiting per user/function",
  "Check user permissions before execution",
  "Sanitize database queries to prevent injection",
  "Limit resource usage (CPU, memory, API calls)",
  "Log all function executions for audit",
  "Implement circuit breakers for failing services",
  "Use environment-specific configurations"
];
```

### 4. Performance Optimization
```javascript
// Cache frequently used results
const toolCache = new Map();

async function cachedToolExecution(functionName, args) {
  const cacheKey = `${functionName}:${JSON.stringify(args)}`;
  
  if (toolCache.has(cacheKey)) {
    return toolCache.get(cacheKey);
  }
  
  const result = await executeTool(functionName, args);
  
  // Cache for 5 minutes
  toolCache.set(cacheKey, result);
  setTimeout(() => toolCache.delete(cacheKey), 5 * 60 * 1000);
  
  return result;
}

// Parallel execution for independent tools
async function executeParallelTools(toolCalls) {
  const results = await Promise.allSettled(
    toolCalls.map(async (toolCall) => {
      const args = JSON.parse(toolCall.function.arguments);
      const result = await executeTool(toolCall.function.name, args);
      return { toolCall, result };
    })
  );
  
  return results.map((result, index) => ({
    tool_call_id: toolCalls[index].id,
    role: "tool",
    content: result.status === 'fulfilled' 
      ? JSON.stringify(result.value) 
      : JSON.stringify({ error: result.reason.message })
  }));
}
```

## Comparison: Tool Calling vs Traditional Approaches

| Aspect | Tool Calling | Traditional API Integration |
|--------|-------------|-----------------------------|
| Flexibility | High - LLM decides when/which tools | Low - Predefined workflows |
| Maintenance | Lower - Declarative definitions | Higher - Code changes needed |
| User Experience | Natural conversation flow | Structured forms/interfaces |
| Error Handling | Built-in retry/error recovery | Manual implementation |
| Scalability | Good - Easy to add new tools | Moderate - Architecture changes |
| Cost | Variable - Per tool execution | Predictable - Fixed API calls |

## Interview Tips
- **Explain the concept**: Break down how LLMs can call external functions
- **Show practical examples**: Weather assistant, e-commerce bot, code assistant
- **Discuss trade-offs**: When to use tool calling vs traditional approaches
- **Cover implementation**: Function definitions, parameter validation, error handling
- **Mention security**: Input validation, rate limiting, permissions
- **Real-world applications**: Share specific projects where you've implemented this
- **Future directions**: Multi-agent systems, autonomous AI agents