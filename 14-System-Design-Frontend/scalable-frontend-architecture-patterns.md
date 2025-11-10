# Scalable Frontend Architecture Patterns

## Question
How do you design a scalable frontend architecture for a large application with millions of users?

## Answer

Designing scalable frontend architecture requires careful consideration of performance, maintainability, team collaboration, and user experience. Here's a comprehensive approach for handling large-scale applications.

## Core Architecture Principles

### 1. **Modular Architecture**

**Micro-Frontend Pattern:**
```typescript
// Shell Application (Host)
// webpack.config.js
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
    plugins: [
        new ModuleFederationPlugin({
            name: 'shell',
            remotes: {
                'user-management': 'userManagement@http://localhost:3001/remoteEntry.js',
                'product-catalog': 'productCatalog@http://localhost:3002/remoteEntry.js',
                'checkout': 'checkout@http://localhost:3003/remoteEntry.js',
            },
        }),
    ],
};

// App.tsx
const UserManagement = React.lazy(() => import('user-management/UserDashboard'));
const ProductCatalog = React.lazy(() => import('product-catalog/ProductGrid'));
const Checkout = React.lazy(() => import('checkout/CheckoutFlow'));

function App() {
    return (
        <Router>
            <Suspense fallback={<PageLoader />}>
                <Routes>
                    <Route path="/users/*" element={<UserManagement />} />
                    <Route path="/products/*" element={<ProductCatalog />} />
                    <Route path="/checkout/*" element={<Checkout />} />
                </Routes>
            </Suspense>
        </Router>
    );
}
```

**Feature-Based Architecture:**
```typescript
// Project Structure
src/
├── shared/
│   ├── components/
│   ├── hooks/
│   ├── utils/
│   └── types/
├── features/
│   ├── authentication/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   └── types/
│   ├── user-profile/
│   └── product-catalog/
└── app/
    ├── store/
    ├── routing/
    └── providers/

// Feature Module Structure
// features/authentication/index.ts
export { LoginForm } from './components/LoginForm';
export { useAuth } from './hooks/useAuth';
export { authService } from './services/authService';
export type { AuthUser } from './types/auth.types';

// Barrel exports for clean imports
import { LoginForm, useAuth, authService } from '@/features/authentication';
```

### 2. **Layer-Based Architecture**

```typescript
// Presentation Layer
interface UserProfileProps {
    userId: string;
}

function UserProfile({ userId }: UserProfileProps) {
    const { user, loading, error } = useUserProfile(userId);
    
    if (loading) return <UserProfileSkeleton />;
    if (error) return <ErrorBoundary error={error} />;
    
    return <UserProfileView user={user} />;
}

// Business Logic Layer
function useUserProfile(userId: string) {
    return useQuery({
        queryKey: ['user-profile', userId],
        queryFn: () => userService.getProfile(userId),
        staleTime: 5 * 60 * 1000, // 5 minutes
        cacheTime: 10 * 60 * 1000, // 10 minutes
    });
}

// Data Access Layer
class UserService {
    private apiClient: ApiClient;
    private cache: Cache;

    async getProfile(userId: string): Promise<User> {
        const cacheKey = `user-profile-${userId}`;
        
        // Check cache first
        const cached = await this.cache.get(cacheKey);
        if (cached) return cached;

        // Fetch from API
        const user = await this.apiClient.get(`/users/${userId}`);
        
        // Cache the result
        await this.cache.set(cacheKey, user, { ttl: 300 });
        
        return user;
    }

    async updateProfile(userId: string, updates: Partial<User>): Promise<User> {
        const user = await this.apiClient.patch(`/users/${userId}`, updates);
        
        // Invalidate cache
        await this.cache.delete(`user-profile-${userId}`);
        
        return user;
    }
}
```

## Scalability Patterns

### 1. **Code Splitting and Lazy Loading**

```typescript
// Route-based Code Splitting
const Home = lazy(() => import('../pages/Home'));
const Products = lazy(() => import('../pages/Products'));
const UserDashboard = lazy(() => 
    import('../pages/UserDashboard').then(module => ({
        default: module.UserDashboard
    }))
);

// Component-based Code Splitting
const HeavyChart = lazy(() => 
    import('../components/HeavyChart').then(module => ({
        default: module.HeavyChart
    }))
);

function AnalyticsDashboard() {
    const [showChart, setShowChart] = useState(false);

    return (
        <div>
            <button onClick={() => setShowChart(true)}>
                Load Chart
            </button>
            
            {showChart && (
                <Suspense fallback={<ChartSkeleton />}>
                    <HeavyChart />
                </Suspense>
            )}
        </div>
    );
}

// Dynamic Imports with Error Handling
async function loadFeature(featureName: string) {
    try {
        const module = await import(`../features/${featureName}`);
        return module.default;
    } catch (error) {
        console.error(`Failed to load feature: ${featureName}`, error);
        // Fallback to basic version
        return import('../features/fallback');
    }
}
```

### 2. **Performance Optimization Patterns**

```typescript
// Virtual Scrolling for Large Lists
import { FixedSizeList as List } from 'react-window';

function VirtualizedProductList({ products }: { products: Product[] }) {
    const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
        <div style={style}>
            <ProductCard product={products[index]} />
        </div>
    );

    return (
        <List
            height={600}
            itemCount={products.length}
            itemSize={120}
            width="100%"
        >
            {Row}
        </List>
    );
}

// Intersection Observer for Lazy Loading
function LazyImage({ src, alt, ...props }: ImageProps) {
    const [isLoaded, setIsLoaded] = useState(false);
    const [isInView, setIsInView] = useState(false);
    const imgRef = useRef<HTMLImageElement>(null);

    useEffect(() => {
        const observer = new IntersectionObserver(
            ([entry]) => {
                if (entry.isIntersecting) {
                    setIsInView(true);
                    observer.disconnect();
                }
            },
            { threshold: 0.1 }
        );

        if (imgRef.current) {
            observer.observe(imgRef.current);
        }

        return () => observer.disconnect();
    }, []);

    return (
        <div ref={imgRef} {...props}>
            {isInView && (
                <img
                    src={src}
                    alt={alt}
                    onLoad={() => setIsLoaded(true)}
                    style={{
                        opacity: isLoaded ? 1 : 0,
                        transition: 'opacity 0.3s ease-in-out'
                    }}
                />
            )}
        </div>
    );
}

// Memoization Strategies
const expensiveSelector = createSelector(
    (state: RootState) => state.products.items,
    (state: RootState) => state.filters.category,
    (state: RootState) => state.filters.priceRange,
    (products, category, priceRange) => {
        return products.filter(product => {
            if (category && product.category !== category) return false;
            if (priceRange && (product.price < priceRange.min || product.price > priceRange.max)) return false;
            return true;
        });
    }
);
```

### 3. **Caching Strategies**

```typescript
// Multi-Level Caching
class CacheManager {
    private memoryCache = new Map();
    private sessionCache = window.sessionStorage;
    private persistentCache = window.localStorage;
    private serviceWorkerCache?: Cache;

    constructor() {
        this.initServiceWorkerCache();
    }

    async get<T>(key: string, options: CacheOptions = {}): Promise<T | null> {
        const { level = 'memory', ttl = 3600 } = options;

        // 1. Check memory cache first (fastest)
        if (level === 'memory' || level === 'all') {
            const memoryData = this.memoryCache.get(key);
            if (memoryData && !this.isExpired(memoryData)) {
                return memoryData.value;
            }
        }

        // 2. Check session cache
        if (level === 'session' || level === 'all') {
            const sessionData = this.sessionCache.getItem(key);
            if (sessionData) {
                const parsed = JSON.parse(sessionData);
                if (!this.isExpired(parsed)) {
                    // Promote to memory cache
                    this.memoryCache.set(key, parsed);
                    return parsed.value;
                }
            }
        }

        // 3. Check persistent cache
        if (level === 'persistent' || level === 'all') {
            const persistentData = this.persistentCache.getItem(key);
            if (persistentData) {
                const parsed = JSON.parse(persistentData);
                if (!this.isExpired(parsed)) {
                    return parsed.value;
                }
            }
        }

        // 4. Check service worker cache
        if (this.serviceWorkerCache) {
            const response = await this.serviceWorkerCache.match(key);
            if (response) {
                return await response.json();
            }
        }

        return null;
    }

    async set<T>(key: string, value: T, options: CacheOptions = {}): Promise<void> {
        const { level = 'memory', ttl = 3600 } = options;
        const cacheEntry = {
            value,
            timestamp: Date.now(),
            ttl: ttl * 1000
        };

        if (level === 'memory' || level === 'all') {
            this.memoryCache.set(key, cacheEntry);
        }

        if (level === 'session' || level === 'all') {
            this.sessionCache.setItem(key, JSON.stringify(cacheEntry));
        }

        if (level === 'persistent' || level === 'all') {
            this.persistentCache.setItem(key, JSON.stringify(cacheEntry));
        }
    }

    private isExpired(entry: CacheEntry): boolean {
        return Date.now() - entry.timestamp > entry.ttl;
    }

    private async initServiceWorkerCache(): Promise<void> {
        if ('caches' in window) {
            this.serviceWorkerCache = await caches.open('app-cache-v1');
        }
    }
}

// HTTP Caching Strategy
class ApiClient {
    private cacheManager = new CacheManager();

    async get<T>(url: string, options: RequestOptions = {}): Promise<T> {
        const { cache = 'default', freshness = 300 } = options;
        const cacheKey = `api:${url}:${JSON.stringify(options.params || {})}`;

        // Try cache first
        if (cache !== 'no-cache') {
            const cached = await this.cacheManager.get<T>(cacheKey);
            if (cached) {
                // Background refresh if stale
                if (cache === 'stale-while-revalidate') {
                    this.backgroundRefresh(url, options, cacheKey);
                }
                return cached;
            }
        }

        // Fetch fresh data
        const response = await fetch(url, {
            ...options,
            headers: {
                'Cache-Control': `max-age=${freshness}`,
                ...options.headers,
            },
        });

        const data = await response.json();

        // Cache the result
        if (cache !== 'no-store') {
            await this.cacheManager.set(cacheKey, data, {
                ttl: freshness,
                level: 'all'
            });
        }

        return data;
    }

    private async backgroundRefresh(url: string, options: RequestOptions, cacheKey: string): Promise<void> {
        try {
            const fresh = await this.get(url, { ...options, cache: 'no-cache' });
            await this.cacheManager.set(cacheKey, fresh);
        } catch (error) {
            console.warn('Background refresh failed:', error);
        }
    }
}
```

## State Management at Scale

### 1. **Distributed State Architecture**

```typescript
// Global State (Redux Toolkit)
// store/index.ts
const store = configureStore({
    reducer: {
        auth: authSlice.reducer,
        ui: uiSlice.reducer,
        api: apiSlice.reducer,
    },
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware({
            serializableCheck: {
                ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
            },
        }).concat(apiSlice.middleware),
});

// Feature-Specific State (Zustand)
interface ProductStore {
    products: Product[];
    filters: ProductFilters;
    setProducts: (products: Product[]) => void;
    updateFilters: (filters: Partial<ProductFilters>) => void;
    clearFilters: () => void;
}

const useProductStore = create<ProductStore>((set, get) => ({
    products: [],
    filters: initialFilters,
    setProducts: (products) => set({ products }),
    updateFilters: (newFilters) => set(state => ({
        filters: { ...state.filters, ...newFilters }
    })),
    clearFilters: () => set({ filters: initialFilters }),
}));

// Component-Local State (useState/useReducer)
function ProductFilters() {
    const [tempFilters, setTempFilters] = useState<ProductFilters>(initialFilters);
    const updateFilters = useProductStore(state => state.updateFilters);

    const applyFilters = useCallback(() => {
        updateFilters(tempFilters);
    }, [tempFilters, updateFilters]);

    return (
        <FilterForm
            filters={tempFilters}
            onChange={setTempFilters}
            onApply={applyFilters}
        />
    );
}

// Server State (React Query)
function useProductsWithFilters() {
    const filters = useProductStore(state => state.filters);
    
    return useInfiniteQuery({
        queryKey: ['products', filters],
        queryFn: ({ pageParam = 0 }) =>
            productApi.getProducts({ ...filters, page: pageParam }),
        getNextPageParam: (lastPage) => lastPage.hasMore ? lastPage.nextPage : undefined,
        staleTime: 5 * 60 * 1000,
    });
}
```

### 2. **Event-Driven Architecture**

```typescript
// Event Bus for Cross-Feature Communication
class EventBus {
    private listeners: Map<string, Set<Function>> = new Map();

    emit<T = any>(event: string, data?: T): void {
        const eventListeners = this.listeners.get(event);
        if (eventListeners) {
            eventListeners.forEach(listener => {
                try {
                    listener(data);
                } catch (error) {
                    console.error(`Error in event listener for ${event}:`, error);
                }
            });
        }
    }

    on<T = any>(event: string, listener: (data?: T) => void): () => void {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, new Set());
        }
        
        this.listeners.get(event)!.add(listener);

        // Return unsubscribe function
        return () => {
            this.listeners.get(event)?.delete(listener);
        };
    }

    once<T = any>(event: string, listener: (data?: T) => void): void {
        const unsubscribe = this.on(event, (data) => {
            listener(data);
            unsubscribe();
        });
    }
}

// Usage in React
const eventBus = new EventBus();

function useEventBus() {
    const emit = useCallback((event: string, data?: any) => {
        eventBus.emit(event, data);
    }, []);

    const on = useCallback((event: string, listener: Function) => {
        return eventBus.on(event, listener);
    }, []);

    return { emit, on };
}

// Feature Integration
function ShoppingCart() {
    const { emit } = useEventBus();

    const addToCart = (product: Product) => {
        // Add to cart logic
        cartService.addItem(product);
        
        // Emit event for other features
        emit('cart:item-added', { product, cartTotal: cartService.getTotal() });
    };

    return <CartInterface onAddItem={addToCart} />;
}

function UserRecommendations() {
    const { on } = useEventBus();
    const [recommendations, setRecommendations] = useState<Product[]>([]);

    useEffect(() => {
        const unsubscribe = on('cart:item-added', ({ product }) => {
            // Update recommendations based on cart changes
            recommendationService.updateBasedOnCartItem(product)
                .then(setRecommendations);
        });

        return unsubscribe;
    }, [on]);

    return <RecommendationsList recommendations={recommendations} />;
}
```

## Error Handling and Resilience

### 1. **Comprehensive Error Boundaries**

```typescript
// Global Error Boundary
interface ErrorBoundaryState {
    hasError: boolean;
    error?: Error;
    errorInfo?: ErrorInfo;
    errorId?: string;
}

class GlobalErrorBoundary extends Component<PropsWithChildren, ErrorBoundaryState> {
    private errorReportingService = new ErrorReportingService();

    state: ErrorBoundaryState = {
        hasError: false,
    };

    static getDerivedStateFromError(error: Error): Partial<ErrorBoundaryState> {
        return {
            hasError: true,
            error,
            errorId: nanoid(),
        };
    }

    componentDidCatch(error: Error, errorInfo: ErrorInfo) {
        this.setState({ errorInfo });

        // Report error to monitoring service
        this.errorReportingService.reportError({
            error,
            errorInfo,
            errorId: this.state.errorId!,
            userAgent: navigator.userAgent,
            url: window.location.href,
            userId: getCurrentUserId(),
            timestamp: new Date().toISOString(),
        });
    }

    render() {
        if (this.state.hasError) {
            return (
                <ErrorFallback
                    error={this.state.error}
                    errorId={this.state.errorId}
                    onRetry={() => this.setState({ hasError: false })}
                    onReport={() => this.reportUserFeedback()}
                />
            );
        }

        return this.props.children;
    }

    private reportUserFeedback(): void {
        // Collect user feedback about the error
        const feedback = prompt('Please describe what you were doing when this error occurred:');
        if (feedback) {
            this.errorReportingService.reportUserFeedback({
                errorId: this.state.errorId!,
                feedback,
                timestamp: new Date().toISOString(),
            });
        }
    }
}

// Feature-Specific Error Boundaries
function withErrorBoundary<T extends object>(
    Component: ComponentType<T>,
    fallback?: ComponentType<{ error: Error; retry: () => void }>
) {
    return function WrappedComponent(props: T) {
        return (
            <ErrorBoundary
                FallbackComponent={fallback || DefaultErrorFallback}
                onError={(error, errorInfo) => {
                    console.error('Feature error:', error, errorInfo);
                }}
            >
                <Component {...props} />
            </ErrorBoundary>
        );
    };
}

// Usage
const ProductListWithErrorBoundary = withErrorBoundary(
    ProductList,
    ProductListErrorFallback
);
```

### 2. **Network Resilience**

```typescript
// Retry Logic with Exponential Backoff
class RetryableRequest {
    async request<T>(
        fn: () => Promise<T>,
        options: RetryOptions = {}
    ): Promise<T> {
        const {
            maxRetries = 3,
            baseDelay = 1000,
            maxDelay = 10000,
            backoffFactor = 2,
        } = options;

        let lastError: Error;

        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                return await fn();
            } catch (error) {
                lastError = error as Error;

                if (attempt === maxRetries) {
                    throw error;
                }

                // Check if error is retryable
                if (!this.isRetryableError(error)) {
                    throw error;
                }

                // Calculate delay with jitter
                const delay = Math.min(
                    baseDelay * Math.pow(backoffFactor, attempt),
                    maxDelay
                );
                const jitter = Math.random() * 0.1 * delay;

                await this.sleep(delay + jitter);
            }
        }

        throw lastError!;
    }

    private isRetryableError(error: any): boolean {
        // Network errors
        if (error.name === 'NetworkError') return true;
        
        // HTTP errors that might be temporary
        if (error.response?.status >= 500) return true;
        if (error.response?.status === 429) return true; // Rate limited
        
        return false;
    }

    private sleep(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Offline Support
class OfflineManager {
    private queue: QueuedRequest[] = [];
    private isOnline = navigator.onLine;

    constructor() {
        window.addEventListener('online', this.handleOnline);
        window.addEventListener('offline', this.handleOffline);
    }

    async request<T>(
        request: () => Promise<T>,
        options: OfflineOptions = {}
    ): Promise<T> {
        const { queueWhenOffline = true, showOfflineMessage = true } = options;

        if (!this.isOnline && queueWhenOffline) {
            return this.queueRequest(request, options);
        }

        try {
            return await request();
        } catch (error) {
            if (this.isNetworkError(error) && queueWhenOffline) {
                return this.queueRequest(request, options);
            }
            throw error;
        }
    }

    private queueRequest<T>(
        request: () => Promise<T>,
        options: OfflineOptions
    ): Promise<T> {
        return new Promise((resolve, reject) => {
            this.queue.push({
                request,
                resolve,
                reject,
                options,
                timestamp: Date.now(),
            });

            if (options.showOfflineMessage) {
                this.showOfflineNotification();
            }
        });
    }

    private handleOnline = async () => {
        this.isOnline = true;
        
        // Process queued requests
        const queuedRequests = [...this.queue];
        this.queue = [];

        for (const queuedRequest of queuedRequests) {
            try {
                const result = await queuedRequest.request();
                queuedRequest.resolve(result);
            } catch (error) {
                queuedRequest.reject(error);
            }
        }
    };

    private handleOffline = () => {
        this.isOnline = false;
    };

    private isNetworkError(error: any): boolean {
        return error.name === 'NetworkError' || error.code === 'NETWORK_ERROR';
    }

    private showOfflineNotification(): void {
        // Show user-friendly offline message
        toast.info('You are offline. Your changes will be saved when connection is restored.');
    }
}
```

## Monitoring and Analytics

### 1. **Performance Monitoring**

```typescript
// Core Web Vitals Tracking
class PerformanceMonitor {
    private observer?: PerformanceObserver;

    init(): void {
        this.trackCoreWebVitals();
        this.trackCustomMetrics();
        this.trackUserInteractions();
    }

    private trackCoreWebVitals(): void {
        // Largest Contentful Paint (LCP)
        this.observeEntries('largest-contentful-paint', (entries) => {
            const lastEntry = entries[entries.length - 1];
            this.sendMetric('lcp', lastEntry.startTime);
        });

        // First Input Delay (FID)
        this.observeEntries('first-input', (entries) => {
            entries.forEach(entry => {
                this.sendMetric('fid', entry.processingStart - entry.startTime);
            });
        });

        // Cumulative Layout Shift (CLS)
        let clsValue = 0;
        this.observeEntries('layout-shift', (entries) => {
            entries.forEach(entry => {
                if (!entry.hadRecentInput) {
                    clsValue += entry.value;
                }
            });
        });

        // Send CLS on page visibility change
        document.addEventListener('visibilitychange', () => {
            if (document.visibilityState === 'hidden') {
                this.sendMetric('cls', clsValue);
            }
        });
    }

    private trackCustomMetrics(): void {
        // Time to Interactive (TTI)
        this.measureTTI();

        // Bundle size metrics
        this.trackBundlePerformance();

        // API response times
        this.trackAPIPerformance();
    }

    private observeEntries(
        entryType: string,
        callback: (entries: PerformanceEntry[]) => void
    ): void {
        if (PerformanceObserver.supportedEntryTypes?.includes(entryType)) {
            const observer = new PerformanceObserver((list) => {
                callback(list.getEntries());
            });
            observer.observe({ entryTypes: [entryType] });
        }
    }

    private sendMetric(name: string, value: number): void {
        // Send to analytics service
        analytics.track('performance_metric', {
            metric: name,
            value,
            timestamp: Date.now(),
            page: window.location.pathname,
        });
    }

    private measureTTI(): void {
        // Custom TTI measurement based on your app's definition
        const startTime = performance.now();
        
        // Wait for main content to be interactive
        const checkInteractive = () => {
            const interactiveElements = document.querySelectorAll('button, input, a');
            const allInteractive = Array.from(interactiveElements).every(
                el => !el.hasAttribute('disabled')
            );

            if (allInteractive) {
                const tti = performance.now() - startTime;
                this.sendMetric('tti', tti);
            } else {
                setTimeout(checkInteractive, 100);
            }
        };

        checkInteractive();
    }
}

// Real User Monitoring (RUM)
class RealUserMonitoring {
    private sessionId: string;
    private startTime: number;

    constructor() {
        this.sessionId = nanoid();
        this.startTime = Date.now();
        this.init();
    }

    private init(): void {
        this.trackPageViews();
        this.trackUserActions();
        this.trackErrors();
        this.trackPerformance();
    }

    private trackPageViews(): void {
        // Initial page load
        this.sendEvent('page_view', {
            page: window.location.pathname,
            referrer: document.referrer,
            loadTime: performance.now(),
        });

        // SPA navigation
        const originalPushState = history.pushState;
        history.pushState = (...args) => {
            originalPushState.apply(history, args);
            this.sendEvent('page_view', {
                page: window.location.pathname,
                type: 'spa_navigation',
            });
        };
    }

    private trackUserActions(): void {
        // Click tracking
        document.addEventListener('click', (event) => {
            const target = event.target as HTMLElement;
            this.sendEvent('user_interaction', {
                type: 'click',
                element: target.tagName.toLowerCase(),
                text: target.textContent?.slice(0, 100),
                path: this.getElementPath(target),
            });
        });

        // Form submissions
        document.addEventListener('submit', (event) => {
            const form = event.target as HTMLFormElement;
            this.sendEvent('form_submit', {
                formId: form.id,
                action: form.action,
                method: form.method,
            });
        });
    }

    private sendEvent(eventType: string, data: Record<string, any>): void {
        analytics.track(eventType, {
            ...data,
            sessionId: this.sessionId,
            timestamp: Date.now(),
            sessionDuration: Date.now() - this.startTime,
        });
    }

    private getElementPath(element: HTMLElement): string {
        const path: string[] = [];
        let current: HTMLElement | null = element;

        while (current && current.nodeType === Node.ELEMENT_NODE) {
            let selector = current.nodeName.toLowerCase();
            
            if (current.id) {
                selector += `#${current.id}`;
                path.unshift(selector);
                break;
            } else if (current.className) {
                selector += `.${current.className.split(' ').join('.')}`;
            }
            
            path.unshift(selector);
            current = current.parentElement;
        }

        return path.join(' > ');
    }
}
```

## Interview Questions & Answers

### Q: How do you handle state management in a large-scale application?

**A:** I use a multi-layered approach: Redux/Zustand for global state (authentication, UI preferences), React Query for server state with intelligent caching, and local state for component-specific data. I implement feature-based state organization and use event-driven patterns for cross-feature communication.

### Q: How do you ensure your frontend can handle millions of users?

**A:** Through multiple strategies: implement micro-frontend architecture for team scalability, use CDN and edge caching, implement progressive loading with code splitting, virtual scrolling for large datasets, implement comprehensive monitoring with Core Web Vitals tracking, and build offline-first capabilities with service workers.

### Q: How do you optimize bundle size for large applications?

**A:** I use route-based and component-based code splitting, dynamic imports for heavy features, tree shaking with proper ES modules, bundle analysis to identify large dependencies, implement micro-frontend architecture to split bundles across teams, and use CDN for shared dependencies.

### Q: How do you handle errors in production at scale?

**A:** I implement multiple error boundary layers (global, feature, component), comprehensive error reporting with user context, retry logic with exponential backoff, offline queue for failed requests, real-user monitoring to track error patterns, and automated alerting for critical errors.

### Interview Tip:
*"Scalable frontend architecture requires careful balance between performance, maintainability, and team productivity. I focus on modular design patterns, intelligent caching strategies, comprehensive monitoring, and resilient error handling to ensure the application can handle growth in both users and development team size."*