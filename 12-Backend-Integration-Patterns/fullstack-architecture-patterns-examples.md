# Full-Stack Architecture Patterns and Examples

## Question
How do you design and implement scalable full-stack applications with modern frontend and backend integration?

## Answer

Full-stack architecture requires careful consideration of data flow, state management, API design, real-time communication, and deployment strategies. Here are comprehensive patterns and examples for building robust, scalable applications that demonstrate deep full-stack expertise.

## Complete Application Architecture Patterns

### 1. **Modern E-commerce Platform Architecture**

```typescript
// Complete e-commerce application architecture
interface ECommerceArchitecture {
    frontend: FrontendArchitecture;
    backend: BackendArchitecture;
    integration: IntegrationLayer;
    infrastructure: InfrastructureLayer;
}

// Frontend architecture with micro-frontends
class ECommerceFrontendArchitecture {
    // Main application shell
    private appShell: ApplicationShell;
    private microFrontends: Map<string, MicroFrontend>;
    private stateManager: GlobalStateManager;
    private apiLayer: APILayer;

    constructor() {
        this.setupApplicationShell();
        this.configureMicroFrontends();
        this.initializeStateManagement();
        this.setupAPILayer();
    }

    private setupApplicationShell(): void {
        this.appShell = {
            // Shared components and layout
            navigation: new NavigationComponent(),
            header: new HeaderComponent(),
            footer: new FooterComponent(),
            
            // Global services
            authService: new AuthenticationService(),
            cartService: new CartService(),
            notificationService: new NotificationService(),
            
            // Routing configuration
            router: new AppRouter({
                routes: [
                    { path: '/products', component: () => import('./products/ProductsApp') },
                    { path: '/cart', component: () => import('./cart/CartApp') },
                    { path: '/checkout', component: () => import('./checkout/CheckoutApp') },
                    { path: '/profile', component: () => import('./profile/ProfileApp') },
                ],
            }),
        };
    }

    private configureMicroFrontends(): void {
        this.microFrontends.set('products', {
            name: 'products',
            entry: '/products',
            technologies: ['React 18', 'Redux Toolkit', 'React Query'],
            responsibilities: [
                'Product catalog and search',
                'Product details and reviews',
                'Inventory management',
                'Recommendation engine',
            ],
            apis: ['/api/products', '/api/search', '/api/recommendations'],
        });

        this.microFrontends.set('cart', {
            name: 'cart',
            entry: '/cart',
            technologies: ['React 18', 'Zustand', 'React Hook Form'],
            responsibilities: [
                'Shopping cart management',
                'Price calculations',
                'Promotion application',
                'Saved items',
            ],
            apis: ['/api/cart', '/api/promotions'],
        });

        this.microFrontends.set('checkout', {
            name: 'checkout',
            entry: '/checkout',
            technologies: ['React 18', 'React Query', 'Stripe Elements'],
            responsibilities: [
                'Payment processing',
                'Order creation',
                'Address management',
                'Order confirmation',
            ],
            apis: ['/api/orders', '/api/payments', '/api/shipping'],
        });
    }

    private initializeStateManagement(): void {
        // Global state for shared data
        this.stateManager = new GlobalStateManager({
            // User state
            user: createUserSlice(),
            
            // Cart state (shared across micro-frontends)
            cart: createCartSlice(),
            
            // UI state
            ui: createUISlice(),
            
            // Configuration
            config: createConfigSlice(),
        });

        // Cross-micro-frontend communication
        this.setupCrossFrontendCommunication();
    }

    private setupCrossFrontendCommunication(): void {
        // Event bus for micro-frontend communication
        const eventBus = new MicroFrontendEventBus();

        // Cart updates
        eventBus.subscribe('cart:item_added', (event) => {
            this.stateManager.dispatch(addItemToCart(event.payload));
            this.showNotification('Item added to cart', 'success');
        });

        eventBus.subscribe('cart:item_removed', (event) => {
            this.stateManager.dispatch(removeItemFromCart(event.payload));
        });

        // User authentication
        eventBus.subscribe('auth:login', (event) => {
            this.stateManager.dispatch(setUser(event.payload.user));
            this.syncCartWithServer();
        });

        eventBus.subscribe('auth:logout', (event) => {
            this.stateManager.dispatch(clearUser());
            this.clearLocalCart();
        });

        // Navigation events
        eventBus.subscribe('navigation:change', (event) => {
            this.updateActiveRoute(event.payload.route);
        });
    }

    // API layer with comprehensive error handling
    private setupAPILayer(): void {
        this.apiLayer = new APILayer({
            baseURL: process.env.REACT_APP_API_URL,
            interceptors: {
                request: [
                    this.addAuthenticationHeader,
                    this.addTrackingHeaders,
                    this.addCSRFProtection,
                ],
                response: [
                    this.handleAuthenticationErrors,
                    this.handleRateLimiting,
                    this.handleServerErrors,
                ],
            },
            retry: {
                maxAttempts: 3,
                backoffStrategy: 'exponential',
                retryableErrors: [408, 429, 500, 502, 503, 504],
            },
        });
    }
}

// Backend microservices architecture
class ECommerceBackendArchitecture {
    private services: Map<string, MicroService>;
    private messageQueue: MessageQueue;
    private database: DatabaseLayer;
    private cache: CacheLayer;

    constructor() {
        this.setupMicroServices();
        this.configureMessageQueue();
        this.initializeDatabases();
        this.setupCaching();
    }

    private setupMicroServices(): void {
        // User service
        this.services.set('user-service', {
            name: 'user-service',
            port: 3001,
            database: 'postgresql',
            responsibilities: [
                'User authentication and authorization',
                'User profile management',
                'Password reset and email verification',
                'Role and permission management',
            ],
            apis: [
                'POST /auth/login',
                'POST /auth/register',
                'GET /users/{id}',
                'PUT /users/{id}',
                'POST /auth/refresh',
            ],
            events: {
                publishes: ['user.created', 'user.updated', 'user.deleted'],
                subscribes: ['order.completed', 'cart.abandoned'],
            },
        });

        // Product service
        this.services.set('product-service', {
            name: 'product-service',
            port: 3002,
            database: 'postgresql + elasticsearch',
            responsibilities: [
                'Product catalog management',
                'Search and filtering',
                'Inventory tracking',
                'Price management',
            ],
            apis: [
                'GET /products',
                'GET /products/{id}',
                'POST /products/search',
                'GET /products/recommendations',
            ],
            events: {
                publishes: ['product.created', 'product.updated', 'inventory.changed'],
                subscribes: ['order.item.reserved', 'order.completed'],
            },
        });

        // Cart service
        this.services.set('cart-service', {
            name: 'cart-service',
            port: 3003,
            database: 'redis + postgresql',
            responsibilities: [
                'Shopping cart management',
                'Session cart persistence',
                'Cart synchronization',
                'Abandoned cart tracking',
            ],
            apis: [
                'GET /cart/{userId}',
                'POST /cart/{userId}/items',
                'PUT /cart/{userId}/items/{itemId}',
                'DELETE /cart/{userId}/items/{itemId}',
            ],
            events: {
                publishes: ['cart.updated', 'cart.abandoned'],
                subscribes: ['user.created', 'order.completed'],
            },
        });

        // Order service
        this.services.set('order-service', {
            name: 'order-service',
            port: 3004,
            database: 'postgresql',
            responsibilities: [
                'Order creation and processing',
                'Payment orchestration',
                'Order status tracking',
                'Invoice generation',
            ],
            apis: [
                'POST /orders',
                'GET /orders/{id}',
                'GET /users/{userId}/orders',
                'PUT /orders/{id}/status',
            ],
            events: {
                publishes: ['order.created', 'order.completed', 'order.cancelled'],
                subscribes: ['payment.completed', 'inventory.reserved'],
            },
        });

        // Payment service
        this.services.set('payment-service', {
            name: 'payment-service',
            port: 3005,
            database: 'postgresql',
            responsibilities: [
                'Payment processing',
                'Payment method management',
                'Refund handling',
                'Payment reconciliation',
            ],
            apis: [
                'POST /payments',
                'GET /payments/{id}',
                'POST /payments/{id}/refund',
                'GET /users/{userId}/payment-methods',
            ],
            events: {
                publishes: ['payment.completed', 'payment.failed', 'refund.processed'],
                subscribes: ['order.created'],
            },
        });
    }

    // Event-driven architecture with message queues
    private configureMessageQueue(): void {
        this.messageQueue = new MessageQueue({
            broker: 'rabbitmq',
            exchanges: {
                'user.events': 'topic',
                'product.events': 'topic',
                'order.events': 'topic',
                'payment.events': 'topic',
            },
            queues: this.generateServiceQueues(),
            deadLetterQueues: this.setupDeadLetterQueues(),
        });

        // Event handlers for cross-service communication
        this.setupEventHandlers();
    }

    private setupEventHandlers(): void {
        // Order service events
        this.messageQueue.subscribe('order.created', async (event) => {
            // Reserve inventory
            await this.services.get('product-service')!.call('POST /inventory/reserve', {
                orderId: event.orderId,
                items: event.items,
            });

            // Process payment
            await this.services.get('payment-service')!.call('POST /payments', {
                orderId: event.orderId,
                amount: event.total,
                paymentMethod: event.paymentMethod,
            });
        });

        // Payment completion events
        this.messageQueue.subscribe('payment.completed', async (event) => {
            // Update order status
            await this.services.get('order-service')!.call('PUT /orders/' + event.orderId + '/status', {
                status: 'paid',
                paymentId: event.paymentId,
            });

            // Clear user cart
            await this.services.get('cart-service')!.call('DELETE /cart/' + event.userId);
        });

        // Inventory management
        this.messageQueue.subscribe('order.cancelled', async (event) => {
            // Release reserved inventory
            await this.services.get('product-service')!.call('POST /inventory/release', {
                orderId: event.orderId,
                items: event.items,
            });
        });
    }

    // Database layer with proper separation
    private initializeDatabases(): void {
        this.database = new DatabaseLayer({
            // User service database
            userDB: new PostgreSQLDatabase({
                host: process.env.USER_DB_HOST,
                database: 'ecommerce_users',
                schema: {
                    users: userTableSchema,
                    roles: roleTableSchema,
                    permissions: permissionTableSchema,
                },
            }),

            // Product service databases
            productDB: new PostgreSQLDatabase({
                host: process.env.PRODUCT_DB_HOST,
                database: 'ecommerce_products',
                schema: {
                    products: productTableSchema,
                    categories: categoryTableSchema,
                    inventory: inventoryTableSchema,
                },
            }),

            searchDB: new ElasticsearchDatabase({
                host: process.env.ELASTICSEARCH_HOST,
                index: 'products',
                mapping: productSearchMapping,
            }),

            // Cart service database
            cartDB: new RedisDatabase({
                host: process.env.REDIS_HOST,
                keyPrefix: 'cart:',
            }),

            // Order and payment databases
            orderDB: new PostgreSQLDatabase({
                host: process.env.ORDER_DB_HOST,
                database: 'ecommerce_orders',
                schema: {
                    orders: orderTableSchema,
                    order_items: orderItemTableSchema,
                    payments: paymentTableSchema,
                },
            }),
        });
    }

    // Caching strategy
    private setupCaching(): void {
        this.cache = new CacheLayer({
            // Application cache
            redis: new RedisCache({
                host: process.env.REDIS_HOST,
                keyPrefix: 'app:',
            }),

            // CDN cache
            cdn: new CDNCache({
                provider: 'cloudflare',
                zones: ['api.ecommerce.com', 'static.ecommerce.com'],
            }),

            // Cache strategies by service
            strategies: {
                'product-service': {
                    'GET /products': { ttl: 300, tags: ['products'] },
                    'GET /products/{id}': { ttl: 600, tags: ['product'] },
                    'POST /products/search': { ttl: 60, tags: ['search'] },
                },
                'user-service': {
                    'GET /users/{id}': { ttl: 300, tags: ['user'] },
                },
                'cart-service': {
                    'GET /cart/{userId}': { ttl: 60, tags: ['cart'] },
                },
            },

            // Cache invalidation
            invalidation: {
                'product.updated': ['products', 'product'],
                'inventory.changed': ['products', 'search'],
                'user.updated': ['user'],
                'cart.updated': ['cart'],
            },
        });
    }
}
```

### 2. **Real-time Collaboration Platform Architecture**

```typescript
// Complete real-time collaboration platform
class CollaborationPlatformArchitecture {
    private frontend: CollaborationFrontend;
    private backend: CollaborationBackend;
    private realTimeEngine: RealTimeEngine;

    constructor() {
        this.initializeFrontend();
        this.setupBackend();
        this.configureRealTimeEngine();
    }

    private initializeFrontend(): void {
        this.frontend = {
            // Core editing experience
            editor: new CollaborativeEditor({
                framework: 'React 18',
                editor: 'Monaco Editor',
                features: [
                    'Syntax highlighting',
                    'Auto-completion',
                    'Multi-cursor editing',
                    'Real-time collaboration',
                    'Version control integration',
                ],
            }),

            // State management
            stateManagement: {
                global: 'Redux Toolkit + RTK Query',
                realTime: 'Custom WebSocket store',
                local: 'React state + useReducer',
                persistence: 'IndexedDB for offline',
            },

            // Real-time features
            collaboration: {
                presence: new PresenceSystem(),
                cursors: new CursorTracker(),
                comments: new CommentingSystem(),
                videoCall: new WebRTCIntegration(),
            },

            // Performance optimization
            optimization: {
                codeLoading: 'Dynamic imports + suspense',
                bundleOptimization: 'Webpack code splitting',
                virtualScrolling: 'React Window',
                memoization: 'React.memo + useMemo',
            },
        };
    }

    private setupBackend(): void {
        this.backend = {
            // Core services
            services: {
                'document-service': {
                    responsibilities: [
                        'Document CRUD operations',
                        'Version control and history',
                        'Document permissions',
                        'Collaborative editing operations',
                    ],
                    technology: 'Node.js + Express + TypeScript',
                    database: 'MongoDB + Redis',
                },

                'user-service': {
                    responsibilities: [
                        'User management and authentication',
                        'Team and workspace management',
                        'Permission and role management',
                        'User presence tracking',
                    ],
                    technology: 'Node.js + Fastify + TypeScript',
                    database: 'PostgreSQL',
                },

                'collaboration-service': {
                    responsibilities: [
                        'Real-time operational transformation',
                        'Conflict resolution',
                        'Session management',
                        'WebSocket connection handling',
                    ],
                    technology: 'Node.js + Socket.IO + TypeScript',
                    database: 'Redis + MongoDB',
                },

                'file-service': {
                    responsibilities: [
                        'File upload and storage',
                        'Image optimization',
                        'File sharing and permissions',
                        'Version control for assets',
                    ],
                    technology: 'Node.js + Express + Sharp',
                    storage: 'AWS S3 + CloudFront',
                },
            },

            // Data layer
            dataLayer: {
                primaryDB: new MongoDBCluster({
                    collections: {
                        documents: documentSchema,
                        versions: versionSchema,
                        operations: operationSchema,
                    },
                }),
                
                cacheDB: new RedisCluster({
                    useCases: [
                        'Session management',
                        'Real-time presence',
                        'Operational transformation cache',
                        'API response caching',
                    ],
                }),
                
                searchEngine: new ElasticsearchCluster({
                    indices: {
                        documents: documentSearchIndex,
                        users: userSearchIndex,
                    },
                }),
            },
        };
    }

    private configureRealTimeEngine(): void {
        this.realTimeEngine = {
            // WebSocket management
            websocketServer: new WebSocketServer({
                port: 3001,
                path: '/collaboration',
                transports: ['websocket', 'polling'],
                cors: {
                    origin: process.env.FRONTEND_URLS?.split(','),
                    credentials: true,
                },
            }),

            // Operational transformation
            operationalTransform: new OperationalTransformEngine({
                operations: [
                    'text-insert',
                    'text-delete',
                    'text-format',
                    'cursor-move',
                    'selection-change',
                ],
                conflictResolution: 'timestamp-based',
                transformationRules: otTransformRules,
            }),

            // Presence system
            presenceSystem: new PresenceSystem({
                trackingInterval: 1000,
                inactivityTimeout: 30000,
                features: [
                    'User cursors',
                    'Active selections',
                    'Typing indicators',
                    'Online status',
                ],
            }),

            // Session management
            sessionManager: new CollaborationSessionManager({
                sessionTimeout: 3600000, // 1 hour
                maxConcurrentUsers: 50,
                persistenceInterval: 5000,
            }),
        };
    }

    // Document collaboration workflow
    async handleDocumentCollaboration(documentId: string, userId: string): Promise<void> {
        // Join collaboration session
        const session = await this.realTimeEngine.sessionManager.joinSession(documentId, userId);
        
        // Load document with operational transform history
        const document = await this.backend.services['document-service'].getDocument(documentId);
        const operations = await this.backend.services['collaboration-service'].getOperations(documentId);
        
        // Initialize client state
        const clientState = {
            document: document.content,
            version: document.version,
            operations: operations,
            cursors: session.activeCursors,
            presence: session.activeUsers,
        };

        // Setup real-time synchronization
        this.realTimeEngine.websocketServer.to(documentId).emit('document_state', clientState);
        
        // Handle incoming operations
        this.realTimeEngine.websocketServer.on('operation', async (operation) => {
            const transformedOp = await this.realTimeEngine.operationalTransform.transform(
                operation,
                session.pendingOperations
            );
            
            // Apply operation to document
            await this.backend.services['document-service'].applyOperation(documentId, transformedOp);
            
            // Broadcast to other clients
            this.realTimeEngine.websocketServer
                .to(documentId)
                .except(operation.clientId)
                .emit('operation', transformedOp);
        });
    }
}

// Operational transformation implementation
class OperationalTransformEngine {
    // Transform operations for concurrent editing
    async transform(operation: Operation, pendingOps: Operation[]): Promise<Operation> {
        let transformedOp = { ...operation };

        for (const pendingOp of pendingOps) {
            transformedOp = this.transformPair(transformedOp, pendingOp);
        }

        return transformedOp;
    }

    private transformPair(op1: Operation, op2: Operation): Operation {
        // Text insertion transformation
        if (op1.type === 'insert' && op2.type === 'insert') {
            if (op1.position <= op2.position) {
                return {
                    ...op2,
                    position: op2.position + op1.text.length,
                };
            }
            return op2;
        }

        // Text deletion transformation
        if (op1.type === 'delete' && op2.type === 'insert') {
            if (op1.position < op2.position) {
                return {
                    ...op2,
                    position: op2.position - op1.length,
                };
            }
            return op2;
        }

        // Handle other operation combinations
        return this.handleComplexTransformation(op1, op2);
    }

    private handleComplexTransformation(op1: Operation, op2: Operation): Operation {
        // Implement complex transformation logic for:
        // - Overlapping deletions
        // - Format operations
        // - Multi-character operations
        // - Cursor movements
        
        return op2; // Simplified implementation
    }
}
```

### 3. **Scalable Social Media Platform Architecture**

```typescript
// Social media platform with real-time feeds
class SocialMediaArchitecture {
    private frontend: SocialFrontend;
    private backend: SocialBackend;
    private realTimeSystem: RealTimeSystem;

    constructor() {
        this.setupFrontendArchitecture();
        this.configureBackendServices();
        this.initializeRealTimeSystems();
    }

    private setupFrontendArchitecture(): void {
        this.frontend = {
            // Main application
            shell: {
                framework: 'Next.js 14 with App Router',
                stateManagement: 'Zustand + TanStack Query',
                styling: 'Tailwind CSS + Framer Motion',
                routing: 'Next.js App Router with layouts',
            },

            // Feature modules
            features: {
                feed: new InfiniteFeedComponent({
                    virtualization: 'React Window',
                    lazyLoading: 'Intersection Observer',
                    caching: 'TanStack Query infinite queries',
                    realTimeUpdates: 'WebSocket + optimistic updates',
                }),

                messaging: new MessagingSystem({
                    realTime: 'WebSocket + WebRTC for video',
                    encryption: 'End-to-end encryption',
                    fileSharing: 'Progressive upload with chunks',
                    typing: 'Real-time typing indicators',
                }),

                media: new MediaHandlingSystem({
                    upload: 'Chunked upload with progress',
                    processing: 'Client-side image optimization',
                    streaming: 'HLS for video streaming',
                    cdn: 'CloudFront with multiple regions',
                }),

                notifications: new NotificationSystem({
                    realTime: 'WebSocket + Service Worker',
                    push: 'Web Push API',
                    inApp: 'Toast notifications with queue',
                    preferences: 'Granular notification settings',
                }),
            },

            // Performance optimizations
            performance: {
                bundling: 'Webpack with module federation',
                caching: 'Service Worker + Cache API',
                prefetching: 'Next.js automatic prefetching',
                imageOptimization: 'Next.js Image component',
                codeLoading: 'Dynamic imports + React Suspense',
            },
        };
    }

    private configureBackendServices(): void {
        this.backend = {
            // Core services
            userService: new UserService({
                authentication: 'JWT + OAuth 2.0',
                authorization: 'RBAC with custom permissions',
                profile: 'Rich user profiles with privacy settings',
                relationships: 'Follow/friend relationships',
                database: 'PostgreSQL with read replicas',
            }),

            contentService: new ContentService({
                posts: 'Text, images, videos with rich metadata',
                comments: 'Nested comments with threading',
                reactions: 'Likes, shares, custom reactions',
                moderation: 'Content moderation and reporting',
                database: 'MongoDB with sharding',
                search: 'Elasticsearch for content discovery',
            }),

            feedService: new FeedService({
                algorithm: 'Machine learning-based feed ranking',
                caching: 'Redis for hot feeds + CDN',
                personalization: 'User preference-based filtering',
                realTime: 'Event-driven feed updates',
                scaling: 'Horizontal scaling with load balancers',
            }),

            messagingService: new MessagingService({
                realTime: 'WebSocket clusters with Redis pub/sub',
                storage: 'MongoDB for message history',
                encryption: 'End-to-end encryption',
                fileSharing: 'S3 with signed URLs',
                groupChat: 'Multi-user chat rooms',
            }),

            mediaService: new MediaService({
                storage: 'S3 with multiple regions',
                processing: 'AWS Lambda for image/video processing',
                cdn: 'CloudFront for global distribution',
                optimization: 'Automatic format conversion',
                streaming: 'HLS video streaming',
            }),

            notificationService: new NotificationService({
                realTime: 'WebSocket + Server-Sent Events',
                push: 'FCM + APNS for mobile push',
                email: 'SES for email notifications',
                sms: 'Twilio for SMS notifications',
                preferences: 'Granular notification preferences',
            }),
        };

        // Inter-service communication
        this.setupServiceCommunication();
    }

    private setupServiceCommunication(): void {
        // Event-driven architecture
        const messageQueue = new MessageQueue({
            broker: 'Apache Kafka',
            topics: {
                'user.events': ['user.created', 'user.updated', 'user.deleted'],
                'content.events': ['post.created', 'post.updated', 'comment.added'],
                'interaction.events': ['like.added', 'share.created', 'follow.added'],
                'notification.events': ['notification.created', 'notification.sent'],
            },
        });

        // Event handlers
        messageQueue.subscribe('post.created', async (event) => {
            // Update feeds for followers
            await this.backend.feedService.updateFollowerFeeds(event.authorId, event.postId);
            
            // Send notifications to mentioned users
            if (event.mentions?.length > 0) {
                await this.backend.notificationService.createMentionNotifications(event);
            }
            
            // Index post for search
            await this.backend.contentService.indexPost(event.postId);
        });

        messageQueue.subscribe('follow.added', async (event) => {
            // Update feed algorithm for new follower
            await this.backend.feedService.updateUserFeedAlgorithm(event.followerId, event.followedId);
            
            // Send notification to followed user
            await this.backend.notificationService.createFollowNotification(event);
        });
    }

    // Real-time feed updates
    private initializeRealTimeSystems(): void {
        this.realTimeSystem = {
            feedUpdates: new RealTimeFeedSystem({
                websocket: 'Socket.IO clusters',
                rooms: 'User-specific feed rooms',
                updates: 'Incremental feed updates',
                fallback: 'Polling for unsupported browsers',
            }),

            messaging: new RealTimeMessaging({
                websocket: 'Socket.IO with Redis adapter',
                encryption: 'Client-side encryption before sending',
                typing: 'Typing indicators with debouncing',
                presence: 'Online/offline status tracking',
            }),

            notifications: new RealTimeNotifications({
                websocket: 'Server-Sent Events + WebSocket',
                push: 'Web Push API integration',
                queueing: 'Notification queue with priorities',
                batching: 'Batch notifications to reduce noise',
            }),
        };

        // Connection management
        this.setupConnectionManagement();
    }

    private setupConnectionManagement(): void {
        // WebSocket connection handling
        const connectionManager = new ConnectionManager({
            maxConnections: 10000,
            authentication: 'JWT token validation',
            rateLimiting: '100 messages per minute per user',
            reconnection: 'Automatic reconnection with backoff',
        });

        // Real-time feed updates
        connectionManager.onConnection('feed', async (socket, userId) => {
            // Join user's personal feed room
            socket.join(`feed:${userId}`);
            
            // Send initial feed data
            const initialFeed = await this.backend.feedService.getUserFeed(userId, { limit: 20 });
            socket.emit('feed:initial', initialFeed);
            
            // Handle feed interactions
            socket.on('feed:like', async (data) => {
                await this.backend.contentService.addLike(data.postId, userId);
                socket.to(`post:${data.postId}`).emit('post:liked', { postId: data.postId, userId });
            });

            socket.on('feed:comment', async (data) => {
                const comment = await this.backend.contentService.addComment(data.postId, userId, data.content);
                socket.to(`post:${data.postId}`).emit('post:commented', comment);
            });
        });

        // Real-time messaging
        connectionManager.onConnection('messaging', async (socket, userId) => {
            // Join user's message rooms
            const userRooms = await this.backend.messagingService.getUserRooms(userId);
            userRooms.forEach(room => socket.join(`chat:${room.id}`));

            // Handle message sending
            socket.on('message:send', async (data) => {
                const message = await this.backend.messagingService.sendMessage(
                    data.roomId, 
                    userId, 
                    data.content
                );
                
                socket.to(`chat:${data.roomId}`).emit('message:received', message);
            });

            // Handle typing indicators
            socket.on('typing:start', (data) => {
                socket.to(`chat:${data.roomId}`).emit('typing:user_started', { userId, roomId: data.roomId });
            });

            socket.on('typing:stop', (data) => {
                socket.to(`chat:${data.roomId}`).emit('typing:user_stopped', { userId, roomId: data.roomId });
            });
        });
    }
}

// Performance optimization for large-scale feeds
class InfiniteFeedOptimization {
    private virtualizer: VirtualizedList;
    private prefetchManager: PrefetchManager;
    private cacheManager: FeedCacheManager;

    constructor() {
        this.setupVirtualization();
        this.configurePrefetching();
        this.initializeCaching();
    }

    private setupVirtualization(): void {
        this.virtualizer = new VirtualizedList({
            itemHeight: 'dynamic', // Dynamic height based on content
            overscan: 5, // Render 5 items outside visible area
            windowSize: 10, // Keep 10 items in memory
            recycling: true, // Recycle DOM elements
        });
    }

    private configurePrefetching(): void {
        this.prefetchManager = new PrefetchManager({
            strategy: 'intersection-observer',
            threshold: 0.8, // Prefetch when 80% scrolled
            batchSize: 10, // Prefetch 10 items at a time
            maxConcurrent: 3, // Max 3 concurrent prefetch requests
        });
    }

    private initializeCaching(): void {
        this.cacheManager = new FeedCacheManager({
            maxSize: 1000, // Cache up to 1000 feed items
            ttl: 300000, // 5 minute TTL
            strategy: 'LRU', // Least Recently Used eviction
            persistence: 'IndexedDB', // Persist cache across sessions
        });
    }
}
```

## Interview Questions & Answers

### Q: How do you handle data consistency across microservices in a full-stack application?

**A:** I implement eventual consistency with event sourcing, use saga patterns for distributed transactions, implement compensating actions for rollbacks, and use message queues for reliable inter-service communication. I also implement proper monitoring and alerting for consistency violations and use database per service with well-defined API contracts.

### Q: What's your approach to real-time features in a scalable application?

**A:** I use WebSocket clustering with Redis pub/sub for horizontal scaling, implement connection management with proper authentication and rate limiting, use operational transformation for conflict resolution in collaborative features, and implement graceful fallbacks to polling for unsupported browsers. I also optimize with message batching and smart subscription management.

### Q: How do you design APIs for both web and mobile clients efficiently?

**A:** I design platform-agnostic APIs with consistent data formats, implement GraphQL for flexible data fetching, use API versioning for backward compatibility, implement proper caching strategies, and optimize payload sizes for mobile. I also provide different endpoints for different client needs while maintaining a unified core API structure.

### Q: How do you handle state management in complex full-stack applications?

**A:** I use layered state management with global state for shared data, local state for component-specific data, server state with proper caching and synchronization, and implement proper separation between UI state and business logic. I also use optimistic updates with rollback capabilities and implement proper error boundaries and loading states.

### Interview Tip:
*"Full-stack architecture requires deep understanding of both frontend and backend concerns, plus their integration points. Focus on data flow, state synchronization, performance optimization, and scalability. Always consider the complete user experience from data entry to real-time updates across all connected clients."*