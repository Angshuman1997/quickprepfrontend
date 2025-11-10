# How to cancel API requests in Redux?

## Question
How to cancel API requests in Redux?

## Answer
Use AbortController to cancel requests when component unmounts or user navigates away. This prevents state updates on unmounted components.

## With createAsyncThunk
```javascript
const fetchUsers = createAsyncThunk(
  'users/fetchUsers',
  async (_, { signal }) => {
    const response = await fetch('/api/users', { signal });
    if (!response.ok) throw new Error('Failed');
    return response.json();
  }
);
```

## In Component
```javascript
function UsersList() {
  const dispatch = useDispatch();

  useEffect(() => {
    const promise = dispatch(fetchUsers());

    return () => {
      promise.abort(); // Cancel when component unmounts
    };
  }, [dispatch]);

  return <div>Users list...</div>;
}
```

## Why Cancel Requests?

**Problems without cancellation:**
- Component unmounts but request completes later
- State update on unmounted component causes memory leaks
- Race conditions when user clicks quickly

**Benefits:**
- Prevents memory leaks
- Avoids unnecessary state updates
- Better performance
- Cleaner user experience

## Interview Q&A

**Q: Why cancel API requests in Redux?**

A: Prevents state updates on unmounted components and avoids memory leaks. When user navigates away quickly, cancelled requests don't update state.

**Q: How do you cancel requests in Redux Toolkit?**

A: Use AbortController signal in createAsyncThunk. Pass signal to fetch, and call abort() in component cleanup function.

**Q: What's AbortController?**

A: Browser API to cancel fetch requests. Create controller, pass signal to fetch, call abort() to cancel the request.
                // Check if the rejection was due to cancellation
                if (action.error.name === 'AbortError') {
                    // Don't update error state for cancelled requests
                    return;
                }
                state.error = action.payload as string;
            });
    },
});
```

### 2. **Manual Cancellation with AbortController**

```typescript
// Thunk with manual AbortController
export const searchUsers = createAsyncThunk(
    'users/searchUsers',
    async (query: string, { rejectWithValue }) => {
        const controller = new AbortController();
        const signal = controller.signal;

        try {
            const response = await fetch(`/api/users/search?q=${query}`, {
                signal,
                method: 'GET',
            });

            if (!response.ok) {
                throw new Error('Search failed');
            }

            const data = await response.json();
            return data;
        } catch (error) {
            if (error instanceof Error && error.name === 'AbortError') {
                // Return a special value to indicate cancellation
                return rejectWithValue('CANCELLED');
            }
            return rejectWithValue(error instanceof Error ? error.message : 'Search failed');
        }
    }
);

// Store AbortController references
const abortControllers = new Map<string, AbortController>();

// Enhanced thunk with external cancellation
export const searchUsersCancellable = createAsyncThunk(
    'users/searchUsersCancellable',
    async (query: string, { rejectWithValue }) => {
        // Cancel previous search
        const prevController = abortControllers.get('searchUsers');
        if (prevController) {
            prevController.abort();
        }

        // Create new controller
        const controller = new AbortController();
        abortControllers.set('searchUsers', controller);

        try {
            const response = await fetch(`/api/users/search?q=${query}`, {
                signal: controller.signal,
                method: 'GET',
            });

            if (!response.ok) {
                throw new Error('Search failed');
            }

            const data = await response.json();

            // Clean up controller on success
            abortControllers.delete('searchUsers');

            return data;
        } catch (error) {
            // Clean up controller on error
            abortControllers.delete('searchUsers');

            if (error instanceof Error && error.name === 'AbortError') {
                return rejectWithValue('CANCELLED');
            }
            return rejectWithValue(error instanceof Error ? error.message : 'Search failed');
        }
    }
);
```

### 3. **Component-Level Cancellation**

```typescript
import React, { useEffect, useRef } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { fetchUsers, searchUsersCancellable } from './usersSlice';

function UsersList() {
    const dispatch = useDispatch();
    const { data: users, loading, error } = useSelector(state => state.users);
    const abortControllerRef = useRef<AbortController | null>(null);

    useEffect(() => {
        // Create AbortController for this component
        abortControllerRef.current = new AbortController();

        // Dispatch with signal
        dispatch(fetchUsers());

        // Cleanup function to cancel request when component unmounts
        return () => {
            if (abortControllerRef.current) {
                abortControllerRef.current.abort();
            }
        };
    }, [dispatch]);

    const handleSearch = (query: string) => {
        if (query.trim()) {
            dispatch(searchUsersCancellable(query));
        }
    };

    return (
        <div>
            <input
                type="text"
                placeholder="Search users..."
                onChange={(e) => handleSearch(e.target.value)}
            />

            {loading && <div>Loading...</div>}

            {error && error !== 'CANCELLED' && (
                <div>Error: {error}</div>
            )}

            <ul>
                {users.map(user => (
                    <li key={user.id}>{user.name}</li>
                ))}
            </ul>
        </div>
    );
}
```

## Advanced Cancellation Patterns

### 1. **Debounced Search with Cancellation**

```typescript
import { useCallback, useRef } from 'react';

function SearchUsers() {
    const dispatch = useDispatch();
    const debounceRef = useRef<NodeJS.Timeout>();
    const abortControllerRef = useRef<AbortController>();

    const debouncedSearch = useCallback((query: string) => {
        // Clear previous timeout
        if (debounceRef.current) {
            clearTimeout(debounceRef.current);
        }

        // Cancel previous request
        if (abortControllerRef.current) {
            abortControllerRef.current.abort();
        }

        // Set new timeout
        debounceRef.current = setTimeout(() => {
            if (query.trim()) {
                // Create new AbortController for this request
                abortControllerRef.current = new AbortController();

                // You could pass the signal to a custom thunk
                dispatch(searchUsers(query));
            }
        }, 300); // 300ms debounce
    }, [dispatch]);

    useEffect(() => {
        return () => {
            // Cleanup on unmount
            if (debounceRef.current) {
                clearTimeout(debounceRef.current);
            }
            if (abortControllerRef.current) {
                abortControllerRef.current.abort();
            }
        };
    }, []);

    return (
        <input
            type="text"
            placeholder="Search users..."
            onChange={(e) => debouncedSearch(e.target.value)}
        />
    );
}
```

### 2. **Cancellation with Axios**

```typescript
import axios from 'axios';

// With Axios CancelToken (deprecated but still works)
export const fetchUsersAxios = createAsyncThunk(
    'users/fetchUsersAxios',
    async (_, { rejectWithValue }) => {
        const source = axios.CancelToken.source();

        try {
            const response = await axios.get('/api/users', {
                cancelToken: source.token,
            });

            return response.data;
        } catch (error) {
            if (axios.isCancel(error)) {
                return rejectWithValue('CANCELLED');
            }
            return rejectWithValue(error instanceof Error ? error.message : 'Request failed');
        }
    }
);

// With AbortController (modern approach)
export const fetchUsersModern = createAsyncThunk(
    'users/fetchUsersModern',
    async (_, { rejectWithValue }) => {
        const controller = new AbortController();

        try {
            const response = await axios.get('/api/users', {
                signal: controller.signal,
            });

            return response.data;
        } catch (error) {
            if (error.name === 'AbortError' || axios.isCancel(error)) {
                return rejectWithValue('CANCELLED');
            }
            return rejectWithValue(error instanceof Error ? error.message : 'Request failed');
        }
    }
);
```

### 3. **Global Request Cancellation Manager**

```typescript
// Request manager for global cancellation
class RequestManager {
    private controllers = new Map<string, AbortController>();

    createController(key: string): AbortController {
        // Cancel existing request with same key
        this.cancel(key);

        const controller = new AbortController();
        this.controllers.set(key, controller);
        return controller;
    }

    cancel(key: string): void {
        const controller = this.controllers.get(key);
        if (controller) {
            controller.abort();
            this.controllers.delete(key);
        }
    }

    cancelAll(): void {
        this.controllers.forEach(controller => controller.abort());
        this.controllers.clear();
    }

    remove(key: string): void {
        this.controllers.delete(key);
    }
}

const requestManager = new RequestManager();

// Usage in thunk
export const fetchUserProfile = createAsyncThunk(
    'user/fetchProfile',
    async (userId: number, { rejectWithValue }) => {
        const controller = requestManager.createController(`userProfile_${userId}`);

        try {
            const response = await fetch(`/api/users/${userId}/profile`, {
                signal: controller.signal,
            });

            if (!response.ok) {
                throw new Error('Failed to fetch profile');
            }

            const data = await response.json();

            // Clean up on success
            requestManager.remove(`userProfile_${userId}`);

            return data;
        } catch (error) {
            // Clean up on error
            requestManager.remove(`userProfile_${userId}`);

            if (error instanceof Error && error.name === 'AbortError') {
                return rejectWithValue('CANCELLED');
            }
            return rejectWithValue(error instanceof Error ? error.message : 'Request failed');
        }
    }
);
```

### 4. **Cancellation in Redux Middleware**

```typescript
// Custom middleware for request cancellation
const cancellationMiddleware = (store: any) => (next: any) => (action: any) => {
    // Check if action has a cancellation key
    if (action.meta?.cancelKey) {
        // Cancel previous requests with same key
        const pendingRequests = store.getState().pendingRequests || [];
        const requestToCancel = pendingRequests.find(
            (req: any) => req.cancelKey === action.meta.cancelKey
        );

        if (requestToCancel) {
            requestToCancel.controller.abort();
        }
    }

    return next(action);
};

// Enhanced thunk with cancellation key
export const searchUsersWithKey = createAsyncThunk(
    'users/searchUsersWithKey',
    async (query: string, { rejectWithValue }) => {
        const controller = new AbortController();

        try {
            const response = await fetch(`/api/users/search?q=${query}`, {
                signal: controller.signal,
            });

            if (!response.ok) {
                throw new Error('Search failed');
            }

            return await response.json();
        } catch (error) {
            if (error instanceof Error && error.name === 'AbortError') {
                return rejectWithValue('CANCELLED');
            }
            return rejectWithValue(error instanceof Error ? error.message : 'Search failed');
        }
    },
    {
        // Add cancellation metadata
        getPendingMeta: ({ arg }: { arg: string }) => ({
            cancelKey: 'searchUsers',
        }),
    }
);
```

## Handling Cancellation in UI

### 1. **Loading States with Cancellation**

```typescript
interface UsersState {
    data: User[];
    loading: boolean;
    error: string | null;
    lastCancelledAt: number | null; // Track when requests were cancelled
}

const usersSlice = createSlice({
    name: 'users',
    initialState: {
        data: [],
        loading: false,
        error: null,
        lastCancelledAt: null,
    } as UsersState,
    reducers: {
        resetCancellation: (state) => {
            state.lastCancelledAt = null;
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
                state.lastCancelledAt = null;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;

                if (action.payload === 'CANCELLED') {
                    state.lastCancelledAt = Date.now();
                    // Don't set error for cancelled requests
                } else {
                    state.error = action.payload as string;
                }
            });
    },
});
```

### 2. **Smart Error Display**

```typescript
function UsersList() {
    const dispatch = useDispatch();
    const { loading, error, lastCancelledAt } = useSelector(state => state.users);

    // Don't show errors if request was cancelled recently
    const shouldShowError = error && (!lastCancelledAt || Date.now() - lastCancelledAt > 1000);

    useEffect(() => {
        // Reset cancellation flag after some time
        if (lastCancelledAt) {
            const timer = setTimeout(() => {
                dispatch(resetCancellation());
            }, 1000);

            return () => clearTimeout(timer);
        }
    }, [lastCancelledAt, dispatch]);

    return (
        <div>
            {loading && <div>Loading...</div>}

            {shouldShowError && (
                <div>Error: {error}</div>
            )}

            {/* Rest of component */}
        </div>
    );
}
```

### 3. **Optimistic Updates with Cancellation**

```typescript
export const updateUserOptimistic = createAsyncThunk(
    'users/updateUserOptimistic',
    async ({ id, updates }: { id: number; updates: Partial<User> }, { rejectWithValue }) => {
        const controller = new AbortController();

        // Store original data for rollback
        const originalData = getState().users.data.find(user => user.id === id);

        // Optimistically update UI
        dispatch(updateUserOptimistic({ id, updates }));

        try {
            const response = await fetch(`/api/users/${id}`, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(updates),
                signal: controller.signal,
            });

            if (!response.ok) {
                throw new Error('Update failed');
            }

            return await response.json();
        } catch (error) {
            // Rollback optimistic update
            if (originalData) {
                dispatch(rollbackUserUpdate(originalData));
            }

            if (error instanceof Error && error.name === 'AbortError') {
                return rejectWithValue('CANCELLED');
            }

            return rejectWithValue(error instanceof Error ? error.message : 'Update failed');
        }
    }
);
```

## Testing Cancelled Requests

```typescript
describe('fetchUsers', () => {
    it('should handle request cancellation', async () => {
        const mockFetch = jest.fn();
        global.fetch = mockFetch;

        // Mock fetch to return a promise that can be aborted
        const abortController = new AbortController();
        mockFetch.mockImplementation(() => {
            return new Promise((_, reject) => {
                // Simulate immediate abortion
                setTimeout(() => {
                    reject(new Error('Aborted'));
                }, 0);
            });
        });

        const dispatch = jest.fn();
        const getState = jest.fn();

        // This would normally be called by Redux
        const thunk = fetchUsers();
        await thunk(dispatch, getState, undefined);

        // Check that rejection was handled
        expect(dispatch).toHaveBeenCalledWith(
            expect.objectContaining({
                type: 'users/fetchUsers/rejected',
                payload: undefined, // Cancellation doesn't set error
            })
        );
    });
});
```

## Best Practices

### 1. **When to Cancel Requests**

```typescript
// ✅ Good: Cancel on component unmount
useEffect(() => {
    const controller = new AbortController();
    dispatch(fetchData());

    return () => controller.abort();
}, []);

// ✅ Good: Cancel previous search on new search
const handleSearch = (query) => {
    // Cancel previous request
    dispatch(searchUsers(query));
};

// ❌ Avoid: Don't cancel important data fetches
// Don't cancel user profile fetch when switching tabs
```

### 2. **Error Handling for Cancelled Requests**

```typescript
// ✅ Good: Differentiate cancellation from real errors
.addCase(fetchUsers.rejected, (state, action) => {
    if (action.error.name === 'AbortError') {
        // Don't update error state
        return;
    }
    state.error = action.payload;
});

// ✅ Good: Use rejectWithValue for custom cancellation
return rejectWithValue('CANCELLED');
```

### 3. **Memory Management**

```typescript
// ✅ Good: Clean up AbortControllers
const controllers = new Map();

function createController(key) {
    const existing = controllers.get(key);
    if (existing) {
        existing.abort();
    }

    const controller = new AbortController();
    controllers.set(key, controller);
    return controller;
}

function cleanupController(key) {
    controllers.delete(key);
}
```

## Common Interview Questions

### Q: Why is request cancellation important in Redux?

**A:** It prevents state updates on unmounted components, avoids race conditions in rapid user interactions (like search), and improves performance by not processing unnecessary requests.

### Q: What's the difference between AbortController and CancelToken?

**A:** AbortController is the modern web standard (also works with fetch), while CancelToken was Axios-specific. AbortController is more widely supported and is the recommended approach.

### Q: How do you handle cancellation in optimistic updates?

**A:** Store the original data before optimistic update, then rollback if the request is cancelled or fails. Don't rollback if the cancellation happens after a successful response.

### Q: When should you NOT cancel requests?

**A:** Don't cancel critical data fetches (user authentication, initial app data), important form submissions, or requests that affect server state and can't be safely repeated.

## Summary

**Key Methods for Cancellation:**
1. **AbortController with createAsyncThunk**: Pass `signal` to fetch/axios
2. **Manual AbortController management**: Track controllers externally
3. **Component-level cancellation**: Cancel in useEffect cleanup
4. **Global request manager**: Centralized cancellation control

**Best Practices:**
- Cancel requests on component unmount
- Handle cancelled requests differently from errors
- Use AbortController over deprecated CancelToken
- Clean up controllers to prevent memory leaks
- Don't cancel critical or state-changing requests

**Interview Tip:** "I cancel API requests in Redux to prevent race conditions and unnecessary state updates. I use AbortController with createAsyncThunk's signal parameter, and handle cancellation in reducers by checking for AbortError to avoid setting error states for cancelled requests."