# What are Axios interceptors?

## Question
What are Axios interceptors?

## Answer
Functions that run before requests or after responses to modify them globally.

## Basic Example
```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: '/api'
});

// Request interceptor - add auth token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor - handle errors
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Redirect to login
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Usage
```javascript
// All requests now automatically include auth token
const getData = () => api.get('/data');
const postData = (data) => api.post('/data', data);
```

## Interview Q&A

**Q: What are Axios interceptors?**

A: Functions that automatically modify requests before sending or responses after receiving.

**Q: When do you use request interceptors?**

A: To add authentication headers, logging, or modify request data globally.

**Q: When do you use response interceptors?**

A: To handle errors, transform response data, or refresh tokens on 401 errors.

### 3. **Response Interceptor**

```typescript
// Response interceptor
apiClient.interceptors.response.use(
    (response: AxiosResponse) => {
        // Log successful responses in development
        if (process.env.NODE_ENV === 'development') {
            console.log('✅ Response:', {
                status: response.status,
                url: response.config.url,
                data: response.data,
            });
        }

        // Transform response data if needed
        if (response.data && response.data.success) {
            return response.data.data; // Return only the data part
        }

        return response;
    },
    (error) => {
        // Handle response errors
        if (error.response) {
            // Server responded with error status
            const { status, data } = error.response;

            switch (status) {
                case 401:
                    // Unauthorized - redirect to login
                    localStorage.removeItem('authToken');
                    window.location.href = '/login';
                    break;
                case 403:
                    // Forbidden
                    console.error('Access denied');
                    break;
                case 404:
                    // Not found
                    console.error('Resource not found');
                    break;
                case 500:
                    // Server error
                    console.error('Server error');
                    break;
                default:
                    console.error(`HTTP ${status} Error:`, data);
            }
        } else if (error.request) {
            // Network error
            console.error('Network Error:', error.message);
        } else {
            // Other error
            console.error('Error:', error.message);
        }

        return Promise.reject(error);
    }
);
```

## Advanced Patterns

### 1. **Authentication with Token Refresh**

```typescript
// Store for managing refresh token state
let isRefreshing = false;
let failedQueue: Array<{
    resolve: (token: string) => void;
    reject: (error: any) => void;
}> = [];

const processQueue = (error: any, token: string | null = null) => {
    failedQueue.forEach(({ resolve, reject }) => {
        if (error) {
            reject(error);
        } else {
            resolve(token!);
        }
    });

    failedQueue = [];
};

// Request interceptor with token refresh logic
apiClient.interceptors.request.use(
    async (config) => {
        const token = localStorage.getItem('authToken');

        if (token) {
            // Check if token is expired
            const tokenData = JSON.parse(atob(token.split('.')[1]));
            const isExpired = Date.now() >= tokenData.exp * 1000;

            if (isExpired) {
                if (!isRefreshing) {
                    isRefreshing = true;

                    try {
                        const refreshToken = localStorage.getItem('refreshToken');
                        const response = await axios.post('/api/auth/refresh', {
                            refreshToken,
                        });

                        const { accessToken } = response.data;
                        localStorage.setItem('authToken', accessToken);

                        processQueue(null, accessToken);
                        isRefreshing = false;
                    } catch (error) {
                        processQueue(error, null);
                        isRefreshing = false;

                        // Redirect to login on refresh failure
                        localStorage.removeItem('authToken');
                        localStorage.removeItem('refreshToken');
                        window.location.href = '/login';
                        return Promise.reject(error);
                    }
                }

                // Wait for token refresh
                return new Promise((resolve, reject) => {
                    failedQueue.push({
                        resolve: (newToken) => {
                            config.headers = {
                                ...config.headers,
                                Authorization: `Bearer ${newToken}`,
                            };
                            resolve(config);
                        },
                        reject: (error) => {
                            reject(error);
                        },
                    });
                });
            }

            config.headers = {
                ...config.headers,
                Authorization: `Bearer ${token}`,
            };
        }

        return config;
    },
    (error) => Promise.reject(error)
);
```

### 2. **Request/Response Logging Interceptor**

```typescript
// Create a logging interceptor
const createLoggingInterceptor = (name: string) => ({
    request: apiClient.interceptors.request.use(
        (config) => {
            const startTime = Date.now();
            config.metadata = { startTime };

            console.group(`🚀 ${name} - ${config.method?.toUpperCase()} ${config.url}`);
            console.log('Request Config:', {
                url: config.url,
                method: config.method,
                headers: config.headers,
                data: config.data,
            });
            console.groupEnd();

            return config;
        },
        (error) => {
            console.error(`❌ ${name} - Request Error:`, error);
            return Promise.reject(error);
        }
    ),

    response: apiClient.interceptors.response.use(
        (response) => {
            const duration = Date.now() - response.config.metadata?.startTime;

            console.group(`✅ ${name} - ${response.status} ${response.config.url}`);
            console.log('Response:', {
                status: response.status,
                statusText: response.statusText,
                data: response.data,
                duration: `${duration}ms`,
            });
            console.groupEnd();

            return response;
        },
        (error) => {
            const duration = error.config?.metadata?.startTime
                ? Date.now() - error.config.metadata.startTime
                : 'unknown';

            console.group(`❌ ${name} - Response Error`);
            console.error('Error Details:', {
                message: error.message,
                status: error.response?.status,
                statusText: error.response?.statusText,
                data: error.response?.data,
                duration: `${duration}ms`,
            });
            console.groupEnd();

            return Promise.reject(error);
        }
    ),
});

// Apply logging to specific client
const loggingInterceptor = createLoggingInterceptor('API Client');
```

### 3. **Retry Interceptor**

```typescript
// Retry configuration
interface RetryConfig {
    retries: number;
    retryDelay: number;
    retryCondition?: (error: any) => boolean;
}

const defaultRetryConfig: RetryConfig = {
    retries: 3,
    retryDelay: 1000,
    retryCondition: (error) => {
        // Retry on network errors or 5xx status codes
        return !error.response || error.response.status >= 500;
    },
};

// Retry interceptor
apiClient.interceptors.response.use(
    (response) => response,
    async (error) => {
        const config = error.config;

        if (!config || !defaultRetryConfig.retryCondition!(error)) {
            return Promise.reject(error);
        }

        // Initialize retry count
        config.retryCount = config.retryCount || 0;

        if (config.retryCount >= defaultRetryConfig.retries) {
            return Promise.reject(error);
        }

        config.retryCount += 1;

        // Exponential backoff
        const delay = defaultRetryConfig.retryDelay * Math.pow(2, config.retryCount - 1);

        console.log(`Retrying request (${config.retryCount}/${defaultRetryConfig.retries}) in ${delay}ms`);

        return new Promise((resolve) => {
            setTimeout(() => resolve(apiClient(config)), delay);
        });
    }
);
```

### 4. **Data Transformation Interceptor**

```typescript
// Request transformation interceptor
apiClient.interceptors.request.use(
    (config) => {
        // Transform request data
        if (config.data && typeof config.data === 'object') {
            // Convert camelCase to snake_case
            config.data = transformKeys(config.data, camelToSnake);

            // Add common fields
            config.data = {
                ...config.data,
                timestamp: new Date().toISOString(),
                clientVersion: process.env.REACT_APP_VERSION || '1.0.0',
            };
        }

        // Transform query parameters
        if (config.params) {
            config.params = transformKeys(config.params, camelToSnake);
        }

        return config;
    }
);

// Response transformation interceptor
apiClient.interceptors.response.use(
    (response) => {
        // Transform response data
        if (response.data && typeof response.data === 'object') {
            // Convert snake_case to camelCase
            response.data = transformKeys(response.data, snakeToCamel);
        }

        return response;
    }
);

// Utility functions for key transformation
const camelToSnake = (str: string) =>
    str.replace(/[A-Z]/g, (letter) => `_${letter.toLowerCase()}`);

const snakeToCamel = (str: string) =>
    str.replace(/_([a-z])/g, (_, letter) => letter.toUpperCase());

const transformKeys = (obj: any, transform: (key: string) => string): any => {
    if (Array.isArray(obj)) {
        return obj.map(item => transformKeys(item, transform));
    }

    if (obj !== null && typeof obj === 'object') {
        return Object.keys(obj).reduce((result, key) => {
            const transformedKey = transform(key);
            result[transformedKey] = transformKeys(obj[key], transform);
            return result;
        }, {} as any);
    }

    return obj;
};
```

## Real-World Examples

### 1. **API Service with Interceptors**

```typescript
// API service class
class ApiService {
    private client: AxiosInstance;

    constructor() {
        this.client = axios.create({
            baseURL: '/api',
            timeout: 15000,
        });

        this.setupInterceptors();
    }

    private setupInterceptors() {
        // Request interceptor
        this.client.interceptors.request.use(
            (config) => {
                const token = this.getAuthToken();
                if (token) {
                    config.headers.Authorization = `Bearer ${token}`;
                }
                return config;
            },
            (error) => Promise.reject(error)
        );

        // Response interceptor
        this.client.interceptors.response.use(
            (response) => response,
            (error) => {
                if (error.response?.status === 401) {
                    this.handleUnauthorized();
                }
                return Promise.reject(error);
            }
        );
    }

    private getAuthToken(): string | null {
        return localStorage.getItem('authToken');
    }

    private handleUnauthorized() {
        localStorage.removeItem('authToken');
        window.location.href = '/login';
    }

    // API methods
    async getUsers(params?: any) {
        return this.client.get('/users', { params });
    }

    async createUser(userData: any) {
        return this.client.post('/users', userData);
    }

    async updateUser(id: number, userData: any) {
        return this.client.put(`/users/${id}`, userData);
    }

    async deleteUser(id: number) {
        return this.client.delete(`/users/${id}`);
    }
}

export const apiService = new ApiService();
```

### 2. **Global Loading State Management**

```typescript
// Loading state store (could be Redux, Context, etc.)
let activeRequests = 0;

const updateLoadingState = (loading: boolean) => {
    // Dispatch to your state management solution
    // For example: store.dispatch(setLoading(loading));
    console.log('Loading state:', loading);
};

// Request interceptor for loading
apiClient.interceptors.request.use(
    (config) => {
        if (!activeRequests) {
            updateLoadingState(true);
        }
        activeRequests++;
        return config;
    },
    (error) => {
        activeRequests = Math.max(0, activeRequests - 1);
        if (!activeRequests) {
            updateLoadingState(false);
        }
        return Promise.reject(error);
    }
);

// Response interceptor for loading
apiClient.interceptors.response.use(
    (response) => {
        activeRequests = Math.max(0, activeRequests - 1);
        if (!activeRequests) {
            updateLoadingState(false);
        }
        return response;
    },
    (error) => {
        activeRequests = Math.max(0, activeRequests - 1);
        if (!activeRequests) {
            updateLoadingState(false);
        }
        return Promise.reject(error);
    }
);
```

### 3. **Caching Interceptor**

```typescript
// Simple in-memory cache
const cache = new Map<string, { data: any; timestamp: number }>();
const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

// Cache interceptor
apiClient.interceptors.request.use(
    (config) => {
        // Only cache GET requests
        if (config.method?.toLowerCase() === 'get') {
            const cacheKey = `${config.method}_${config.url}_${JSON.stringify(config.params)}`;
            const cached = cache.get(cacheKey);

            if (cached && Date.now() - cached.timestamp < CACHE_DURATION) {
                // Return cached data
                config.adapter = () => Promise.resolve({
                    data: cached.data,
                    status: 200,
                    statusText: 'OK',
                    headers: {},
                    config,
                    request: {},
                });
            }
        }

        return config;
    }
);

// Cache response data
apiClient.interceptors.response.use(
    (response) => {
        if (response.config.method?.toLowerCase() === 'get') {
            const cacheKey = `${response.config.method}_${response.config.url}_${JSON.stringify(response.config.params)}`;
            cache.set(cacheKey, {
                data: response.data,
                timestamp: Date.now(),
            });
        }

        return response;
    }
);
```

## Best Practices

### 1. **Interceptor Organization**

```typescript
// ✅ Good: Separate interceptor files
// interceptors/auth.ts
export const authRequestInterceptor = (config) => { /* ... */ };
export const authResponseInterceptor = (error) => { /* ... */ };

// interceptors/logging.ts
export const loggingRequestInterceptor = (config) => { /* ... */ };
export const loggingResponseInterceptor = (response, error) => { /* ... */ };

// interceptors/retry.ts
export const retryResponseInterceptor = (error) => { /* ... */ };

// Setup in main file
import * as authInterceptors from './interceptors/auth';
import * as loggingInterceptors from './interceptors/logging';
import * as retryInterceptors from './interceptors/retry';

apiClient.interceptors.request.use(authInterceptors.authRequestInterceptor);
apiClient.interceptors.request.use(loggingInterceptors.loggingRequestInterceptor);
apiClient.interceptors.response.use(null, authInterceptors.authResponseInterceptor);
apiClient.interceptors.response.use(loggingInterceptors.loggingResponseInterceptor);
apiClient.interceptors.response.use(null, retryInterceptors.retryResponseInterceptor);
```

### 2. **Error Handling**

```typescript
// ✅ Good: Centralized error handling
const handleApiError = (error: any) => {
    const errorMessage = error.response?.data?.message || error.message || 'An error occurred';

    // Log error
    console.error('API Error:', errorMessage);

    // Show user-friendly message
    // toast.error(errorMessage);

    // Track error for monitoring
    // analytics.track('api_error', { error: errorMessage });

    throw error;
};

// Use in interceptors
apiClient.interceptors.response.use(
    (response) => response,
    (error) => handleApiError(error)
);
```

### 3. **Testing Interceptors**

```typescript
// Example test for request interceptor
describe('Auth Request Interceptor', () => {
    it('adds authorization header when token exists', () => {
        localStorage.setItem('authToken', 'test-token');

        const config = { headers: {} };
        const result = authRequestInterceptor(config);

        expect(result.headers.Authorization).toBe('Bearer test-token');
    });

    it('does not add authorization header when no token', () => {
        localStorage.removeItem('authToken');

        const config = { headers: {} };
        const result = authRequestInterceptor(config);

        expect(result.headers.Authorization).toBeUndefined();
    });
});
```

## Common Interview Questions

### Q: What's the difference between interceptors and middleware?

**A:** Interceptors are specific to Axios and run for every request/response. Middleware (like in Express.js) runs on the server for incoming requests. Interceptors can modify requests before sending and responses after receiving.

### Q: How do you handle token refresh with multiple concurrent requests?

**A:** Use a flag to prevent multiple refresh attempts and queue failed requests. When refresh completes, retry all queued requests with the new token.

### Q: Can interceptors be removed or modified?

**A:** Yes, `axios.interceptors.request.use()` returns an ID that can be used with `axios.interceptors.request.eject(id)` to remove the interceptor.

### Q: What's the order of interceptor execution?

**A:** Request interceptors run in the order they were added. Response interceptors run in reverse order (last added runs first).

## Summary

**Key Benefits:**
- Centralized cross-cutting concerns
- Automatic request/response processing
- Cleaner API service code
- Consistent error handling
- Easy testing and debugging

**Common Use Cases:**
- Authentication (token management)
- Logging and monitoring
- Error handling and retry logic
- Data transformation
- Loading states
- Caching

**Best Practices:**
- Separate interceptors by concern
- Handle errors appropriately
- Test interceptors thoroughly
- Use TypeScript for type safety
- Keep interceptors simple and focused

**Interview Tip:** "Axios interceptors allow me to handle cross-cutting concerns like authentication, logging, and error handling globally. I use request interceptors to add auth tokens and response interceptors to handle common errors like 401s by refreshing tokens or redirecting to login."