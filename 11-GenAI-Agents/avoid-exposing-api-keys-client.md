# How do you avoid exposing API keys on the client?

## Question
How do you avoid exposing API keys on the client?

# How do you avoid exposing API keys on the client?

## Question
How do you avoid exposing API keys on the client?

## Answer

Exposing API keys on the client side is a major security risk because anyone can view the source code and extract the keys. The solution is to never send API keys to the client and instead make API calls from the server side.

## The Problem with Client-Side API Keys

### Why It's Dangerous
```javascript
// ❌ DANGEROUS: API key in client-side code
const OPENAI_API_KEY = 'sk-your-secret-key-here';

async function callOpenAI() {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${OPENAI_API_KEY}` // Key exposed!
    },
    body: JSON.stringify({
      model: 'gpt-4',
      messages: [{ role: 'user', content: 'Hello!' }]
    })
  });
  
  return response.json();
}
```

### Security Risks
```javascript
const securityRisks = {
  theft: 'Anyone can copy your API key from browser dev tools',
  abuse: 'Malicious users can make unlimited API calls at your expense',
  quota: 'Your API limits can be exhausted quickly',
  cost: 'You pay for all usage, including abuse',
  rateLimits: 'Your application can be rate-limited or banned',
  dataExposure: 'Sensitive data might be exposed through API calls'
};
```

## Server-Side API Calls

### Backend API Proxy
```javascript
// ✅ SECURE: Server-side API call
// server.js (Node.js/Express)
const express = require('express');
const { Configuration, OpenAIApi } = require('openai');

const app = express();
app.use(express.json());

// Keep API key secure on server
const configuration = new Configuration({
  apiKey: process.env.OPENAI_API_KEY, // From environment variables
});
const openai = new OpenAIApi(configuration);

app.post('/api/chat', async (req, res) => {
  try {
    const { message } = req.body;
    
    const completion = await openai.createChatCompletion({
      model: 'gpt-4',
      messages: [{ role: 'user', content: message }],
    });
    
    res.json({ response: completion.data.choices[0].message.content });
  } catch (error) {
    res.status(500).json({ error: 'API call failed' });
  }
});

app.listen(3001);
```

### Client-Side Implementation
```javascript
// client.js - No API keys here!
async function sendMessage(message) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ message })
  });
  
  const data = await response.json();
  return data.response;
}
```

## Environment Variables

### Server-Side Configuration
```bash
# .env file (never commit to git)
OPENAI_API_KEY=sk-your-secret-key-here
ANTHROPIC_API_KEY=sk-ant-your-key-here
DATABASE_URL=postgresql://user:pass@localhost:5432/db
```

```javascript
// Load environment variables
require('dotenv').config();

// Use in code
const openai = new OpenAIApi({
  apiKey: process.env.OPENAI_API_KEY,
});
```

### Different Environments
```javascript
// config.js
const config = {
  development: {
    openai: {
      apiKey: process.env.OPENAI_API_KEY_DEV,
    }
  },
  production: {
    openai: {
      apiKey: process.env.OPENAI_API_KEY_PROD,
    }
  }
};

module.exports = config[process.env.NODE_ENV || 'development'];
```

## Next.js API Routes

### API Route Implementation
```javascript
// pages/api/chat.js
import { Configuration, OpenAIApi } from 'openai';

const configuration = new Configuration({
  apiKey: process.env.OPENAI_API_KEY,
});
const openai = new OpenAIApi(configuration);

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  try {
    const { messages } = req.body;
    
    const completion = await openai.createChatCompletion({
      model: 'gpt-4',
      messages,
    });
    
    res.status(200).json({
      response: completion.data.choices[0].message.content
    });
  } catch (error) {
    console.error('OpenAI API error:', error);
    res.status(500).json({ error: 'Failed to get AI response' });
  }
}
```

### Client-Side Call
```javascript
// components/Chat.js
import { useState } from 'react';

export default function Chat() {
  const [message, setMessage] = useState('');
  const [response, setResponse] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    
    try {
      const res = await fetch('/api/chat', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          messages: [{ role: 'user', content: message }]
        }),
      });
      
      const data = await res.json();
      setResponse(data.response);
    } catch (error) {
      console.error('Error:', error);
      setResponse('Sorry, something went wrong.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          value={message}
          onChange={(e) => setMessage(e.target.value)}
          placeholder="Ask me anything..."
        />
        <button type="submit" disabled={loading}>
          {loading ? 'Thinking...' : 'Send'}
        </button>
      </form>
      {response && <p>{response}</p>}
    </div>
  );
}
```

## Authentication and Authorization

### API Key Validation
```javascript
// middleware/auth.js
export function validateApiKey(req) {
  const apiKey = req.headers['x-api-key'];
  const validKeys = process.env.VALID_API_KEYS?.split(',') || [];
  
  if (!apiKey || !validKeys.includes(apiKey)) {
    throw new Error('Invalid API key');
  }
  
  return true;
}

// In API route
import { validateApiKey } from '../../middleware/auth';

export default async function handler(req, res) {
  try {
    validateApiKey(req);
    
    // Proceed with API call...
  } catch (error) {
    res.status(401).json({ error: 'Unauthorized' });
  }
}
```

### User-Based Authentication
```javascript
// With user authentication
import { getSession } from 'next-auth/react';

export default async function handler(req, res) {
  const session = await getSession({ req });
  
  if (!session) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  
  // User is authenticated, proceed with API call
  // You can also implement user-based rate limiting here
}
```

## Rate Limiting and Security

### Rate Limiting Implementation
```javascript
// middleware/rateLimit.js
import rateLimit from 'express-rate-limit';

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP, please try again later.',
  standardHeaders: true,
  legacyHeaders: false,
});

// Apply to API routes
app.use('/api/', apiLimiter);
```

### Input Validation and Sanitization
```javascript
// middleware/validation.js
import validator from 'validator';

export function validateChatInput(req, res, next) {
  const { messages } = req.body;
  
  if (!messages || !Array.isArray(messages)) {
    return res.status(400).json({ error: 'Invalid messages format' });
  }
  
  // Validate each message
  for (const message of messages) {
    if (!message.role || !message.content) {
      return res.status(400).json({ error: 'Invalid message format' });
    }
    
    if (!['user', 'assistant', 'system'].includes(message.role)) {
      return res.status(400).json({ error: 'Invalid message role' });
    }
    
    // Sanitize content
    message.content = validator.escape(message.content);
  }
  
  next();
}
```

## CORS Configuration

### Secure CORS Setup
```javascript
// middleware/cors.js
import cors from 'cors';

const corsOptions = {
  origin: function (origin, callback) {
    // Allow requests from your domain only
    const allowedOrigins = [
      'http://localhost:3000',
      'https://yourdomain.com'
    ];
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  optionsSuccessStatus: 200
};

export default cors(corsOptions);
```

## Error Handling

### Secure Error Responses
```javascript
// Don't expose internal errors
export default async function handler(req, res) {
  try {
    // API call logic
    const result = await callExternalAPI();
    res.status(200).json(result);
  } catch (error) {
    console.error('API Error:', error);
    
    // Don't send error details to client
    res.status(500).json({ 
      error: 'Something went wrong. Please try again later.' 
    });
  }
}
```

## Deployment Security

### Environment Variables in Production
```javascript
// vercel.json or deployment config
{
  "env": {
    "OPENAI_API_KEY": "@openai-api-key",
    "ANTHROPIC_API_KEY": "@anthropic-api-key"
  }
}
```

### Build-Time vs Runtime
```javascript
// ❌ Don't do this - exposed at build time
const API_KEY = process.env.OPENAI_API_KEY;

// ✅ Do this - only available at runtime
export default async function handler(req, res) {
  const apiKey = process.env.OPENAI_API_KEY;
  // Use in server-side code only
}
```

## Monitoring and Logging

### API Usage Monitoring
```javascript
// middleware/monitoring.js
export function logApiUsage(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`API Call: ${req.method} ${req.url} - ${res.statusCode} - ${duration}ms`);
    
    // Send to monitoring service (e.g., DataDog, New Relic)
    // monitor.track('api_call', { method: req.method, url: req.url, status: res.statusCode, duration });
  });
  
  next();
}
```

## Best Practices Summary

### Security Checklist
```javascript
const securityChecklist = [
  '✅ Never expose API keys in client-side code',
  '✅ Use environment variables for secrets',
  '✅ Implement server-side API calls',
  '✅ Add authentication and authorization',
  '✅ Implement rate limiting',
  '✅ Validate and sanitize inputs',
  '✅ Configure CORS properly',
  '✅ Handle errors securely',
  '✅ Monitor API usage',
  '✅ Use HTTPS in production'
];
```

### Architecture Patterns
```javascript
const architecturePatterns = {
  apiProxy: 'Client → Your API → External API',
  bff: 'Backend for Frontend pattern',
  microservices: 'Separate API services',
  serverless: 'API routes in serverless functions'
};
```

## Common Mistakes to Avoid

### ❌ Client-Side API Keys
```javascript
// Don't do this!
const API_KEY = 'sk-your-key';
fetch('https://api.openai.com/v1/chat/completions', {
  headers: { Authorization: `Bearer ${API_KEY}` }
});
```

### ❌ Hardcoded Secrets
```javascript
// Don't do this!
const config = {
  apiKey: 'sk-your-secret-key'
};
```

### ❌ Logging Sensitive Data
```javascript
// Don't do this!
console.log('API Response:', response); // Might contain sensitive data
```

## Interview Tips
- **Never expose API keys**: Client-side code is visible to everyone
- **Server-side proxy**: Always route API calls through your backend
- **Environment variables**: Store secrets securely, not in code
- **Authentication**: Add user auth and API key validation
- **Rate limiting**: Protect against abuse and control costs
- **Input validation**: Sanitize all user inputs
- **Error handling**: Don't expose internal errors to clients
- **Monitoring**: Track API usage and performance
- **HTTPS**: Always use encrypted connections