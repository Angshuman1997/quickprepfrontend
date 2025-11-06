# How to implement API response caching?

## Question
How to implement API response caching?

## Answer

API response caching is crucial for improving application performance by reducing network requests and server load. Effective caching strategies can significantly reduce response times and improve user experience. The key is choosing the right caching layer and invalidation strategy for your use case.

## Browser-Level Caching

### 1. **HTTP Cache Headers**

**Cache-Control header:**
```javascript
// Server response with caching headers
app.get('/api/users', (req, res) => {
    res.set({
        'Cache-Control': 'public, max-age=300', // Cache for 5 minutes
        'ETag': generateETag(usersData),
        'Last-Modified': new Date().toUTCString()
    });
    res.json(usersData);
});

// Client-side fetch with cache control
const response = await fetch('/api/users', {
    headers: {
        'Cache-Control': 'max-age=300' // Respect server cache
    }
});
```

**Common Cache-Control directives:**
```javascript
// No caching
'Cache-Control': 'no-cache, no-store, must-revalidate'

// Short-term caching (5 minutes)
'Cache-Control': 'public, max-age=300'

// Long-term caching with revalidation (1 hour)
'Cache-Control': 'public, max-age=3600, must-revalidate'

// Private caching (user-specific)
'Cache-Control': 'private, max-age=600'
```

### 2. **ETag and Last-Modified**

**ETag-based caching:**
```javascript
// Server generates ETag
const generateETag = (data) => {
    return crypto.createHash('md5').update(JSON.stringify(data)).digest('hex');
};

app.get('/api/data', (req, res) => {
    const data = getData();
    const etag = generateETag(data);

    if (req.headers['if-none-match'] === etag) {
        return res.status(304).end(); // Not modified
    }

    res.set('ETag', etag);
    res.json(data);
});

// Client sends ETag for revalidation
fetch('/api/data', {
    headers: {
        'If-None-Match': storedETag
    }
}).then(response => {
    if (response.status === 304) {
        return cachedData; // Use cached data
    }
    return response.json();
});
```

## Application-Level Caching

### 1. **In-Memory Cache**

**Simple in-memory cache:**
```typescript
class MemoryCache {
    private cache = new Map<string, { data: any; timestamp: number; ttl: number }>();

    set(key: string, data: any, ttl = 300000): void { // 5 minutes default
        this.cache.set(key, {
            data,
            timestamp: Date.now(),
            ttl
        });
    }

    get(key: string): any | null {
        const item = this.cache.get(key);

        if (!item) return null;

        if (Date.now() - item.timestamp > item.ttl) {
            this.cache.delete(key);
            return null;
        }

        return item.data;
    }

    clear(): void {
        this.cache.clear();
    }

    size(): number {
        return this.cache.size;
    }
}

// Usage
const cache = new MemoryCache();

async function fetchWithCache(url: string, ttl = 300000) {
    const cached = cache.get(url);
    if (cached) {
        console.log('Returning cached data');
        return cached;
    }

    const response = await fetch(url);
    const data = await response.json();

    cache.set(url, data, ttl);
    return data;
}
```

**LRU (Least Recently Used) cache:**
```typescript
class LRUCache {
    private cache = new Map<string, any>();
    private maxSize: number;

    constructor(maxSize = 100) {
        this.maxSize = maxSize;
    }

    get(key: string): any | null {
        const value = this.cache.get(key);
        if (value !== undefined) {
            // Move to end (most recently used)
            this.cache.delete(key);
            this.cache.set(key, value);
            return value;
        }
        return null;
    }

    set(key: string, value: any): void {
        if (this.cache.has(key)) {
            this.cache.delete(key);
        } else if (this.cache.size >= this.maxSize) {
            // Remove least recently used (first item)
            const firstKey = this.cache.keys().next().value;
            this.cache.delete(firstKey);
        }

        this.cache.set(key, value);
    }

    clear(): void {
        this.cache.clear();
    }
}
```

### 2. **React Query/SWR for Client-Side Caching**

**React Query implementation:**
```typescript
import { useQuery, useQueryClient, QueryClient, QueryClientProvider } from 'react-query';

const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            staleTime: 5 * 60 * 1000, // 5 minutes
            cacheTime: 10 * 60 * 1000, // 10 minutes
            refetchOnWindowFocus: false,
        },
    },
});

function App() {
    return (
        <QueryClientProvider client={queryClient}>
            <UserList />
        </QueryClientProvider>
    );
}

function UserList() {
    const { data, isLoading, error } = useQuery(
        'users',
        () => fetch('/api/users').then(res => res.json()),
        {
            staleTime: 2 * 60 * 1000, // Consider data fresh for 2 minutes
            cacheTime: 5 * 60 * 1000, // Keep in cache for 5 minutes
        }
    );

    if (isLoading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;

    return (
        <ul>
            {data.map(user => (
                <li key={user.id}>{user.name}</li>
            ))}
        </ul>
    );
}

// Manual cache management
function UpdateUser({ userId }) {
    const queryClient = useQueryClient();

    const updateUser = async (updates) => {
        await fetch(`/api/users/${userId}`, {
            method: 'PATCH',
            body: JSON.stringify(updates)
        });

        // Update cache optimistically
        queryClient.setQueryData(['users', userId], oldData => ({
            ...oldData,
            ...updates
        }));

        // Invalidate related queries
        queryClient.invalidateQueries('users');
    };

    return <button onClick={() => updateUser({ name: 'New Name' })}>Update</button>;
}
```

**SWR implementation:**
```typescript
import useSWR, { mutate } from 'swr';

const fetcher = (url) => fetch(url).then(res => res.json());

function UserProfile({ userId }) {
    const { data, error, mutate: revalidate } = useSWR(
        `/api/users/${userId}`,
        fetcher,
        {
            refreshInterval: 0, // No automatic refresh
            revalidateOnFocus: false,
            dedupingInterval: 2000, // Dedupe requests within 2s
        }
    );

    if (error) return <div>Failed to load</div>;
    if (!data) return <div>Loading...</div>;

    return (
        <div>
            <h1>{data.name}</h1>
            <button onClick={revalidate}>Refresh</button>
        </div>
    );
}

// Global cache mutation
function updateUserGlobally(userId, newData) {
    mutate(`/api/users/${userId}`, newData, false); // Update cache without revalidation
}
```

### 3. **Service Worker Caching**

**Cache API with service worker:**
```javascript
// service-worker.js
const CACHE_NAME = 'api-cache-v1';
const API_CACHE_NAME = 'api-responses-v1';

self.addEventListener('install', (event) => {
    console.log('Service worker installing');
});

self.addEventListener('fetch', (event) => {
    if (event.request.url.includes('/api/')) {
        event.respondWith(cacheApiResponse(event.request));
    }
});

async function cacheApiResponse(request) {
    const cache = await caches.open(API_CACHE_NAME);

    // Try cache first
    const cachedResponse = await cache.match(request);
    if (cachedResponse) {
        // Check if cache is still fresh
        const cacheDate = new Date(cachedResponse.headers.get('sw-cache-date'));
        const age = Date.now() - cacheDate.getTime();

        if (age < 5 * 60 * 1000) { // 5 minutes
            return cachedResponse;
        }
    }

    try {
        // Fetch from network
        const response = await fetch(request);

        if (response.ok) {
            // Clone response for caching
            const responseClone = response.clone();

            // Add cache timestamp
            const headers = new Headers(responseClone.headers);
            headers.set('sw-cache-date', new Date().toISOString());

            const cachedResponse = new Response(responseClone.body, {
                status: responseClone.status,
                statusText: responseClone.statusText,
                headers
            });

            cache.put(request, cachedResponse);
        }

        return response;
    } catch (error) {
        // Network failed, return cached if available
        if (cachedResponse) {
            return cachedResponse;
        }
        throw error;
    }
}

// Cache invalidation
self.addEventListener('message', (event) => {
    if (event.data.type === 'INVALIDATE_CACHE') {
        caches.open(API_CACHE_NAME).then(cache => {
            return cache.delete(event.data.url);
        });
    }
});
```

**Registering service worker:**
```typescript
// In main app
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/service-worker.js')
        .then(registration => {
            console.log('SW registered');
        })
        .catch(error => {
            console.log('SW registration failed');
        });
}

// Invalidate cache from app
function invalidateCache(url: string) {
    if ('serviceWorker' in navigator) {
        navigator.serviceWorker.controller?.postMessage({
            type: 'INVALIDATE_CACHE',
            url
        });
    }
}
```

## Server-Side Caching

### 1. **Redis/Memcached**

**Node.js with Redis:**
```javascript
const redis = require('redis');
const client = redis.createClient();

class RedisCache {
    constructor(redisClient) {
        this.client = redisClient;
    }

    async get(key) {
        const data = await this.client.get(key);
        return data ? JSON.parse(data) : null;
    }

    async set(key, value, ttl = 300) {
        await this.client.setex(key, ttl, JSON.stringify(value));
    }

    async del(key) {
        await this.client.del(key);
    }

    async clear() {
        await this.client.flushdb();
    }
}

// Express middleware
function cacheMiddleware(ttl = 300) {
    return async (req, res, next) => {
        const key = `api:${req.originalUrl}`;

        // Try cache first
        const cached = await cache.get(key);
        if (cached) {
            return res.json(cached);
        }

        // Store original json method
        const originalJson = res.json;
        res.json = function(data) {
            // Cache the response
            cache.set(key, data, ttl);
            // Call original method
            originalJson.call(this, data);
        };

        next();
    };
}

// Usage
app.get('/api/users', cacheMiddleware(600), (req, res) => {
    // Expensive operation
    const users = getUsersFromDatabase();
    res.json(users);
});
```

### 2. **Database Query Caching**

**SQL query result caching:**
```typescript
class QueryCache {
    constructor(dbConnection) {
        this.db = dbConnection;
        this.cache = new Map();
    }

    async query(sql, params = [], ttl = 300) {
        const key = `${sql}:${JSON.stringify(params)}`;

        // Check cache
        const cached = this.cache.get(key);
        if (cached && Date.now() - cached.timestamp < ttl * 1000) {
            return cached.result;
        }

        // Execute query
        const result = await this.db.query(sql, params);

        // Cache result
        this.cache.set(key, {
            result,
            timestamp: Date.now()
        });

        return result;
    }

    invalidate(pattern) {
        for (const [key] of this.cache) {
            if (key.includes(pattern)) {
                this.cache.delete(key);
            }
        }
    }
}

// Usage
const queryCache = new QueryCache(db);

app.get('/api/posts', async (req, res) => {
    const posts = await queryCache.query(
        'SELECT * FROM posts WHERE category = ?',
        [req.query.category],
        600 // 10 minutes
    );
    res.json(posts);
});

// Invalidate on update
app.post('/api/posts', async (req, res) => {
    await db.query('INSERT INTO posts ...');
    queryCache.invalidate('SELECT * FROM posts'); // Clear related cache
    res.json({ success: true });
});
```

## Advanced Caching Patterns

### 1. **Cache-Aside Pattern**

**Lazy loading with cache:**
```typescript
class CacheAsideService {
    constructor(cache, dataSource) {
        this.cache = cache;
        this.dataSource = dataSource;
    }

    async get(key) {
        // Try cache first
        let data = await this.cache.get(key);

        if (!data) {
            // Cache miss - fetch from source
            data = await this.dataSource.get(key);

            // Store in cache
            await this.cache.set(key, data);
        }

        return data;
    }

    async set(key, value) {
        // Update data source
        await this.dataSource.set(key, value);

        // Update cache
        await this.cache.set(key, value);
    }

    async invalidate(key) {
        await this.cache.delete(key);
    }
}
```

### 2. **Write-Through Caching**

**Immediate cache updates:**
```typescript
class WriteThroughCache {
    async set(key, value) {
        // Write to cache and data source simultaneously
        const promises = [
            this.cache.set(key, value),
            this.dataSource.set(key, value)
        ];

        await Promise.all(promises);
    }

    async get(key) {
        // Always try cache first
        return await this.cache.get(key);
    }
}
```

### 3. **Cache Invalidation Strategies**

**Time-based expiration:**
```typescript
class TimeBasedCache {
    set(key, value, ttl = 300000) { // 5 minutes
        const expiration = Date.now() + ttl;
        this.cache.set(key, { value, expiration });
    }

    get(key) {
        const item = this.cache.get(key);

        if (!item) return null;

        if (Date.now() > item.expiration) {
            this.cache.delete(key);
            return null;
        }

        return item.value;
    }
}
```

**Event-based invalidation:**
```typescript
class EventDrivenCache {
    constructor() {
        this.cache = new Map();
        this.listeners = new Map();
    }

    subscribe(event, callback) {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, []);
        }
        this.listeners.get(event).push(callback);
    }

    async invalidate(event, data) {
        // Notify listeners
        const listeners = this.listeners.get(event) || [];
        await Promise.all(listeners.map(listener => listener(data)));

        // Clear related cache entries
        for (const [key, value] of this.cache) {
            if (this.shouldInvalidate(key, event, data)) {
                this.cache.delete(key);
            }
        }
    }

    shouldInvalidate(key, event, data) {
        // Custom logic to determine if key should be invalidated
        if (event === 'user_updated' && key.includes(`user:${data.userId}`)) {
            return true;
        }
        return false;
    }
}
```

## React-Specific Caching

### 1. **Custom Hook for API Caching**

```typescript
import { useState, useEffect, useRef } from 'react';

function useApiCache(url, options = {}) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const cache = useRef(new Map());

    const {
        ttl = 5 * 60 * 1000, // 5 minutes
        enabled = true,
        onSuccess,
        onError
    } = options;

    useEffect(() => {
        if (!enabled) return;

        const fetchData = async () => {
            setLoading(true);
            setError(null);

            // Check cache
            const cacheKey = url;
            const cached = cache.current.get(cacheKey);

            if (cached && Date.now() - cached.timestamp < ttl) {
                setData(cached.data);
                setLoading(false);
                onSuccess?.(cached.data);
                return;
            }

            try {
                const response = await fetch(url);
                if (!response.ok) throw new Error('API Error');

                const result = await response.json();
                setData(result);

                // Cache the result
                cache.current.set(cacheKey, {
                    data: result,
                    timestamp: Date.now()
                });

                onSuccess?.(result);
            } catch (err) {
                setError(err);
                onError?.(err);
            } finally {
                setLoading(false);
            }
        };

        fetchData();
    }, [url, enabled, ttl, onSuccess, onError]);

    const invalidate = () => {
        cache.current.delete(url);
    };

    const refetch = () => {
        cache.current.delete(url);
        // Trigger re-fetch by updating a dependency
        setData(null);
    };

    return { data, loading, error, invalidate, refetch };
}

// Usage
function UserProfile({ userId }) {
    const { data, loading, error, refetch } = useApiCache(
        `/api/users/${userId}`,
        {
            ttl: 10 * 60 * 1000, // 10 minutes
            onSuccess: (data) => console.log('User loaded:', data.name)
        }
    );

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;

    return (
        <div>
            <h1>{data.name}</h1>
            <button onClick={refetch}>Refresh</button>
        </div>
    );
}
```

### 2. **Context-Based Cache Provider**

```typescript
import React, { createContext, useContext, useReducer } from 'react';

interface CacheState {
    [key: string]: {
        data: any;
        timestamp: number;
        ttl: number;
    };
}

type CacheAction =
    | { type: 'SET'; key: string; data: any; ttl: number }
    | { type: 'INVALIDATE'; key: string }
    | { type: 'CLEAR' };

const CacheContext = createContext<{
    state: CacheState;
    dispatch: React.Dispatch<CacheAction>;
} | null>(null);

function cacheReducer(state: CacheState, action: CacheAction): CacheState {
    switch (action.type) {
        case 'SET':
            return {
                ...state,
                [action.key]: {
                    data: action.data,
                    timestamp: Date.now(),
                    ttl: action.ttl
                }
            };
        case 'INVALIDATE':
            const newState = { ...state };
            delete newState[action.key];
            return newState;
        case 'CLEAR':
            return {};
        default:
            return state;
    }
}

export function CacheProvider({ children }: { children: React.ReactNode }) {
    const [state, dispatch] = useReducer(cacheReducer, {});

    return (
        <CacheContext.Provider value={{ state, dispatch }}>
            {children}
        </CacheContext.Provider>
    );
}

export function useCache() {
    const context = useContext(CacheContext);
    if (!context) {
        throw new Error('useCache must be used within CacheProvider');
    }

    const { state, dispatch } = context;

    const get = (key: string) => {
        const item = state[key];
        if (!item) return null;

        if (Date.now() - item.timestamp > item.ttl) {
            dispatch({ type: 'INVALIDATE', key });
            return null;
        }

        return item.data;
    };

    const set = (key: string, data: any, ttl = 5 * 60 * 1000) => {
        dispatch({ type: 'SET', key, data, ttl });
    };

    const invalidate = (key: string) => {
        dispatch({ type: 'INVALIDATE', key });
    };

    const clear = () => {
        dispatch({ type: 'CLEAR' });
    };

    return { get, set, invalidate, clear };
}

// Usage
function CachedUserList() {
    const { get, set } = useCache();
    const [users, setUsers] = useState(null);

    useEffect(() => {
        const cached = get('users');
        if (cached) {
            setUsers(cached);
            return;
        }

        fetch('/api/users')
            .then(res => res.json())
            .then(data => {
                setUsers(data);
                set('users', data, 10 * 60 * 1000); // Cache for 10 minutes
            });
    }, [get, set]);

    return (
        <ul>
            {users?.map(user => (
                <li key={user.id}>{user.name}</li>
            ))}
        </ul>
    );
}
```

## Cache Management Best Practices

### 1. **Cache Key Strategies**

**Consistent key generation:**
```typescript
function generateCacheKey(base, params = {}) {
    // Sort params for consistent keys
    const sortedParams = Object.keys(params)
        .sort()
        .reduce((result, key) => {
            result[key] = params[key];
            return result;
        }, {});

    return `${base}:${JSON.stringify(sortedParams)}`;
}

// Usage
const key1 = generateCacheKey('users', { page: 1, limit: 10 });
const key2 = generateCacheKey('users', { limit: 10, page: 1 });
// Both generate: "users:{"limit":10,"page":1}"
```

### 2. **Cache Size Management**

**Automatic cleanup:**
```typescript
class SelfCleaningCache extends Map {
    private maxSize: number;
    private ttl: number;

    constructor(maxSize = 1000, ttl = 5 * 60 * 1000) {
        super();
        this.maxSize = maxSize;
        this.ttl = ttl;

        // Periodic cleanup
        setInterval(() => this.cleanup(), 60000); // Every minute
    }

    set(key: string, value: any) {
        // Remove oldest entries if at max size
        if (this.size >= this.maxSize) {
            const firstKey = this.keys().next().value;
            this.delete(firstKey);
        }

        return super.set(key, {
            value,
            timestamp: Date.now()
        });
    }

    get(key: string) {
        const item = super.get(key);

        if (!item) return undefined;

        // Check TTL
        if (Date.now() - item.timestamp > this.ttl) {
            this.delete(key);
            return undefined;
        }

        return item.value;
    }

    private cleanup() {
        const now = Date.now();
        for (const [key, item] of this.entries()) {
            if (now - item.timestamp > this.ttl) {
                this.delete(key);
            }
        }
    }
}
```

### 3. **Cache Monitoring**

**Cache hit/miss tracking:**
```typescript
class MonitoredCache {
    private hits = 0;
    private misses = 0;
    private cache = new Map();

    get(key: string) {
        if (this.cache.has(key)) {
            this.hits++;
            return this.cache.get(key);
        }
        this.misses++;
        return null;
    }

    set(key: string, value: any) {
        this.cache.set(key, value);
    }

    getStats() {
        const total = this.hits + this.misses;
        const hitRate = total > 0 ? (this.hits / total) * 100 : 0;

        return {
            hits: this.hits,
            misses: this.misses,
            hitRate: hitRate.toFixed(2) + '%',
            size: this.cache.size
        };
    }

    resetStats() {
        this.hits = 0;
        this.misses = 0;
    }
}

// Usage with logging
setInterval(() => {
    const stats = cache.getStats();
    console.log('Cache stats:', stats);

    if (stats.hitRate < 50) {
        console.warn('Low cache hit rate - consider adjusting TTL or cache size');
    }

    cache.resetStats();
}, 5 * 60 * 1000); // Every 5 minutes
```

## Common Interview Questions

### Q: How do you implement API response caching?

**A:** I implement multiple layers of caching: HTTP caching with Cache-Control headers for browser-level caching, in-memory caching for session data, and React Query/SWR for client-side caching with automatic invalidation. For server-side, I use Redis with appropriate TTL settings and cache invalidation strategies.

### Q: What's the difference between Cache-Control and Expires headers?

**A:** Cache-Control is more flexible and powerful than Expires. Expires sets an absolute expiration date, while Cache-Control uses relative time (max-age) and provides more directives like no-cache, no-store, must-revalidate, and private/public. Cache-Control is preferred in HTTP/1.1.

### Q: How do you handle cache invalidation?

**A:** I use multiple strategies: time-based expiration (TTL), event-driven invalidation (when data changes), and manual invalidation. For React Query, I use invalidateQueries to clear related cache entries. For server-side, I implement cache tags or patterns to invalidate related entries.

### Q: When should you NOT cache API responses?

**A:** Don't cache highly dynamic data, user-specific data that changes frequently, sensitive information, or data that must always be fresh (like real-time stock prices). Also avoid caching error responses or responses with authentication requirements.

### Q: How do you handle cache consistency?

**A:** I ensure cache consistency by implementing proper invalidation strategies, using optimistic updates for better UX, and implementing cache versioning. For critical data, I use cache-aside pattern where cache is updated only when data is requested, ensuring consistency with the data source.

## Summary

**Caching Layers:**
1. **Browser HTTP caching** - Cache-Control, ETags, Last-Modified
2. **Service Worker caching** - Offline capability, custom logic
3. **Application caching** - React Query, SWR, custom hooks
4. **Server-side caching** - Redis, Memcached, database query caching

**Key Strategies:**
- **Cache-Aside** - Lazy loading pattern
- **Write-Through** - Immediate cache updates
- **Time-based expiration** - TTL for automatic cleanup
- **Event-driven invalidation** - Reactive cache updates

**Best Practices:**
- **Monitor cache hit rates** and adjust TTL accordingly
- **Implement proper invalidation** to prevent stale data
- **Use consistent cache keys** for reliable lookups
- **Handle cache failures gracefully** with fallbacks
- **Consider cache size limits** to prevent memory issues

**Interview Tip:** "I implement API response caching using multiple layers: HTTP caching with proper Cache-Control headers for browser-level caching, React Query for client-side caching with automatic background refetching, and Redis for server-side caching. The key is choosing appropriate TTL values and implementing proper cache invalidation strategies."