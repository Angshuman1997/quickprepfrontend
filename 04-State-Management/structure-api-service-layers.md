# How to structure API service layers?

## Question
How to structure API service layers?

## Answer

API service layers provide a clean separation between your application's business logic and external API communications. A well-structured service layer handles HTTP requests, response transformation, error handling, authentication, and caching, making your components and Redux logic cleaner and more testable.

## Why Structure API Services?

### 1. **Problems with Inline API Calls**

**Before (API calls in components):**
```typescript
// Component with mixed concerns
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    useEffect(() => {
        const fetchUser = async () => {
            setLoading(true);
            try {
                const token = localStorage.getItem('authToken');
                const response = await fetch(`/api/users/${userId}`, {
                    headers: {
                        'Authorization': `Bearer ${token}`,
                        'Content-Type': 'application/json',
                    },
                });

                if (!response.ok) {
                    if (response.status === 401) {
                        // Handle token refresh
                        const newToken = await refreshToken();
                        localStorage.setItem('authToken', newToken);
                        // Retry request...
                    }
                    throw new Error('Failed to fetch user');
                }

                const userData = await response.json();
                setUser(userData);
            } catch (error) {
                setError(error.message);
                // Log error to service
                console.error('User fetch failed:', error);
            } finally {
                setLoading(false);
            }
        };

        fetchUser();
    }, [userId]);

    // Component logic mixed with API concerns
}
```

**Problems:**
- **Mixed concerns**: UI logic mixed with HTTP details
- **Code duplication**: Same fetch logic repeated across components
- **Hard to test**: HTTP calls make component testing difficult
- **Maintenance nightmare**: API changes require updating multiple files
- **Error handling inconsistency**: Different error handling patterns

### 2. **Benefits of Service Layers**

- **Separation of concerns**: UI focuses on presentation, services handle data
- **Reusability**: Same API calls used across multiple components
- **Testability**: Services can be mocked easily
- **Maintainability**: API changes isolated to service layer
- **Consistency**: Standardized error handling and response transformation

## Basic Service Layer Structure

### 1. **Simple API Service**

```
src/
├── services/
│   ├── api/
│   │   ├── base.ts          # Base API configuration
│   │   ├── users.ts         # User-related API calls
│   │   ├── products.ts      # Product-related API calls
│   │   └── index.ts         # Service exports
│   ├── types/
│   │   ├── user.ts
│   │   ├── product.ts
│   │   └── index.ts
│   └── utils/
│       ├── errorHandling.ts
│       └── responseTransformers.ts
```

**Base API configuration:**
```typescript
// services/api/base.ts
export interface ApiConfig {
    baseURL: string;
    timeout: number;
    headers: Record<string, string>;
}

export const apiConfig: ApiConfig = {
    baseURL: process.env.REACT_APP_API_URL || 'http://localhost:3001/api',
    timeout: 10000,
    headers: {
        'Content-Type': 'application/json',
    },
};

export class ApiError extends Error {
    constructor(
        message: string,
        public status: number,
        public data?: any
    ) {
        super(message);
        this.name = 'ApiError';
    }
}

export const handleApiResponse = async (response: Response) => {
    if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new ApiError(
            errorData.message || `HTTP ${response.status}`,
            response.status,
            errorData
        );
    }

    return response.json();
};
```

**Feature-specific service:**
```typescript
// services/api/users.ts
import { apiConfig, handleApiResponse, ApiError } from './base';

export interface User {
    id: number;
    name: string;
    email: string;
    role: string;
}

export interface CreateUserData {
    name: string;
    email: string;
    password: string;
}

class UsersService {
    private getAuthHeaders(): Record<string, string> {
        const token = localStorage.getItem('authToken');
        return token ? { Authorization: `Bearer ${token}` } : {};
    }

    async getUsers(): Promise<User[]> {
        const response = await fetch(`${apiConfig.baseURL}/users`, {
            headers: {
                ...apiConfig.headers,
                ...this.getAuthHeaders(),
            },
        });

        return handleApiResponse(response);
    }

    async getUserById(id: number): Promise<User> {
        const response = await fetch(`${apiConfig.baseURL}/users/${id}`, {
            headers: {
                ...apiConfig.headers,
                ...this.getAuthHeaders(),
            },
        });

        return handleApiResponse(response);
    }

    async createUser(userData: CreateUserData): Promise<User> {
        const response = await fetch(`${apiConfig.baseURL}/users`, {
            method: 'POST',
            headers: {
                ...apiConfig.headers,
                ...this.getAuthHeaders(),
            },
            body: JSON.stringify(userData),
        });

        return handleApiResponse(response);
    }

    async updateUser(id: number, updates: Partial<User>): Promise<User> {
        const response = await fetch(`${apiConfig.baseURL}/users/${id}`, {
            method: 'PATCH',
            headers: {
                ...apiConfig.headers,
                ...this.getAuthHeaders(),
            },
            body: JSON.stringify(updates),
        });

        return handleApiResponse(response);
    }

    async deleteUser(id: number): Promise<void> {
        const response = await fetch(`${apiConfig.baseURL}/users/${id}`, {
            method: 'DELETE',
            headers: {
                ...apiConfig.headers,
                ...this.getAuthHeaders(),
            },
        });

        if (!response.ok) {
            const errorData = await response.json().catch(() => ({}));
            throw new ApiError(
                errorData.message || `HTTP ${response.status}`,
                response.status,
                errorData
            );
        }
    }
}

export const usersService = new UsersService();
```

**Service usage in components:**
```typescript
// Component using service layer
function UserProfile({ userId }) {
    const [user, setUser] = useState<User | null>(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState<string | null>(null);

    useEffect(() => {
        const fetchUser = async () => {
            setLoading(true);
            setError(null);

            try {
                const userData = await usersService.getUserById(userId);
                setUser(userData);
            } catch (err) {
                setError(err instanceof ApiError ? err.message : 'An error occurred');
            } finally {
                setLoading(false);
            }
        };

        fetchUser();
    }, [userId]);

    // Clean component focused on UI logic
}
```

## Advanced Service Layer Patterns

### 1. **HTTP Client Wrapper**

**Axios-based service:**
```typescript
// services/api/httpClient.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import { apiConfig } from './base';

class HttpClient {
    private client: AxiosInstance;

    constructor() {
        this.client = axios.create({
            baseURL: apiConfig.baseURL,
            timeout: apiConfig.timeout,
            headers: apiConfig.headers,
        });

        this.setupInterceptors();
    }

    private setupInterceptors() {
        // Request interceptor for auth
        this.client.interceptors.request.use(
            (config) => {
                const token = localStorage.getItem('authToken');
                if (token) {
                    config.headers.Authorization = `Bearer ${token}`;
                }
                return config;
            },
            (error) => Promise.reject(error)
        );

        // Response interceptor for token refresh
        this.client.interceptors.response.use(
            (response) => response,
            async (error) => {
                if (error.response?.status === 401) {
                    try {
                        // Attempt token refresh
                        const refreshResponse = await axios.post(
                            `${apiConfig.baseURL}/auth/refresh`,
                            {
                                refreshToken: localStorage.getItem('refreshToken'),
                            }
                        );

                        const { accessToken } = refreshResponse.data;
                        localStorage.setItem('authToken', accessToken);

                        // Retry original request
                        return this.client(error.config);
                    } catch (refreshError) {
                        // Refresh failed, redirect to login
                        localStorage.removeItem('authToken');
                        localStorage.removeItem('refreshToken');
                        window.location.href = '/login';
                        return Promise.reject(refreshError);
                    }
                }

                return Promise.reject(error);
            }
        );
    }

    async get<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
        const response = await this.client.get(url, config);
        return response.data;
    }

    async post<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
        const response = await this.client.post(url, data, config);
        return response.data;
    }

    async put<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
        const response = await this.client.put(url, data, config);
        return response.data;
    }

    async patch<T>(url: string, data?: any, config?: AxiosRequestConfig): Promise<T> {
        const response = await this.client.patch(url, data, config);
        return response.data;
    }

    async delete<T>(url: string, config?: AxiosRequestConfig): Promise<T> {
        const response = await this.client.delete(url, config);
        return response.data;
    }
}

export const httpClient = new HttpClient();
```

**Service using HTTP client:**
```typescript
// services/api/users.ts
import { httpClient } from './httpClient';

export class UsersService {
    async getUsers(params?: { page?: number; limit?: number }): Promise<User[]> {
        return httpClient.get('/users', { params });
    }

    async getUserById(id: number): Promise<User> {
        return httpClient.get(`/users/${id}`);
    }

    async createUser(userData: CreateUserData): Promise<User> {
        return httpClient.post('/users', userData);
    }

    async updateUser(id: number, updates: Partial<User>): Promise<User> {
        return httpClient.patch(`/users/${id}`, updates);
    }

    async deleteUser(id: number): Promise<void> {
        return httpClient.delete(`/users/${id}`);
    }

    async searchUsers(query: string): Promise<User[]> {
        return httpClient.get('/users/search', {
            params: { q: query },
        });
    }

    async uploadAvatar(userId: number, file: File): Promise<{ avatarUrl: string }> {
        const formData = new FormData();
        formData.append('avatar', file);

        return httpClient.post(`/users/${userId}/avatar`, formData, {
            headers: {
                'Content-Type': 'multipart/form-data',
            },
        });
    }
}

export const usersService = new UsersService();
```

### 2. **Repository Pattern**

**Repository abstraction:**
```typescript
// services/repositories/interfaces/IUserRepository.ts
export interface IUserRepository {
    findAll(): Promise<User[]>;
    findById(id: number): Promise<User | null>;
    create(data: CreateUserData): Promise<User>;
    update(id: number, data: Partial<User>): Promise<User>;
    delete(id: number): Promise<void>;
    search(query: string): Promise<User[]>;
}

// services/repositories/UserRepository.ts
import { IUserRepository } from './interfaces/IUserRepository';
import { usersService } from '../api/users';

export class UserRepository implements IUserRepository {
    async findAll(): Promise<User[]> {
        return usersService.getUsers();
    }

    async findById(id: number): Promise<User | null> {
        try {
            return await usersService.getUserById(id);
        } catch (error) {
            if (error instanceof ApiError && error.status === 404) {
                return null;
            }
            throw error;
        }
    }

    async create(data: CreateUserData): Promise<User> {
        return usersService.createUser(data);
    }

    async update(id: number, data: Partial<User>): Promise<User> {
        return usersService.updateUser(id, data);
    }

    async delete(id: number): Promise<void> {
        return usersService.deleteUser(id);
    }

    async search(query: string): Promise<User[]> {
        return usersService.searchUsers(query);
    }
}

// Dependency injection
export const userRepository = new UserRepository();
```

**Usage in business logic:**
```typescript
// services/useCases/CreateUserUseCase.ts
import { userRepository } from '../repositories/UserRepository';
import { CreateUserData, User } from '../types';

export class CreateUserUseCase {
    async execute(userData: CreateUserData): Promise<User> {
        // Business logic validation
        if (!userData.email.includes('@')) {
            throw new Error('Invalid email format');
        }

        if (userData.password.length < 8) {
            throw new Error('Password must be at least 8 characters');
        }

        // Check if user already exists
        const existingUsers = await userRepository.search(userData.email);
        if (existingUsers.length > 0) {
            throw new Error('User with this email already exists');
        }

        // Create user
        const user = await userRepository.create(userData);

        // Send welcome email (could be another service)
        await this.sendWelcomeEmail(user.email);

        return user;
    }

    private async sendWelcomeEmail(email: string): Promise<void> {
        // Email service integration
        console.log(`Sending welcome email to ${email}`);
    }
}

export const createUserUseCase = new CreateUserUseCase();
```

### 3. **Service Layer with Caching**

**Cached service implementation:**
```typescript
// services/api/CachedHttpClient.ts
import { httpClient } from './httpClient';

interface CacheEntry<T> {
    data: T;
    timestamp: number;
    ttl: number;
}

export class CachedHttpClient {
    private cache = new Map<string, CacheEntry<any>>();

    private getCacheKey(url: string, params?: any): string {
        return `${url}${params ? JSON.stringify(params) : ''}`;
    }

    private isExpired(entry: CacheEntry<any>): boolean {
        return Date.now() - entry.timestamp > entry.ttl;
    }

    private get<T>(key: string): T | null {
        const entry = this.cache.get(key);
        if (!entry || this.isExpired(entry)) {
            this.cache.delete(key);
            return null;
        }
        return entry.data;
    }

    private set<T>(key: string, data: T, ttl: number = 5 * 60 * 1000): void {
        this.cache.set(key, {
            data,
            timestamp: Date.now(),
            ttl,
        });
    }

    async getCached<T>(
        url: string,
        params?: any,
        ttl: number = 5 * 60 * 1000
    ): Promise<T> {
        const key = this.getCacheKey(url, params);

        // Check cache first
        const cached = this.get<T>(key);
        if (cached) {
            return cached;
        }

        // Fetch from API
        const data = await httpClient.get<T>(url, { params });

        // Cache the result
        this.set(key, data, ttl);

        return data;
    }

    // Invalidate cache
    invalidate(pattern: string): void {
        for (const key of this.cache.keys()) {
            if (key.includes(pattern)) {
                this.cache.delete(key);
            }
        }
    }

    // Clear all cache
    clear(): void {
        this.cache.clear();
    }
}

export const cachedHttpClient = new CachedHttpClient();
```

**Service with caching:**
```typescript
// services/api/products.ts
import { cachedHttpClient } from './CachedHttpClient';

export class ProductsService {
    async getProducts(category?: string): Promise<Product[]> {
        const params = category ? { category } : undefined;
        return cachedHttpClient.getCached('/products', params, 10 * 60 * 1000); // 10 min cache
    }

    async getProductById(id: number): Promise<Product> {
        return cachedHttpClient.getCached(`/products/${id}`, undefined, 5 * 60 * 1000);
    }

    async createProduct(productData: CreateProductData): Promise<Product> {
        const product = await httpClient.post('/products', productData);

        // Invalidate products cache
        cachedHttpClient.invalidate('/products');

        return product;
    }

    async updateProduct(id: number, updates: Partial<Product>): Promise<Product> {
        const product = await httpClient.patch(`/products/${id}`, updates);

        // Invalidate specific product and list cache
        cachedHttpClient.invalidate(`/products/${id}`);
        cachedHttpClient.invalidate('/products');

        return product;
    }
}

export const productsService = new ProductsService();
```

## Error Handling and Resilience

### 1. **Global Error Handler**

```typescript
// services/utils/errorHandler.ts
import { ApiError } from '../api/base';

export interface ErrorContext {
    operation: string;
    userId?: string;
    metadata?: Record<string, any>;
}

export class ErrorHandler {
    async handle(error: unknown, context: ErrorContext): Promise<void> {
        // Log error
        console.error(`Error in ${context.operation}:`, error);

        // Send to error reporting service
        if (process.env.NODE_ENV === 'production') {
            await this.reportError(error, context);
        }

        // Handle specific error types
        if (error instanceof ApiError) {
            await this.handleApiError(error, context);
        }
    }

    private async handleApiError(error: ApiError, context: ErrorContext): Promise<void> {
        switch (error.status) {
            case 401:
                // Handle authentication errors
                this.handleAuthError();
                break;
            case 403:
                // Handle permission errors
                this.handlePermissionError();
                break;
            case 429:
                // Handle rate limiting
                await this.handleRateLimit(error);
                break;
            case 500:
            case 502:
            case 503:
            case 504:
                // Handle server errors
                await this.handleServerError(error, context);
                break;
        }
    }

    private handleAuthError(): void {
        localStorage.removeItem('authToken');
        localStorage.removeItem('refreshToken');
        window.location.href = '/login';
    }

    private handlePermissionError(): void {
        // Show permission denied message
        console.warn('Permission denied');
    }

    private async handleRateLimit(error: ApiError): Promise<void> {
        const retryAfter = error.data?.retryAfter || 60;
        console.warn(`Rate limited. Retry after ${retryAfter} seconds`);

        // Could implement automatic retry with backoff
    }

    private async handleServerError(error: ApiError, context: ErrorContext): Promise<void> {
        // Log server errors for monitoring
        await this.logServerError(error, context);
    }

    private async reportError(error: unknown, context: ErrorContext): Promise<void> {
        // Send to error reporting service (e.g., Sentry, LogRocket)
        try {
            await fetch('/api/errors', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    error: error instanceof Error ? error.message : String(error),
                    stack: error instanceof Error ? error.stack : undefined,
                    context,
                    timestamp: new Date().toISOString(),
                    userAgent: navigator.userAgent,
                }),
            });
        } catch (reportingError) {
            console.error('Failed to report error:', reportingError);
        }
    }

    private async logServerError(error: ApiError, context: ErrorContext): Promise<void> {
        // Additional server error logging
        console.error('Server error:', {
            status: error.status,
            message: error.message,
            operation: context.operation,
            userId: context.userId,
        });
    }
}

export const errorHandler = new ErrorHandler();
```

### 2. **Resilient Service with Retry Logic**

```typescript
// services/api/ResilientService.ts
import { httpClient } from './httpClient';
import { errorHandler } from '../utils/errorHandler';

export interface RetryConfig {
    maxRetries: number;
    baseDelay: number;
    maxDelay: number;
    backoffFactor: number;
}

export class ResilientService {
    private defaultRetryConfig: RetryConfig = {
        maxRetries: 3,
        baseDelay: 1000,
        maxDelay: 30000,
        backoffFactor: 2,
    };

    async executeWithRetry<T>(
        operation: () => Promise<T>,
        context: { operation: string; userId?: string },
        retryConfig: Partial<RetryConfig> = {}
    ): Promise<T> {
        const config = { ...this.defaultRetryConfig, ...retryConfig };
        let lastError: unknown;

        for (let attempt = 0; attempt <= config.maxRetries; attempt++) {
            try {
                return await operation();
            } catch (error) {
                lastError = error;

                // Don't retry certain errors
                if (error instanceof ApiError) {
                    if (error.status === 400 || error.status === 401 || error.status === 403) {
                        throw error; // Don't retry client errors
                    }
                }

                if (attempt === config.maxRetries) {
                    break; // Max retries reached
                }

                // Calculate delay with exponential backoff
                const delay = Math.min(
                    config.baseDelay * Math.pow(config.backoffFactor, attempt),
                    config.maxDelay
                );

                // Add jitter to prevent thundering herd
                const jitter = Math.random() * 0.1 * delay;
                const finalDelay = delay + jitter;

                console.warn(`Attempt ${attempt + 1} failed, retrying in ${finalDelay}ms`);
                await new Promise(resolve => setTimeout(resolve, finalDelay));
            }
        }

        // Handle final error
        await errorHandler.handle(lastError, context);
        throw lastError;
    }

    async get<T>(url: string, context: { operation: string; userId?: string }): Promise<T> {
        return this.executeWithRetry(
            () => httpClient.get<T>(url),
            context
        );
    }

    async post<T>(url: string, data: any, context: { operation: string; userId?: string }): Promise<T> {
        return this.executeWithRetry(
            () => httpClient.post<T>(url, data),
            context
        );
    }

    // Similar methods for put, patch, delete...
}

export const resilientService = new ResilientService();
```

## Testing Service Layers

### 1. **Unit Testing Services**

```typescript
// services/api/users.test.ts
import { usersService } from './users';
import { httpClient } from './httpClient';

// Mock the HTTP client
jest.mock('./httpClient');
const mockHttpClient = httpClient as jest.Mocked<typeof httpClient>;

describe('UsersService', () => {
    beforeEach(() => {
        jest.clearAllMocks();
    });

    describe('getUsers', () => {
        it('should fetch users successfully', async () => {
            const mockUsers = [
                { id: 1, name: 'John Doe', email: 'john@example.com' },
                { id: 2, name: 'Jane Doe', email: 'jane@example.com' },
            ];

            mockHttpClient.get.mockResolvedValue(mockUsers);

            const result = await usersService.getUsers();

            expect(mockHttpClient.get).toHaveBeenCalledWith('/users');
            expect(result).toEqual(mockUsers);
        });

        it('should handle API errors', async () => {
            const errorMessage = 'Network error';
            mockHttpClient.get.mockRejectedValue(new Error(errorMessage));

            await expect(usersService.getUsers()).rejects.toThrow(errorMessage);
        });
    });

    describe('createUser', () => {
        it('should create user with valid data', async () => {
            const userData = {
                name: 'New User',
                email: 'new@example.com',
                password: 'password123',
            };
            const createdUser = { id: 3, ...userData };

            mockHttpClient.post.mockResolvedValue(createdUser);

            const result = await usersService.createUser(userData);

            expect(mockHttpClient.post).toHaveBeenCalledWith('/users', userData);
            expect(result).toEqual(createdUser);
        });

        it('should validate input data', async () => {
            const invalidData = { name: '', email: 'invalid-email' };

            // If validation is in service layer
            await expect(usersService.createUser(invalidData)).rejects.toThrow();
        });
    });
});
```

### 2. **Integration Testing**

```typescript
// services/api/users.integration.test.ts
import { usersService } from './users';
import { setupTestServer, teardownTestServer } from '../../test/server';

describe('UsersService Integration', () => {
    beforeAll(async () => {
        await setupTestServer();
    });

    afterAll(async () => {
        await teardownTestServer();
    });

    it('should create and retrieve user', async () => {
        const userData = {
            name: 'Integration Test User',
            email: 'integration@example.com',
            password: 'testpass123',
        };

        // Create user
        const createdUser = await usersService.createUser(userData);
        expect(createdUser).toHaveProperty('id');
        expect(createdUser.name).toBe(userData.name);

        // Retrieve user
        const retrievedUser = await usersService.getUserById(createdUser.id);
        expect(retrievedUser).toEqual(createdUser);

        // Clean up
        await usersService.deleteUser(createdUser.id);
    });
});
```

## Best Practices

### 1. **Service Layer Guidelines**

```typescript
// ✅ Good: Clear separation of concerns
services/
├── api/              # HTTP communication
├── repositories/     # Data access abstraction
├── useCases/         # Business logic
├── utils/            # Shared utilities
└── types/            # Type definitions

// ✅ Good: Consistent error handling
export class ApiService {
    async request<T>(config: RequestConfig): Promise<T> {
        try {
            // Implementation
        } catch (error) {
            await errorHandler.handle(error, {
                operation: config.operation,
                userId: config.userId,
            });
            throw error;
        }
    }
}

// ❌ Avoid: Business logic in service methods
export class UsersService {
    // Bad: Business logic mixed with data access
    async createUserWithValidation(userData: CreateUserData): Promise<User> {
        // Validation logic here...
        // Email sending here...
        // Data transformation here...
    }
}
```

### 2. **Type Safety**

```typescript
// ✅ Good: Strongly typed services
export interface UserFilters {
    role?: string;
    status?: 'active' | 'inactive';
    search?: string;
}

export interface PaginatedResponse<T> {
    data: T[];
    total: number;
    page: number;
    limit: number;
}

export class UsersService {
    async getUsers(
        filters?: UserFilters,
        pagination?: { page: number; limit: number }
    ): Promise<PaginatedResponse<User>> {
        // Implementation
    }
}
```

### 3. **Environment Configuration**

```typescript
// ✅ Good: Environment-specific configuration
export const apiConfig = {
    development: {
        baseURL: 'http://localhost:3001/api',
        timeout: 30000,
    },
    staging: {
        baseURL: 'https://api-staging.example.com',
        timeout: 15000,
    },
    production: {
        baseURL: 'https://api.example.com',
        timeout: 10000,
    },
};

const currentEnv = process.env.NODE_ENV || 'development';
export const currentConfig = apiConfig[currentEnv];
```

### 4. **Monitoring and Observability**

```typescript
// ✅ Good: Service metrics and monitoring
export class MonitoredHttpClient {
    async get<T>(url: string): Promise<T> {
        const startTime = Date.now();

        try {
            const result = await httpClient.get<T>(url);

            // Track successful requests
            this.trackMetric('api_request_success', {
                url,
                duration: Date.now() - startTime,
                method: 'GET',
            });

            return result;
        } catch (error) {
            // Track failed requests
            this.trackMetric('api_request_error', {
                url,
                duration: Date.now() - startTime,
                method: 'GET',
                error: error.message,
            });

            throw error;
        }
    }

    private trackMetric(name: string, data: any): void {
        // Send to monitoring service (e.g., DataDog, New Relic)
        console.log(`Metric: ${name}`, data);
    }
}
```

## Common Interview Questions

### Q: Why should you use a service layer instead of direct API calls in components?

**A:** Service layers provide separation of concerns, making components focus on UI logic while services handle data fetching, error handling, and API communication. This improves testability, reusability, and maintainability, and provides a single place to handle cross-cutting concerns like authentication and caching.

### Q: How do you handle authentication in service layers?

**A:** I use HTTP interceptors or request headers to automatically include authentication tokens. For token refresh, I implement response interceptors that catch 401 errors, attempt to refresh the token, and retry the original request. Failed refresh attempts redirect to login.

### Q: What's the difference between a service layer and a repository pattern?

**A:** Service layers handle HTTP communication and external API interactions, while repositories provide an abstraction over data access, hiding whether data comes from an API, database, or cache. Services are about external communication, repositories are about data access patterns.

### Q: How do you handle errors in service layers?

**A:** I use custom error classes for different error types, implement global error handlers for logging and user feedback, and use retry logic with exponential backoff for transient failures. Critical errors trigger user notifications, while non-critical ones are logged for monitoring.

## Summary

**Service Layer Benefits:**
- **Separation of concerns**: UI logic separate from data logic
- **Reusability**: Same services used across multiple components
- **Testability**: Easy to mock HTTP calls and test business logic
- **Maintainability**: API changes isolated to service layer
- **Consistency**: Standardized error handling and response processing

**Key Patterns:**
1. **HTTP Client Wrapper**: Axios/fetch wrapper with interceptors
2. **Feature Services**: Domain-specific service classes
3. **Repository Pattern**: Data access abstraction
4. **Error Handling**: Global error handlers with retry logic
5. **Caching Layer**: Intelligent caching with invalidation

**Best Practices:**
- Strong TypeScript typing
- Comprehensive error handling
- Authentication interceptors
- Retry logic with backoff
- Proper testing (unit + integration)
- Environment-specific configuration
- Monitoring and observability

**Interview Tip:** "Service layers create a clean separation between UI components and API communication. I structure them with HTTP clients for low-level requests, feature-specific services for domain logic, and repositories for data access patterns. This approach improves testability, maintainability, and provides consistent error handling across the application."