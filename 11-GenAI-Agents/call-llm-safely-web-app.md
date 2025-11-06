# How do you call an LLM safely from a web app?

## Question
How do you call an LLM safely from a web app?

# How do you call an LLM safely from a web app?

## Question
How do you call an LLM safely from a web app?

## Answer

Calling LLMs safely from web applications requires addressing security, performance, cost, and user experience concerns. The key is to make API calls server-side, implement proper safeguards, and handle errors gracefully.

## Security First: Server-Side API Calls

### Never Expose API Keys
```javascript
// ❌ DANGEROUS: Client-side API call
const callLLM = async () => {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`, // Exposed!
    }
  });
};

// ✅ SECURE: Server-side API call
// pages/api/chat.js (Next.js)
export default async function handler(req, res) {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`, // Secure
    }
  });
  
  const data = await response.json();
  res.status(200).json(data);
}
```

### Environment Variables
```bash
# .env.local (never commit to git)
OPENAI_API_KEY=sk-your-secret-key
ANTHROPIC_API_KEY=sk-ant-your-key
```

```javascript
// Use in server-side code only
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

## Input Validation and Sanitization

### Validate User Input
```javascript
// middleware/validation.js
import validator from 'validator';

export function validateLLMInput(req, res, next) {
  const { messages } = req.body;
  
  // Check if messages exist and is array
  if (!messages || !Array.isArray(messages)) {
    return res.status(400).json({ error: 'Messages must be an array' });
  }
  
  // Validate message structure
  for (const message of messages) {
    if (!message.role || !message.content) {
      return res.status(400).json({ error: 'Invalid message format' });
    }
    
    if (!['user', 'assistant', 'system'].includes(message.role)) {
      return res.status(400).json({ error: 'Invalid message role' });
    }
    
    // Sanitize content to prevent injection
    message.content = validator.escape(message.content);
    
    // Check content length
    if (message.content.length > 10000) {
      return res.status(400).json({ error: 'Message too long' });
    }
  }
  
  // Check total messages
  if (messages.length > 50) {
    return res.status(400).json({ error: 'Too many messages' });
  }
  
  next();
}
```

### Content Filtering
```javascript
// Check for harmful content
const harmfulPatterns = [
  /system.*prompt/i,
  /ignore.*previous.*instructions/i,
  /override.*safety/i,
  // Add more patterns as needed
];

function containsHarmfulContent(text) {
  return harmfulPatterns.some(pattern => pattern.test(text));
}

// Use in validation
if (containsHarmfulContent(message.content)) {
  return res.status(400).json({ error: 'Invalid content' });
}
```

## Rate Limiting and Abuse Prevention

### User-Based Rate Limiting
```javascript
// middleware/rateLimit.js
import rateLimit from 'express-rate-limit';

export const llmRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 50, // 50 requests per window
  message: {
    error: 'Too many requests. Please wait before trying again.'
  },
  standardHeaders: true,
  legacyHeaders: false,
  // Use user ID for rate limiting if authenticated
  keyGenerator: (req) => req.user?.id || req.ip,
});
```

### Cost-Based Limiting
```javascript
// Track API usage per user
const userUsage = new Map();

function checkUsageLimit(userId, tokensUsed) {
  const currentUsage = userUsage.get(userId) || 0;
  const newUsage = currentUsage + tokensUsed;
  
  // Set daily limit (adjust based on your needs)
  const dailyLimit = 100000; // tokens
  
  if (newUsage > dailyLimit) {
    throw new Error('Daily usage limit exceeded');
  }
  
  userUsage.set(userId, newUsage);
  
  // Reset usage daily
  setTimeout(() => userUsage.delete(userId), 24 * 60 * 60 * 1000);
}
```

## Error Handling and Resilience

### Comprehensive Error Handling
```javascript
// api/llm.js
export async function callLLMSafely(prompt, options = {}) {
  try {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      },
      body: JSON.stringify({
        model: options.model || 'gpt-4',
        messages: [{ role: 'user', content: prompt }],
        max_tokens: options.maxTokens || 1000,
        temperature: options.temperature || 0.7,
      }),
      // Set reasonable timeout
      signal: AbortSignal.timeout(30000), // 30 seconds
    });

    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }

    const data = await response.json();
    return data.choices[0].message.content;
    
  } catch (error) {
    // Log error for monitoring
    console.error('LLM API error:', error);
    
    // Handle different error types
    if (error.name === 'AbortError') {
      throw new Error('Request timed out');
    }
    
    if (error.message.includes('429')) {
      throw new Error('Rate limit exceeded. Please try again later.');
    }
    
    if (error.message.includes('401')) {
      throw new Error('Authentication failed');
    }
    
    // Generic error
    throw new Error('AI service temporarily unavailable');
  }
}
```

### Fallback Strategies
```javascript
// Implement fallback to simpler model or cached responses
export async function callLLMWithFallback(prompt) {
  try {
    // Try primary model
    return await callLLMSafely(prompt, { model: 'gpt-4' });
  } catch (error) {
    console.warn('Primary LLM failed, trying fallback:', error.message);
    
    try {
      // Fallback to simpler model
      return await callLLMSafely(prompt, { model: 'gpt-3.5-turbo' });
    } catch (fallbackError) {
      console.error('Fallback LLM also failed:', fallbackError.message);
      
      // Return cached or default response
      return getCachedResponse(prompt) || 'I apologize, but I\'m unable to assist right now.';
    }
  }
}
```

## Authentication and Authorization

### User Authentication
```javascript
// middleware/auth.js
import { getSession } from 'next-auth/react';

export async function requireAuth(req, res) {
  const session = await getSession({ req });
  
  if (!session) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  
  req.user = session.user;
  return null; // Continue to next middleware
}

// In API route
export default async function handler(req, res) {
  const authError = await requireAuth(req, res);
  if (authError) return;
  
  // Proceed with LLM call
}
```

### Feature Access Control
```javascript
// Check if user has access to LLM features
function checkLLMAccess(user) {
  // Check subscription tier
  if (user.tier === 'free') {
    // Free users get limited access
    return { allowed: true, limits: { requestsPerDay: 10 } };
  }
  
  if (user.tier === 'premium') {
    return { allowed: true, limits: { requestsPerDay: 1000 } };
  }
  
  return { allowed: false, reason: 'Upgrade required for AI features' };
}
```

## Monitoring and Logging

### API Usage Tracking
```javascript
// middleware/monitoring.js
export function trackLLMUsage(req, res, next) {
  const startTime = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    
    // Log usage for monitoring
    console.log(`LLM Call: ${req.method} ${req.url} - ${res.statusCode} - ${duration}ms`);
    
    // Send to monitoring service
    if (process.env.NODE_ENV === 'production') {
      // monitor.track('llm_api_call', {
      //   userId: req.user?.id,
      //   endpoint: req.url,
      //   status: res.statusCode,
      //   duration,
      //   tokens: res.locals.tokensUsed
      // });
    }
  });
  
  next();
}
```

### Cost Monitoring
```javascript
// Track API costs
function calculateCost(model, tokens) {
  const costs = {
    'gpt-4': { input: 0.03, output: 0.06 }, // per 1K tokens
    'gpt-3.5-turbo': { input: 0.0015, output: 0.002 },
  };
  
  const modelCost = costs[model];
  if (!modelCost) return 0;
  
  // Simplified calculation
  return (tokens * modelCost.output) / 1000;
}

// Alert on high usage
function checkCostAlert(cost, userId) {
  const dailyLimit = 10; // $10 per day
  
  if (cost > dailyLimit) {
    console.warn(`High LLM cost for user ${userId}: $${cost}`);
    // Send alert notification
  }
}
```

## Content Safety and Moderation

### Input Moderation
```javascript
// Check input for inappropriate content
async function moderateInput(text) {
  // Use OpenAI's moderation API
  const response = await fetch('https://api.openai.com/v1/moderations', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ input: text }),
  });
  
  const result = await response.json();
  return result.results[0].flagged;
}

// Use in API route
if (await moderateInput(userInput)) {
  return res.status(400).json({ error: 'Content violates usage policy' });
}
```

### Output Filtering
```javascript
// Filter potentially harmful outputs
function filterOutput(text) {
  const harmfulPatterns = [
    /how to.*(hack|exploit|damage)/i,
    /instructions for.*illegal/i,
    // Add more patterns
  ];
  
  if (harmfulPatterns.some(pattern => pattern.test(text))) {
    return 'I cannot provide information on that topic.';
  }
  
  return text;
}
```

## Performance Optimization

### Caching Layer
```javascript
// Cache frequent queries
const responseCache = new Map();

function getCacheKey(prompt, options) {
  return `${prompt}:${JSON.stringify(options)}`;
}

async function callLLMWithCache(prompt, options = {}) {
  const cacheKey = getCacheKey(prompt, options);
  
  // Check cache first
  if (responseCache.has(cacheKey)) {
    return responseCache.get(cacheKey);
  }
  
  // Make API call
  const response = await callLLMSafely(prompt, options);
  
  // Cache response (with TTL)
  responseCache.set(cacheKey, response);
  setTimeout(() => responseCache.delete(cacheKey), 3600000); // 1 hour
  
  return response;
}
```

### Streaming Responses
```javascript
// Handle streaming for better UX
export default async function handler(req, res) {
  const { prompt } = req.body;
  
  try {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }],
        stream: true, // Enable streaming
      }),
    });

    // Set headers for streaming
    res.writeHead(200, {
      'Content-Type': 'text/plain',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    });

    // Stream response to client
    const reader = response.body.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      
      const chunk = new TextDecoder().decode(value);
      res.write(chunk);
    }
    
    res.end();
  } catch (error) {
    res.status(500).json({ error: 'Streaming failed' });
  }
}
```

## Client-Side Implementation

### React Hook for Safe LLM Calls
```javascript
// hooks/useLLM.js
import { useState, useCallback } from 'react';

export function useLLM() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  const callLLM = useCallback(async (prompt, options = {}) => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('/api/llm', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ prompt, ...options }),
      });
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      
      const data = await response.json();
      return data.response;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  }, []);
  
  return { callLLM, loading, error };
}
```

### Usage in Component
```javascript
// components/AIChat.js
import { useLLM } from '../hooks/useLLM';

export default function AIChat() {
  const { callLLM, loading, error } = useLLM();
  const [messages, setMessages] = useState([]);
  
  const handleSubmit = async (input) => {
    try {
      const response = await callLLM(input);
      setMessages([...messages, { role: 'user', content: input }, { role: 'assistant', content: response }]);
    } catch (err) {
      // Error already handled by hook
    }
  };
  
  return (
    <div>
      {/* Chat UI */}
      {error && <div className="error">{error}</div>}
      <button disabled={loading}>
        {loading ? 'Thinking...' : 'Send'}
      </button>
    </div>
  );
}
```

## Best Practices Summary

### Security Checklist
```javascript
const securityChecklist = [
  '✅ Server-side API calls only',
  '✅ API keys in environment variables',
  '✅ Input validation and sanitization',
  '✅ Rate limiting implemented',
  '✅ Authentication required',
  '✅ Content moderation',
  '✅ Error handling without information leakage',
  '✅ Usage monitoring and logging'
];
```

### Performance Checklist
```javascript
const performanceChecklist = [
  '✅ Reasonable timeouts',
  '✅ Caching for frequent queries',
  '✅ Streaming for long responses',
  '✅ Fallback strategies',
  '✅ Cost monitoring',
  '✅ Rate limiting to prevent abuse'
];
```

## Interview Tips
- **Server-side calls**: Never expose API keys to client
- **Input validation**: Sanitize and validate all user inputs
- **Rate limiting**: Protect against abuse and control costs
- **Error handling**: Graceful degradation and user-friendly messages
- **Authentication**: Require user auth for LLM features
- **Monitoring**: Track usage, costs, and performance
- **Content safety**: Moderate inputs and filter outputs
- **Caching**: Improve performance and reduce costs
- **Fallbacks**: Handle API failures gracefully