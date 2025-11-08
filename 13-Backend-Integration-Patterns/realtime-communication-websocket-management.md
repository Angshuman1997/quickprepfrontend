# Real-time Communication and WebSocket Management

## Question
How do you implement reliable real-time communication in modern web applications?

## Answer

Real-time communication is essential for modern interactive applications. This requires robust WebSocket management, efficient data synchronization, conflict resolution, and fallback mechanisms. Here's a comprehensive approach to building scalable real-time features with excellent user experience.

## WebSocket Connection Management

### 1. **Enterprise-Grade WebSocket Client**

```typescript
// Advanced WebSocket manager with enterprise features
class EnterpriseWebSocketManager {
    private ws: WebSocket | null = null;
    private url: string;
    private protocols: string[];
    private connectionId: string | null = null;
    private reconnectAttempts = 0;
    private maxReconnectAttempts = 10;
    private reconnectDelay = 1000;
    private maxReconnectDelay = 30000;
    private heartbeatInterval: NodeJS.Timeout | null = null;
    private heartbeatFrequency = 30000;
    private lastHeartbeat = 0;
    private messageQueue: QueuedMessage[] = [];
    private subscriptions = new Map<string, Set<MessageHandler>>();
    private connectionState: ConnectionState = 'disconnected';
    private metrics: ConnectionMetrics;

    constructor(config: WebSocketConfig) {
        this.url = config.url;
        this.protocols = config.protocols || [];
        this.maxReconnectAttempts = config.maxReconnectAttempts || 10;
        this.heartbeatFrequency = config.heartbeatFrequency || 30000;
        this.metrics = new ConnectionMetrics();
    }

    // Connect with comprehensive error handling
    async connect(): Promise<void> {
        return new Promise((resolve, reject) => {
            if (this.connectionState === 'connected' || this.connectionState === 'connecting') {
                resolve();
                return;
            }

            this.connectionState = 'connecting';
            this.metrics.recordConnectionAttempt();

            try {
                this.ws = new WebSocket(this.url, this.protocols);
                this.setupEventHandlers(resolve, reject);
            } catch (error) {
                this.connectionState = 'disconnected';
                this.metrics.recordConnectionFailure(error);
                reject(error);
            }
        });
    }

    private setupEventHandlers(resolve: () => void, reject: (error: any) => void): void {
        if (!this.ws) return;

        this.ws.onopen = (event) => {
            console.log('WebSocket connected');
            this.connectionState = 'connected';
            this.reconnectAttempts = 0;
            this.reconnectDelay = 1000;
            
            this.metrics.recordSuccessfulConnection();
            this.startHeartbeat();
            this.processMessageQueue();
            this.emit('connected', { connectionId: this.connectionId });
            resolve();
        };

        this.ws.onmessage = (event) => {
            this.handleMessage(event);
        };

        this.ws.onclose = (event) => {
            console.log(`WebSocket closed: ${event.code} - ${event.reason}`);
            this.connectionState = 'disconnected';
            this.stopHeartbeat();
            this.metrics.recordDisconnection(event.code, event.reason);
            
            this.emit('disconnected', { 
                code: event.code, 
                reason: event.reason,
                wasClean: event.wasClean 
            });

            // Attempt reconnection for abnormal closures
            if (event.code !== 1000 && event.code !== 1001) {
                this.scheduleReconnect();
            }
        };

        this.ws.onerror = (event) => {
            console.error('WebSocket error:', event);
            this.metrics.recordError(event);
            this.emit('error', event);
            
            if (this.connectionState === 'connecting') {
                reject(event);
            }
        };
    }

    // Send message with delivery guarantees
    async send(message: any, options: SendOptions = {}): Promise<void> {
        const messageData: QueuedMessage = {
            id: this.generateMessageId(),
            payload: message,
            timestamp: Date.now(),
            retries: 0,
            maxRetries: options.maxRetries || 3,
            priority: options.priority || 'normal',
            requiresAck: options.requiresAck || false,
            timeout: options.timeout || 5000,
        };

        if (this.connectionState !== 'connected') {
            if (options.queueWhenDisconnected !== false) {
                this.queueMessage(messageData);
            }
            throw new Error('WebSocket not connected');
        }

        return this.sendMessage(messageData);
    }

    private async sendMessage(message: QueuedMessage): Promise<void> {
        return new Promise((resolve, reject) => {
            if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
                this.queueMessage(message);
                reject(new Error('WebSocket not ready'));
                return;
            }

            try {
                const payload = {
                    id: message.id,
                    data: message.payload,
                    timestamp: message.timestamp,
                    requiresAck: message.requiresAck,
                };

                this.ws.send(JSON.stringify(payload));
                this.metrics.recordMessageSent();

                if (message.requiresAck) {
                    this.setupAckTimeout(message, resolve, reject);
                } else {
                    resolve();
                }
            } catch (error) {
                this.metrics.recordSendFailure(error);
                reject(error);
            }
        });
    }

    // Subscribe to specific message types
    subscribe(messageType: string, handler: MessageHandler): () => void {
        if (!this.subscriptions.has(messageType)) {
            this.subscriptions.set(messageType, new Set());
        }
        
        this.subscriptions.get(messageType)!.add(handler);

        // Return unsubscribe function
        return () => {
            this.unsubscribe(messageType, handler);
        };
    }

    unsubscribe(messageType: string, handler: MessageHandler): void {
        const handlers = this.subscriptions.get(messageType);
        if (handlers) {
            handlers.delete(handler);
            if (handlers.size === 0) {
                this.subscriptions.delete(messageType);
            }
        }
    }

    // Handle incoming messages with routing
    private handleMessage(event: MessageEvent): void {
        try {
            const message = JSON.parse(event.data);
            this.metrics.recordMessageReceived();

            // Handle heartbeat
            if (message.type === 'heartbeat') {
                this.handleHeartbeat(message);
                return;
            }

            // Handle acknowledgments
            if (message.type === 'ack') {
                this.handleAcknowledment(message);
                return;
            }

            // Handle connection info
            if (message.type === 'connection_info') {
                this.connectionId = message.connectionId;
                return;
            }

            // Route to subscribers
            const messageType = message.type || 'default';
            const handlers = this.subscriptions.get(messageType);
            
            if (handlers) {
                handlers.forEach(handler => {
                    try {
                        handler(message.data || message, messageType);
                    } catch (error) {
                        console.error(`Message handler error for ${messageType}:`, error);
                    }
                });
            }

            // Send acknowledgment if required
            if (message.requiresAck) {
                this.sendAcknowledment(message.id);
            }

        } catch (error) {
            console.error('Error parsing WebSocket message:', error);
            this.metrics.recordMessageError(error);
        }
    }

    // Heartbeat mechanism
    private startHeartbeat(): void {
        this.stopHeartbeat();
        
        this.heartbeatInterval = setInterval(() => {
            if (this.connectionState === 'connected') {
                this.sendHeartbeat();
            }
        }, this.heartbeatFrequency);
    }

    private sendHeartbeat(): void {
        this.send({
            type: 'heartbeat',
            timestamp: Date.now(),
        }, { queueWhenDisconnected: false }).catch(error => {
            console.warn('Heartbeat failed:', error);
        });
    }

    private handleHeartbeat(message: any): void {
        this.lastHeartbeat = Date.now();
        
        // Respond to server heartbeat
        if (message.requiresResponse) {
            this.send({
                type: 'heartbeat_response',
                originalTimestamp: message.timestamp,
                timestamp: Date.now(),
            });
        }
    }

    // Reconnection logic with exponential backoff
    private scheduleReconnect(): void {
        if (this.reconnectAttempts >= this.maxReconnectAttempts) {
            console.error('Max reconnection attempts reached');
            this.emit('reconnectFailed');
            return;
        }

        this.reconnectAttempts++;
        const delay = Math.min(
            this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1),
            this.maxReconnectDelay
        );

        console.log(`Scheduling reconnection attempt ${this.reconnectAttempts} in ${delay}ms`);
        
        setTimeout(() => {
            if (this.connectionState === 'disconnected') {
                this.connect().catch(error => {
                    console.error('Reconnection failed:', error);
                });
            }
        }, delay);
    }

    // Message queue management
    private queueMessage(message: QueuedMessage): void {
        // Sort by priority and timestamp
        this.messageQueue.push(message);
        this.messageQueue.sort((a, b) => {
            const priorityOrder = { high: 3, normal: 2, low: 1 };
            const aPriority = priorityOrder[a.priority];
            const bPriority = priorityOrder[b.priority];
            
            if (aPriority !== bPriority) {
                return bPriority - aPriority;
            }
            
            return a.timestamp - b.timestamp;
        });

        // Limit queue size
        if (this.messageQueue.length > 100) {
            this.messageQueue = this.messageQueue.slice(0, 100);
        }
    }

    private async processMessageQueue(): Promise<void> {
        const queue = [...this.messageQueue];
        this.messageQueue = [];

        for (const message of queue) {
            try {
                await this.sendMessage(message);
            } catch (error) {
                if (message.retries < message.maxRetries) {
                    message.retries++;
                    this.queueMessage(message);
                } else {
                    console.error('Message failed after max retries:', message);
                }
            }
        }
    }

    // Connection metrics and monitoring
    getConnectionMetrics(): ConnectionMetrics {
        return this.metrics.getMetrics();
    }

    // Cleanup and disconnection
    disconnect(): void {
        if (this.ws) {
            this.ws.close(1000, 'Client disconnecting');
        }
        
        this.stopHeartbeat();
        this.connectionState = 'disconnected';
        this.subscriptions.clear();
        this.messageQueue = [];
    }

    private emit(event: string, data?: any): void {
        // Custom event emission - would integrate with event system
        window.dispatchEvent(new CustomEvent(`ws_${event}`, { detail: data }));
    }
}

// Connection metrics tracking
class ConnectionMetrics {
    private metrics = {
        connectionAttempts: 0,
        successfulConnections: 0,
        failedConnections: 0,
        totalDisconnections: 0,
        messagesSent: 0,
        messagesReceived: 0,
        sendFailures: 0,
        messageErrors: 0,
        averageLatency: 0,
        connectionUptime: 0,
        lastConnectionTime: 0,
    };

    recordConnectionAttempt(): void {
        this.metrics.connectionAttempts++;
    }

    recordSuccessfulConnection(): void {
        this.metrics.successfulConnections++;
        this.metrics.lastConnectionTime = Date.now();
    }

    recordConnectionFailure(error: any): void {
        this.metrics.failedConnections++;
    }

    recordDisconnection(code: number, reason: string): void {
        this.metrics.totalDisconnections++;
        
        if (this.metrics.lastConnectionTime > 0) {
            this.metrics.connectionUptime += Date.now() - this.metrics.lastConnectionTime;
        }
    }

    recordMessageSent(): void {
        this.metrics.messagesSent++;
    }

    recordMessageReceived(): void {
        this.metrics.messagesReceived++;
    }

    recordSendFailure(error: any): void {
        this.metrics.sendFailures++;
    }

    recordMessageError(error: any): void {
        this.metrics.messageErrors++;
    }

    recordError(error: any): void {
        // Track various error types
    }

    getMetrics(): ConnectionMetrics {
        return { ...this.metrics };
    }
}
```

### 2. **Real-time Data Synchronization**

```typescript
// Comprehensive data synchronization system
class RealTimeDataSynchronizer {
    private wsManager: EnterpriseWebSocketManager;
    private localState = new Map<string, any>();
    private subscriptions = new Map<string, Set<StateUpdateHandler>>();
    private conflictResolver: ConflictResolver;
    private operationalTransform: OperationalTransform;
    private syncQueue: SyncOperation[] = [];
    private optimisticUpdates = new Map<string, OptimisticUpdate>();

    constructor(wsConfig: WebSocketConfig) {
        this.wsManager = new EnterpriseWebSocketManager(wsConfig);
        this.conflictResolver = new ConflictResolver();
        this.operationalTransform = new OperationalTransform();
        this.setupEventHandlers();
    }

    // Subscribe to data changes
    subscribe(path: string, handler: StateUpdateHandler): () => void {
        if (!this.subscriptions.has(path)) {
            this.subscriptions.set(path, new Set());
            this.requestInitialData(path);
        }

        this.subscriptions.get(path)!.add(handler);

        // Return unsubscribe function
        return () => {
            this.unsubscribe(path, handler);
        };
    }

    unsubscribe(path: string, handler: StateUpdateHandler): void {
        const handlers = this.subscriptions.get(path);
        if (handlers) {
            handlers.delete(handler);
            if (handlers.size === 0) {
                this.subscriptions.delete(path);
                this.unsubscribeFromServer(path);
            }
        }
    }

    // Update data with optimistic updates
    async updateData(path: string, operation: UpdateOperation, options: UpdateOptions = {}): Promise<void> {
        const updateId = this.generateUpdateId();
        const timestamp = Date.now();

        // Apply optimistic update locally
        if (options.optimistic !== false) {
            const optimisticUpdate: OptimisticUpdate = {
                id: updateId,
                path,
                operation,
                timestamp,
                originalValue: this.getLocalValue(path),
            };

            this.applyOptimisticUpdate(optimisticUpdate);
        }

        // Send update to server
        try {
            await this.wsManager.send({
                type: 'data_update',
                updateId,
                path,
                operation,
                timestamp,
                clientId: this.wsManager.connectionId,
            }, {
                requiresAck: true,
                timeout: options.timeout || 10000,
            });

        } catch (error) {
            // Rollback optimistic update on failure
            if (options.optimistic !== false) {
                this.rollbackOptimisticUpdate(updateId);
            }
            throw error;
        }
    }

    // Batch operations for efficiency
    async batchUpdate(operations: BatchOperation[]): Promise<void> {
        const batchId = this.generateBatchId();
        const timestamp = Date.now();

        // Apply optimistic updates
        const optimisticUpdates: OptimisticUpdate[] = [];
        for (const op of operations) {
            if (op.optimistic !== false) {
                const updateId = this.generateUpdateId();
                const optimisticUpdate: OptimisticUpdate = {
                    id: updateId,
                    path: op.path,
                    operation: op.operation,
                    timestamp,
                    originalValue: this.getLocalValue(op.path),
                };
                
                optimisticUpdates.push(optimisticUpdate);
                this.applyOptimisticUpdate(optimisticUpdate);
            }
        }

        try {
            await this.wsManager.send({
                type: 'batch_update',
                batchId,
                operations,
                timestamp,
                clientId: this.wsManager.connectionId,
            }, {
                requiresAck: true,
                timeout: 15000,
            });

        } catch (error) {
            // Rollback all optimistic updates
            for (const update of optimisticUpdates) {
                this.rollbackOptimisticUpdate(update.id);
            }
            throw error;
        }
    }

    // Handle real-time updates from server
    private setupEventHandlers(): void {
        this.wsManager.subscribe('data_update', (data) => {
            this.handleServerUpdate(data);
        });

        this.wsManager.subscribe('conflict_resolution', (data) => {
            this.handleConflictResolution(data);
        });

        this.wsManager.subscribe('sync_complete', (data) => {
            this.handleSyncComplete(data);
        });

        this.wsManager.subscribe('connected', () => {
            this.resyncAllSubscriptions();
        });
    }

    private handleServerUpdate(data: ServerUpdate): void {
        const { path, operation, timestamp, clientId, updateId } = data;

        // Skip if this is our own update
        if (clientId === this.wsManager.connectionId) {
            this.confirmOptimisticUpdate(updateId);
            return;
        }

        // Check for conflicts with pending optimistic updates
        const conflictingUpdates = this.findConflictingUpdates(path, timestamp);
        
        if (conflictingUpdates.length > 0) {
            this.handleConflict(data, conflictingUpdates);
        } else {
            this.applyServerUpdate(path, operation, timestamp);
        }
    }

    private handleConflict(serverUpdate: ServerUpdate, conflictingUpdates: OptimisticUpdate[]): void {
        const resolution = this.conflictResolver.resolve(serverUpdate, conflictingUpdates);
        
        switch (resolution.strategy) {
            case 'server_wins':
                // Rollback conflicting updates and apply server update
                this.rollbackConflictingUpdates(conflictingUpdates);
                this.applyServerUpdate(serverUpdate.path, serverUpdate.operation, serverUpdate.timestamp);
                break;
                
            case 'client_wins':
                // Keep local changes and reject server update
                break;
                
            case 'merge':
                // Apply operational transformation to merge changes
                const mergedOperation = this.operationalTransform.transform(
                    serverUpdate.operation,
                    conflictingUpdates.map(u => u.operation)
                );
                this.applyServerUpdate(serverUpdate.path, mergedOperation, serverUpdate.timestamp);
                break;
        }
    }

    // Operational transformation for conflict resolution
    private applyOperationalTransform(
        serverOp: UpdateOperation, 
        clientOps: UpdateOperation[]
    ): UpdateOperation {
        return this.operationalTransform.transform(serverOp, clientOps);
    }

    // Optimistic update management
    private applyOptimisticUpdate(update: OptimisticUpdate): void {
        this.optimisticUpdates.set(update.id, update);
        this.applyOperation(update.path, update.operation);
        this.notifySubscribers(update.path);
    }

    private rollbackOptimisticUpdate(updateId: string): void {
        const update = this.optimisticUpdates.get(updateId);
        if (update) {
            this.localState.set(update.path, update.originalValue);
            this.optimisticUpdates.delete(updateId);
            this.notifySubscribers(update.path);
        }
    }

    private confirmOptimisticUpdate(updateId: string): void {
        this.optimisticUpdates.delete(updateId);
    }

    // Apply operations to local state
    private applyOperation(path: string, operation: UpdateOperation): void {
        const currentValue = this.getLocalValue(path);
        let newValue;

        switch (operation.type) {
            case 'set':
                newValue = operation.value;
                break;
            case 'merge':
                newValue = { ...currentValue, ...operation.value };
                break;
            case 'array_push':
                newValue = [...(currentValue || []), operation.value];
                break;
            case 'array_remove':
                newValue = (currentValue || []).filter((item: any) => 
                    item.id !== operation.value.id
                );
                break;
            case 'increment':
                newValue = (currentValue || 0) + operation.value;
                break;
            default:
                console.warn(`Unknown operation type: ${operation.type}`);
                return;
        }

        this.localState.set(path, newValue);
    }

    private getLocalValue(path: string): any {
        return this.localState.get(path);
    }

    private notifySubscribers(path: string): void {
        const handlers = this.subscriptions.get(path);
        if (handlers) {
            const value = this.getLocalValue(path);
            handlers.forEach(handler => {
                try {
                    handler(value, path);
                } catch (error) {
                    console.error(`State update handler error for ${path}:`, error);
                }
            });
        }
    }

    // Connection management
    async connect(): Promise<void> {
        await this.wsManager.connect();
    }

    disconnect(): void {
        this.wsManager.disconnect();
    }

    // Utility methods
    private generateUpdateId(): string {
        return `${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }

    private generateBatchId(): string {
        return `batch_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }
}

// Conflict resolution strategies
class ConflictResolver {
    resolve(serverUpdate: ServerUpdate, conflictingUpdates: OptimisticUpdate[]): ConflictResolution {
        // Timestamp-based resolution (Last Writer Wins)
        const latestLocalUpdate = Math.max(...conflictingUpdates.map(u => u.timestamp));
        
        if (serverUpdate.timestamp > latestLocalUpdate) {
            return { strategy: 'server_wins' };
        }

        // Check operation compatibility
        const isCompatible = this.checkOperationCompatibility(
            serverUpdate.operation,
            conflictingUpdates.map(u => u.operation)
        );

        if (isCompatible) {
            return { strategy: 'merge' };
        }

        // Default to server wins for safety
        return { strategy: 'server_wins' };
    }

    private checkOperationCompatibility(
        serverOp: UpdateOperation,
        clientOps: UpdateOperation[]
    ): boolean {
        // Simple compatibility check - in production, implement more sophisticated logic
        return serverOp.type === 'merge' && clientOps.every(op => op.type === 'merge');
    }
}

// Operational transformation implementation
class OperationalTransform {
    transform(serverOp: UpdateOperation, clientOps: UpdateOperation[]): UpdateOperation {
        let transformedOp = { ...serverOp };

        for (const clientOp of clientOps) {
            transformedOp = this.transformPair(transformedOp, clientOp);
        }

        return transformedOp;
    }

    private transformPair(op1: UpdateOperation, op2: UpdateOperation): UpdateOperation {
        // Implement operational transformation logic based on operation types
        if (op1.type === 'merge' && op2.type === 'merge') {
            return {
                type: 'merge',
                value: { ...op2.value, ...op1.value },
            };
        }

        // For other operation types, implement appropriate transformation logic
        return op1;
    }
}
```

### 3. **Real-time Collaboration Features**

```typescript
// Real-time collaboration system
class CollaborationManager {
    private dataSync: RealTimeDataSynchronizer;
    private presenceManager: PresenceManager;
    private cursorManager: CursorManager;
    private documentManager: DocumentManager;

    constructor(wsConfig: WebSocketConfig) {
        this.dataSync = new RealTimeDataSynchronizer(wsConfig);
        this.presenceManager = new PresenceManager(this.dataSync);
        this.cursorManager = new CursorManager(this.dataSync);
        this.documentManager = new DocumentManager(this.dataSync);
    }

    // Initialize collaboration for a document
    async joinDocument(documentId: string, userId: string): Promise<void> {
        await this.dataSync.connect();
        
        // Subscribe to document updates
        this.documentManager.subscribeToDocument(documentId);
        
        // Join presence
        await this.presenceManager.join(documentId, userId);
        
        // Initialize cursor tracking
        this.cursorManager.initialize(documentId, userId);
    }

    // Leave collaboration session
    async leaveDocument(documentId: string): Promise<void> {
        await this.presenceManager.leave(documentId);
        this.cursorManager.cleanup();
        this.documentManager.unsubscribeFromDocument(documentId);
    }

    // Document editing operations
    async insertText(documentId: string, position: number, text: string): Promise<void> {
        const operation: TextOperation = {
            type: 'insert',
            position,
            content: text,
            length: text.length,
        };

        await this.documentManager.applyOperation(documentId, operation);
    }

    async deleteText(documentId: string, position: number, length: number): Promise<void> {
        const operation: TextOperation = {
            type: 'delete',
            position,
            length,
        };

        await this.documentManager.applyOperation(documentId, operation);
    }

    // Cursor and selection tracking
    updateCursor(documentId: string, position: number): void {
        this.cursorManager.updatePosition(documentId, position);
    }

    updateSelection(documentId: string, start: number, end: number): void {
        this.cursorManager.updateSelection(documentId, { start, end });
    }
}

// Presence management for collaborative features
class PresenceManager {
    private currentPresence = new Map<string, UserPresence>();
    private presenceUpdateInterval: NodeJS.Timeout | null = null;

    constructor(private dataSync: RealTimeDataSynchronizer) {
        this.setupPresenceUpdates();
    }

    async join(roomId: string, userId: string, metadata?: any): Promise<void> {
        const presence: UserPresence = {
            userId,
            roomId,
            joinedAt: Date.now(),
            lastSeen: Date.now(),
            isActive: true,
            metadata: metadata || {},
        };

        this.currentPresence.set(userId, presence);

        await this.dataSync.updateData(`presence/${roomId}/${userId}`, {
            type: 'set',
            value: presence,
        });
    }

    async leave(roomId: string): Promise<void> {
        const userId = this.getCurrentUserId();
        if (userId) {
            await this.dataSync.updateData(`presence/${roomId}/${userId}`, {
                type: 'set',
                value: null,
            });
            
            this.currentPresence.delete(userId);
        }
    }

    updateActivity(roomId: string): void {
        const userId = this.getCurrentUserId();
        if (userId) {
            const presence = this.currentPresence.get(userId);
            if (presence) {
                presence.lastSeen = Date.now();
                presence.isActive = true;
                
                this.dataSync.updateData(`presence/${roomId}/${userId}/lastSeen`, {
                    type: 'set',
                    value: presence.lastSeen,
                });
            }
        }
    }

    private setupPresenceUpdates(): void {
        // Send periodic presence updates
        this.presenceUpdateInterval = setInterval(() => {
            this.sendPresenceUpdate();
        }, 30000);

        // Track user activity
        document.addEventListener('mousemove', this.handleUserActivity);
        document.addEventListener('keydown', this.handleUserActivity);
        document.addEventListener('click', this.handleUserActivity);
    }

    private handleUserActivity = () => {
        const presence = Array.from(this.currentPresence.values())[0];
        if (presence) {
            this.updateActivity(presence.roomId);
        }
    };

    private sendPresenceUpdate(): void {
        // Update presence for all active rooms
        this.currentPresence.forEach((presence) => {
            this.updateActivity(presence.roomId);
        });
    }

    private getCurrentUserId(): string | null {
        // Get current user ID from auth context
        return null; // Implementation depends on auth setup
    }
}

// Real-time cursor tracking
class CursorManager {
    private cursors = new Map<string, CursorData>();
    private cursorElements = new Map<string, HTMLElement>();

    constructor(private dataSync: RealTimeDataSynchronizer) {
        this.setupCursorSync();
    }

    initialize(documentId: string, userId: string): void {
        // Subscribe to cursor updates
        this.dataSync.subscribe(`cursors/${documentId}`, (cursors) => {
            this.handleCursorUpdates(cursors);
        });
    }

    updatePosition(documentId: string, position: number): void {
        const userId = this.getCurrentUserId();
        if (!userId) return;

        const cursorData: CursorData = {
            userId,
            position,
            timestamp: Date.now(),
        };

        this.dataSync.updateData(`cursors/${documentId}/${userId}`, {
            type: 'set',
            value: cursorData,
        });
    }

    updateSelection(documentId: string, selection: Selection): void {
        const userId = this.getCurrentUserId();
        if (!userId) return;

        const selectionData: SelectionData = {
            userId,
            start: selection.start,
            end: selection.end,
            timestamp: Date.now(),
        };

        this.dataSync.updateData(`selections/${documentId}/${userId}`, {
            type: 'set',
            value: selectionData,
        });
    }

    private handleCursorUpdates(cursors: Record<string, CursorData>): void {
        Object.entries(cursors).forEach(([userId, cursorData]) => {
            if (userId !== this.getCurrentUserId()) {
                this.renderCursor(userId, cursorData);
            }
        });
    }

    private renderCursor(userId: string, cursorData: CursorData): void {
        let cursorElement = this.cursorElements.get(userId);
        
        if (!cursorElement) {
            cursorElement = this.createCursorElement(userId);
            this.cursorElements.set(userId, cursorElement);
        }

        // Position cursor element based on text position
        const coordinates = this.getCoordinatesFromPosition(cursorData.position);
        cursorElement.style.left = `${coordinates.x}px`;
        cursorElement.style.top = `${coordinates.y}px`;
        cursorElement.style.display = 'block';

        // Hide cursor after inactivity
        setTimeout(() => {
            if (Date.now() - cursorData.timestamp > 5000) {
                cursorElement!.style.display = 'none';
            }
        }, 5000);
    }

    private createCursorElement(userId: string): HTMLElement {
        const element = document.createElement('div');
        element.className = 'collaboration-cursor';
        element.style.cssText = `
            position: absolute;
            width: 2px;
            height: 20px;
            background-color: #007acc;
            pointer-events: none;
            z-index: 1000;
            transition: all 0.1s ease;
        `;
        
        document.body.appendChild(element);
        return element;
    }

    private getCoordinatesFromPosition(position: number): { x: number; y: number } {
        // Convert text position to screen coordinates
        // Implementation depends on the text editor being used
        return { x: 0, y: 0 };
    }

    cleanup(): void {
        this.cursorElements.forEach(element => {
            element.remove();
        });
        this.cursorElements.clear();
        this.cursors.clear();
    }

    private getCurrentUserId(): string | null {
        return null; // Implementation depends on auth setup
    }

    private setupCursorSync(): void {
        // Implementation for cursor synchronization
    }
}
```

## Interview Questions & Answers

### Q: How do you handle WebSocket connection failures and ensure data consistency?

**A:** I implement automatic reconnection with exponential backoff, maintain message queues for offline periods, use heartbeat mechanisms to detect connection issues, and implement optimistic updates with rollback capabilities. I also use message acknowledgments for critical operations and maintain local state synchronization to ensure consistency across connection interruptions.

### Q: What's your approach to resolving conflicts in real-time collaborative editing?

**A:** I use operational transformation to merge conflicting operations, implement last-writer-wins with timestamps for simple conflicts, use optimistic updates with server reconciliation, and provide conflict resolution UI when automatic resolution isn't possible. I also implement proper cursor tracking and presence awareness to minimize conflicts through better user coordination.

### Q: How do you optimize WebSocket performance for high-frequency updates?

**A:** I implement message batching to reduce network overhead, use binary protocols for large data transfers, implement client-side throttling for rapid updates, and use differential updates instead of full state synchronization. I also implement smart subscription management and connection pooling for multiple real-time features.

### Q: How do you ensure WebSocket security in production applications?

**A:** I implement proper authentication and authorization for WebSocket connections, use secure WebSocket (WSS) with proper TLS configuration, validate all incoming messages on both client and server, implement rate limiting to prevent abuse, and use connection tokens with expiration. I also implement proper CORS policies and monitor for suspicious activity patterns.

### Interview Tip:
*"Real-time communication requires careful balance between performance, reliability, and user experience. Focus on robust error handling, efficient data synchronization, and graceful degradation. Always implement proper conflict resolution and ensure your real-time features enhance rather than complicate the user experience."*