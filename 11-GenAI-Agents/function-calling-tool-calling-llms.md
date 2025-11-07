# Function Calling / Tool Calling in LLMs

## Question
What is function calling (tool calling) in LLMs and how does it work?

## Answer
Function calling allows LLMs to interact with external tools, APIs, and functions to perform real-world actions and access real-time information beyond their training data.

## How Function Calling Works

### Basic Flow
1. **User Query**: User asks something requiring external data
2. **LLM Decision**: LLM determines it needs a tool
3. **Function Call**: LLM outputs structured request to call a function
4. **Execution**: System runs the function and gets results
5. **Response**: LLM uses results to generate final answer

### Simple Example
```javascript
// User asks: "What's the weather in New York?"

// LLM outputs function call request
{
  "function_call": {
    "name": "get_weather",
    "arguments": {
      "location": "New York",
      "unit": "celsius"
    }
  }
}

// System executes function
const weather = await getWeather("New York", "celsius");
// Returns: { temperature: 22, condition: "sunny" }

// LLM generates final response
"The weather in New York is 22°C and sunny."
```

## OpenAI Function Calling Implementation

### Define Functions
```javascript
const functions = [
  {
    name: "get_weather",
    description: "Get current weather for a location",
    parameters: {
      type: "object",
      properties: {
        location: { type: "string", description: "City name" },
        unit: { type: "string", enum: ["celsius", "fahrenheit"] }
      },
      required: ["location"]
    }
  }
];
```

### Make API Call
```javascript
import OpenAI from 'openai';

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

async function chatWithTools(userMessage) {
  const response = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [{ role: "user", content: userMessage }],
    functions: functions,
    function_call: "auto"
  });

  const message = response.choices[0].message;

  if (message.function_call) {
    // Execute the function
    const result = await executeFunction(message.function_call);
    
    // Call LLM again with results
    const finalResponse = await openai.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "user", content: userMessage },
        message,
        { role: "function", name: message.function_call.name, content: JSON.stringify(result) }
      ]
    });
    
    return finalResponse.choices[0].message.content;
  }

  return message.content;
}
```

### Execute Function
```javascript
async function executeFunction(functionCall) {
  const { name, arguments: args } = functionCall;
  
  switch (name) {
    case "get_weather":
      return await getWeatherData(JSON.parse(args));
    default:
      throw new Error(`Unknown function: ${name}`);
  }
}
```

## Common Use Cases

### Real-Time Data
- Weather information
- Stock prices
- Current news
- Live sports scores

### External Actions
- Send emails
- Create calendar events
- Database queries
- API integrations

### Calculations
- Math operations
- Unit conversions
- Data analysis

## Interview Q&A

**Q: What's the difference between regular LLM responses and function calling?**

**A:** Regular responses generate text directly. Function calling allows LLMs to request external data/actions, get results, then continue the conversation.

**Q: When should you use function calling?**

**A:** When you need real-time data, external API access, or actions beyond the LLM's training knowledge.

**Q: How do you handle errors in function calling?**

**A:** Validate inputs, wrap function execution in try-catch, return error information to the LLM so it can handle gracefully.

**Q: What's the security concern with function calling?**

**A:** LLMs could potentially call dangerous functions. Always validate inputs, check permissions, and limit available tools.

## Key Interview Points
- **Purpose**: Extend LLM capabilities with real-world actions
- **Mechanism**: Structured JSON output → function execution → results back to LLM
- **Implementation**: Define tool schemas, handle function_call responses
- **Best Practices**: Clear descriptions, input validation, error handling
- **Use Cases**: Weather APIs, calculations, database queries, email sending</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\11-GenAI-Agents\function-calling-tool-calling-llms-simple.md