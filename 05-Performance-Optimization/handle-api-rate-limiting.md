# How to handle API rate limiting?

## Question
How to handle API rate limiting?

## Answer

API rate limiting is a technique used by servers to control the amount of requests a client can make within a specific time window. When rate limits are exceeded, servers return error responses (typically 429 Too Many Requests). Effective rate limiting handling improves application reliability, user experience, and prevents service disruptions.

## Understanding Rate Limiting

### 1. **Common Rate Limiting Patterns**

**Fixed Window:**
- Allows X requests per time window (e.g., 100 requests per minute)
- Resets at fixed intervals regardless of when requests were made

**Sliding Window:**
- Tracks requests in a moving time window
- More precise but computationally expensive

**Token Bucket:**
- Client accumulates tokens over time
- Each request consumes a token
- Allows burst traffic when tokens are available

**Leaky Bucket:**
- Requests are processed at a constant rate
- Excess requests are queued or rejected

### 2. **Rate Limit Headers**

APIs often provide rate limit information in response headers:

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 50
X-RateLimit-Reset: 1634567890
X-RateLimit-Retry-After: 60
```

## Client-Side Rate Limiting Strategies

### 1. **Request Queuing with Retry**

**Basic retry with exponential backoff:**
```typescript
interface RateLimitConfig {
    maxRetries: number;
    baseDelay: number;
    maxDelay: number;
}

class ApiClient {
    private retryQueue: Array<() => Promise<any>> = [];
    private isProcessing = false;

    async request(url: string, options: RequestInit = {}): Promise<Response> {
        const response = await fetch(url, options);

        if (response.status === 429) {
            return this.handleRateLimit(response, url, options);
        }

        return response;
    }

    private async handleRateLimit(
        response: Response,
        url: string,
        options: RequestInit
    ): Promise<Response> {
        const retryAfter = response.headers.get('Retry-After');
        const delay = retryAfter ? parseInt(retryAfter) * 1000 : 1000;

        console.warn(`Rate limited. Retrying in ${delay}ms`);

        await this.delay(delay);
        return this.request(url, options);
    }

    private delay(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

**Advanced retry with exponential backoff:**
```typescript
class RateLimitHandler {
    private config: RateLimitConfig;

    constructor(config: RateLimitConfig = {
        maxRetries: 3,
        baseDelay: 1000,
        maxDelay: 30000
    }) {
        this.config = config;
    }

    async executeWithRetry<T>(
        operation: () => Promise<T>,
        attempt = 1
    ): Promise<T> {
        try {
            return await operation();
        } catch (error) {
            if (error.status === 429 && attempt <= this.config.maxRetries) {
                const delay = Math.min(
                    this.config.baseDelay * Math.pow(2, attempt - 1),
                    this.config.maxDelay
                );

                console.warn(`Rate limited. Retry ${attempt}/${this.config.maxRetries} in ${delay}ms`);
                await this.delay(delay);

                return this.executeWithRetry(operation, attempt + 1);
            }

            throw error;
        }
    }

    private delay(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Usage
const apiClient = new RateLimitHandler();

const fetchData = async () => {
    return apiClient.executeWithRetry(async () => {
        const response = await fetch('/api/data');

        if (!response.ok) {
            const error = new Error('API Error');
            (error as any).status = response.status;
            throw error;
        }

        return response.json();
    });
};
```

### 2. **Request Throttling**

**Throttle requests to stay within limits:**
```typescript
class RequestThrottler {
    private queue: Array<() => void> = [];
    private processing = false;
    private lastRequestTime = 0;
    private minInterval: number;

    constructor(requestsPerSecond: number = 10) {
        this.minInterval = 1000 / requestsPerSecond;
    }

    async throttle<T>(operation: () => Promise<T>): Promise<T> {
        return new Promise((resolve, reject) => {
            this.queue.push(async () => {
                try {
                    const result = await operation();
                    resolve(result);
                } catch (error) {
                    reject(error);
                }
            });

            this.processQueue();
        });
    }

    private async processQueue() {
        if (this.processing || this.queue.length === 0) return;

        this.processing = true;

        while (this.queue.length > 0) {
            const now = Date.now();
            const timeSinceLastRequest = now - this.lastRequestTime;

            if (timeSinceLastRequest < this.minInterval) {
                await this.delay(this.minInterval - timeSinceLastRequest);
            }

            const operation = this.queue.shift();
            if (operation) {
                await operation();
                this.lastRequestTime = Date.now();
            }
        }

        this.processing = false;
    }

    private delay(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Usage
const throttler = new RequestThrottler(5); // 5 requests per second

const makeThrottledRequest = async (url: string) => {
    return throttler.throttle(async () => {
        const response = await fetch(url);
        return response.json();
    });
};
```

### 3. **Token Bucket Implementation**

**Client-side token bucket:**
```typescript
interface TokenBucketConfig {
    capacity: number;      // Max tokens
    refillRate: number;    // Tokens per second
    initialTokens?: number;
}

class TokenBucket {
    private tokens: number;
    private lastRefill: number;
    private config: TokenBucketConfig;

    constructor(config: TokenBucketConfig) {
        this.config = config;
        this.tokens = config.initialTokens || config.capacity;
        this.lastRefill = Date.now();
    }

    async consume(): Promise<boolean> {
        this.refill();

        if (this.tokens >= 1) {
            this.tokens -= 1;
            return true;
        }

        return false;
    }

    private refill() {
        const now = Date.now();
        const timePassed = (now - this.lastRefill) / 1000; // seconds
        const tokensToAdd = timePassed * this.config.refillRate;

        this.tokens = Math.min(
            this.config.capacity,
            this.tokens + tokensToAdd
        );

        this.lastRefill = now;
    }

    getAvailableTokens(): number {
        this.refill();
        return this.tokens;
    }
}

// Usage with API client
class RateLimitedApiClient {
    private bucket: TokenBucket;

    constructor() {
        // 10 requests per second, burst up to 20
        this.bucket = new TokenBucket({
            capacity: 20,
            refillRate: 10
        });
    }

    async request(url: string): Promise<any> {
        if (!await this.bucket.consume()) {
            throw new Error('Rate limit exceeded - too many requests');
        }

        const response = await fetch(url);

        if (response.status === 429) {
            // Could implement retry logic here
            throw new Error('Server rate limit exceeded');
        }

        return response.json();
    }
}
```

### 4. **Request Batching**

**Combine multiple requests into fewer API calls:**
```typescript
class RequestBatcher {
    private batch: Array<{ id: string; request: any }> = [];
    private batchTimeout: NodeJS.Timeout | null = null;
    private batchSize: number;
    private batchDelay: number;

    constructor(batchSize = 10, batchDelay = 100) {
        this.batchSize = batchSize;
        this.batchDelay = batchDelay;
    }

    async addRequest<T>(requestData: any): Promise<T> {
        return new Promise((resolve, reject) => {
            const id = Math.random().toString(36);

            this.batch.push({
                id,
                request: requestData,
                resolve,
                reject
            });

            this.scheduleBatch();
        });
    }

    private scheduleBatch() {
        if (this.batchTimeout) return;

        if (this.batch.length >= this.batchSize) {
            this.executeBatch();
        } else {
            this.batchTimeout = setTimeout(() => {
                this.executeBatch();
            }, this.batchDelay);
        }
    }

    private async executeBatch() {
        if (this.batchTimeout) {
            clearTimeout(this.batchTimeout);
            this.batchTimeout = null;
        }

        const currentBatch = [...this.batch];
        this.batch = [];

        try {
            const response = await fetch('/api/batch', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    requests: currentBatch.map(item => item.request)
                })
            });

            const results = await response.json();

            currentBatch.forEach((item, index) => {
                if (results[index]?.error) {
                    item.reject(new Error(results[index].error));
                } else {
                    item.resolve(results[index]);
                }
            });
        } catch (error) {
            currentBatch.forEach(item => item.reject(error));
        }
    }
}

// Usage
const batcher = new RequestBatcher();

// Instead of multiple individual requests
const result1 = await batcher.addRequest({ type: 'user', id: 1 });
const result2 = await batcher.addRequest({ type: 'user', id: 2 });
const result3 = await batcher.addRequest({ type: 'post', id: 1 });
```

## React-Specific Patterns

### 1. **Custom Hook for Rate Limiting**

```typescript
import { useCallback, useRef } from 'react';

function useRateLimitedApi(maxRequests = 10, windowMs = 1000) {
    const requestTimes = useRef<number[]>([]);

    const makeRequest = useCallback(async (url: string, options?: RequestInit) => {
        const now = Date.now();

        // Remove old requests outside the window
        requestTimes.current = requestTimes.current.filter(
            time => now - time < windowMs
        );

        if (requestTimes.current.length >= maxRequests) {
            const oldestRequest = Math.min(...requestTimes.current);
            const waitTime = windowMs - (now - oldestRequest);

            await new Promise(resolve => setTimeout(resolve, waitTime));
        }

        requestTimes.current.push(now);

        const response = await fetch(url, options);

        if (response.status === 429) {
            // Implement retry logic
            const retryAfter = response.headers.get('Retry-After');
            const delay = retryAfter ? parseInt(retryAfter) * 1000 : 1000;

            await new Promise(resolve => setTimeout(resolve, delay));
            return makeRequest(url, options); // Retry
        }

        return response;
    }, [maxRequests, windowMs]);

    return makeRequest;
}

// Usage in component
function DataComponent() {
    const makeRequest = useRateLimitedApi(5, 1000); // 5 requests per second

    const fetchData = async () => {
        try {
            const response = await makeRequest('/api/data');
            const data = await response.json();
            setData(data);
        } catch (error) {
            console.error('Request failed:', error);
        }
    };

    return (
        <button onClick={fetchData}>Fetch Data</button>
    );
}
```

### 2. **Rate Limit Context Provider**

```typescript
import React, { createContext, useContext, useCallback, useRef } from 'react';

interface RateLimitContextType {
    makeRequest: (url: string, options?: RequestInit) => Promise<Response>;
    isRateLimited: boolean;
}

const RateLimitContext = createContext<RateLimitContextType | null>(null);

export function RateLimitProvider({
    children,
    maxRequests = 10,
    windowMs = 1000
}: {
    children: React.ReactNode;
    maxRequests?: number;
    windowMs?: number;
}) {
    const requestTimes = useRef<number[]>([]);
    const isRateLimitedRef = useRef(false);

    const makeRequest = useCallback(async (url: string, options?: RequestInit) => {
        const now = Date.now();

        // Clean old requests
        requestTimes.current = requestTimes.current.filter(
            time => now - time < windowMs
        );

        if (requestTimes.current.length >= maxRequests) {
            isRateLimitedRef.current = true;
            const oldestRequest = Math.min(...requestTimes.current);
            const waitTime = windowMs - (now - oldestRequest);

            await new Promise(resolve => setTimeout(resolve, waitTime));
            isRateLimitedRef.current = false;
        }

        requestTimes.current.push(now);

        try {
            const response = await fetch(url, options);

            if (response.status === 429) {
                isRateLimitedRef.current = true;
                const retryAfter = response.headers.get('Retry-After');
                const delay = retryAfter ? parseInt(retryAfter) * 1000 : 1000;

                await new Promise(resolve => setTimeout(resolve, delay));
                isRateLimitedRef.current = false;

                // Retry the request
                return makeRequest(url, options);
            }

            return response;
        } catch (error) {
            isRateLimitedRef.current = false;
            throw error;
        }
    }, [maxRequests, windowMs]);

    const contextValue = {
        makeRequest,
        isRateLimited: isRateLimitedRef.current
    };

    return (
        <RateLimitContext.Provider value={contextValue}>
            {children}
        </RateLimitContext.Provider>
    );
}

export function useRateLimit() {
    const context = useContext(RateLimitContext);
    if (!context) {
        throw new Error('useRateLimit must be used within RateLimitProvider');
    }
    return context;
}

// Usage
function App() {
    return (
        <RateLimitProvider maxRequests={5} windowMs={1000}>
            <DataComponent />
        </RateLimitProvider>
    );
}

function DataComponent() {
    const { makeRequest, isRateLimited } = useRateLimit();

    return (
        <div>
            {isRateLimited && <div>Rate limited, please wait...</div>}
            <button onClick={() => makeRequest('/api/data')}>
                Fetch Data
            </button>
        </div>
    );
}
```

## Advanced Strategies

### 1. **Circuit Breaker Pattern**

**Prevent cascading failures:**
```typescript
enum CircuitState {
    CLOSED = 'closed',
    OPEN = 'open',
    HALF_OPEN = 'half_open'
}

class CircuitBreaker {
    private state: CircuitState = CircuitState.CLOSED;
    private failureCount = 0;
    private lastFailureTime = 0;
    private config: {
        failureThreshold: number;
        recoveryTimeout: number;
        monitoringPeriod: number;
    };

    constructor(config = {
        failureThreshold: 5,
        recoveryTimeout: 60000, // 1 minute
        monitoringPeriod: 10000 // 10 seconds
    }) {
        this.config = config;
    }

    async execute<T>(operation: () => Promise<T>): Promise<T> {
        if (this.state === CircuitState.OPEN) {
            if (Date.now() - this.lastFailureTime > this.config.recoveryTimeout) {
                this.state = CircuitState.HALF_OPEN;
            } else {
                throw new Error('Circuit breaker is OPEN');
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

    private onSuccess() {
        this.failureCount = 0;
        this.state = CircuitState.CLOSED;
    }

    private onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();

        if (this.failureCount >= this.config.failureThreshold) {
            this.state = CircuitState.OPEN;
        }
    }
}

// Usage with rate limiting
const circuitBreaker = new CircuitBreaker();

const makeResilientRequest = async (url: string) => {
    return circuitBreaker.execute(async () => {
        const response = await fetch(url);

        if (response.status === 429) {
            throw new Error('Rate limited');
        }

        if (!response.ok) {
            throw new Error('Request failed');
        }

        return response.json();
    });
};
```

### 2. **Adaptive Rate Limiting**

**Adjust request rate based on server feedback:**
```typescript
class AdaptiveRateLimiter {
    private currentRate: number;
    private minRate: number;
    private maxRate: number;
    private adjustmentFactor: number;
    private lastAdjustment: number;

    constructor(
        initialRate = 10,
        minRate = 1,
        maxRate = 100,
        adjustmentFactor = 0.8
    ) {
        this.currentRate = initialRate;
        this.minRate = minRate;
        this.maxRate = maxRate;
        this.adjustmentFactor = adjustmentFactor;
        this.lastAdjustment = Date.now();
    }

    async makeRequest(url: string): Promise<any> {
        // Wait based on current rate
        const interval = 1000 / this.currentRate;
        const now = Date.now();

        if (now - this.lastAdjustment < interval) {
            await this.delay(interval - (now - this.lastAdjustment));
        }

        try {
            const response = await fetch(url);

            if (response.status === 429) {
                // Reduce rate on rate limit
                this.currentRate = Math.max(
                    this.minRate,
                    this.currentRate * this.adjustmentFactor
                );
                console.log(`Rate limited. Reducing rate to ${this.currentRate} req/s`);
                throw new Error('Rate limited');
            } else {
                // Gradually increase rate on success
                this.currentRate = Math.min(
                    this.maxRate,
                    this.currentRate * 1.1
                );
            }

            this.lastAdjustment = Date.now();
            return response.json();
        } catch (error) {
            this.lastAdjustment = Date.now();
            throw error;
        }
    }

    private delay(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

### 3. **Request Prioritization**

**Handle critical vs non-critical requests:**
```typescript
type RequestPriority = 'high' | 'medium' | 'low';

interface PrioritizedRequest {
    id: string;
    priority: RequestPriority;
    operation: () => Promise<any>;
    resolve: (value: any) => void;
    reject: (error: any) => void;
}

class PriorityQueue {
    private queues = {
        high: [] as PrioritizedRequest[],
        medium: [] as PrioritizedRequest[],
        low: [] as PrioritizedRequest[]
    };

    private processing = false;
    private rateLimiter: TokenBucket;

    constructor() {
        this.rateLimiter = new TokenBucket({ capacity: 10, refillRate: 2 });
    }

    async addRequest<T>(
        operation: () => Promise<T>,
        priority: RequestPriority = 'medium'
    ): Promise<T> {
        return new Promise((resolve, reject) => {
            const request: PrioritizedRequest = {
                id: Math.random().toString(36),
                priority,
                operation,
                resolve,
                reject
            };

            this.queues[priority].push(request);
            this.processQueue();
        });
    }

    private async processQueue() {
        if (this.processing) return;

        this.processing = true;

        while (this.hasRequests()) {
            // Process high priority first, then medium, then low
            const request = this.getNextRequest();

            if (request && await this.rateLimiter.consume()) {
                try {
                    const result = await request.operation();
                    request.resolve(result);
                } catch (error) {
                    request.reject(error);
                }
            } else {
                // Rate limited, wait a bit
                await this.delay(100);
            }
        }

        this.processing = false;
    }

    private getNextRequest(): PrioritizedRequest | null {
        return this.queues.high.shift() ||
               this.queues.medium.shift() ||
               this.queues.low.shift() ||
               null;
    }

    private hasRequests(): boolean {
        return this.queues.high.length > 0 ||
               this.queues.medium.length > 0 ||
               this.queues.low.length > 0;
    }

    private delay(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

## Monitoring and Analytics

### 1. **Rate Limit Metrics**

```typescript
interface RateLimitMetrics {
    totalRequests: number;
    rateLimitedRequests: number;
    averageResponseTime: number;
    retryCount: number;
    successRate: number;
}

class RateLimitMonitor {
    private metrics: RateLimitMetrics = {
        totalRequests: 0,
        rateLimitedRequests: 0,
        averageResponseTime: 0,
        retryCount: 0,
        successRate: 0
    };

    recordRequest(responseTime: number, wasRateLimited = false, retryAttempt = 0) {
        this.metrics.totalRequests++;
        this.metrics.retryCount += retryAttempt;

        if (wasRateLimited) {
            this.metrics.rateLimitedRequests++;
        }

        // Update average response time
        this.metrics.averageResponseTime =
            (this.metrics.averageResponseTime * (this.metrics.totalRequests - 1) + responseTime) /
            this.metrics.totalRequests;

        this.metrics.successRate =
            ((this.metrics.totalRequests - this.metrics.rateLimitedRequests) / this.metrics.totalRequests) * 100;
    }

    getMetrics(): RateLimitMetrics {
        return { ...this.metrics };
    }

    logMetrics() {
        console.log('Rate Limit Metrics:', {
            ...this.metrics,
            rateLimitPercentage: (this.metrics.rateLimitedRequests / this.metrics.totalRequests * 100).toFixed(2) + '%'
        });
    }
}
```

### 2. **Error Handling and User Feedback**

```typescript
function RateLimitErrorBoundary({ children }: { children: React.ReactNode }) {
    const [rateLimitError, setRateLimitError] = useState<string | null>(null);

    useEffect(() => {
        const handleRateLimit = (event: CustomEvent) => {
            setRateLimitError(event.detail.message);
            // Auto-clear after 5 seconds
            setTimeout(() => setRateLimitError(null), 5000);
        };

        window.addEventListener('rate-limit-error', handleRateLimit as EventListener);
        return () => window.removeEventListener('rate-limit-error', handleRateLimit as EventListener);
    }, []);

    return (
        <>
            {rateLimitError && (
                <div className="rate-limit-banner">
                    <p>{rateLimitError}</p>
                    <button onClick={() => setRateLimitError(null)}>×</button>
                </div>
            )}
            {children}
        </>
    );
}

// Dispatch custom event on rate limit
const handleApiError = (error: any) => {
    if (error.status === 429) {
        const event = new CustomEvent('rate-limit-error', {
            detail: { message: 'Too many requests. Please wait a moment.' }
        });
        window.dispatchEvent(event);
    }
};
```

## Testing Rate Limiting

### 1. **Unit Testing**

```typescript
describe('RateLimitHandler', () => {
    beforeEach(() => {
        jest.useFakeTimers();
    });

    it('should retry on 429 status', async () => {
        const mockFetch = jest.fn()
            .mockRejectedValueOnce({ status: 429 })
            .mockResolvedValueOnce({ ok: true, json: () => Promise.resolve({ data: 'success' }) });

        global.fetch = mockFetch;

        const handler = new RateLimitHandler({ maxRetries: 1, baseDelay: 1000 });
        const result = await handler.executeWithRetry(() => fetch('/api/test'));

        expect(mockFetch).toHaveBeenCalledTimes(2);
        expect(result).toEqual({ data: 'success' });
    });

    it('should implement exponential backoff', async () => {
        const delays: number[] = [];
        const originalDelay = global.setTimeout;
        global.setTimeout = jest.fn((cb, delay) => {
            delays.push(delay);
            return originalDelay(cb, delay);
        });

        const mockFetch = jest.fn()
            .mockRejectedValueOnce({ status: 429 })
            .mockRejectedValueOnce({ status: 429 })
            .mockResolvedValueOnce({ ok: true });

        const handler = new RateLimitHandler({ maxRetries: 2, baseDelay: 1000 });
        await handler.executeWithRetry(() => mockFetch());

        expect(delays).toEqual([1000, 2000]); // 1s, then 2s
    });
});
```

### 2. **Integration Testing**

```typescript
describe('Rate Limited API Client', () => {
    it('should handle server rate limiting', async () => {
        // Mock server that returns 429 on first request
        const mockServer = nock('http://api.example.com')
            .get('/data')
            .reply(429, {}, { 'Retry-After': '1' })
            .get('/data')
            .reply(200, { data: 'success' });

        const client = new ApiClient();
        const response = await client.request('http://api.example.com/data');

        expect(response.status).toBe(200);
        const data = await response.json();
        expect(data).toEqual({ data: 'success' });
    });
});
```

## Common Interview Questions

### Q: How do you handle API rate limiting?

**A:** I implement client-side rate limiting using techniques like request queuing with exponential backoff, token bucket algorithms, and request throttling. For React apps, I create custom hooks that manage request frequency and handle 429 responses gracefully with automatic retries.

### Q: What's the difference between throttling and rate limiting?

**A:** Throttling controls the rate of requests on the client side to prevent hitting server limits, while rate limiting is enforced by the server. Throttling is proactive prevention, rate limiting is reactive enforcement.

### Q: How do you implement exponential backoff?

**A:** Exponential backoff starts with a base delay and doubles it with each retry attempt, up to a maximum delay. For example, retry 1: 1s, retry 2: 2s, retry 3: 4s. This prevents thundering herd problems and gives servers time to recover.

### Q: What headers should you check for rate limiting?

**A:** Check `X-RateLimit-Limit` (total allowed), `X-RateLimit-Remaining` (requests left), `X-RateLimit-Reset` (when limit resets), and `Retry-After` (seconds to wait before retrying).

### Q: How do you handle rate limiting in a React application?

**A:** I create a custom hook that wraps fetch calls, implements retry logic with exponential backoff, and provides user feedback. I also use context providers to share rate limiting state across components and implement request batching for efficiency.

## Summary

**Key Strategies:**
1. **Retry with exponential backoff** - Handle 429 responses gracefully
2. **Request throttling** - Control client-side request frequency
3. **Token bucket algorithm** - Allow burst traffic while maintaining limits
4. **Request batching** - Combine multiple requests to reduce API calls
5. **Circuit breaker pattern** - Prevent cascading failures

**React-Specific Approaches:**
- **Custom hooks** for rate limiting logic
- **Context providers** for shared rate limit state
- **Error boundaries** for user-friendly error handling
- **Suspense** for loading states during retries

**Best Practices:**
- **Monitor rate limit metrics** and adjust strategies
- **Implement proper error handling** and user feedback
- **Use request prioritization** for critical operations
- **Test rate limiting scenarios** thoroughly
- **Respect server headers** and implement adaptive behavior

**Interview Tip:** "I handle API rate limiting by implementing client-side throttling to prevent hitting limits, retry logic with exponential backoff for 429 responses, and user feedback during rate-limited periods. In React, I use custom hooks and context to manage rate limiting state across the application."