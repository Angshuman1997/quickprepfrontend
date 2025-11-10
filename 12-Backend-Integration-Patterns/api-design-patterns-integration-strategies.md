# API Design Patterns and Integration Strategies

## Question
How do you design and implement robust API integrations in modern frontend applications?

## Answer

API integration is fundamental to modern frontend development, requiring careful consideration of design patterns, error handling, performance optimization, and user experience. Here's a comprehensive framework for building scalable, maintainable API integrations.

## RESTful API Integration Patterns

### 1. **API Client Architecture**

```typescript
// Comprehensive API client with interceptors and error handling
class APIClient {
    private baseURL: string;
    private defaultHeaders: Record<string, string>;
    private interceptors: Interceptors;
    private retryConfig: RetryConfig;

    constructor(config: APIClientConfig) {
        this.baseURL = config.baseURL;
        this.defaultHeaders = config.defaultHeaders || {};
        this.interceptors = new Interceptors();
        this.retryConfig = config.retryConfig || this.getDefaultRetryConfig();
        
        this.setupInterceptors();
    }

    // Request interceptor for authentication, logging, and transformation
    private setupInterceptors() {
        // Request interceptors
        this.interceptors.request.use(
            (config) => {
                // Add authentication
                const token = this.getAuthToken();
                if (token) {
                    config.headers.Authorization = `Bearer ${token}`;
                }

                // Add request ID for tracing
                config.headers['X-Request-ID'] = this.generateRequestId();

                // Log request (development only)
                if (process.env.NODE_ENV === 'development') {
                    console.log(`API Request: ${config.method?.toUpperCase()} ${config.url}`);
                }

                return config;
            },
            (error) => {
                console.error('Request interceptor error:', error);
                return Promise.reject(error);
            }
        );

        // Response interceptors
        this.interceptors.response.use(
            (response) => {
                // Log successful responses (development only)
                if (process.env.NODE_ENV === 'development') {
                    console.log(`API Response: ${response.status} ${response.config.url}`);
                }

                return response;
            },
            async (error) => {
                const originalRequest = error.config;

                // Handle token expiration
                if (error.response?.status === 401 && !originalRequest._retry) {
                    originalRequest._retry = true;
                    
                    try {
                        await this.refreshToken();
                        return this.request(originalRequest);
                    } catch (refreshError) {
                        this.handleAuthenticationFailure();
                        return Promise.reject(refreshError);
                    }
                }

                // Handle rate limiting with retry
                if (error.response?.status === 429) {
                    const retryAfter = this.getRetryAfterDelay(error.response);
                    return this.retryWithDelay(originalRequest, retryAfter);
                }

                // Handle network errors with retry
                if (this.isRetryableError(error)) {
                    return this.retryRequest(originalRequest);
                }

                return Promise.reject(this.normalizeError(error));
            }
        );
    }

    // Generic HTTP methods with type safety
    async get<T>(url: string, config?: RequestConfig): Promise<APIResponse<T>> {
        return this.request<T>({ ...config, method: 'GET', url });
    }

    async post<T>(url: string, data?: any, config?: RequestConfig): Promise<APIResponse<T>> {
        return this.request<T>({ ...config, method: 'POST', url, data });
    }

    async put<T>(url: string, data?: any, config?: RequestConfig): Promise<APIResponse<T>> {
        return this.request<T>({ ...config, method: 'PUT', url, data });
    }

    async patch<T>(url: string, data?: any, config?: RequestConfig): Promise<APIResponse<T>> {
        return this.request<T>({ ...config, method: 'PATCH', url, data });
    }

    async delete<T>(url: string, config?: RequestConfig): Promise<APIResponse<T>> {
        return this.request<T>({ ...config, method: 'DELETE', url });
    }

    // Main request method with comprehensive error handling
    private async request<T>(config: RequestConfig): Promise<APIResponse<T>> {
        const fullURL = `${this.baseURL}${config.url}`;
        const requestConfig = {
            ...config,
            url: fullURL,
            headers: { ...this.defaultHeaders, ...config.headers },
        };

        try {
            const response = await fetch(requestConfig.url, {
                method: requestConfig.method,
                headers: requestConfig.headers,
                body: requestConfig.data ? JSON.stringify(requestConfig.data) : undefined,
                signal: config.signal, // Support for AbortController
            });

            if (!response.ok) {
                throw new APIError(
                    `HTTP ${response.status}: ${response.statusText}`,
                    response.status,
                    await response.text()
                );
            }

            const data = await response.json();
            return {
                data,
                status: response.status,
                headers: response.headers,
                config: requestConfig,
            };
        } catch (error) {
            if (error instanceof APIError) {
                throw error;
            }
            throw new APIError('Network Error', 0, error.message);
        }
    }

    // Retry logic with exponential backoff
    private async retryRequest(config: RequestConfig): Promise<any> {
        const maxRetries = this.retryConfig.maxRetries;
        const baseDelay = this.retryConfig.baseDelay;

        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                return await this.request(config);
            } catch (error) {
                if (attempt === maxRetries || !this.isRetryableError(error)) {
                    throw error;
                }

                const delay = this.calculateExponentialBackoff(attempt, baseDelay);
                await this.sleep(delay);
            }
        }
    }

    private calculateExponentialBackoff(attempt: number, baseDelay: number): number {
        return Math.min(baseDelay * Math.pow(2, attempt - 1), this.retryConfig.maxDelay);
    }
}

// Type-safe API service layer
class UserService {
    constructor(private apiClient: APIClient) {}

    async getUser(id: string): Promise<User> {
        const response = await this.apiClient.get<User>(`/users/${id}`);
        return response.data;
    }

    async getUsersWithPagination(params: PaginationParams): Promise<PaginatedResponse<User>> {
        const response = await this.apiClient.get<PaginatedResponse<User>>('/users', {
            params: {
                page: params.page,
                limit: params.limit,
                sortBy: params.sortBy,
                sortOrder: params.sortOrder,
                ...params.filters,
            },
        });
        return response.data;
    }

    async createUser(userData: CreateUserRequest): Promise<User> {
        const response = await this.apiClient.post<User>('/users', userData);
        return response.data;
    }

    async updateUser(id: string, userData: Partial<UpdateUserRequest>): Promise<User> {
        const response = await this.apiClient.patch<User>(`/users/${id}`, userData);
        return response.data;
    }

    async deleteUser(id: string): Promise<void> {
        await this.apiClient.delete(`/users/${id}`);
    }

    // Bulk operations
    async bulkCreateUsers(users: CreateUserRequest[]): Promise<BulkOperationResult<User>> {
        const response = await this.apiClient.post<BulkOperationResult<User>>('/users/bulk', {
            users,
        });
        return response.data;
    }

    // Search with debouncing
    async searchUsers(query: string, signal?: AbortSignal): Promise<User[]> {
        const response = await this.apiClient.get<User[]>('/users/search', {
            params: { q: query },
            signal,
        });
        return response.data;
    }
}
```

### 2. **GraphQL Integration Patterns**

```typescript
// GraphQL client with caching and optimistic updates
class GraphQLClient {
    private endpoint: string;
    private headers: Record<string, string>;
    private cache: GraphQLCache;

    constructor(config: GraphQLClientConfig) {
        this.endpoint = config.endpoint;
        this.headers = config.headers || {};
        this.cache = new GraphQLCache(config.cacheConfig);
    }

    // Execute GraphQL query with caching
    async query<T>(
        query: string, 
        variables?: Record<string, any>,
        options?: QueryOptions
    ): Promise<GraphQLResponse<T>> {
        // Check cache first
        const cacheKey = this.generateCacheKey(query, variables);
        if (!options?.skipCache) {
            const cachedResult = this.cache.get<T>(cacheKey);
            if (cachedResult && !this.isCacheExpired(cachedResult)) {
                return cachedResult;
            }
        }

        const response = await this.executeRequest({
            query,
            variables,
        });

        // Cache successful responses
        if (response.data && !response.errors) {
            this.cache.set(cacheKey, response, options?.cacheTTL);
        }

        return response;
    }

    // Execute GraphQL mutation with optimistic updates
    async mutate<T>(
        mutation: string,
        variables?: Record<string, any>,
        options?: MutationOptions<T>
    ): Promise<GraphQLResponse<T>> {
        // Apply optimistic update
        if (options?.optimisticUpdate) {
            this.cache.applyOptimisticUpdate(options.optimisticUpdate);
        }

        try {
            const response = await this.executeRequest({
                query: mutation,
                variables,
            });

            // Update cache with real data
            if (response.data && options?.updateCache) {
                options.updateCache(this.cache, response.data);
            }

            return response;
        } catch (error) {
            // Rollback optimistic update on error
            if (options?.optimisticUpdate) {
                this.cache.rollbackOptimisticUpdate();
            }
            throw error;
        }
    }

    // GraphQL subscription handling
    subscribe<T>(
        subscription: string,
        variables?: Record<string, any>,
        onNext?: (data: T) => void,
        onError?: (error: Error) => void
    ): Subscription {
        const ws = new WebSocket(this.getWebSocketEndpoint());
        
        ws.onopen = () => {
            ws.send(JSON.stringify({
                type: 'start',
                payload: { query: subscription, variables },
            }));
        };

        ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            
            if (message.type === 'data' && onNext) {
                onNext(message.payload.data);
            }
            
            if (message.type === 'error' && onError) {
                onError(new Error(message.payload.message));
            }
        };

        return {
            unsubscribe: () => {
                ws.close();
            },
        };
    }

    private async executeRequest(request: GraphQLRequest): Promise<GraphQLResponse<any>> {
        const response = await fetch(this.endpoint, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                ...this.headers,
            },
            body: JSON.stringify(request),
        });

        if (!response.ok) {
            throw new GraphQLError(`HTTP ${response.status}: ${response.statusText}`);
        }

        return response.json();
    }
}

// GraphQL cache implementation with normalization
class GraphQLCache {
    private data: Map<string, CacheEntry> = new Map();
    private normalizedCache: NormalizedCache = new Map();
    private optimisticUpdates: OptimisticUpdate[] = [];

    // Normalize and store data
    set<T>(key: string, response: GraphQLResponse<T>, ttl?: number): void {
        const entry: CacheEntry = {
            data: response,
            timestamp: Date.now(),
            ttl: ttl || 300000, // 5 minutes default
        };

        this.data.set(key, entry);
        this.normalizeData(response.data);
    }

    // Retrieve from cache
    get<T>(key: string): GraphQLResponse<T> | null {
        const entry = this.data.get(key);
        if (!entry) return null;

        // Apply optimistic updates
        const dataWithOptimistic = this.applyOptimisticUpdatesTo(entry.data);
        
        return {
            ...entry.data,
            data: dataWithOptimistic,
        };
    }

    // Normalize GraphQL response data
    private normalizeData(data: any, typeName?: string): void {
        if (!data || typeof data !== 'object') return;

        if (Array.isArray(data)) {
            data.forEach(item => this.normalizeData(item));
            return;
        }

        // Store normalized entity
        if (data.id && data.__typename) {
            const entityKey = `${data.__typename}:${data.id}`;
            this.normalizedCache.set(entityKey, data);
        }

        // Recursively normalize nested objects
        Object.values(data).forEach(value => {
            if (typeof value === 'object') {
                this.normalizeData(value);
            }
        });
    }

    // Apply optimistic update
    applyOptimisticUpdate(update: OptimisticUpdate): void {
        this.optimisticUpdates.push(update);
    }

    // Rollback optimistic updates
    rollbackOptimisticUpdate(): void {
        this.optimisticUpdates.pop();
    }

    private applyOptimisticUpdatesTo(data: any): any {
        return this.optimisticUpdates.reduce((acc, update) => {
            return update.apply(acc);
        }, data);
    }
}
```

### 3. **Real-time Communication Patterns**

```typescript
// WebSocket connection manager with reconnection
class WebSocketManager {
    private ws: WebSocket | null = null;
    private url: string;
    private protocols?: string[];
    private reconnectAttempts = 0;
    private maxReconnectAttempts = 5;
    private reconnectInterval = 1000;
    private listeners: Map<string, Set<Function>> = new Map();
    private heartbeatInterval: NodeJS.Timeout | null = null;
    private connectionState: ConnectionState = 'disconnected';

    constructor(url: string, protocols?: string[]) {
        this.url = url;
        this.protocols = protocols;
    }

    // Connect with automatic reconnection
    connect(): Promise<void> {
        return new Promise((resolve, reject) => {
            try {
                this.ws = new WebSocket(this.url, this.protocols);
                this.connectionState = 'connecting';

                this.ws.onopen = () => {
                    console.log('WebSocket connected');
                    this.connectionState = 'connected';
                    this.reconnectAttempts = 0;
                    this.startHeartbeat();
                    this.emit('connected');
                    resolve();
                };

                this.ws.onmessage = (event) => {
                    this.handleMessage(event);
                };

                this.ws.onclose = (event) => {
                    console.log('WebSocket closed:', event.code, event.reason);
                    this.connectionState = 'disconnected';
                    this.stopHeartbeat();
                    this.emit('disconnected', { code: event.code, reason: event.reason });
                    
                    if (event.code !== 1000) { // Not a normal closure
                        this.attemptReconnect();
                    }
                };

                this.ws.onerror = (error) => {
                    console.error('WebSocket error:', error);
                    this.emit('error', error);
                    reject(error);
                };

            } catch (error) {
                reject(error);
            }
        });
    }

    // Send message with queuing for disconnected state
    send(message: any): Promise<void> {
        return new Promise((resolve, reject) => {
            if (this.connectionState !== 'connected' || !this.ws) {
                // Queue message for later
                this.queueMessage(message);
                reject(new Error('WebSocket not connected'));
                return;
            }

            try {
                const messageString = typeof message === 'string' ? message : JSON.stringify(message);
                this.ws.send(messageString);
                resolve();
            } catch (error) {
                reject(error);
            }
        });
    }

    // Event listener management
    on(event: string, callback: Function): void {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, new Set());
        }
        this.listeners.get(event)!.add(callback);
    }

    off(event: string, callback?: Function): void {
        if (!this.listeners.has(event)) return;
        
        if (callback) {
            this.listeners.get(event)!.delete(callback);
        } else {
            this.listeners.delete(event);
        }
    }

    // Close connection
    close(): void {
        if (this.ws) {
            this.ws.close(1000, 'Client closing connection');
            this.ws = null;
        }
        this.stopHeartbeat();
        this.connectionState = 'disconnected';
    }

    private handleMessage(event: MessageEvent): void {
        try {
            const message = JSON.parse(event.data);
            
            // Handle heartbeat
            if (message.type === 'ping') {
                this.send({ type: 'pong' });
                return;
            }

            // Emit message event
            this.emit('message', message);
            
            // Emit specific message type if available
            if (message.type) {
                this.emit(message.type, message.payload);
            }
        } catch (error) {
            console.error('Error parsing WebSocket message:', error);
            this.emit('message', event.data);
        }
    }

    private attemptReconnect(): void {
        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.error('Max reconnection attempts reached');
            this.emit('reconnectFailed');
            return;
        }

        this.reconnectAttempts++;
        const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
        
        setTimeout(() => {
            console.log(`Attempting to reconnect (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
            this.connect().catch(() => {
                // Will automatically retry due to onerror handler
            });
        }, delay);
    }

    private startHeartbeat(): void {
        this.heartbeatInterval = setInterval(() => {
            this.send({ type: 'ping' });
        }, 30000);
    }

    private stopHeartbeat(): void {
        if (this.heartbeatInterval) {
            clearInterval(this.heartbeatInterval);
            this.heartbeatInterval = null;
        }
    }

    private emit(event: string, data?: any): void {
        const eventListeners = this.listeners.get(event);
        if (eventListeners) {
            eventListeners.forEach(callback => callback(data));
        }
    }

    private queueMessage(message: any): void {
        // Implementation for message queuing when disconnected
        // Could use localStorage or in-memory queue
    }
}

// Real-time data synchronization
class RealTimeDataSync {
    private wsManager: WebSocketManager;
    private dataStore: Map<string, any> = new Map();
    private subscribers: Map<string, Set<Function>> = new Map();

    constructor(wsUrl: string) {
        this.wsManager = new WebSocketManager(wsUrl);
        this.setupEventHandlers();
    }

    // Subscribe to data changes
    subscribe(key: string, callback: (data: any) => void): () => void {
        if (!this.subscribers.has(key)) {
            this.subscribers.set(key, new Set());
            // Request initial data
            this.wsManager.send({
                type: 'subscribe',
                key: key,
            });
        }

        this.subscribers.get(key)!.add(callback);

        // Return unsubscribe function
        return () => {
            this.unsubscribe(key, callback);
        };
    }

    // Unsubscribe from data changes
    unsubscribe(key: string, callback: Function): void {
        const keySubscribers = this.subscribers.get(key);
        if (keySubscribers) {
            keySubscribers.delete(callback);
            
            if (keySubscribers.size === 0) {
                this.subscribers.delete(key);
                this.wsManager.send({
                    type: 'unsubscribe',
                    key: key,
                });
            }
        }
    }

    // Update data with optimistic updates
    updateData(key: string, data: any, optimistic = false): void {
        if (optimistic) {
            // Apply optimistic update locally
            this.dataStore.set(key, data);
            this.notifySubscribers(key, data);
        }

        // Send update to server
        this.wsManager.send({
            type: 'update',
            key: key,
            data: data,
        });
    }

    private setupEventHandlers(): void {
        this.wsManager.on('connected', () => {
            // Resubscribe to all keys after reconnection
            this.subscribers.forEach((_, key) => {
                this.wsManager.send({
                    type: 'subscribe',
                    key: key,
                });
            });
        });

        this.wsManager.on('data_update', (message: any) => {
            const { key, data } = message;
            this.dataStore.set(key, data);
            this.notifySubscribers(key, data);
        });

        this.wsManager.on('data_deleted', (message: any) => {
            const { key } = message;
            this.dataStore.delete(key);
            this.notifySubscribers(key, null);
        });
    }

    private notifySubscribers(key: string, data: any): void {
        const keySubscribers = this.subscribers.get(key);
        if (keySubscribers) {
            keySubscribers.forEach(callback => callback(data));
        }
    }
}
```

### 4. **API Error Handling and User Experience**

```typescript
// Comprehensive error handling framework
class APIErrorHandler {
    private errorStrategies: Map<string, ErrorStrategy> = new Map();

    constructor() {
        this.setupDefaultStrategies();
    }

    // Handle API errors with context-aware strategies
    handleError(error: APIError, context: ErrorContext): ErrorHandlingResult {
        const strategy = this.selectStrategy(error, context);
        return strategy.handle(error, context);
    }

    private setupDefaultStrategies(): void {
        // Network errors
        this.errorStrategies.set('network', {
            handle: (error: APIError, context: ErrorContext) => ({
                userMessage: 'Connection problem. Please check your internet connection.',
                action: 'retry',
                retryable: true,
                severity: 'warning',
            }),
        });

        // Authentication errors
        this.errorStrategies.set('auth', {
            handle: (error: APIError, context: ErrorContext) => ({
                userMessage: 'Session expired. Please log in again.',
                action: 'redirect',
                redirectTo: '/login',
                retryable: false,
                severity: 'error',
            }),
        });

        // Server errors
        this.errorStrategies.set('server', {
            handle: (error: APIError, context: ErrorContext) => ({
                userMessage: 'Server error. Please try again later.',
                action: 'retry',
                retryable: true,
                delay: 5000,
                severity: 'error',
            }),
        });

        // Validation errors
        this.errorStrategies.set('validation', {
            handle: (error: APIError, context: ErrorContext) => ({
                userMessage: 'Please check your input and try again.',
                action: 'show_details',
                details: error.details,
                retryable: true,
                severity: 'warning',
            }),
        });

        // Rate limiting
        this.errorStrategies.set('rate_limit', {
            handle: (error: APIError, context: ErrorContext) => ({
                userMessage: 'Too many requests. Please wait before trying again.',
                action: 'delay',
                delay: this.getRetryAfterDelay(error),
                retryable: true,
                severity: 'info',
            }),
        });
    }

    private selectStrategy(error: APIError, context: ErrorContext): ErrorStrategy {
        if (error.status === 0) return this.errorStrategies.get('network')!;
        if (error.status === 401) return this.errorStrategies.get('auth')!;
        if (error.status === 422 || error.status === 400) return this.errorStrategies.get('validation')!;
        if (error.status === 429) return this.errorStrategies.get('rate_limit')!;
        if (error.status >= 500) return this.errorStrategies.get('server')!;

        return this.errorStrategies.get('network')!; // Default
    }
}

// User-friendly error boundaries and notifications
class ErrorNotificationSystem {
    private notifications: ErrorNotification[] = [];
    private maxNotifications = 5;

    // Show contextual error notification
    showError(error: ErrorHandlingResult, context: ErrorContext): void {
        const notification: ErrorNotification = {
            id: this.generateId(),
            message: error.userMessage,
            severity: error.severity,
            action: error.action,
            context: context.operation,
            timestamp: Date.now(),
            dismissable: error.severity !== 'error',
        };

        // Add retry action if applicable
        if (error.retryable && context.retryFunction) {
            notification.actions = [{
                label: 'Retry',
                handler: () => {
                    this.dismissNotification(notification.id);
                    setTimeout(context.retryFunction!, error.delay || 0);
                },
            }];
        }

        this.addNotification(notification);
    }

    // Batch error handling for multiple operations
    showBatchErrors(errors: BatchErrorResult[]): void {
        const groupedErrors = this.groupErrorsByType(errors);
        
        Object.entries(groupedErrors).forEach(([type, errors]) => {
            if (errors.length === 1) {
                this.showError(errors[0].result, errors[0].context);
            } else {
                this.showBatchNotification(type, errors);
            }
        });
    }

    private addNotification(notification: ErrorNotification): void {
        this.notifications.unshift(notification);
        
        // Limit number of notifications
        if (this.notifications.length > this.maxNotifications) {
            this.notifications = this.notifications.slice(0, this.maxNotifications);
        }

        this.renderNotifications();

        // Auto-dismiss info notifications
        if (notification.severity === 'info' && notification.dismissable) {
            setTimeout(() => {
                this.dismissNotification(notification.id);
            }, 5000);
        }
    }

    private renderNotifications(): void {
        // Emit event for UI to update
        window.dispatchEvent(new CustomEvent('errorNotificationsUpdated', {
            detail: this.notifications,
        }));
    }
}
```

## Interview Questions & Answers

### Q: How do you handle API versioning in a large frontend application?

**A:** I implement versioning at multiple levels: use API versioning headers to specify versions per request, maintain backward compatibility adapters for gradual migration, implement feature flags to toggle between API versions, and create abstraction layers that hide versioning complexity from components. I also establish clear migration timelines and deprecation strategies with stakeholders.

### Q: What's your approach to handling offline scenarios in API-dependent applications?

**A:** I implement a multi-layered offline strategy: use service workers for request caching and background sync, implement local storage for critical data persistence, create optimistic updates with conflict resolution, and provide clear offline indicators and functionality. I also queue failed requests for retry when connectivity returns and gracefully degrade features that require network access.

### Q: How do you optimize API performance in React applications?

**A:** I use multiple optimization techniques: implement intelligent caching with cache invalidation strategies, use request deduplication to prevent duplicate calls, implement pagination and virtual scrolling for large datasets, use GraphQL for efficient data fetching, and optimize with request batching and parallel loading. I also implement proper loading states and error boundaries for better UX.

### Q: How do you handle real-time updates while maintaining data consistency?

**A:** I implement optimistic updates with rollback mechanisms, use WebSocket connections with automatic reconnection, implement conflict resolution strategies for concurrent updates, and maintain proper state synchronization between local and server state. I also use message queuing for reliable delivery and implement proper error handling for real-time failures.

### Interview Tip:
*"Effective API integration combines technical robustness with excellent user experience. Focus on error handling, performance optimization, and offline capabilities. Always design with failure scenarios in mind and provide clear feedback to users about what's happening with their data and requests."*