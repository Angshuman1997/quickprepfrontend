# How do you avoid exposing API keys on the client?

## Simple Answer
Never put API keys in client-side code. Instead, make API calls from your server and proxy them through your backend. Store API keys in environment variables on the server.

## The Problem

```javascript
// ❌ DANGEROUS - Key exposed in browser
const OPENAI_API_KEY = 'sk-your-secret-key-here';

fetch('https://api.openai.com/v1/chat/completions', {
  headers: {
    'Authorization': `Bearer ${OPENAI_API_KEY}` // Anyone can see this!
  }
});
```

**Risks:**
- Anyone can copy your API key
- Malicious users can abuse your API
- You pay for all usage
- Your app can get rate-limited or banned

## The Solution: Server-Side Proxy

```javascript
// ✅ SECURE - API call on server
// server.js
const express = require('express');
const app = express();

app.post('/api/chat', async (req, res) => {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`, // Key stays on server
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(req.body)
  });
  
  const data = await response.json();
  res.json(data);
});
```

```javascript
// client.js - No API keys here!
async function sendMessage(message) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({ message })
  });
  
  return response.json();
}
```

## Environment Variables

```bash
# .env file (never commit to git)
OPENAI_API_KEY=sk-your-secret-key-here
ANTHROPIC_API_KEY=sk-ant-your-key-here
```

```javascript
// server.js
require('dotenv').config();

const apiKey = process.env.OPENAI_API_KEY; // Secure on server only
```

## Next.js API Routes

```javascript
// pages/api/chat.js
export default async function handler(req, res) {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(req.body)
  });
  
  const data = await response.json();
  res.status(200).json(data);
}
```

```javascript
// components/Chat.js
const sendMessage = async (message) => {
  const res = await fetch('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages: [{ role: 'user', content: message }] })
  });
  
  return res.json();
};
```

## Security Best Practices

### 1. Authentication
```javascript
// Check if user is logged in before allowing API calls
export default async function handler(req, res) {
  const session = await getSession({ req });
  if (!session) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  
  // Proceed with API call...
}
```

### 2. Rate Limiting
```javascript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests
});

app.use('/api/', limiter);
```

### 3. Input Validation
```javascript
// Validate and sanitize inputs
export default async function handler(req, res) {
  const { message } = req.body;
  
  if (!message || typeof message !== 'string' || message.length > 1000) {
    return res.status(400).json({ error: 'Invalid message' });
  }
  
  // Proceed...
}
```

### 4. Error Handling
```javascript
// Don't expose internal errors
try {
  const result = await callExternalAPI();
  res.json(result);
} catch (error) {
  console.error('API Error:', error); // Log for debugging
  res.status(500).json({ error: 'Something went wrong' }); // Generic message
}
```

## Interview Q&A

**Q: Why can't you put API keys in client-side code?**
A: Client-side code is visible to anyone who opens the browser dev tools. Anyone can copy your API key and abuse your API at your expense.

**Q: How do you securely call external APIs from a web app?**
A: Create API routes on your server that proxy the requests. The client calls your API, which then calls the external API using the key stored securely on the server.

**Q: Where should you store API keys?**
A: In environment variables on the server, never in the code repository. Use .env files that are gitignored, and load them with libraries like dotenv.

**Q: What are some additional security measures?**
A: Add user authentication, rate limiting, input validation, CORS configuration, and proper error handling. Monitor API usage and use HTTPS in production.

**Q: Can you use API keys client-side for some services?**
A: Only for services that are designed for client-side use and have proper security measures like API key restrictions, rate limiting, and domain whitelisting. But for sensitive APIs like OpenAI, always use server-side proxy.

## Architecture Patterns

### API Proxy Pattern
```
Client → Your Server API → External API
```

### Backend for Frontend (BFF)
```
Client → BFF Service → Multiple External APIs
```

### Serverless Functions
```
Client → Serverless Function → External API
```

## Common Mistakes
- ❌ Hardcoding API keys in JavaScript files
- ❌ Committing .env files to git
- ❌ Logging sensitive data in client-side code
- ❌ No rate limiting or input validation
- ❌ Exposing detailed error messages to users

## Key Takeaways
- **Server-side only**: API keys belong on the server
- **Environment variables**: Store secrets securely
- **Proxy pattern**: Route all external API calls through your backend
- **Authentication**: Add user auth and API key validation
- **Validation**: Sanitize all inputs and handle errors securely
- **Monitoring**: Track API usage and costs</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\11-GenAI-Agents\avoid-exposing-api-keys-client-simple.md