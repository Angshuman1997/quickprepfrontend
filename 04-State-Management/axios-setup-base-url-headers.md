# How to set up Axios with base URL and default headers?

## Question
How to set up Axios with base URL and default headers?

## Answer

Setting up Axios with a base URL and default headers is essential for maintaining clean, consistent API calls. Axios allows you to create configured instances that can be reused across your application, eliminating code duplication and making configuration management easier.

## Basic Setup

### 1. **Creating an Axios Instance**

```typescript
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';

// Create a configured Axios instance
const apiClient: AxiosInstance = axios.create({
    baseURL: process.env.REACT_APP_API_BASE_URL || 'http://localhost:3001/api',
    timeout: 10000, // 10 seconds
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
    },
});

export default apiClient;
```

### 2. **Environment-Based Configuration**

```typescript
// config/api.ts
interface ApiConfig {
    baseURL: string;
    timeout: number;
    headers: Record<string, string>;
}

const getApiConfig = (): ApiConfig => {
    const isProduction = process.env.NODE_ENV === 'production';
    const isStaging = process.env.REACT_APP_ENV === 'staging';

    return {
        baseURL: isProduction
            ? process.env.REACT_APP_PROD_API_URL || 'https://api.myapp.com'
            : isStaging
                ? process.env.REACT_APP_STAGING_API_URL || 'https://staging-api.myapp.com'
                : process.env.REACT_APP_DEV_API_URL || 'http://localhost:3001/api',
        timeout: isProduction ? 15000 : 10000,
        headers: {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
            'X-App-Version': process.env.REACT_APP_VERSION || '1.0.0',
            'X-Environment': isProduction ? 'production' : isStaging ? 'staging' : 'development',
        },
    };
};

// Create configured instance
const apiClient = axios.create(getApiConfig());
```

### 3. **Using the Configured Client**

```typescript
// Instead of:
const response = await axios.get('http://localhost:3001/api/users', {
    headers: { 'Authorization': 'Bearer token' }
});

// Use:
const response = await apiClient.get('/users');
// The full URL becomes: http://localhost:3001/api/users
// Default headers are automatically included
```

## Advanced Header Configuration

### 1. **Dynamic Headers with Interceptors**

```typescript
// Request interceptor for dynamic headers
apiClient.interceptors.request.use(
    (config: AxiosRequestConfig): AxiosRequestConfig => {
        // Add authorization token
        const token = localStorage.getItem('authToken');
        if (token) {
            config.headers = {
                ...config.headers,
                Authorization: `Bearer ${token}`,
            };
        }

        // Add request ID for tracking
        config.headers = {
            ...config.headers,
            'X-Request-ID': generateRequestId(),
        };

        // Add timestamp
        config.headers = {
            ...config.headers,
            'X-Request-Time': new Date().toISOString(),
        };

        // Add user context if available
        const userId = getCurrentUserId();
        if (userId) {
            config.headers = {
                ...config.headers,
                'X-User-ID': userId,
            };
        }

        return config;
    },
    (error) => Promise.reject(error)
);

// Utility functions
function generateRequestId(): string {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}

function getCurrentUserId(): string | null {
    // Get from your auth state management
    return localStorage.getItem('userId');
}
```

### 2. **Content-Type Headers for Different Request Types**

```typescript
// Different clients for different content types
const jsonClient = axios.create({
    baseURL: process.env.REACT_APP_API_BASE_URL,
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
    },
});

const formDataClient = axios.create({
    baseURL: process.env.REACT_APP_API_BASE_URL,
    headers: {
        'Content-Type': 'multipart/form-data',
        'Accept': 'application/json',
    },
});

const textClient = axios.create({
    baseURL: process.env.REACT_APP_API_BASE_URL,
    headers: {
        'Content-Type': 'text/plain',
        'Accept': 'text/plain',
    },
});

// Usage
const uploadFile = async (file: File) => {
    const formData = new FormData();
    formData.append('file', file);

    return formDataClient.post('/upload', formData);
};

const sendJson = async (data: object) => {
    return jsonClient.post('/data', data);
};
```

### 3. **API Versioning Headers**

```typescript
// Versioned API clients
const createVersionedClient = (version: string) => {
    return axios.create({
        baseURL: process.env.REACT_APP_API_BASE_URL,
        headers: {
            'Content-Type': 'application/json',
            'Accept': `application/vnd.myapp.v${version}+json`,
            'X-API-Version': version,
        },
    });
};

const v1Client = createVersionedClient('1');
const v2Client = createVersionedClient('2');

// Usage
const getUsersV1 = () => v1Client.get('/users');
const getUsersV2 = () => v2Client.get('/users');
```

## Configuration Patterns

### 1. **Multiple API Endpoints**

```typescript
// Different clients for different services
const authClient = axios.create({
    baseURL: process.env.REACT_APP_AUTH_API_URL,
    headers: {
        'Content-Type': 'application/json',
    },
});

const dataClient = axios.create({
    baseURL: process.env.REACT_APP_DATA_API_URL,
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
    },
});

const fileClient = axios.create({
    baseURL: process.env.REACT_APP_FILE_API_URL,
    headers: {
        'Content-Type': 'multipart/form-data',
    },
});

// API service class
class ApiService {
    auth = {
        login: (credentials: any) => authClient.post('/login', credentials),
        logout: () => authClient.post('/logout'),
        refresh: () => authClient.post('/refresh'),
    };

    users = {
        getAll: (params?: any) => dataClient.get('/users', { params }),
        getById: (id: number) => dataClient.get(`/users/${id}`),
        create: (user: any) => dataClient.post('/users', user),
        update: (id: number, user: any) => dataClient.put(`/users/${id}`, user),
        delete: (id: number) => dataClient.delete(`/users/${id}`),
    };

    files = {
        upload: (file: File) => {
            const formData = new FormData();
            formData.append('file', file);
            return fileClient.post('/upload', formData);
        },
        download: (id: string) => fileClient.get(`/download/${id}`, {
            responseType: 'blob',
        }),
    };
}

export const apiService = new ApiService();
```

### 2. **Environment-Specific Headers**

```typescript
// Development headers
const devHeaders = {
    'X-Debug': 'true',
    'X-Environment': 'development',
    'Cache-Control': 'no-cache',
};

// Production headers
const prodHeaders = {
    'X-Environment': 'production',
    'Cache-Control': 'public, max-age=300',
};

// Create client based on environment
const createApiClient = () => {
    const isDev = process.env.NODE_ENV === 'development';

    return axios.create({
        baseURL: process.env.REACT_APP_API_BASE_URL,
        timeout: isDev ? 30000 : 10000, // Longer timeout in dev
        headers: {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
            ...getEnvironmentHeaders(),
        },
    });
};

function getEnvironmentHeaders() {
    const isDev = process.env.NODE_ENV === 'development';
    const isTest = process.env.NODE_ENV === 'test';

    if (isTest) {
        return {
            'X-Environment': 'test',
            'Authorization': 'Bearer test-token',
        };
    }

    return isDev ? devHeaders : prodHeaders;
}
```

### 3. **Custom Instance Methods**

```typescript
// Extend Axios instance with custom methods
class CustomApiClient {
    private client: AxiosInstance;

    constructor(config: AxiosRequestConfig) {
        this.client = axios.create(config);
        this.setupInterceptors();
    }

    private setupInterceptors() {
        // Add your interceptors here
        this.client.interceptors.request.use(
            (config) => {
                // Add auth token, etc.
                return config;
            }
        );
    }

    // Custom methods
    async getWithCache<T = any>(url: string, config?: AxiosRequestConfig): Promise<T> {
        const cacheKey = `cache_${url}`;
        const cached = sessionStorage.getItem(cacheKey);

        if (cached) {
            return JSON.parse(cached);
        }

        const response = await this.client.get(url, config);
        sessionStorage.setItem(cacheKey, JSON.stringify(response.data));

        return response.data;
    }

    async postWithRetry<T = any>(
        url: string,
        data?: any,
        retries: number = 3
    ): Promise<T> {
        let lastError: Error;

        for (let i = 0; i < retries; i++) {
            try {
                const response = await this.client.post(url, data);
                return response.data;
            } catch (error) {
                lastError = error as Error;

                if (i < retries - 1) {
                    // Wait before retry (exponential backoff)
                    await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, i)));
                }
            }
        }

        throw lastError!;
    }

    // Expose standard methods
    get = (url: string, config?: AxiosRequestConfig) => this.client.get(url, config);
    post = (url: string, data?: any, config?: AxiosRequestConfig) => this.client.post(url, data, config);
    put = (url: string, data?: any, config?: AxiosRequestConfig) => this.client.put(url, data, config);
    delete = (url: string, config?: AxiosRequestConfig) => this.client.delete(url, config);
}

// Usage
const apiClient = new CustomApiClient({
    baseURL: process.env.REACT_APP_API_BASE_URL,
    headers: {
        'Content-Type': 'application/json',
    },
});

// Use custom methods
const users = await apiClient.getWithCache('/users');
const newUser = await apiClient.postWithRetry('/users', userData);
```

## Best Practices

### 1. **Environment Variables**

```typescript
// .env.development
REACT_APP_API_BASE_URL=http://localhost:3001/api
REACT_APP_AUTH_API_URL=http://localhost:3001/auth

// .env.production
REACT_APP_API_BASE_URL=https://api.myapp.com
REACT_APP_AUTH_API_URL=https://auth.myapp.com

// .env.test
REACT_APP_API_BASE_URL=http://localhost:3001/api
REACT_APP_AUTH_API_URL=http://localhost:3001/auth
```

### 2. **Type Safety**

```typescript
// Define API response types
interface ApiResponse<T = any> {
    data: T;
    message?: string;
    success: boolean;
}

interface User {
    id: number;
    name: string;
    email: string;
}

// Typed API functions
const getUsers = (): Promise<ApiResponse<User[]>> => {
    return apiClient.get('/users');
};

const createUser = (user: Omit<User, 'id'>): Promise<ApiResponse<User>> => {
    return apiClient.post('/users', user);
};
```

### 3. **Error Handling**

```typescript
// Centralized error handling
apiClient.interceptors.response.use(
    (response) => response,
    (error) => {
        if (error.response?.status === 401) {
            // Handle unauthorized
            localStorage.removeItem('authToken');
            window.location.href = '/login';
        } else if (error.response?.status >= 500) {
            // Handle server errors
            console.error('Server error:', error.response.data);
        } else if (error.code === 'ECONNABORTED') {
            // Handle timeout
            console.error('Request timeout');
        }

        return Promise.reject(error);
    }
);
```

### 4. **Testing Configuration**

```typescript
// jest.setup.js or test setup file
import axios from 'axios';

// Mock Axios for all tests
jest.mock('axios');
axios.create = jest.fn(() => ({
    get: jest.fn(),
    post: jest.fn(),
    put: jest.fn(),
    delete: jest.fn(),
    interceptors: {
        request: { use: jest.fn() },
        response: { use: jest.fn() },
    },
}));

// Test configuration
describe('API Client', () => {
    it('creates client with correct base URL', () => {
        const client = axios.create({
            baseURL: 'http://test.com/api',
        });

        expect(client.defaults.baseURL).toBe('http://test.com/api');
    });
});
```

## Common Interview Questions

### Q: Why use axios.create() instead of the default axios instance?

**A:** `axios.create()` allows you to create isolated instances with their own configuration, interceptors, and base URLs. This prevents configuration conflicts and makes testing easier.

### Q: How do you handle different environments (dev, staging, prod)?

**A:** Use environment variables and conditional logic to set different base URLs and headers based on `NODE_ENV` or custom environment variables.

### Q: What's the difference between default headers and interceptor headers?

**A:** Default headers are set during instance creation and apply to all requests. Interceptor headers are added dynamically for each request, allowing conditional logic.

### Q: How do you handle API versioning with Axios?

**A:** You can add version headers, use different base URLs for different versions, or include version in the URL path. Accept headers with vendor media types are commonly used.

## Summary

**Key Benefits:**
- Centralized configuration management
- Environment-specific settings
- Consistent headers across requests
- Type safety with TypeScript
- Easy testing and mocking

**Configuration Options:**
- `baseURL`: Base URL for all requests
- `timeout`: Request timeout in milliseconds
- `headers`: Default headers for all requests
- `params`: Default query parameters

**Best Practices:**
- Use environment variables for URLs
- Create separate instances for different APIs
- Use interceptors for dynamic headers
- Implement proper error handling
- Add TypeScript types for responses

**Interview Tip:** "I use `axios.create()` to set up configured instances with base URLs and default headers. This ensures all API calls use consistent configuration and makes it easy to manage different environments and API versions."