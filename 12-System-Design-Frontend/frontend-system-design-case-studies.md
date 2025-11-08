# Frontend System Design Case Studies

## Question
Walk me through designing a frontend system for [specific application type]. Consider scalability, performance, and user experience.

## Answer

Frontend system design requires understanding user needs, technical constraints, and scalability requirements. Here are comprehensive case studies for common application types that frequently appear in interviews.

## Case Study 1: E-commerce Platform (Amazon/Flipkart Scale)

### Requirements Analysis
- **Users:** 100M+ users globally
- **Peak Load:** Black Friday - 10x normal traffic
- **Core Features:** Product catalog, search, cart, checkout, user accounts
- **Performance:** <2s page load, <100ms search response
- **Availability:** 99.99% uptime required

### Architecture Design

```typescript
// Micro-Frontend Architecture
const EcommerceApp = {
    // Shell Application
    shell: {
        responsibilities: ['Authentication', 'Navigation', 'Global State'],
        technologies: ['React 18', 'Single-SPA', 'Shared Design System'],
    },
    
    // Independent Micro-Frontends
    microFrontends: {
        productCatalog: {
            team: 'Catalog Team',
            tech: ['React', 'GraphQL', 'Elasticsearch'],
            deployment: 'Independent CI/CD',
        },
        shoppingCart: {
            team: 'Commerce Team', 
            tech: ['Vue 3', 'Pinia', 'Real-time WebSockets'],
            deployment: 'Blue-green deployment',
        },
        userAccount: {
            team: 'User Team',
            tech: ['Angular', 'NgRx', 'Progressive Enhancement'],
            deployment: 'Canary releases',
        },
        checkout: {
            team: 'Payments Team',
            tech: ['React', 'Redux Toolkit', 'Strict CSP'],
            deployment: 'Feature flags',
        },
    },
};

// Module Federation Configuration
// webpack.config.js for Shell
const ModuleFederationPlugin = require('@module-federation/webpack');

module.exports = {
    plugins: [
        new ModuleFederationPlugin({
            name: 'shell',
            remotes: {
                catalog: 'catalog@https://catalog-cdn.example.com/remoteEntry.js',
                cart: 'cart@https://cart-cdn.example.com/remoteEntry.js',
                account: 'account@https://account-cdn.example.com/remoteEntry.js',
                checkout: 'checkout@https://checkout-cdn.example.com/remoteEntry.js',
            },
            shared: {
                react: { singleton: true },
                'react-dom': { singleton: true },
                '@company/design-system': { singleton: true },
            },
        }),
    ],
};

// Dynamic Remote Loading with Fallbacks
class MicroFrontendLoader {
    private loadedRemotes = new Map<string, any>();
    private fallbacks = new Map<string, ComponentType>();

    async loadRemote(remoteName: string): Promise<ComponentType> {
        try {
            // Check if already loaded
            if (this.loadedRemotes.has(remoteName)) {
                return this.loadedRemotes.get(remoteName);
            }

            // Load remote module
            const remote = await import(/* webpackChunkName: "[request]" */ remoteName);
            this.loadedRemotes.set(remoteName, remote.default);
            
            return remote.default;
        } catch (error) {
            console.error(`Failed to load remote ${remoteName}:`, error);
            
            // Return fallback component
            const fallback = this.fallbacks.get(remoteName);
            if (fallback) {
                return fallback;
            }
            
            // Generic error component
            return () => (
                <div className="micro-frontend-error">
                    <h3>Service Temporarily Unavailable</h3>
                    <p>We're working to restore this feature. Please try again later.</p>
                    <button onClick={() => window.location.reload()}>
                        Refresh Page
                    </button>
                </div>
            );
        }
    }

    registerFallback(remoteName: string, fallback: ComponentType): void {
        this.fallbacks.set(remoteName, fallback);
    }
}
```

### Performance Optimization Strategy

```typescript
// Product Catalog Optimization
class ProductCatalogOptimizer {
    // Virtualized Product Grid
    renderProductGrid(products: Product[]): JSX.Element {
        return (
            <FixedSizeGrid
                columnCount={4}
                columnWidth={250}
                height={600}
                rowCount={Math.ceil(products.length / 4)}
                rowHeight={300}
                itemData={products}
            >
                {ProductGridItem}
            </FixedSizeGrid>
        );
    }

    // Image Optimization with WebP/AVIF
    optimizeProductImages(product: Product): ImageSet {
        return {
            webp: `${product.image}?format=webp&quality=80`,
            avif: `${product.image}?format=avif&quality=70`,
            jpeg: `${product.image}?format=jpeg&quality=85`,
            sizes: {
                thumbnail: '&width=150&height=150',
                medium: '&width=400&height=400',
                large: '&width=800&height=800',
            },
        };
    }

    // Progressive Image Loading
    ProgressiveImage({ src, alt, ...props }: ImageProps): JSX.Element {
        const [loaded, setLoaded] = useState(false);
        const [error, setError] = useState(false);
        
        return (
            <div className="progressive-image-container" {...props}>
                {/* Low-quality placeholder */}
                <img
                    src={`${src}?width=50&quality=20&blur=5`}
                    alt={alt}
                    className={`placeholder ${loaded ? 'fade-out' : ''}`}
                />
                
                {/* High-quality image */}
                <img
                    src={src}
                    alt={alt}
                    className={`main-image ${loaded ? 'fade-in' : ''}`}
                    onLoad={() => setLoaded(true)}
                    onError={() => setError(true)}
                    loading="lazy"
                />
                
                {error && (
                    <div className="image-error">
                        <ImageIcon />
                        <span>Image unavailable</span>
                    </div>
                )}
            </div>
        );
    }
}

// Search Performance with Debouncing and Caching
function useOptimizedSearch() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState<SearchResult[]>([]);
    const searchCache = useRef(new Map<string, SearchResult[]>());
    
    const debouncedSearch = useMemo(
        () => debounce(async (searchQuery: string) => {
            if (!searchQuery) {
                setResults([]);
                return;
            }

            // Check cache first
            const cached = searchCache.current.get(searchQuery);
            if (cached) {
                setResults(cached);
                return;
            }

            try {
                const searchResults = await searchAPI.search(searchQuery, {
                    fuzzy: true,
                    boost: {
                        title: 2,
                        brand: 1.5,
                        category: 1.2,
                    },
                });

                // Cache results
                searchCache.current.set(searchQuery, searchResults);
                setResults(searchResults);
            } catch (error) {
                console.error('Search failed:', error);
                setResults([]);
            }
        }, 300),
        []
    );

    useEffect(() => {
        debouncedSearch(query);
    }, [query, debouncedSearch]);

    return { query, setQuery, results };
}

// Shopping Cart Real-time Updates
function useRealtimeCart() {
    const [cart, setCart] = useState<CartItem[]>([]);
    const ws = useRef<WebSocket>();

    useEffect(() => {
        // Establish WebSocket connection
        ws.current = new WebSocket('wss://api.example.com/cart/realtime');
        
        ws.current.onmessage = (event) => {
            const update = JSON.parse(event.data);
            
            switch (update.type) {
                case 'CART_UPDATED':
                    setCart(update.cart);
                    break;
                case 'ITEM_OUT_OF_STOCK':
                    handleOutOfStock(update.itemId);
                    break;
                case 'PRICE_CHANGED':
                    handlePriceChange(update.itemId, update.newPrice);
                    break;
            }
        };

        return () => {
            ws.current?.close();
        };
    }, []);

    const addToCart = useCallback((item: CartItem) => {
        // Optimistic update
        setCart(prev => [...prev, item]);
        
        // Send to server
        ws.current?.send(JSON.stringify({
            type: 'ADD_ITEM',
            item,
        }));
    }, []);

    return { cart, addToCart };
}
```

### Scalability Solutions

```typescript
// CDN and Edge Configuration
const cdnConfig = {
    staticAssets: {
        path: '/static/*',
        headers: {
            'Cache-Control': 'public, max-age=31536000, immutable',
            'Content-Encoding': 'gzip, br',
        },
        edgeLocations: ['US', 'EU', 'APAC'],
    },
    
    apiResponses: {
        path: '/api/*',
        headers: {
            'Cache-Control': 'public, s-maxage=300, stale-while-revalidate=86400',
            'Vary': 'Accept-Language, Authorization',
        },
        edgeLogic: `
            // Cloudflare Worker for API caching
            addEventListener('fetch', event => {
                event.respondWith(handleAPIRequest(event.request));
            });

            async function handleAPIRequest(request) {
                const url = new URL(request.url);
                
                // Cache product data aggressively
                if (url.pathname.startsWith('/api/products/')) {
                    const cached = await caches.default.match(request);
                    if (cached) {
                        return cached;
                    }
                }
                
                const response = await fetch(request);
                
                // Cache successful responses
                if (response.ok) {
                    const clone = response.clone();
                    await caches.default.put(request, clone);
                }
                
                return response;
            }
        `,
    },
};

// Database Query Optimization
class ProductDataLayer {
    async getProductsWithFilters(filters: ProductFilters): Promise<Product[]> {
        // Use compound indexes for efficient filtering
        const query = {
            bool: {
                must: [
                    { match: { status: 'active' } },
                    ...(filters.category ? [{ term: { category: filters.category } }] : []),
                    ...(filters.brand ? [{ term: { brand: filters.brand } }] : []),
                    ...(filters.priceRange ? [
                        { range: { 
                            price: { 
                                gte: filters.priceRange.min,
                                lte: filters.priceRange.max 
                            } 
                        }}
                    ] : []),
                ],
            },
        };

        // Include aggregations for faceted search
        const aggregations = {
            categories: { terms: { field: 'category' } },
            brands: { terms: { field: 'brand' } },
            priceRanges: {
                range: {
                    field: 'price',
                    ranges: [
                        { to: 50 },
                        { from: 50, to: 100 },
                        { from: 100, to: 500 },
                        { from: 500 },
                    ],
                },
            },
        };

        const results = await this.elasticsearch.search({
            index: 'products',
            body: {
                query,
                aggs: aggregations,
                size: filters.limit || 20,
                from: filters.offset || 0,
                sort: this.getSortOptions(filters.sortBy),
            },
        });

        return {
            products: results.body.hits.hits.map(hit => hit._source),
            facets: results.body.aggregations,
            total: results.body.hits.total.value,
        };
    }

    private getSortOptions(sortBy?: string): any[] {
        switch (sortBy) {
            case 'price_low':
                return [{ price: 'asc' }];
            case 'price_high':
                return [{ price: 'desc' }];
            case 'rating':
                return [{ rating: 'desc' }, { reviewCount: 'desc' }];
            case 'newest':
                return [{ createdAt: 'desc' }];
            default:
                return [{ _score: 'desc' }, { popularity: 'desc' }];
        }
    }
}
```

## Case Study 2: Social Media Platform (Twitter/Instagram Scale)

### Requirements Analysis
- **Users:** 500M+ daily active users
- **Content:** Real-time feeds, media uploads, interactions
- **Performance:** <200ms feed load, real-time notifications
- **Features:** Infinite scroll, offline support, push notifications

### Architecture Design

```typescript
// Real-time Feed Architecture
class SocialFeedSystem {
    private feedCache: Map<string, FeedItem[]> = new Map();
    private websocket: WebSocket;
    private virtualizer: VirtualizedList;

    constructor() {
        this.initializeRealtimeConnection();
        this.setupVirtualization();
    }

    // Infinite Scroll with Virtualization
    renderFeed(userId: string): JSX.Element {
        const { data, fetchNextPage, hasNextPage, isFetching } = useInfiniteQuery({
            queryKey: ['feed', userId],
            queryFn: ({ pageParam = 0 }) => this.fetchFeedPage(userId, pageParam),
            getNextPageParam: (lastPage) => lastPage.nextCursor,
        });

        const allItems = data?.pages.flatMap(page => page.items) ?? [];

        return (
            <VirtualizedList
                items={allItems}
                renderItem={({ item, index }) => (
                    <FeedItem
                        key={item.id}
                        item={item}
                        onVisible={() => this.trackImpression(item.id)}
                        onInteraction={(type) => this.trackInteraction(item.id, type)}
                    />
                )}
                onEndReached={() => {
                    if (hasNextPage && !isFetching) {
                        fetchNextPage();
                    }
                }}
                estimatedItemSize={200}
                overscan={5}
            />
        );
    }

    // Real-time Updates
    private initializeRealtimeConnection(): void {
        this.websocket = new WebSocket('wss://api.social.com/feed/realtime');
        
        this.websocket.onmessage = (event) => {
            const update = JSON.parse(event.data);
            
            switch (update.type) {
                case 'NEW_POST':
                    this.handleNewPost(update.post);
                    break;
                case 'POST_UPDATED':
                    this.handlePostUpdate(update.postId, update.changes);
                    break;
                case 'POST_DELETED':
                    this.handlePostDeletion(update.postId);
                    break;
            }
        };
    }

    private handleNewPost(post: FeedItem): void {
        // Add to top of feed with smooth animation
        queryClient.setQueryData(['feed'], (old: any) => {
            if (!old) return old;
            
            return {
                ...old,
                pages: [
                    {
                        ...old.pages[0],
                        items: [post, ...old.pages[0].items],
                    },
                    ...old.pages.slice(1),
                ],
            };
        });

        // Show notification for new posts
        this.showNewPostNotification();
    }
}

// Media Upload with Progress and Optimization
class MediaUploadSystem {
    async uploadMedia(file: File): Promise<MediaUploadResult> {
        // Client-side image optimization
        const optimizedFile = await this.optimizeImage(file);
        
        // Chunk upload for large files
        if (optimizedFile.size > 10 * 1024 * 1024) { // 10MB
            return this.uploadInChunks(optimizedFile);
        }

        return this.uploadDirect(optimizedFile);
    }

    private async optimizeImage(file: File): Promise<File> {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d')!;
        const img = new Image();

        return new Promise((resolve) => {
            img.onload = () => {
                // Calculate optimal dimensions
                const maxWidth = 1920;
                const maxHeight = 1080;
                let { width, height } = img;

                if (width > maxWidth) {
                    height = (height * maxWidth) / width;
                    width = maxWidth;
                }

                if (height > maxHeight) {
                    width = (width * maxHeight) / height;
                    height = maxHeight;
                }

                canvas.width = width;
                canvas.height = height;

                // Draw and compress
                ctx.drawImage(img, 0, 0, width, height);
                
                canvas.toBlob(
                    (blob) => {
                        const optimizedFile = new File([blob!], file.name, {
                            type: 'image/jpeg',
                            lastModified: Date.now(),
                        });
                        resolve(optimizedFile);
                    },
                    'image/jpeg',
                    0.8 // 80% quality
                );
            };

            img.src = URL.createObjectURL(file);
        });
    }

    private async uploadInChunks(file: File): Promise<MediaUploadResult> {
        const chunkSize = 1024 * 1024; // 1MB chunks
        const totalChunks = Math.ceil(file.size / chunkSize);
        const uploadId = nanoid();

        for (let i = 0; i < totalChunks; i++) {
            const start = i * chunkSize;
            const end = Math.min(start + chunkSize, file.size);
            const chunk = file.slice(start, end);

            await this.uploadChunk(uploadId, i, chunk, totalChunks);
            
            // Update progress
            const progress = ((i + 1) / totalChunks) * 100;
            this.updateUploadProgress(uploadId, progress);
        }

        // Finalize upload
        return this.finalizeUpload(uploadId);
    }
}

// Offline-First Architecture
class OfflineSocialMedia {
    private syncQueue: SyncOperation[] = [];
    private offlineStorage: IndexedDB;

    constructor() {
        this.setupOfflineStorage();
        this.setupSyncQueue();
    }

    async createPost(content: string, media?: File[]): Promise<void> {
        const post: OfflinePost = {
            id: nanoid(),
            content,
            media: media ? await this.storeMediaOffline(media) : [],
            createdAt: Date.now(),
            status: 'pending',
        };

        // Store locally first
        await this.offlineStorage.posts.add(post);

        // Add to sync queue
        this.syncQueue.push({
            type: 'CREATE_POST',
            data: post,
            retries: 0,
            maxRetries: 3,
        });

        // Try immediate sync if online
        if (navigator.onLine) {
            this.processSyncQueue();
        }
    }

    private async processSyncQueue(): Promise<void> {
        const operations = [...this.syncQueue];
        this.syncQueue = [];

        for (const operation of operations) {
            try {
                await this.executeSync(operation);
            } catch (error) {
                console.error('Sync failed:', error);
                
                if (operation.retries < operation.maxRetries) {
                    operation.retries++;
                    this.syncQueue.push(operation);
                } else {
                    // Mark as failed
                    await this.markSyncFailed(operation);
                }
            }
        }
    }

    private async executeSync(operation: SyncOperation): Promise<void> {
        switch (operation.type) {
            case 'CREATE_POST':
                const serverPost = await api.createPost(operation.data);
                await this.offlineStorage.posts.update(operation.data.id, {
                    serverId: serverPost.id,
                    status: 'synced',
                });
                break;
            // Handle other sync operations...
        }
    }
}
```

## Case Study 3: Real-time Collaboration Platform (Google Docs/Figma Scale)

### Requirements Analysis
- **Users:** Multiple users editing simultaneously
- **Latency:** <50ms for real-time updates
- **Conflict Resolution:** Operational Transform (OT) or CRDTs
- **Features:** Live cursors, comments, version history

### Architecture Design

```typescript
// Operational Transform Implementation
class CollaborativeEditor {
    private operationQueue: Operation[] = [];
    private documentState: DocumentState;
    private websocket: WebSocket;
    private otEngine: OperationalTransform;

    constructor(documentId: string) {
        this.documentId = documentId;
        this.otEngine = new OperationalTransform();
        this.setupRealtimeConnection();
    }

    // Apply local operation and broadcast
    applyOperation(operation: Operation): void {
        // Transform against pending operations
        const transformedOp = this.transformAgainstQueue(operation);
        
        // Apply locally
        this.documentState = this.otEngine.apply(this.documentState, transformedOp);
        
        // Add to pending queue
        this.operationQueue.push(transformedOp);
        
        // Broadcast to server
        this.websocket.send(JSON.stringify({
            type: 'OPERATION',
            operation: transformedOp,
            clientId: this.clientId,
            timestamp: Date.now(),
        }));
        
        // Update UI
        this.updateEditor();
    }

    // Handle remote operation
    handleRemoteOperation(operation: Operation): void {
        // Transform against pending operations
        const transformedOp = this.transformAgainstQueue(operation);
        
        // Apply to document state
        this.documentState = this.otEngine.apply(this.documentState, transformedOp);
        
        // Update UI
        this.updateEditor();
    }

    private transformAgainstQueue(operation: Operation): Operation {
        return this.operationQueue.reduce(
            (op, pendingOp) => this.otEngine.transform(op, pendingOp),
            operation
        );
    }
}

// Real-time Cursor Tracking
class CursorManager {
    private cursors: Map<string, CursorPosition> = new Map();
    private localCursor: CursorPosition | null = null;

    updateLocalCursor(position: CursorPosition): void {
        this.localCursor = position;
        
        // Throttle cursor updates to avoid spam
        this.throttledBroadcast(position);
    }

    private throttledBroadcast = throttle((position: CursorPosition) => {
        this.websocket.send(JSON.stringify({
            type: 'CURSOR_UPDATE',
            position,
            clientId: this.clientId,
        }));
    }, 50); // 50ms throttle

    handleRemoteCursor(clientId: string, position: CursorPosition): void {
        this.cursors.set(clientId, position);
        this.renderCursors();
    }

    private renderCursors(): void {
        // Render other users' cursors in the editor
        this.cursors.forEach((position, clientId) => {
            if (clientId !== this.clientId) {
                this.renderUserCursor(clientId, position);
            }
        });
    }
}

// Conflict-free Replicated Data Types (CRDT) Implementation
class CRDTDocument {
    private content: CRDTString;
    private metadata: Map<string, any> = new Map();

    constructor() {
        this.content = new CRDTString();
    }

    insert(position: number, text: string, clientId: string): void {
        const operation = {
            type: 'insert',
            position,
            text,
            clientId,
            timestamp: Date.now(),
            id: nanoid(),
        };

        this.content.insert(operation);
        this.broadcast(operation);
    }

    delete(position: number, length: number, clientId: string): void {
        const operation = {
            type: 'delete',
            position,
            length,
            clientId,
            timestamp: Date.now(),
            id: nanoid(),
        };

        this.content.delete(operation);
        this.broadcast(operation);
    }

    // CRDT guarantees eventual consistency without conflicts
    merge(remoteOperations: Operation[]): void {
        remoteOperations.forEach(op => {
            this.content.applyRemoteOperation(op);
        });
    }
}
```

## Performance Monitoring & Analytics

```typescript
// Comprehensive Performance Tracking
class SystemDesignAnalytics {
    private performanceObserver: PerformanceObserver;
    private userBehaviorTracker: UserBehaviorTracker;

    init(): void {
        this.trackCoreWebVitals();
        this.trackUserFlows();
        this.trackBusinessMetrics();
        this.setupErrorTracking();
    }

    private trackCoreWebVitals(): void {
        // LCP, FID, CLS tracking with business context
        new PerformanceObserver((list) => {
            list.getEntries().forEach(entry => {
                if (entry.entryType === 'largest-contentful-paint') {
                    this.sendMetric('lcp', entry.startTime, {
                        page: window.location.pathname,
                        userType: this.getUserType(),
                        experiment: this.getActiveExperiments(),
                    });
                }
            });
        }).observe({ entryTypes: ['largest-contentful-paint'] });
    }

    private trackUserFlows(): void {
        // E-commerce specific metrics
        this.trackConversionFunnel([
            'product_view',
            'add_to_cart', 
            'checkout_start',
            'payment_info',
            'purchase'
        ]);

        // Social media specific metrics
        this.trackEngagementMetrics([
            'post_view',
            'like',
            'comment',
            'share',
            'follow'
        ]);
    }

    private trackBusinessMetrics(): void {
        // Revenue impact tracking
        this.correlatePerformanceWithRevenue();
        
        // User retention impact
        this.trackRetentionByPerformance();
        
        // Feature adoption rates
        this.trackFeatureUsage();
    }
}

// A/B Testing Framework
class PerformanceExperiments {
    async runExperiment(experimentId: string): Promise<void> {
        const variant = await this.getVariant(experimentId);
        
        switch (experimentId) {
            case 'image-loading-strategy':
                if (variant === 'lazy') {
                    this.enableLazyLoading();
                } else if (variant === 'eager') {
                    this.enableEagerLoading();
                }
                break;
                
            case 'bundle-splitting':
                if (variant === 'route-based') {
                    this.enableRouteBundling();
                } else if (variant === 'component-based') {
                    this.enableComponentBundling();
                }
                break;
        }
        
        this.trackExperimentPerformance(experimentId, variant);
    }
}
```

## Interview Questions & Answers

### Q: How would you design the frontend for a video streaming platform like Netflix?

**A:** I'd use adaptive bitrate streaming with MSE/EME APIs, implement progressive web app features for offline viewing, use service workers for content prefetching based on viewing history, implement virtualized grids for large catalogs, and use CDN edge locations for global content delivery with regional caching strategies.

### Q: Design a real-time chat application frontend that scales to millions of users.

**A:** I'd implement WebSocket connections with connection pooling, use message queuing for offline/online sync, implement virtual scrolling for chat history, use optimistic updates for message sending, implement message threading and reactions with conflict resolution, and use horizontal scaling with WebSocket server clustering.

### Q: How would you architect a collaborative design tool like Figma?

**A:** I'd use WebRTC for real-time cursor sharing, implement operational transform or CRDTs for conflict-free editing, use Canvas API with efficient rendering optimizations, implement layer-based architecture with selective updates, use WebAssembly for complex geometric calculations, and implement robust undo/redo with operation history.

### Interview Tip:
*"When designing frontend systems, always start with understanding the user requirements and business constraints, then design the architecture to handle the specific scalability and performance needs. Focus on real-world constraints like network latency, device capabilities, and team organization when making technology choices."*