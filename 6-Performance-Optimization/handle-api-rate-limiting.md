# How to Handle API Rate Limiting

## Question
How do you handle API rate limiting in your applications?

## Answer
API rate limiting happens when you make too many requests to a server too quickly. The server says "slow down!" and returns a 429 error. Here's how to handle it gracefully so your app keeps working smoothly.

## What is Rate Limiting?

**Simple explanation:** APIs limit how many requests you can make per minute/hour to prevent overload. When you hit the limit, you get a "Too Many Requests" error.

**Why it happens:**
- Server protection (prevents crashes)
- Fair usage (everyone gets a turn)
- Cost control (APIs aren't free)

## Basic Solution: Retry with Backoff

When you get a 429 error, wait a bit and try again. This is the simplest approach.

```javascript
// Basic retry function
async function fetchWithRetry(url, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            const response = await fetch(url);

            if (response.status === 429) {
                // Wait before retrying
                const retryAfter = response.headers.get('Retry-After');
                const waitTime = retryAfter ? parseInt(retryAfter) * 1000 : 1000 * (i + 1);

                console.log(`Rate limited. Waiting ${waitTime}ms before retry ${i + 1}/${maxRetries}`);
                await new Promise(resolve => setTimeout(resolve, waitTime));
                continue;
            }

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}`);
            }

            return response.json();
        } catch (error) {
            if (i === maxRetries - 1) throw error;
        }
    }
}

// Usage
const data = await fetchWithRetry('/api/users');
```

**When to use:** Most cases. Simple and effective.

## React Hook for Rate Limiting

Create a custom hook that handles rate limiting automatically.

```javascript
import { useState, useCallback } from 'react';

function useApiWithRetry(maxRetries = 3) {
    const [isLoading, setIsLoading] = useState(false);
    const [error, setError] = useState(null);

    const makeRequest = useCallback(async (url, options = {}) => {
        setIsLoading(true);
        setError(null);

        for (let i = 0; i < maxRetries; i++) {
            try {
                const response = await fetch(url, options);

                if (response.status === 429) {
                    const retryAfter = response.headers.get('Retry-After');
                    const waitTime = retryAfter ? parseInt(retryAfter) * 1000 : 1000 * Math.pow(2, i);

                    await new Promise(resolve => setTimeout(resolve, waitTime));
                    continue;
                }

                if (!response.ok) {
                    throw new Error(`Request failed: ${response.status}`);
                }

                const data = await response.json();
                setIsLoading(false);
                return data;

            } catch (err) {
                if (i === maxRetries - 1) {
                    setError(err.message);
                    setIsLoading(false);
                    throw err;
                }
            }
        }
    }, [maxRetries]);

    return { makeRequest, isLoading, error };
}

// Usage in component
function UserList() {
    const { makeRequest, isLoading, error } = useApiWithRetry();

    const loadUsers = async () => {
        try {
            const users = await makeRequest('/api/users');
            setUsers(users);
        } catch (err) {
            console.error('Failed to load users:', err);
        }
    };

    return (
        <div>
            <button onClick={loadUsers} disabled={isLoading}>
                {isLoading ? 'Loading...' : 'Load Users'}
            </button>
            {error && <p>Error: {error}</p>}
        </div>
    );
}
```

**When to use:** React applications where you want automatic loading states and error handling.

**When to use:** React applications where you want automatic loading states and error handling.

## Prevent Hitting Limits: Request Throttling

Instead of hitting the limit and retrying, slow down your requests proactively.

```javascript
class RequestThrottler {
    constructor(requestsPerSecond = 5) {
        this.requestsPerSecond = requestsPerSecond;
        this.lastRequestTime = 0;
    }

    async throttleRequest(url, options = {}) {
        const now = Date.now();
        const timeSinceLastRequest = now - this.lastRequestTime;
        const minInterval = 1000 / this.requestsPerSecond;

        if (timeSinceLastRequest < minInterval) {
            const waitTime = minInterval - timeSinceLastRequest;
            await new Promise(resolve => setTimeout(resolve, waitTime));
        }

        this.lastRequestTime = Date.now();
        return fetch(url, options);
    }
}

// Usage
const throttler = new RequestThrottler(3); // Max 3 requests per second

// All requests are automatically spaced out
const response1 = await throttler.throttleRequest('/api/users');
const response2 = await throttler.throttleRequest('/api/posts');
const response3 = await throttler.throttleRequest('/api/comments');
```

**When to use:** When you know the API limits and want to stay under them consistently.

**When to use:** When you know the API limits and want to stay under them consistently.

## Batch Multiple Requests

Instead of making many small requests, combine them into fewer larger ones.

```javascript
// Instead of:
// await fetch('/api/user/1');
// await fetch('/api/user/2');
// await fetch('/api/user/3');

// Do this:
async function batchRequests(urls) {
    const responses = await Promise.all(
        urls.map(url => fetch(url))
    );

    return Promise.all(
        responses.map(response => {
            if (response.status === 429) {
                throw new Error('Rate limited');
            }
            return response.json();
        })
    );
}

// Usage
const userIds = [1, 2, 3, 4, 5];
const urls = userIds.map(id => `/api/user/${id}`);
const users = await batchRequests(urls);
```

**When to use:** When fetching multiple related items. Reduces total requests significantly.

**When to use:** When fetching multiple related items. Reduces total requests significantly.

## Handle Rate Limits Gracefully in UI

Show users what's happening instead of just failing silently.

```javascript
function RateLimitedButton({ onClick, children }) {
    const [isRateLimited, setIsRateLimited] = useState(false);
    const [retryIn, setRetryIn] = useState(0);

    const handleClick = async () => {
        try {
            setIsRateLimited(false);
            await onClick();
        } catch (error) {
            if (error.message.includes('429') || error.message.includes('rate limit')) {
                setIsRateLimited(true);
                setRetryIn(30); // Start countdown from 30 seconds

                const interval = setInterval(() => {
                    setRetryIn(prev => {
                        if (prev <= 1) {
                            clearInterval(interval);
                            setIsRateLimited(false);
                            return 0;
                        }
                        return prev - 1;
                    });
                }, 1000);
            }
        }
    };

    return (
        <button onClick={handleClick} disabled={isRateLimited}>
            {isRateLimited ? `Try again in ${retryIn}s` : children}
        </button>
    );
}

// Usage
<RateLimitedButton onClick={saveData}>
    Save Changes
</RateLimitedButton>
```

**When to use:** User-facing buttons that might trigger rate-limited APIs.

## Advanced: Circuit Breaker Pattern

Stop making requests temporarily when the API is consistently rate limiting you.

```javascript
class CircuitBreaker {
    constructor(failureThreshold = 3, recoveryTime = 30000) {
        this.failureThreshold = failureThreshold;
        this.recoveryTime = recoveryTime;
        this.failureCount = 0;
        this.lastFailureTime = 0;
        this.state = 'closed'; // 'closed', 'open', 'half-open'
    }

    async execute(operation) {
        if (this.state === 'open') {
            if (Date.now() - this.lastFailureTime > this.recoveryTime) {
                this.state = 'half-open';
            } else {
                throw new Error('Circuit breaker is OPEN - API is rate limiting');
            }
        }

        try {
            const result = await operation();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }

    onSuccess() {
        this.failureCount = 0;
        this.state = 'closed';
    }

    onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();

        if (this.failureCount >= this.failureThreshold) {
            this.state = 'open';
        }
    }
}

// Usage
const breaker = new CircuitBreaker();

async function safeApiCall(url) {
    return breaker.execute(async () => {
        const response = await fetch(url);
        if (response.status === 429) {
            throw new Error('Rate limited');
        }
        return response.json();
    });
}
```

**When to use:** High-traffic apps where you want to protect against cascading failures.

## Check These Headers

APIs tell you about rate limits through headers:

```javascript
const response = await fetch('/api/data');

console.log('Rate limit info:');
console.log('Total allowed:', response.headers.get('X-RateLimit-Limit'));
console.log('Remaining:', response.headers.get('X-RateLimit-Remaining'));
console.log('Resets at:', response.headers.get('X-RateLimit-Reset'));
console.log('Retry after:', response.headers.get('Retry-After'));
```

## Common Patterns Summary

**For simple apps:**
- Use retry with exponential backoff
- Show loading states to users

**For complex apps:**
- Add request throttling
- Implement circuit breakers
- Batch requests when possible

**For user-facing features:**
- Show rate limit status in UI
- Disable buttons during cooldown
- Provide clear error messages

## Interview Answer

"I handle API rate limiting by implementing retry logic with exponential backoff for 429 errors, and using request throttling to prevent hitting limits in the first place. In React apps, I create custom hooks that manage loading states and provide user feedback during rate-limited periods. For critical operations, I implement circuit breakers to prevent cascading failures."

## Key Takeaways

- **Retry with backoff** is your first line of defense
- **Throttle requests** to stay under limits proactively
- **Show users what's happening** instead of silent failures
- **Check response headers** for rate limit information
- **Batch requests** when fetching multiple items
- **Use circuit breakers** for resilient systems