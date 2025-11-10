# How do you define a reusable type for API responses?

## Question
How do you define a reusable type for API responses?

## Answer

Defining reusable types for API responses is crucial for maintaining type safety, consistency, and developer experience in TypeScript applications. A well-designed API response type system should handle success/error states, pagination, metadata, and be flexible enough for different data types.

## Basic API Response Structure

### 1. **Simple Success/Error Response**

```typescript
// Base response interface
interface APIResponse<T = any> {
    success: boolean;
    data?: T;
    error?: string;
    message?: string;
}

// Usage examples
type UserResponse = APIResponse<User>;
type UsersResponse = APIResponse<User[]>;
type EmptyResponse = APIResponse<null>;

// API functions
async function getUser(id: number): Promise<UserResponse> {
    try {
        const response = await fetch(`/api/users/${id}`);
        const data = await response.json();

        if (response.ok) {
            return {
                success: true,
                data,
                message: 'User retrieved successfully'
            };
        } else {
            return {
                success: false,
                error: data.message || 'Failed to fetch user'
            };
        }
    } catch (error) {
        return {
            success: false,
            error: error instanceof Error ? error.message : 'Network error'
        };
    }
}
```

### 2. **Discriminated Union Response**

```typescript
// Discriminated union for better type safety
type APIResponse<T> =
    | { success: true; data: T; message?: string }
    | { success: false; error: string; code?: string };

// Usage - TypeScript knows which fields are available based on success
function handleUserResponse(response: APIResponse<User>) {
    if (response.success) {
        // TypeScript knows response.data exists and is User
        console.log('User:', response.data.name);
        console.log('Message:', response.message);
    } else {
        // TypeScript knows response.error exists
        console.error('Error:', response.error);
        console.log('Error code:', response.code);
    }
}
```

## Advanced Response Patterns

### 1. **Paginated Responses**

```typescript
interface PaginationMeta {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
}

interface Links {
    self: string;
    next?: string;
    prev?: string;
    first: string;
    last: string;
}

interface PaginatedResponse<T> extends APIResponse<T[]> {
    meta?: PaginationMeta;
    links?: Links;
}

// Specific paginated responses
type UsersPaginatedResponse = PaginatedResponse<User>;
type PostsPaginatedResponse = PaginatedResponse<Post>;

// API function with pagination
async function getUsers(page = 1, limit = 10): Promise<UsersPaginatedResponse> {
    try {
        const response = await fetch(`/api/users?page=${page}&limit=${limit}`);
        const data = await response.json();

        if (response.ok) {
            return {
                success: true,
                data: data.users,
                meta: data.meta,
                links: data.links,
                message: 'Users retrieved successfully'
            };
        } else {
            return {
                success: false,
                error: data.message || 'Failed to fetch users'
            };
        }
    } catch (error) {
        return {
            success: false,
            error: error instanceof Error ? error.message : 'Network error'
        };
    }
}
```

### 2. **Response with Metadata**

```typescript
interface ResponseMetadata {
    timestamp: string;
    requestId: string;
    version: string;
    processingTime?: number;
}

interface DetailedResponse<T> extends APIResponse<T> {
    meta: ResponseMetadata;
}

// Usage
async function getUserDetails(id: number): Promise<DetailedResponse<User>> {
    const startTime = Date.now();

    try {
        const response = await fetch(`/api/users/${id}/details`);
        const data = await response.json();
        const processingTime = Date.now() - startTime;

        if (response.ok) {
            return {
                success: true,
                data,
                meta: {
                    timestamp: new Date().toISOString(),
                    requestId: generateRequestId(),
                    version: '1.0.0',
                    processingTime
                }
            };
        } else {
            return {
                success: false,
                error: data.message,
                meta: {
                    timestamp: new Date().toISOString(),
                    requestId: generateRequestId(),
                    version: '1.0.0',
                    processingTime
                }
            };
        }
    } catch (error) {
        return {
            success: false,
            error: error instanceof Error ? error.message : 'Network error',
            meta: {
                timestamp: new Date().toISOString(),
                requestId: generateRequestId(),
                version: '1.0.0'
            }
        };
    }
}

function generateRequestId(): string {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}
```

## Generic API Response System

### 1. **Comprehensive API Response Type**

```typescript
// HTTP status codes
type HTTPStatus = 200 | 201 | 204 | 400 | 401 | 403 | 404 | 422 | 500;

// Error codes
type ErrorCode =
    | 'VALIDATION_ERROR'
    | 'AUTHENTICATION_ERROR'
    | 'AUTHORIZATION_ERROR'
    | 'NOT_FOUND'
    | 'CONFLICT'
    | 'RATE_LIMITED'
    | 'SERVER_ERROR';

// Generic API response
interface APIResponse<T = any, M = any> {
    // Status
    success: boolean;
    status: HTTPStatus;

    // Data (only present on success)
    data?: T;

    // Error details (only present on failure)
    error?: {
        code: ErrorCode;
        message: string;
        details?: Record<string, any>;
        fieldErrors?: Record<string, string[]>;
    };

    // Metadata
    meta?: M & {
        timestamp: string;
        requestId: string;
        version: string;
    };

    // Links (for HATEOAS or pagination)
    links?: {
        self: string;
        related?: Record<string, string>;
    };
}

// Type aliases for common responses
type SuccessResponse<T, M = {}> = APIResponse<T, M> & { success: true };
type ErrorResponse<M = {}> = APIResponse<never, M> & { success: false };
type PaginatedResponse<T> = APIResponse<T[], { pagination: PaginationMeta }>;

// Factory functions for creating responses
function createSuccessResponse<T, M = {}>(
    data: T,
    status: HTTPStatus = 200,
    meta?: M,
    links?: APIResponse['links']
): SuccessResponse<T, M> {
    return {
        success: true,
        status,
        data,
        meta: {
            ...meta,
            timestamp: new Date().toISOString(),
            requestId: generateRequestId(),
            version: '1.0.0'
        } as any,
        links
    };
}

function createErrorResponse<M = {}>(
    error: ErrorResponse['error'],
    status: HTTPStatus = 500,
    meta?: M
): ErrorResponse<M> {
    return {
        success: false,
        status,
        error,
        meta: {
            ...meta,
            timestamp: new Date().toISOString(),
            requestId: generateRequestId(),
            version: '1.0.0'
        } as any
    };
}
```

### 2. **Typed API Client**

```typescript
// API method types
type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';

// API endpoint configuration
interface APIEndpoint<TRequest = any, TResponse = any> {
    path: string;
    method: HTTPMethod;
    requiresAuth?: boolean;
    transformRequest?: (data: TRequest) => any;
    transformResponse?: (data: any) => TResponse;
}

// API client class
class APIClient {
    private baseURL: string;
    private defaultHeaders: Record<string, string>;

    constructor(baseURL: string, defaultHeaders: Record<string, string> = {}) {
        this.baseURL = baseURL;
        this.defaultHeaders = defaultHeaders;
    }

    async request<TRequest, TResponse, M = {}>(
        endpoint: APIEndpoint<TRequest, TResponse>,
        data?: TRequest,
        meta?: M
    ): Promise<APIResponse<TResponse, M>> {
        try {
            const url = `${this.baseURL}${endpoint.path}`;
            const headers = {
                'Content-Type': 'application/json',
                ...this.defaultHeaders
            };

            // Add auth header if required
            if (endpoint.requiresAuth) {
                const token = this.getAuthToken();
                if (token) {
                    headers.Authorization = `Bearer ${token}`;
                }
            }

            // Transform request data
            const requestData = endpoint.transformRequest
                ? endpoint.transformRequest(data!)
                : data;

            const response = await fetch(url, {
                method: endpoint.method,
                headers,
                body: requestData && ['POST', 'PUT', 'PATCH'].includes(endpoint.method)
                    ? JSON.stringify(requestData)
                    : undefined
            });

            const responseData = await response.json();

            if (response.ok) {
                const transformedData = endpoint.transformResponse
                    ? endpoint.transformResponse(responseData)
                    : responseData;

                return createSuccessResponse(transformedData, response.status as HTTPStatus, meta);
            } else {
                return createErrorResponse({
                    code: this.mapStatusToErrorCode(response.status),
                    message: responseData.message || 'Request failed',
                    details: responseData
                }, response.status as HTTPStatus, meta);
            }
        } catch (error) {
            return createErrorResponse({
                code: 'SERVER_ERROR',
                message: error instanceof Error ? error.message : 'Network error'
            }, 500, meta);
        }
    }

    private getAuthToken(): string | null {
        return localStorage.getItem('authToken');
    }

    private mapStatusToErrorCode(status: number): ErrorCode {
        switch (status) {
            case 400: return 'VALIDATION_ERROR';
            case 401: return 'AUTHENTICATION_ERROR';
            case 403: return 'AUTHORIZATION_ERROR';
            case 404: return 'NOT_FOUND';
            case 409: return 'CONFLICT';
            case 429: return 'RATE_LIMITED';
            default: return 'SERVER_ERROR';
        }
    }
}
```

## React Integration

### 1. **Custom Hook for API Calls**

```typescript
import { useState, useEffect, useCallback } from 'react';

interface UseAPIState<T, M = {}> {
    data: T | null;
    loading: boolean;
    error: APIResponse<never, M>['error'] | null;
    meta: M | null;
}

function useAPI<T, M = {}>(
    endpoint: APIEndpoint<any, T>,
    autoFetch = true,
    initialData?: T
): UseAPIState<T, M> & {
    refetch: (data?: any) => Promise<void>;
    mutate: (data: T | ((prev: T | null) => T)) => void;
} {
    const [state, setState] = useState<UseAPIState<T, M>>({
        data: initialData || null,
        loading: autoFetch,
        error: null,
        meta: null
    });

    const apiClient = new APIClient('/api');

    const executeRequest = useCallback(async (requestData?: any) => {
        setState(prev => ({ ...prev, loading: true, error: null }));

        const response = await apiClient.request(endpoint, requestData);

        if (response.success) {
            setState({
                data: response.data || null,
                loading: false,
                error: null,
                meta: (response.meta || null) as M
            });
        } else {
            setState(prev => ({
                ...prev,
                loading: false,
                error: response.error || null
            }));
        }
    }, [endpoint]);

    const refetch = useCallback((data?: any) => executeRequest(data), [executeRequest]);

    const mutate = useCallback((data: T | ((prev: T | null) => T)) => {
        setState(prev => ({
            ...prev,
            data: typeof data === 'function' ? data(prev.data) : data
        }));
    }, []);

    useEffect(() => {
        if (autoFetch) {
            executeRequest();
        }
    }, [autoFetch, executeRequest]);

    return {
        ...state,
        refetch,
        mutate
    };
}

// Usage
function UserProfile({ userId }: { userId: number }) {
    const { data: user, loading, error, refetch } = useAPI<User>(
        { path: `/users/${userId}`, method: 'GET' }
    );

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;
    if (!user) return <div>User not found</div>;

    return (
        <div>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
            <button onClick={() => refetch()}>Refresh</button>
        </div>
    );
}
```

### 2. **Form Submission with API Response**

```typescript
interface UseFormSubmitState {
    submitting: boolean;
    submitError: APIResponse['error'] | null;
    submitSuccess: boolean;
}

function useFormSubmit<TRequest, TResponse = void>() {
    const [state, setState] = useState<UseFormSubmitState>({
        submitting: false,
        submitError: null,
        submitSuccess: false
    });

    const submit = useCallback(async (
        endpoint: APIEndpoint<TRequest, TResponse>,
        data: TRequest
    ): Promise<APIResponse<TResponse>> => {
        setState({
            submitting: true,
            submitError: null,
            submitSuccess: false
        });

        const apiClient = new APIClient('/api');
        const response = await apiClient.request(endpoint, data);

        if (response.success) {
            setState({
                submitting: false,
                submitError: null,
                submitSuccess: true
            });
        } else {
            setState({
                submitting: false,
                submitError: response.error || null,
                submitSuccess: false
            });
        }

        return response;
    }, []);

    const reset = useCallback(() => {
        setState({
            submitting: false,
            submitError: null,
            submitSuccess: false
        });
    }, []);

    return {
        ...state,
        submit,
        reset
    };
}

// Usage
function CreateUserForm() {
    const { submitting, submitError, submitSuccess, submit, reset } = useFormSubmit<
        Omit<User, 'id'>,
        User
    >();

    const handleSubmit = async (formData: FormData) => {
        const userData = {
            name: formData.get('name') as string,
            email: formData.get('email') as string,
            password: formData.get('password') as string
        };

        const response = await submit(
            { path: '/users', method: 'POST', requiresAuth: false },
            userData
        );

        if (response.success) {
            console.log('User created:', response.data);
            // Redirect or show success message
        }
    };

    if (submitSuccess) {
        return <div>User created successfully!</div>;
    }

    return (
        <form onSubmit={handleSubmit}>
            <input name="name" required />
            <input name="email" type="email" required />
            <input name="password" type="password" required />

            {submitError && (
                <div className="error">
                    {submitError.message}
                    {submitError.fieldErrors && (
                        <ul>
                            {Object.entries(submitError.fieldErrors).map(([field, errors]) => (
                                <li key={field}>{field}: {errors.join(', ')}</li>
                            ))}
                        </ul>
                    )}
                </div>
            )}

            <button type="submit" disabled={submitting}>
                {submitting ? 'Creating...' : 'Create User'}
            </button>
        </form>
    );
}
```

## Best Practices

### 1. **Consistent Response Structure**

```typescript
// ✅ Good: Consistent structure across all endpoints
interface APIResponse<T = any> {
    success: boolean;
    data?: T;
    error?: string;
    message?: string;
}

// ❌ Avoid: Inconsistent structures
// Some endpoints return { data, error }
// Others return { result, success }
// Others return { payload, status }
```

### 2. **Use Discriminated Unions**

```typescript
// ✅ Good: Type-safe discriminated unions
type APIResponse<T> =
    | { success: true; data: T; message?: string }
    | { success: false; error: string; code?: string };

// ❌ Avoid: Optional fields that create unsafe unions
// interface APIResponse<T> {
//     success: boolean;
//     data?: T;        // Unsafe - data might not exist when success is false
//     error?: string;  // Unsafe - error might not exist when success is true
// }
```

### 3. **Generic Response Types**

```typescript
// ✅ Good: Reusable generic types
type ListResponse<T> = APIResponse<T[]> & { pagination?: PaginationMeta };
type SingleResponse<T> = APIResponse<T>;
type EmptyResponse = APIResponse<null>;

// ❌ Avoid: Specific types for each endpoint
// type GetUsersResponse = APIResponse<User[]> & { pagination: PaginationMeta };
// type GetUserResponse = APIResponse<User>;
// type DeleteUserResponse = APIResponse<null>;
```

### 4. **Error Handling Types**

```typescript
// ✅ Good: Structured error information
interface APIError {
    code: string;
    message: string;
    fieldErrors?: Record<string, string[]>;
    details?: any;
}

// ✅ Good: HTTP status mapping
const ERROR_STATUS_MAP: Record<number, ErrorCode> = {
    400: 'VALIDATION_ERROR',
    401: 'AUTHENTICATION_ERROR',
    403: 'AUTHORIZATION_ERROR',
    404: 'NOT_FOUND',
    409: 'CONFLICT',
    429: 'RATE_LIMITED',
    500: 'SERVER_ERROR'
};
```

### 5. **Versioning and Evolution**

```typescript
// ✅ Good: Version-aware responses
interface APIResponse<T = any, V extends string = 'v1'> {
    version: V;
    // ... other fields
}

// ✅ Good: Backward compatibility
type V1Response<T> = APIResponse<T, 'v1'>;
type V2Response<T> = APIResponse<T, 'v2'> & {
    meta: { processingTime: number };
};
```

## Common Interview Questions

### Q: Why use a generic API response type instead of specific types for each endpoint?

**A:** Generic types ensure consistency across all API endpoints, make error handling predictable, and allow for reusable utility functions and hooks. They also make it easier to add features like logging, caching, or retry logic.

### Q: What's the benefit of discriminated unions over optional fields in API responses?

**A:** Discriminated unions provide better type safety. When `success: true`, TypeScript knows `data` exists and `error` doesn't. With optional fields, you have to do runtime checks or use non-null assertions.

### Q: How do you handle pagination in API responses?

**A:** Create a specific `PaginatedResponse<T>` type that extends the base response with pagination metadata including page, limit, total, and navigation links.

### Q: Should API response types include HTTP status codes?

**A:** Yes, including status codes in the response type helps with proper error handling and allows the client to make decisions based on the specific HTTP status, not just success/failure.

## Summary

Reusable API response types should:

1. **Be Generic**: Work with any data type using generics
2. **Handle Success/Error States**: Use discriminated unions for type safety
3. **Include Metadata**: Support pagination, links, timestamps, etc.
4. **Be Consistent**: Same structure across all endpoints
5. **Support Evolution**: Versioning and backward compatibility

**Key Components:**
- `APIResponse<T>`: Base generic response type
- `SuccessResponse<T>` / `ErrorResponse`: Discriminated unions
- `PaginatedResponse<T>`: For list endpoints
- Factory functions for creating responses
- Typed API client with automatic response handling

**Interview Tip:** "I define reusable API response types using generics and discriminated unions. The base `APIResponse<T>` includes success/error states, data, error details, and metadata. For pagination, I extend it with `PaginatedResponse<T>`. This ensures type safety, consistency, and excellent developer experience."