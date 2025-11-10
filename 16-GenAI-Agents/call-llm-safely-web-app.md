# How do you call an LLM safely from a web app?

## Simple Answer
Make LLM API calls server-side only, never expose API keys to the client. Implement input validation, rate limiting, error handling, and monitoring to ensure safe and reliable operation.

## Security: Server-Side Only

```javascript
// ❌ DANGEROUS - Client exposes API key
const callLLM = async () => {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    headers: {
      'Authorization': `Bearer ${OPENAI_API_KEY}`, // Key visible to users!
    }
  });
};

// ✅ SECURE - Server-side proxy
// pages/api/chat.js (Next.js)
export default async function handler(req, res) {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`, // Key stays secure
    },
    body: JSON.stringify(req.body)
  });
  
  const data = await response.json();
  res.status(200).json(data);
}
```

## Environment Variables

```bash
# .env.local (never commit to git)
OPENAI_API_KEY=sk-your-secret-key-here
```

```javascript
// Only use in server-side code
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY,
});
```

## Input Validation

```javascript
// Validate and sanitize inputs
export default async function handler(req, res) {
  const { messages } = req.body;
  
  // Check structure
  if (!messages || !Array.isArray(messages)) {
    return res.status(400).json({ error: 'Invalid messages format' });
  }
  
  // Validate each message
  for (const message of messages) {
    if (!message.role || !message.content) {
      return res.status(400).json({ error: 'Invalid message format' });
    }
    
    // Check length limits
    if (message.content.length > 10000) {
      return res.status(400).json({ error: 'Message too long' });
    }
  }
  
  // Proceed with LLM call...
}
```

## Rate Limiting

```javascript
import rateLimit from 'express-rate-limit';

const llmLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 50, // 50 requests per window
  message: 'Too many requests. Please wait before trying again.',
  keyGenerator: (req) => req.user?.id || req.ip, // Per user or IP
});

app.use('/api/llm', llmLimiter);
```

## Error Handling

```javascript
export default async function handler(req, res) {
  try {
    const response = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(req.body),
      signal: AbortSignal.timeout(30000), // 30 second timeout
    });

    if (!response.ok) {
      throw new Error(`API error: ${response.status}`);
    }

    const data = await response.json();
    return res.status(200).json(data);
    
  } catch (error) {
    console.error('LLM API error:', error);
    
    // Handle different error types
    if (error.name === 'AbortError') {
      return res.status(408).json({ error: 'Request timed out' });
    }
    
    if (error.message.includes('429')) {
      return res.status(429).json({ error: 'Rate limit exceeded' });
    }
    
    // Generic error - don't expose details
    return res.status(500).json({ error: 'AI service temporarily unavailable' });
  }
}
```

## Authentication

```javascript
// Require user authentication
import { getSession } from 'next-auth/react';

export default async function handler(req, res) {
  const session = await getSession({ req });
  
  if (!session) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  
  // Proceed with LLM call...
}
```

## Cost Monitoring

```javascript
// Track usage and costs
function trackUsage(userId, tokensUsed, model) {
  // Log for monitoring
  console.log(`User ${userId} used ${tokensUsed} tokens on ${model}`);
  
  // Calculate approximate cost
  const costPerToken = model === 'gpt-4' ? 0.06 : 0.002; // per 1K tokens
  const cost = (tokensUsed * costPerToken) / 1000;
  
  // Alert if cost is too high
  if (cost > 1.0) { // $1 threshold
    console.warn(`High cost alert: $${cost} for user ${userId}`);
  }
}
```

## Content Safety

```javascript
// Moderate input content
async function moderateContent(text) {
  // Use OpenAI moderation API
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

// Check before processing
if (await moderateContent(userInput)) {
  return res.status(400).json({ error: 'Content violates usage policy' });
}
```

## Client-Side Hook

```javascript
// hooks/useLLM.js
import { useState } from 'react';

export function useLLM() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  const callLLM = async (prompt) => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('/api/llm', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt }),
      });
      
      if (!response.ok) throw new Error('API call failed');
      
      const data = await response.json();
      return data.response;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };
  
  return { callLLM, loading, error };
}
```

## Interview Q&A

**Q: How do you securely call LLMs from a web app?**
A: Always make API calls server-side through your backend. Store API keys in environment variables on the server, never expose them to the client. The client calls your API, which then calls the LLM service securely.

**Q: What are the main security concerns with LLM integration?**
A: API key exposure, input validation (preventing injection attacks), rate limiting (preventing abuse), cost control, content moderation, and proper error handling without leaking sensitive information.

**Q: How do you handle rate limiting for LLM calls?**
A: Implement rate limiting middleware that tracks requests per user or IP address. Set reasonable limits (like 50 requests per 15 minutes) and return appropriate error messages when limits are exceeded.

**Q: What should you validate when accepting user input for LLM calls?**
A: Message format and structure, content length limits, allowed message roles (user/assistant/system), and potentially harmful content patterns. Sanitize inputs to prevent injection attacks.

**Q: How do you handle LLM API failures?**
A: Implement comprehensive error handling with different responses for different error types (timeouts, rate limits, authentication failures). Provide user-friendly error messages and consider fallback strategies like simpler models or cached responses.

## Best Practices
- **Server-side only**: Never expose API keys to client
- **Environment variables**: Store secrets securely
- **Input validation**: Sanitize and validate all inputs
- **Rate limiting**: Prevent abuse and control costs
- **Authentication**: Require user login for LLM features
- **Error handling**: Graceful degradation with user-friendly messages
- **Monitoring**: Track usage, costs, and performance
- **Content safety**: Moderate inputs and filter harmful content
- **Timeouts**: Set reasonable request timeouts
- **Caching**: Cache frequent queries to improve performance</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\11-GenAI-Agents\call-llm-safely-web-app-simple.md