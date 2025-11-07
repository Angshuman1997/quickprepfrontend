# How to set up Axios with base URL and default headers?

## Question
How to set up Axios with base URL and default headers?

## Answer
Use `axios.create()` to set up a configured Axios instance with base URL and default headers. This creates a reusable client for all API calls.

## Basic Setup

**Create configured Axios instance:**
```typescript
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'https://api.example.com',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer your-token'
  }
});

export default apiClient;
```

**Usage in components:**
```typescript
// Instead of full URL each time
const response = await axios.get('https://api.example.com/users');

// Use configured client
const response = await apiClient.get('/users');
// Automatically becomes: https://api.example.com/users
// Headers are included automatically
```

## Environment Variables

**Use environment variables for different environments:**
```typescript
const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:3001/api',
  headers: {
    'Content-Type': 'application/json'
  }
});
```

## Interview Q&A

**Q: Why use axios.create() instead of default axios?**

A: `axios.create()` lets you set base URL and default headers once, then reuse for all API calls. No need to repeat configuration.

**Q: How do you set a base URL in Axios?**

A: Pass `baseURL` option to `axios.create()`. All requests will automatically prepend this URL.

**Q: How do you add default headers to all requests?**

A: Include `headers` object in the `axios.create()` configuration. These headers will be sent with every request.