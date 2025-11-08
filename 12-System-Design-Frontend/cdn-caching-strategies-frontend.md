# CDN and Caching Strategies for Frontend

## Question
How do you implement effective caching strategies for a frontend application at scale?

## Answer

Effective caching is crucial for frontend performance at scale. It involves multiple layers from browser cache to CDN edge locations, each serving different purposes in the caching hierarchy.

## Multi-Layer Caching Architecture

### 1. **Browser Caching Strategy**

```typescript
// Cache-Control Headers Configuration
const cacheStrategies = {
    // Static assets (images, fonts, CSS, JS with hashes)
    staticAssets: {
        'Cache-Control': 'public, max-age=31536000, immutable', // 1 year
        'ETag': 'W/"abc123"',
    },
    
    // HTML pages
    htmlPages: {
        'Cache-Control': 'public, max-age=0, must-revalidate',
        'ETag': 'W/"def456"',
    },
    
    // API responses
    apiData: {
        'Cache-Control': 'private, max-age=300, stale-while-revalidate=86400', // 5min fresh, 24h stale
        'Vary': 'Authorization, Accept-Encoding',
    },
    
    // User-specific content
    userContent: {
        'Cache-Control': 'private, no-cache, no-store, must-revalidate',
        'Pragma': 'no-cache',
        'Expires': '0',
    },
};

// Service Worker Cache Implementation
class ServiceWorkerCache {
    private readonly CACHE_NAME = 'app-cache-v1';
    private readonly STATIC_CACHE_NAME = 'static-cache-v1';
    private readonly API_CACHE_NAME = 'api-cache-v1';

    async install(): Promise<void> {
        const cache = await caches.open(this.STATIC_CACHE_NAME);
        
        // Pre-cache critical resources
        await cache.addAll([
            '/',
            '/static/css/main.css',
            '/static/js/main.js',
            '/static/fonts/main.woff2',
            '/offline.html',
        ]);
    }

    async handleRequest(request: Request): Promise<Response> {
        const url = new URL(request.url);
        
        // Handle different request types
        if (url.pathname.startsWith('/api/')) {
            return this.handleAPIRequest(request);
        } else if (url.pathname.startsWith('/static/')) {
            return this.handleStaticRequest(request);
        } else {
            return this.handlePageRequest(request);
        }
    }

    private async handleStaticRequest(request: Request): Promise<Response> {
        // Cache first strategy for static assets
        const cached = await caches.match(request);
        if (cached) {
            return cached;
        }

        try {
            const response = await fetch(request);
            if (response.ok) {
                const cache = await caches.open(this.STATIC_CACHE_NAME);
                cache.put(request, response.clone());
            }
            return response;
        } catch (error) {
            // Return fallback for critical assets
            return caches.match('/offline.html') || new Response('Offline');
        }
    }

    private async handleAPIRequest(request: Request): Promise<Response> {
        // Stale-while-revalidate strategy for API calls
        const cached = await caches.match(request);
        
        // Always try to fetch fresh data
        const fetchPromise = fetch(request).then(async (response) => {
            if (response.ok) {
                const cache = await caches.open(this.API_CACHE_NAME);
                cache.put(request, response.clone());
            }
            return response;
        });

        // Return cached data immediately if available, otherwise wait for fetch
        return cached || fetchPromise;
    }

    private async handlePageRequest(request: Request): Promise<Response> {
        // Network first strategy for pages
        try {
            const response = await fetch(request);
            if (response.ok) {
                const cache = await caches.open(this.CACHE_NAME);
                cache.put(request, response.clone());
            }
            return response;
        } catch (error) {
            const cached = await caches.match(request);
            return cached || caches.match('/offline.html') || new Response('Offline');
        }
    }
}
```

### 2. **Application-Level Caching**

```typescript
// Intelligent Memory Cache with LRU Eviction
class IntelligentCache<T = any> {
    private cache = new Map<string, CacheEntry<T>>();
    private readonly maxSize: number;
    private readonly defaultTTL: number;

    constructor(options: CacheOptions = {}) {
        this.maxSize = options.maxSize || 100;
        this.defaultTTL = options.defaultTTL || 5 * 60 * 1000; // 5 minutes
    }

    async get(key: string): Promise<T | null> {
        const entry = this.cache.get(key);
        
        if (!entry) {
            return null;
        }

        // Check if expired
        if (Date.now() > entry.expiresAt) {
            this.cache.delete(key);
            return null;
        }

        // Update access time for LRU
        entry.lastAccessed = Date.now();
        
        // Move to end (most recently used)
        this.cache.delete(key);
        this.cache.set(key, entry);

        return entry.value;
    }

    async set(key: string, value: T, ttl?: number): Promise<void> {
        const expiresAt = Date.now() + (ttl || this.defaultTTL);
        
        // Evict if at capacity
        if (this.cache.size >= this.maxSize && !this.cache.has(key)) {
            this.evictLeastRecentlyUsed();
        }

        this.cache.set(key, {
            value,
            expiresAt,
            createdAt: Date.now(),
            lastAccessed: Date.now(),
        });
    }

    private evictLeastRecentlyUsed(): void {
        let oldestKey: string | null = null;
        let oldestTime = Infinity;

        for (const [key, entry] of this.cache) {
            if (entry.lastAccessed < oldestTime) {
                oldestTime = entry.lastAccessed;
                oldestKey = key;
            }
        }

        if (oldestKey) {
            this.cache.delete(oldestKey);
        }
    }

    clear(): void {
        this.cache.clear();
    }

    size(): number {
        return this.cache.size;
    }

    // Cache statistics for monitoring
    getStats(): CacheStats {
        const now = Date.now();
        let hits = 0;
        let expired = 0;

        for (const entry of this.cache.values()) {
            if (now <= entry.expiresAt) {
                hits++;
            } else {
                expired++;
            }
        }

        return {
            size: this.cache.size,
            hits,
            expired,
            hitRate: hits / (hits + expired) || 0,
        };
    }
}

// React Query Integration with Advanced Caching
const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            staleTime: 5 * 60 * 1000, // 5 minutes
            cacheTime: 30 * 60 * 1000, // 30 minutes
            retry: (failureCount, error) => {
                // Don't retry on 4xx errors
                if (error?.response?.status >= 400 && error?.response?.status < 500) {
                    return false;
                }
                return failureCount < 3;
            },
        },
    },
});

// Smart prefetching based on user behavior
function useSmartPrefetch() {
    const queryClient = useQueryClient();

    const prefetchRelatedData = useCallback(async (primaryData: any) => {
        // Prefetch related resources
        const relatedQueries = getRelatedQueries(primaryData);
        
        await Promise.all(
            relatedQueries.map(query =>
                queryClient.prefetchQuery({
                    queryKey: query.key,
                    queryFn: query.fn,
                    staleTime: 10 * 60 * 1000, // Keep prefetched data fresh for 10 minutes
                })
            )
        );
    }, [queryClient]);

    return { prefetchRelatedData };
}

// Intersection Observer for preloading
function usePreloadOnView() {
    const [ref, inView] = useInView({
        triggerOnce: true,
        threshold: 0.1,
    });

    const preload = useCallback((resources: string[]) => {
        if (inView) {
            resources.forEach(resource => {
                const link = document.createElement('link');
                link.rel = 'prefetch';
                link.href = resource;
                document.head.appendChild(link);
            });
        }
    }, [inView]);

    return [ref, preload] as const;
}
```

### 3. **CDN Configuration and Edge Caching**

```typescript
// Next.js Edge Caching Configuration
// next.config.js
module.exports = {
    async headers() {
        return [
            {
                source: '/api/products/:path*',
                headers: [
                    {
                        key: 'Cache-Control',
                        value: 'public, s-maxage=300, stale-while-revalidate=86400',
                    },
                ],
            },
            {
                source: '/static/:path*',
                headers: [
                    {
                        key: 'Cache-Control',
                        value: 'public, max-age=31536000, immutable',
                    },
                ],
            },
        ];
    },

    async rewrites() {
        return [
            {
                source: '/api/:path*',
                destination: 'https://api.example.com/:path*',
            },
        ];
    },
};

// Cloudflare Workers for Advanced Edge Logic
class EdgeCacheWorker {
    async handleRequest(request: Request): Promise<Response> {
        const url = new URL(request.url);
        const cacheKey = this.generateCacheKey(request);
        
        // Check edge cache first
        const cached = await caches.default.match(cacheKey);
        if (cached) {
            // Add cache hit header for debugging
            const response = new Response(cached.body, cached);
            response.headers.set('X-Cache', 'HIT');
            return response;
        }

        // Fetch from origin
        const response = await fetch(request);
        
        if (response.ok && this.shouldCache(request, response)) {
            // Clone response for caching
            const cacheResponse = response.clone();
            
            // Set appropriate cache headers
            cacheResponse.headers.set('Cache-Control', this.getCacheControl(url));
            cacheResponse.headers.set('X-Cache', 'MISS');
            
            // Cache at the edge
            await caches.default.put(cacheKey, cacheResponse);
        }

        return response;
    }

    private generateCacheKey(request: Request): string {
        const url = new URL(request.url);
        
        // Include relevant headers in cache key
        const varyHeaders = ['Accept-Language', 'Accept-Encoding'];
        const headerParts = varyHeaders
            .map(header => `${header}:${request.headers.get(header) || ''}`)
            .join(',');

        return `${url.pathname}${url.search}|${headerParts}`;
    }

    private shouldCache(request: Request, response: Response): boolean {
        // Don't cache errors or non-GET requests
        if (!response.ok || request.method !== 'GET') {
            return false;
        }

        // Don't cache personalized content
        const url = new URL(request.url);
        if (url.pathname.includes('/user/') || url.pathname.includes('/account/')) {
            return false;
        }

        return true;
    }

    private getCacheControl(url: URL): string {
        if (url.pathname.startsWith('/api/')) {
            return 'public, s-maxage=300, stale-while-revalidate=3600';
        } else if (url.pathname.startsWith('/static/')) {
            return 'public, max-age=31536000, immutable';
        } else {
            return 'public, s-maxage=60, stale-while-revalidate=300';
        }
    }
}

// Geographic Cache Distribution
class GeoDistributedCache {
    private regions = [
        { name: 'us-east-1', endpoint: 'https://us-east-1.api.example.com' },
        { name: 'eu-west-1', endpoint: 'https://eu-west-1.api.example.com' },
        { name: 'ap-southeast-1', endpoint: 'https://ap-southeast-1.api.example.com' },
    ];

    async request(path: string, options: RequestOptions = {}): Promise<Response> {
        const nearestRegion = await this.getNearestRegion();
        const endpoint = nearestRegion.endpoint;
        
        try {
            // Try nearest region first
            return await fetch(`${endpoint}${path}`, options);
        } catch (error) {
            // Fallback to other regions
            return await this.fallbackRequest(path, options, nearestRegion.name);
        }
    }

    private async getNearestRegion(): Promise<{ name: string; endpoint: string }> {
        // Use geolocation API or CloudFlare headers to determine region
        const userRegion = await this.getUserRegion();
        
        const regionMap = {
            'US': 'us-east-1',
            'CA': 'us-east-1',
            'GB': 'eu-west-1',
            'DE': 'eu-west-1',
            'SG': 'ap-southeast-1',
            'JP': 'ap-southeast-1',
        };

        const preferredRegion = regionMap[userRegion] || 'us-east-1';
        return this.regions.find(r => r.name === preferredRegion) || this.regions[0];
    }

    private async getUserRegion(): Promise<string> {
        // Try CloudFlare header first
        const cfCountry = self.request?.headers?.get?.('CF-IPCountry');
        if (cfCountry) return cfCountry;

        // Fallback to geolocation API
        return new Promise((resolve) => {
            navigator.geolocation.getCurrentPosition(
                (position) => {
                    // Convert coordinates to country code
                    resolve(this.coordinatesToCountry(position.coords));
                },
                () => resolve('US') // Default fallback
            );
        });
    }
}
```

### 4. **Database and API Response Caching**

```typescript
// Redis Cache Integration
class ApiCacheLayer {
    private redis: RedisClient;
    private localCache: IntelligentCache;

    constructor() {
        this.redis = new RedisClient({ url: process.env.REDIS_URL });
        this.localCache = new IntelligentCache({ maxSize: 1000 });
    }

    async get<T>(key: string): Promise<T | null> {
        // L1 Cache: Memory
        let result = await this.localCache.get(key);
        if (result) {
            return result;
        }

        // L2 Cache: Redis
        const redisData = await this.redis.get(key);
        if (redisData) {
            result = JSON.parse(redisData);
            // Populate L1 cache
            await this.localCache.set(key, result);
            return result;
        }

        return null;
    }

    async set<T>(key: string, value: T, ttl: number = 300): Promise<void> {
        // Set in both caches
        await Promise.all([
            this.localCache.set(key, value, ttl * 1000),
            this.redis.setex(key, ttl, JSON.stringify(value)),
        ]);
    }

    async invalidate(pattern: string): Promise<void> {
        // Invalidate pattern-based keys
        const keys = await this.redis.keys(pattern);
        
        if (keys.length > 0) {
            await this.redis.del(...keys);
        }

        // Clear local cache
        this.localCache.clear();
    }

    // Cache warming
    async warmCache(warmupTasks: WarmupTask[]): Promise<void> {
        const promises = warmupTasks.map(async (task) => {
            try {
                const data = await task.fetchData();
                await this.set(task.key, data, task.ttl);
            } catch (error) {
                console.error(`Failed to warm cache for ${task.key}:`, error);
            }
        });

        await Promise.all(promises);
    }
}

// Intelligent Cache Invalidation
class SmartCacheInvalidator {
    private dependencies = new Map<string, Set<string>>();

    // Register cache dependencies
    registerDependency(cacheKey: string, dependsOn: string[]): void {
        dependsOn.forEach(dep => {
            if (!this.dependencies.has(dep)) {
                this.dependencies.set(dep, new Set());
            }
            this.dependencies.get(dep)!.add(cacheKey);
        });
    }

    // Invalidate cache and all dependents
    async invalidateCascade(key: string): Promise<void> {
        const toInvalidate = new Set<string>([key]);
        const visited = new Set<string>();

        // Build invalidation tree
        const addDependents = (currentKey: string) => {
            if (visited.has(currentKey)) return;
            visited.add(currentKey);

            const dependents = this.dependencies.get(currentKey);
            if (dependents) {
                dependents.forEach(dependent => {
                    toInvalidate.add(dependent);
                    addDependents(dependent);
                });
            }
        };

        addDependents(key);

        // Invalidate all keys
        await Promise.all(
            Array.from(toInvalidate).map(k => cache.invalidate(k))
        );
    }
}

// Usage example
const cacheInvalidator = new SmartCacheInvalidator();

// Set up dependencies
cacheInvalidator.registerDependency('user-profile-123', ['user-123']);
cacheInvalidator.registerDependency('user-posts-123', ['user-123']);
cacheInvalidator.registerDependency('user-friends-123', ['user-123']);

// When user data changes, invalidate all related caches
await cacheInvalidator.invalidateCascade('user-123');
```

## Cache Performance Optimization

### 1. **Cache Preloading Strategies**

```typescript
// Route-based Preloading
function RoutePreloader() {
    const router = useRouter();
    const queryClient = useQueryClient();

    useEffect(() => {
        const preloadRouteData = async (route: string) => {
            const routeConfig = getRouteConfig(route);
            
            // Preload critical data for the route
            await Promise.all(
                routeConfig.criticalQueries.map(query =>
                    queryClient.prefetchQuery({
                        queryKey: query.key,
                        queryFn: query.fn,
                        staleTime: 10 * 60 * 1000,
                    })
                )
            );
        };

        // Preload on route hover
        const handleLinkHover = (event: MouseEvent) => {
            const link = event.target as HTMLAnchorElement;
            if (link.tagName === 'A' && link.href) {
                const route = new URL(link.href).pathname;
                preloadRouteData(route);
            }
        };

        // Preload on route focus
        const handleLinkFocus = (event: FocusEvent) => {
            const link = event.target as HTMLAnchorElement;
            if (link.tagName === 'A' && link.href) {
                const route = new URL(link.href).pathname;
                preloadRouteData(route);
            }
        };

        document.addEventListener('mouseover', handleLinkHover);
        document.addEventListener('focus', handleLinkFocus, true);

        return () => {
            document.removeEventListener('mouseover', handleLinkHover);
            document.removeEventListener('focus', handleLinkFocus, true);
        };
    }, [queryClient]);

    return null;
}

// Predictive Preloading
class PredictiveCache {
    private userBehavior: UserBehaviorTracker;
    private mlModel: PreloadingModel;

    constructor() {
        this.userBehavior = new UserBehaviorTracker();
        this.mlModel = new PreloadingModel();
    }

    async predictAndPreload(): Promise<void> {
        const currentContext = {
            route: window.location.pathname,
            time: new Date().getHours(),
            dayOfWeek: new Date().getDay(),
            userHistory: this.userBehavior.getRecentActions(),
        };

        const predictions = await this.mlModel.predict(currentContext);
        
        // Preload top predictions
        const topPredictions = predictions
            .filter(p => p.confidence > 0.7)
            .slice(0, 3);

        await Promise.all(
            topPredictions.map(prediction => 
                this.preloadResource(prediction.resource)
            )
        );
    }

    private async preloadResource(resource: string): Promise<void> {
        const link = document.createElement('link');
        link.rel = 'prefetch';
        link.href = resource;
        document.head.appendChild(link);
    }
}
```

### 2. **Cache Monitoring and Analytics**

```typescript
// Cache Performance Monitor
class CacheMonitor {
    private metrics = {
        hits: 0,
        misses: 0,
        errors: 0,
        totalRequests: 0,
    };

    recordHit(cacheLayer: string, key: string): void {
        this.metrics.hits++;
        this.metrics.totalRequests++;
        
        this.sendMetric('cache_hit', {
            layer: cacheLayer,
            key: this.sanitizeKey(key),
            hitRate: this.getHitRate(),
        });
    }

    recordMiss(cacheLayer: string, key: string): void {
        this.metrics.misses++;
        this.metrics.totalRequests++;
        
        this.sendMetric('cache_miss', {
            layer: cacheLayer,
            key: this.sanitizeKey(key),
            hitRate: this.getHitRate(),
        });
    }

    recordError(cacheLayer: string, key: string, error: Error): void {
        this.metrics.errors++;
        this.metrics.totalRequests++;
        
        this.sendMetric('cache_error', {
            layer: cacheLayer,
            key: this.sanitizeKey(key),
            error: error.message,
        });
    }

    getHitRate(): number {
        return this.metrics.hits / (this.metrics.hits + this.metrics.misses) || 0;
    }

    private sanitizeKey(key: string): string {
        // Remove sensitive data from cache keys for logging
        return key.replace(/user-\d+/g, 'user-***').replace(/token-\w+/g, 'token-***');
    }

    private sendMetric(event: string, data: Record<string, any>): void {
        analytics.track(event, {
            ...data,
            timestamp: Date.now(),
        });
    }

    // Generate cache performance report
    generateReport(): CacheReport {
        return {
            hitRate: this.getHitRate(),
            totalRequests: this.metrics.totalRequests,
            breakdown: {
                hits: this.metrics.hits,
                misses: this.metrics.misses,
                errors: this.metrics.errors,
            },
            recommendations: this.generateRecommendations(),
        };
    }

    private generateRecommendations(): string[] {
        const recommendations: string[] = [];
        const hitRate = this.getHitRate();

        if (hitRate < 0.7) {
            recommendations.push('Consider increasing cache TTL or improving cache key strategies');
        }

        if (this.metrics.errors > this.metrics.totalRequests * 0.1) {
            recommendations.push('High cache error rate detected, investigate cache infrastructure');
        }

        return recommendations;
    }
}

// Real-time Cache Dashboard
function CacheDashboard() {
    const [cacheStats, setCacheStats] = useState<CacheStats | null>(null);
    const [refreshInterval, setRefreshInterval] = useState(5000);

    useEffect(() => {
        const updateStats = async () => {
            const stats = await cacheMonitor.generateReport();
            setCacheStats(stats);
        };

        updateStats();
        const interval = setInterval(updateStats, refreshInterval);

        return () => clearInterval(interval);
    }, [refreshInterval]);

    if (!cacheStats) {
        return <div>Loading cache statistics...</div>;
    }

    return (
        <div className="cache-dashboard">
            <div className="metric-card">
                <h3>Hit Rate</h3>
                <div className={`metric-value ${cacheStats.hitRate > 0.8 ? 'good' : 'warning'}`}>
                    {(cacheStats.hitRate * 100).toFixed(1)}%
                </div>
            </div>
            
            <div className="metric-card">
                <h3>Total Requests</h3>
                <div className="metric-value">
                    {cacheStats.totalRequests.toLocaleString()}
                </div>
            </div>

            <div className="recommendations">
                <h3>Recommendations</h3>
                {cacheStats.recommendations.map((rec, index) => (
                    <div key={index} className="recommendation">
                        {rec}
                    </div>
                ))}
            </div>
        </div>
    );
}
```

## Interview Questions & Answers

### Q: How do you implement caching for a global application?

**A:** I implement multi-layer caching: browser cache with proper Cache-Control headers, CDN edge caching for static assets and API responses, application-level memory cache with LRU eviction, Redis for shared cache across instances, and service workers for offline capabilities. I use geographic distribution and cache warming strategies for optimal performance.

### Q: How do you handle cache invalidation at scale?

**A:** I use smart invalidation with dependency tracking - when core data changes, I cascade invalidate all dependent cache entries. I implement time-based TTL with stale-while-revalidate for graceful updates, tag-based invalidation for grouped content, and event-driven invalidation across microservices. I also use versioned cache keys for breaking changes.

### Q: What's your strategy for CDN optimization?

**A:** I configure different cache policies per content type: long-term caching for static assets with immutable headers, short-term with stale-while-revalidate for API responses, and no-cache for user-specific content. I use edge workers for advanced logic, geographic distribution for global users, and HTTP/2 push for critical resources.

### Q: How do you monitor cache performance?

**A:** I track cache hit rates across all layers, monitor response times and error rates, implement real-time dashboards for cache metrics, set up alerts for performance degradation, and use distributed tracing to identify cache bottlenecks. I also analyze user behavior patterns for predictive preloading.

### Interview Tip:
*"Effective caching requires understanding the data access patterns and user behavior. I implement progressive caching strategies from browser to edge to origin, with intelligent invalidation and comprehensive monitoring to ensure optimal performance while maintaining data freshness."*