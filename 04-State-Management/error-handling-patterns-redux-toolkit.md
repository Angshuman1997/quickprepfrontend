# Error handling patterns in Redux Toolkit

## Question
Error handling patterns in Redux Toolkit

## Answer

Redux Toolkit provides powerful patterns for handling errors in async operations through `createAsyncThunk` and `createSlice`. Proper error handling ensures a good user experience, prevents crashes, and provides meaningful feedback. Understanding these patterns is crucial for building robust Redux applications.

## Basic Error Handling with createAsyncThunk

### 1. **Standard Async Thunk with Error Handling**

```typescript
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';

interface User {
    id: number;
    name: string;
    email: string;
}

interface UsersState {
    data: User[];
    loading: boolean;
    error: string | null;
}

// Async thunk with error handling
export const fetchUsers = createAsyncThunk(
    'users/fetchUsers',
    async (_, { rejectWithValue }) => {
        try {
            const response = await fetch('/api/users');

            if (!response.ok) {
                // Handle HTTP errors
                const errorData = await response.json().catch(() => ({}));
                return rejectWithValue({
                    message: errorData.message || `HTTP ${response.status}: ${response.statusText}`,
                    status: response.status,
                    code: errorData.code,
                });
            }

            const data = await response.json();
            return data;
        } catch (error) {
            // Handle network errors
            return rejectWithValue({
                message: error instanceof Error ? error.message : 'Network error',
                code: 'NETWORK_ERROR',
            });
        }
    }
);

// Slice with error handling
const usersSlice = createSlice({
    name: 'users',
    initialState: {
        data: [],
        loading: false,
        error: null,
    } as UsersState,
    reducers: {
        clearError: (state) => {
            state.error = null;
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchUsers.pending, (state) => {
                state.loading = true;
                state.error = null;
            })
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.loading = false;
                state.data = action.payload;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;
                // Handle different error types
                if (action.payload) {
                    // Error from rejectWithValue
                    state.error = (action.payload as any).message;
                } else if (action.error) {
                    // Generic error
                    state.error = action.error.message || 'An error occurred';
                }
            });
    },
});

export const { clearError } = usersSlice.actions;
```

### 2. **Typed Error Handling**

```typescript
// Define error types
interface ApiError {
    message: string;
    code: string;
    status?: number;
    details?: Record<string, any>;
}

interface UsersState {
    data: User[];
    loading: boolean;
    error: ApiError | null;
}

// Typed async thunk
export const fetchUsers = createAsyncThunk<
    User[], // Return type
    void,   // First argument type
    { rejectValue: ApiError } // Thunk config
>(
    'users/fetchUsers',
    async (_, { rejectWithValue }) => {
        try {
            const response = await fetch('/api/users');

            if (!response.ok) {
                const errorData = await response.json().catch(() => ({}));
                return rejectWithValue({
                    message: errorData.message || `HTTP ${response.status}`,
                    code: errorData.code || 'HTTP_ERROR',
                    status: response.status,
                    details: errorData.details,
                });
            }

            return await response.json();
        } catch (error) {
            return rejectWithValue({
                message: error instanceof Error ? error.message : 'Network error',
                code: 'NETWORK_ERROR',
            });
        }
    }
);

// Slice with typed error handling
const usersSlice = createSlice({
    name: 'users',
    initialState: {
        data: [],
        loading: false,
        error: null,
    } as UsersState,
    reducers: {
        clearError: (state) => {
            state.error = null;
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchUsers.pending, (state) => {
                state.loading = true;
                state.error = null;
            })
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.loading = false;
                state.data = action.payload;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload || {
                    message: action.error?.message || 'Unknown error',
                    code: 'UNKNOWN_ERROR',
                };
            });
    },
});
```

## Advanced Error Handling Patterns

### 1. **Retry Logic with Exponential Backoff**

```typescript
export const fetchUsersWithRetry = createAsyncThunk(
    'users/fetchUsersWithRetry',
    async (_, { rejectWithValue }) => {
        const maxRetries = 3;
        let lastError: ApiError;

        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                const response = await fetch('/api/users');

                if (!response.ok) {
                    // Only retry on server errors (5xx) or network errors
                    if (response.status >= 500 || attempt === maxRetries) {
                        const errorData = await response.json().catch(() => ({}));
                        return rejectWithValue({
                            message: errorData.message || `HTTP ${response.status}`,
                            code: 'HTTP_ERROR',
                            status: response.status,
                            attempt,
                        });
                    }

                    // Wait before retrying with exponential backoff
                    await new Promise(resolve =>
                        setTimeout(resolve, Math.pow(2, attempt) * 1000)
                    );
                    continue;
                }

                return await response.json();
            } catch (error) {
                lastError = {
                    message: error instanceof Error ? error.message : 'Network error',
                    code: 'NETWORK_ERROR',
                    attempt,
                };

                if (attempt === maxRetries) {
                    return rejectWithValue(lastError);
                }

                // Wait before retrying
                await new Promise(resolve =>
                    setTimeout(resolve, Math.pow(2, attempt) * 1000)
                );
            }
        }

        return rejectWithValue(lastError!);
    }
);

// Enhanced state with retry information
interface UsersState {
    data: User[];
    loading: boolean;
    error: ApiError | null;
    retryCount: number;
    lastAttempt: number | null;
}

const usersSlice = createSlice({
    name: 'users',
    initialState: {
        data: [],
        loading: false,
        error: null,
        retryCount: 0,
        lastAttempt: null,
    } as UsersState,
    reducers: {
        resetRetry: (state) => {
            state.retryCount = 0;
            state.lastAttempt = null;
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchUsersWithRetry.pending, (state) => {
                state.loading = true;
                state.error = null;
                state.lastAttempt = Date.now();
            })
            .addCase(fetchUsersWithRetry.fulfilled, (state, action) => {
                state.loading = false;
                state.data = action.payload;
                state.retryCount = 0;
            })
            .addCase(fetchUsersWithRetry.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload as ApiError;
                state.retryCount = (action.payload as ApiError)?.attempt || 0;
            });
    },
});
```

### 2. **Conditional Error Handling**

```typescript
export const updateUser = createAsyncThunk(
    'users/updateUser',
    async ({ id, updates }: { id: number; updates: Partial<User> }, { rejectWithValue, getState }) => {
        const state = getState() as RootState;
        const existingUser = state.users.data.find(user => user.id === id);

        if (!existingUser) {
            return rejectWithValue({
                message: 'User not found',
                code: 'USER_NOT_FOUND',
                status: 404,
            });
        }

        try {
            const response = await fetch(`/api/users/${id}`, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(updates),
            });

            if (!response.ok) {
                const errorData = await response.json().catch(() => ({}));

                // Handle specific error codes
                if (response.status === 409) {
                    return rejectWithValue({
                        message: 'User data has been modified by another user',
                        code: 'CONFLICT',
                        status: 409,
                        details: errorData,
                    });
                }

                return rejectWithValue({
                    message: errorData.message || 'Update failed',
                    code: 'UPDATE_ERROR',
                    status: response.status,
                });
            }

            return await response.json();
        } catch (error) {
            return rejectWithValue({
                message: error instanceof Error ? error.message : 'Network error',
                code: 'NETWORK_ERROR',
            });
        }
    }
);

// Slice with conditional error handling
const usersSlice = createSlice({
    name: 'users',
    initialState,
    reducers: {
        clearSpecificError: (state, action) => {
            if (state.error?.code === action.payload) {
                state.error = null;
            }
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(updateUser.rejected, (state, action) => {
                state.loading = false;
                const error = action.payload as ApiError;

                // Handle specific errors differently
                switch (error?.code) {
                    case 'USER_NOT_FOUND':
                        // Could redirect to user list or show specific message
                        state.error = { ...error, message: 'User no longer exists' };
                        break;
                    case 'CONFLICT':
                        // Could show merge conflict UI
                        state.error = error;
                        break;
                    default:
                        state.error = error;
                }
            });
    },
});
```

### 3. **Global Error Handling Middleware**

```typescript
// Error handling middleware
const errorHandlingMiddleware = (store: any) => (next: any) => (action: any) => {
    if (action.type.endsWith('/rejected')) {
        const error = action.payload || action.error;

        // Log errors in development
        if (process.env.NODE_ENV === 'development') {
            console.error('Async action failed:', {
                action: action.type,
                error,
                state: store.getState(),
            });
        }

        // Send to error reporting service
        if (error && typeof error === 'object' && error.code !== 'CANCELLED') {
            errorReporting.captureException(error, {
                tags: {
                    action: action.type,
                    userId: store.getState().auth.user?.id,
                },
                extra: {
                    payload: action.payload,
                },
            });
        }

        // Handle authentication errors globally
        if (error?.status === 401) {
            store.dispatch(logoutUser());
            // Could redirect to login page
        }

        // Handle rate limiting
        if (error?.status === 429) {
            store.dispatch(showNotification({
                type: 'warning',
                message: 'Too many requests. Please try again later.',
            }));
        }
    }

    return next(action);
};

// Apply middleware
const store = configureStore({
    reducer: rootReducer,
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware().concat(errorHandlingMiddleware),
});
```

### 4. **Error Recovery Patterns**

```typescript
// Slice with error recovery
const usersSlice = createSlice({
    name: 'users',
    initialState: {
        data: [],
        loading: false,
        error: null,
        failedAttempts: 0,
        lastErrorTime: null,
    } as UsersState & { failedAttempts: number; lastErrorTime: number | null },
    reducers: {
        clearError: (state) => {
            state.error = null;
            state.failedAttempts = 0;
            state.lastErrorTime = null;
        },
        incrementFailedAttempts: (state) => {
            state.failedAttempts += 1;
            state.lastErrorTime = Date.now();
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload as ApiError;
                state.failedAttempts += 1;
                state.lastErrorTime = Date.now();

                // Implement circuit breaker pattern
                if (state.failedAttempts >= 5) {
                    state.error = {
                        message: 'Service temporarily unavailable. Please try again later.',
                        code: 'CIRCUIT_BREAKER',
                    };
                }
            })
            .addCase(fetchUsers.fulfilled, (state) => {
                // Reset on success
                state.failedAttempts = 0;
                state.lastErrorTime = null;
            });
    },
});

// Recovery thunk
export const retryFailedRequest = createAsyncThunk(
    'users/retryFailedRequest',
    async (_, { getState, dispatch }) => {
        const state = getState() as RootState;

        if (state.users.failedAttempts >= 5) {
            const timeSinceLastError = Date.now() - (state.users.lastErrorTime || 0);

            // Wait 30 seconds before allowing retry after circuit breaker
            if (timeSinceLastError < 30000) {
                throw new Error('Circuit breaker active. Please wait before retrying.');
            }
        }

        // Retry the original request
        await dispatch(fetchUsers());
    }
);
```

## Component-Level Error Handling

### 1. **Error Boundary Integration**

```typescript
// Error boundary for Redux errors
class ReduxErrorBoundary extends React.Component {
    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
        // Dispatch error to Redux store
        this.props.dispatch({
            type: 'APP_ERROR',
            payload: {
                error: error.message,
                stack: error.stack,
                componentStack: errorInfo.componentStack,
            },
        });
    }

    render() {
        return this.props.children;
    }
}

// Connect to Redux
const ConnectedErrorBoundary = connect()(ReduxErrorBoundary);

// Usage
<ConnectedErrorBoundary>
    <App />
</ConnectedErrorBoundary>
```

### 2. **User-Friendly Error Messages**

```typescript
// Error message mapper
const getErrorMessage = (error: ApiError | null): string => {
    if (!error) return '';

    switch (error.code) {
        case 'NETWORK_ERROR':
            return 'Unable to connect. Please check your internet connection.';
        case 'USER_NOT_FOUND':
            return 'The requested user could not be found.';
        case 'VALIDATION_ERROR':
            return 'Please check your input and try again.';
        case 'UNAUTHORIZED':
            return 'You are not authorized to perform this action.';
        case 'CONFLICT':
            return 'This item has been modified by someone else.';
        case 'RATE_LIMITED':
            return 'Too many requests. Please wait a moment.';
        default:
            return error.message || 'An unexpected error occurred.';
    }
};

// Component usage
function UsersList() {
    const { error, loading } = useSelector(state => state.users);
    const errorMessage = getErrorMessage(error);

    if (loading) return <div>Loading...</div>;

    return (
        <div>
            {error && (
                <div className="error-banner">
                    <p>{errorMessage}</p>
                    <button onClick={() => dispatch(clearError())}>
                        Dismiss
                    </button>
                </div>
            )}

            {/* Rest of component */}
        </div>
    );
}
```

### 3. **Optimistic Updates with Error Rollback**

```typescript
export const deleteUserOptimistic = createAsyncThunk(
    'users/deleteUserOptimistic',
    async (userId: number, { rejectWithValue, getState }) => {
        const state = getState() as RootState;
        const userToDelete = state.users.data.find(user => user.id === userId);

        if (!userToDelete) {
            return rejectWithValue({
                message: 'User not found',
                code: 'USER_NOT_FOUND',
            });
        }

        // Optimistically remove from state
        // (This would be handled in the slice)

        try {
            const response = await fetch(`/api/users/${userId}`, {
                method: 'DELETE',
            });

            if (!response.ok) {
                throw new Error('Delete failed');
            }

            return userId; // Return deleted ID
        } catch (error) {
            // Error will trigger rollback in the slice
            return rejectWithValue({
                message: error instanceof Error ? error.message : 'Delete failed',
                code: 'DELETE_ERROR',
                originalUser: userToDelete, // For rollback
            });
        }
    }
);

// Slice with optimistic updates and rollback
const usersSlice = createSlice({
    name: 'users',
    initialState,
    reducers: {
        // Optimistically remove user
        removeUserOptimistic: (state, action) => {
            state.data = state.data.filter(user => user.id !== action.payload);
        },
        // Rollback on error
        rollbackUser: (state, action) => {
            state.data.push(action.payload);
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(deleteUserOptimistic.pending, (state, action) => {
                // Optimistically remove
                state.data = state.data.filter(user => user.id !== action.payload);
            })
            .addCase(deleteUserOptimistic.fulfilled, (state) => {
                // Success - no additional action needed
            })
            .addCase(deleteUserOptimistic.rejected, (state, action) => {
                // Rollback on error
                const error = action.payload as any;
                if (error?.originalUser) {
                    state.data.push(error.originalUser);
                }
                state.error = {
                    message: error?.message || 'Delete failed',
                    code: error?.code || 'DELETE_ERROR',
                };
            });
    },
});
```

## Testing Error Scenarios

```typescript
describe('usersSlice error handling', () => {
    it('handles network errors', async () => {
        // Mock fetch to throw network error
        global.fetch = jest.fn(() =>
            Promise.reject(new Error('Network error'))
        );

        const store = configureStore({ reducer: usersSlice.reducer });
        await store.dispatch(fetchUsers());

        const state = store.getState();
        expect(state.error).toEqual({
            message: 'Network error',
            code: 'NETWORK_ERROR',
        });
    });

    it('handles HTTP errors', async () => {
        // Mock fetch to return error response
        global.fetch = jest.fn(() =>
            Promise.resolve({
                ok: false,
                status: 404,
                json: () => Promise.resolve({ message: 'Not found' }),
            })
        );

        const store = configureStore({ reducer: usersSlice.reducer });
        await store.dispatch(fetchUsers());

        const state = store.getState();
        expect(state.error).toEqual({
            message: 'Not found',
            code: 'HTTP_ERROR',
            status: 404,
        });
    });

    it('clears errors on successful request', async () => {
        // First dispatch with error
        global.fetch = jest.fn(() =>
            Promise.reject(new Error('Network error'))
        );

        const store = configureStore({ reducer: usersSlice.reducer });
        await store.dispatch(fetchUsers());

        expect(store.getState().error).not.toBeNull();

        // Then successful request
        global.fetch = jest.fn(() =>
            Promise.resolve({
                ok: true,
                json: () => Promise.resolve([{ id: 1, name: 'John' }]),
            })
        );

        await store.dispatch(fetchUsers());
        expect(store.getState().error).toBeNull();
    });
});
```

## Best Practices

### 1. **Consistent Error Structure**

```typescript
// ✅ Good: Consistent error interface
interface ApiError {
    message: string;
    code: string;
    status?: number;
    details?: any;
    timestamp?: string;
}

// ❌ Avoid: Inconsistent error shapes
// Sometimes { error: string }
// Sometimes { message: string, code: number }
// Sometimes nested objects
```

### 2. **Error Codes vs Messages**

```typescript
// ✅ Good: Use error codes for logic, messages for display
switch (error.code) {
    case 'VALIDATION_ERROR':
        // Handle validation logic
        break;
    case 'NETWORK_ERROR':
        // Handle network logic
        break;
}

// Use messages for user display
const displayMessage = getUserFriendlyMessage(error.code, error.message);
```

### 3. **Graceful Degradation**

```typescript
// ✅ Good: Handle errors gracefully
.addCase(fetchUsers.rejected, (state, action) => {
    // Don't break the app on error
    state.loading = false;
    state.error = action.payload;

    // Maybe show cached data if available
    if (state.cachedData) {
        state.data = state.cachedData;
    }
});
```

### 4. **Error Boundaries**

```typescript
// ✅ Good: Use error boundaries for UI errors
class ErrorBoundary extends React.Component {
    state = { hasError: false };

    static getDerivedStateFromError() {
        return { hasError: true };
    }

    componentDidCatch(error, errorInfo) {
        // Log to error reporting service
        errorReporting.captureException(error, { extra: errorInfo });
    }

    render() {
        if (this.state.hasError) {
            return <ErrorFallback onRetry={() => this.setState({ hasError: false })} />;
        }
        return this.props.children;
    }
}
```

## Common Interview Questions

### Q: How do you handle errors in Redux Toolkit async thunks?

**A:** I use `rejectWithValue` to return custom error objects from thunks, then handle them in the `rejected` case of the slice. This allows typed error handling and user-friendly error messages.

### Q: What's the difference between action.error and action.payload in rejected cases?

**A:** `action.payload` contains the value passed to `rejectWithValue`, while `action.error` contains generic thunk error information. I prefer using `action.payload` for custom error objects.

### Q: How do you handle different types of errors (network, validation, auth)?

**A:** I create typed error objects with codes and handle them conditionally in reducers. For global errors like auth failures, I use middleware to intercept and handle them consistently.

### Q: What patterns do you use for error recovery?

**A:** I implement retry logic with exponential backoff, circuit breaker patterns for repeated failures, and optimistic updates with rollback on errors. I also provide user actions to retry failed operations.

## Summary

**Key Error Handling Patterns:**
1. **rejectWithValue**: Return custom error objects from thunks
2. **Typed Errors**: Define consistent error interfaces
3. **Conditional Handling**: Handle different error types appropriately
4. **Retry Logic**: Implement exponential backoff for transient errors
5. **Global Middleware**: Handle cross-cutting errors consistently
6. **User-Friendly Messages**: Map error codes to user-friendly text

**Best Practices:**
- Use consistent error structures
- Handle errors gracefully without breaking the app
- Provide recovery mechanisms (retry, rollback)
- Log errors for debugging
- Test error scenarios thoroughly

**Interview Tip:** "In Redux Toolkit, I handle errors using `rejectWithValue` in thunks to return typed error objects, then handle them in slice `rejected` cases. I implement retry logic, user-friendly messages, and global error handling through middleware for robust error management."