# How to handle loading states in Redux Toolkit?

## Question
How to handle loading states in Redux Toolkit?

## Answer
Use createAsyncThunk which generates pending/fulfilled/rejected actions. Handle them in extraReducers to manage loading state.

## Basic Example
```javascript
const usersSlice = createSlice({
  name: 'users',
  initialState: { data: [], loading: false, error: null },
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
        state.error = action.error.message;
      });
  }
});
```

## In Component
```javascript
function UsersList() {
  const dispatch = useDispatch();
  const { data, loading, error } = useSelector(state => state.users);

  useEffect(() => {
    dispatch(fetchUsers());
  }, [dispatch]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <ul>
      {data.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

## Interview Q&A

**Q: How do you handle loading states in Redux Toolkit?**

A: Use createAsyncThunk which generates pending/fulfilled/rejected actions, then handle them in extraReducers to set loading: true/false.

**Q: What's the pattern for loading states?**

A: Set loading: true in pending action, set loading: false in both fulfilled and rejected actions.

**Q: How do you show loading in components?**

A: Use useSelector to get the loading state from Redux store, conditionally render loading spinner or message.

### 2. **Component Usage**

```typescript
import { useSelector, useDispatch } from 'react-redux';
import { fetchUsers } from './usersSlice';

function UsersList() {
    const dispatch = useDispatch();
    const { data: users, loading, error } = useSelector((state: RootState) => state.users);

    React.useEffect(() => {
        dispatch(fetchUsers());
    }, [dispatch]);

    if (loading) {
        return <div>Loading users...</div>;
    }

    if (error) {
        return <div>Error: {error}</div>;
    }

    return (
        <ul>
            {users.map(user => (
                <li key={user.id}>{user.name}</li>
            ))}
        </ul>
    );
}
```

## Advanced Loading Patterns

### 1. **Multiple Loading States**

```typescript
interface UsersState {
    data: User[];
    loading: {
        fetch: boolean;
        create: boolean;
        update: boolean;
        delete: boolean;
    };
    error: {
        fetch: string | null;
        create: string | null;
        update: string | null;
        delete: string | null;
    };
}

const initialState: UsersState = {
    data: [],
    loading: {
        fetch: false,
        create: false,
        update: false,
        delete: false,
    },
    error: {
        fetch: null,
        create: null,
        update: null,
        delete: null,
    },
};

// Async thunks for different operations
export const fetchUsers = createAsyncThunk('users/fetchUsers', async () => {
    const response = await fetch('/api/users');
    return response.json();
});

export const createUser = createAsyncThunk('users/createUser', async (user: Omit<User, 'id'>) => {
    const response = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(user),
    });
    return response.json();
});

export const updateUser = createAsyncThunk('users/updateUser', async ({ id, updates }: { id: number; updates: Partial<User> }) => {
    const response = await fetch(`/api/users/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates),
    });
    return response.json();
});

export const deleteUser = createAsyncThunk('users/deleteUser', async (id: number) => {
    await fetch(`/api/users/${id}`, { method: 'DELETE' });
    return id;
});

// Slice with granular loading states
const usersSlice = createSlice({
    name: 'users',
    initialState,
    reducers: {
        clearError: (state, action) => {
            const operation = action.payload;
            state.error[operation] = null;
        },
    },
    extraReducers: (builder) => {
        // Fetch users
        builder
            .addCase(fetchUsers.pending, (state) => {
                state.loading.fetch = true;
                state.error.fetch = null;
            })
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.loading.fetch = false;
                state.data = action.payload;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading.fetch = false;
                state.error.fetch = action.payload as string;
            })

            // Create user
            .addCase(createUser.pending, (state) => {
                state.loading.create = true;
                state.error.create = null;
            })
            .addCase(createUser.fulfilled, (state, action) => {
                state.loading.create = false;
                state.data.push(action.payload);
            })
            .addCase(createUser.rejected, (state, action) => {
                state.loading.create = false;
                state.error.create = action.payload as string;
            })

            // Update user
            .addCase(updateUser.pending, (state) => {
                state.loading.update = true;
                state.error.update = null;
            })
            .addCase(updateUser.fulfilled, (state, action) => {
                state.loading.update = false;
                const index = state.data.findIndex(user => user.id === action.payload.id);
                if (index !== -1) {
                    state.data[index] = action.payload;
                }
            })
            .addCase(updateUser.rejected, (state, action) => {
                state.loading.update = false;
                state.error.update = action.payload as string;
            })

            // Delete user
            .addCase(deleteUser.pending, (state) => {
                state.loading.delete = true;
                state.error.delete = null;
            })
            .addCase(deleteUser.fulfilled, (state, action) => {
                state.loading.delete = false;
                state.data = state.data.filter(user => user.id !== action.payload);
            })
            .addCase(deleteUser.rejected, (state, action) => {
                state.loading.delete = false;
                state.error.delete = action.payload as string;
            });
    },
});

export const { clearError } = usersSlice.actions;
```

### 2. **Global Loading State**

```typescript
// Global loading slice
interface LoadingState {
    activeRequests: number;
    globalLoading: boolean;
}

const loadingSlice = createSlice({
    name: 'loading',
    initialState: {
        activeRequests: 0,
        globalLoading: false,
    } as LoadingState,
    reducers: {
        startLoading: (state) => {
            state.activeRequests += 1;
            state.globalLoading = true;
        },
        stopLoading: (state) => {
            state.activeRequests = Math.max(0, state.activeRequests - 1);
            state.globalLoading = state.activeRequests > 0;
        },
    },
});

// Loading middleware
const loadingMiddleware = (store: any) => (next: any) => (action: any) => {
    // Check if action is async thunk
    if (action.type && action.type.endsWith('/pending')) {
        store.dispatch(startLoading());
    }

    if (action.type && (action.type.endsWith('/fulfilled') || action.type.endsWith('/rejected'))) {
        store.dispatch(stopLoading());
    }

    return next(action);
};

// Apply middleware
const store = configureStore({
    reducer: {
        users: usersSlice.reducer,
        loading: loadingSlice.reducer,
    },
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware().concat(loadingMiddleware),
});

// Global loading component
function GlobalLoadingIndicator() {
    const { globalLoading } = useSelector((state: RootState) => state.loading);

    if (!globalLoading) return null;

    return (
        <div className="global-loading">
            <div className="spinner"></div>
            <span>Loading...</span>
        </div>
    );
}
```

### 3. **Loading States with Entity IDs**

```typescript
interface UsersState {
    entities: Record<number, User>;
    ids: number[];
    loading: {
        global: boolean;
        byId: Record<number, boolean>;
    };
    error: {
        global: string | null;
        byId: Record<number, string | null>;
    };
}

const initialState: UsersState = {
    entities: {},
    ids: [],
    loading: {
        global: false,
        byId: {},
    },
    error: {
        global: null,
        byId: {},
    },
};

// Thunk for updating specific user
export const updateUser = createAsyncThunk(
    'users/updateUser',
    async ({ id, updates }: { id: number; updates: Partial<User> }) => {
        const response = await fetch(`/api/users/${id}`, {
            method: 'PATCH',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(updates),
        });
        return { id, user: await response.json() };
    }
);

// Slice with entity-specific loading
const usersSlice = createSlice({
    name: 'users',
    initialState,
    extraReducers: (builder) => {
        builder
            .addCase(updateUser.pending, (state, action) => {
                const userId = action.meta.arg.id;
                state.loading.byId[userId] = true;
                state.error.byId[userId] = null;
            })
            .addCase(updateUser.fulfilled, (state, action) => {
                const { id, user } = action.payload;
                state.loading.byId[id] = false;
                state.entities[id] = user;
            })
            .addCase(updateUser.rejected, (state, action) => {
                const userId = action.meta.arg.id;
                state.loading.byId[userId] = false;
                state.error.byId[userId] = action.payload as string;
            });
    },
});

// Component with specific loading states
function UserItem({ user }: { user: User }) {
    const dispatch = useDispatch();
    const loading = useSelector((state: RootState) =>
        state.users.loading.byId[user.id]
    );
    const error = useSelector((state: RootState) =>
        state.users.error.byId[user.id]
    );

    const handleUpdate = () => {
        dispatch(updateUser({ id: user.id, updates: { name: 'Updated Name' } }));
    };

    return (
        <div>
            <span>{user.name}</span>
            {loading && <span>Updating...</span>}
            {error && <span>Error: {error}</span>}
            <button onClick={handleUpdate} disabled={loading}>
                Update
            </button>
        </div>
    );
}
```

## Loading State Patterns

### 1. **Optimistic Updates with Loading**

```typescript
export const toggleUserStatus = createAsyncThunk(
    'users/toggleStatus',
    async (userId: number, { rejectWithValue }) => {
        try {
            // Optimistically update UI first
            const response = await fetch(`/api/users/${userId}/toggle-status`, {
                method: 'POST',
            });
            return { userId, status: await response.json() };
        } catch (error) {
            return rejectWithValue({ userId, error: 'Failed to toggle status' });
        }
    }
);

interface UsersState {
    entities: Record<number, User>;
    optimisticUpdates: Record<number, Partial<User>>;
    loading: Record<number, boolean>;
    error: Record<number, string | null>;
}

const usersSlice = createSlice({
    name: 'users',
    initialState: {
        entities: {},
        optimisticUpdates: {},
        loading: {},
        error: {},
    } as UsersState,
    reducers: {
        // Apply optimistic update
        applyOptimisticUpdate: (state, action) => {
            const { userId, updates } = action.payload;
            state.optimisticUpdates[userId] = {
                ...state.optimisticUpdates[userId],
                ...updates,
            };
        },
        // Revert optimistic update on error
        revertOptimisticUpdate: (state, action) => {
            const userId = action.payload;
            delete state.optimisticUpdates[userId];
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(toggleUserStatus.pending, (state, action) => {
                const userId = action.meta.arg;
                state.loading[userId] = true;
                // Apply optimistic update
                const currentUser = state.entities[userId];
                if (currentUser) {
                    state.optimisticUpdates[userId] = {
                        ...currentUser,
                        active: !currentUser.active,
                    };
                }
            })
            .addCase(toggleUserStatus.fulfilled, (state, action) => {
                const { userId, status } = action.payload;
                state.loading[userId] = false;
                state.entities[userId] = { ...state.entities[userId], ...status };
                delete state.optimisticUpdates[userId];
            })
            .addCase(toggleUserStatus.rejected, (state, action) => {
                const { userId } = action.payload as any;
                state.loading[userId] = false;
                state.error[userId] = 'Failed to toggle status';
                delete state.optimisticUpdates[userId];
            });
    },
});

// Selector for user with optimistic updates
export const selectUserWithOptimistic = (state: RootState, userId: number) => {
    const user = state.users.entities[userId];
    const optimistic = state.users.optimisticUpdates[userId];

    return optimistic ? { ...user, ...optimistic } : user;
};
```

### 2. **Sequential Loading States**

```typescript
// For operations that need to happen in sequence
export const createUserAndFetchList = createAsyncThunk(
    'users/createAndFetch',
    async (userData: Omit<User, 'id'>, { dispatch }) => {
        // First create user
        await dispatch(createUser(userData));

        // Then fetch updated list
        await dispatch(fetchUsers());

        return 'Success';
    }
);

interface UsersState {
    data: User[];
    loading: {
        create: boolean;
        fetch: boolean;
        createAndFetch: boolean; // Combined operation
    };
    error: string | null;
}

const usersSlice = createSlice({
    name: 'users',
    initialState,
    extraReducers: (builder) => {
        builder
            // Handle individual operations
            .addCase(createUser.pending, (state) => {
                state.loading.create = true;
            })
            .addCase(createUser.fulfilled, (state) => {
                state.loading.create = false;
            })
            .addCase(createUser.rejected, (state) => {
                state.loading.create = false;
            })

            // Handle combined operation
            .addCase(createUserAndFetchList.pending, (state) => {
                state.loading.createAndFetch = true;
                state.error = null;
            })
            .addCase(createUserAndFetchList.fulfilled, (state) => {
                state.loading.createAndFetch = false;
            })
            .addCase(createUserAndFetchList.rejected, (state, action) => {
                state.loading.createAndFetch = false;
                state.error = action.payload as string;
            });
    },
});
```

### 3. **Loading States with Timeouts**

```typescript
// Add timeout handling to prevent infinite loading
const loadingSlice = createSlice({
    name: 'loading',
    initialState: {
        operations: {} as Record<string, {
            startTime: number;
            timeoutId: number | null;
        }>,
    },
    reducers: {
        startOperation: (state, action) => {
            const operationId = action.payload;
            const timeoutId = setTimeout(() => {
                // Dispatch timeout action
                store.dispatch(operationTimeout(operationId));
            }, 30000); // 30 second timeout

            state.operations[operationId] = {
                startTime: Date.now(),
                timeoutId,
            };
        },
        finishOperation: (state, action) => {
            const operationId = action.payload;
            const operation = state.operations[operationId];

            if (operation?.timeoutId) {
                clearTimeout(operation.timeoutId);
            }

            delete state.operations[operationId];
        },
    },
});

export const operationTimeout = createAction('loading/operationTimeout');

// Middleware to track operations
const operationTrackingMiddleware = (store: any) => (next: any) => (action: any) => {
    if (action.type?.endsWith('/pending')) {
        const operationId = action.type.replace('/pending', '');
        store.dispatch(startOperation(operationId));
    }

    if (action.type?.endsWith('/fulfilled') || action.type?.endsWith('/rejected')) {
        const operationId = action.type.replace('/fulfilled', '').replace('/rejected', '');
        store.dispatch(finishOperation(operationId));
    }

    return next(action);
};
```

## Component Patterns

### 1. **Loading HOC (Higher-Order Component)**

```typescript
interface WithLoadingProps {
    loading: boolean;
    error: string | null;
}

function withLoading<P extends object>(
    WrappedComponent: React.ComponentType<P & WithLoadingProps>,
    loadingSelector: (state: RootState) => boolean,
    errorSelector: (state: RootState) => string | null
) {
    return function WithLoadingComponent(props: P) {
        const loading = useSelector(loadingSelector);
        const error = useSelector(errorSelector);

        if (loading) {
            return <div>Loading...</div>;
        }

        if (error) {
            return <div>Error: {error}</div>;
        }

        return <WrappedComponent {...props} loading={loading} error={error} />;
    };
}

// Usage
const UsersListWithLoading = withLoading(
    UsersList,
    (state) => state.users.loading,
    (state) => state.users.error
);
```

### 2. **Loading Hook**

```typescript
function useLoadingState(operationType?: string) {
    return useSelector((state: RootState) => {
        if (operationType) {
            return {
                loading: state.loading.operations[operationType]?.loading || false,
                error: state.loading.operations[operationType]?.error || null,
            };
        }

        return {
            loading: state.loading.globalLoading,
            error: state.loading.globalError,
        };
    });
}

// Usage
function MyComponent() {
    const { loading, error } = useLoadingState('fetchUsers');

    if (loading) return <div>Loading users...</div>;
    if (error) return <div>Error: {error}</div>;

    return <div>Content loaded</div>;
}
```

### 3. **Skeleton Loading**

```typescript
function UserSkeleton() {
    return (
        <div className="user-skeleton">
            <div className="skeleton avatar"></div>
            <div className="skeleton text"></div>
            <div className="skeleton text short"></div>
        </div>
    );
}

function UsersList() {
    const { data: users, loading } = useSelector((state: RootState) => state.users);

    if (loading && users.length === 0) {
        return (
            <div>
                {Array.from({ length: 5 }).map((_, i) => (
                    <UserSkeleton key={i} />
                ))}
            </div>
        );
    }

    return (
        <div>
            {users.map(user => (
                <UserItem key={user.id} user={user} />
            ))}
        </div>
    );
}
```

## Best Practices

### 1. **Consistent Loading Patterns**

```typescript
// ✅ Good: Consistent loading state structure
interface LoadingState {
    status: 'idle' | 'pending' | 'fulfilled' | 'rejected';
    error: string | null;
}

// ❌ Avoid: Inconsistent loading states
// Sometimes loading: boolean
// Sometimes status: string
// Sometimes isLoading: boolean
```

### 2. **Avoid Over-fetching**

```typescript
// ✅ Good: Check if data already exists
.addCase(fetchUsers.pending, (state) => {
    if (state.data.length === 0) {
        state.loading = true;
    }
});

// ✅ Good: Cache with TTL
interface CachedData<T> {
    data: T;
    timestamp: number;
    ttl: number;
}

const isExpired = (cached: CachedData) =>
    Date.now() - cached.timestamp > cached.ttl;
```

### 3. **Loading State Cleanup**

```typescript
// ✅ Good: Clear loading states on unmount
useEffect(() => {
    dispatch(fetchUsers());

    return () => {
        // Clear loading state if component unmounts
        dispatch(clearLoading());
    };
}, [dispatch]);
```

### 4. **User Feedback**

```typescript
// ✅ Good: Provide clear feedback
function LoadingButton({ loading, children, ...props }) {
    return (
        <button disabled={loading} {...props}>
            {loading ? (
                <>
                    <Spinner size="small" />
                    Loading...
                </>
            ) : (
                children
            )}
        </button>
    );
}
```

## Testing Loading States

```typescript
describe('usersSlice loading states', () => {
    it('sets loading to true when fetchUsers is pending', () => {
        const state = usersSlice.reducer(
            { ...initialState, loading: false },
            fetchUsers.pending()
        );

        expect(state.loading).toBe(true);
    });

    it('sets loading to false when fetchUsers is fulfilled', () => {
        const users = [{ id: 1, name: 'John' }];
        const state = usersSlice.reducer(
            { ...initialState, loading: true },
            fetchUsers.fulfilled(users)
        );

        expect(state.loading).toBe(false);
        expect(state.data).toEqual(users);
    });

    it('handles multiple loading states correctly', () => {
        let state = usersSlice.reducer(initialState, createUser.pending());

        expect(state.loading.create).toBe(true);

        state = usersSlice.reducer(state, createUser.fulfilled({ id: 1, name: 'John' }));

        expect(state.loading.create).toBe(false);
        expect(state.data).toContainEqual({ id: 1, name: 'John' });
    });
});
```

## Common Interview Questions

### Q: How do you handle loading states in Redux Toolkit?

**A:** I use the `pending`, `fulfilled`, and `rejected` cases in `extraReducers` to update loading states. For simple cases, I use a boolean `loading` property. For complex apps, I use objects with specific loading states for different operations.

### Q: What's the difference between global and granular loading states?

**A:** Global loading shows a general loading indicator for the entire app. Granular loading allows specific parts of the UI to show loading states independently, providing better user experience by not blocking the entire interface.

### Q: How do you handle loading states for optimistic updates?

**A:** I apply the optimistic update immediately, show a loading indicator, and either confirm the update on success or revert it on failure. This provides instant feedback while handling potential errors.

### Q: What are some common loading state anti-patterns?

**A:** Over-using global loading that blocks the entire UI, not handling loading state cleanup on component unmount, and not providing proper user feedback during loading operations.

## Summary

**Key Loading State Patterns:**
1. **Simple Boolean**: Single loading state for basic operations
2. **Granular Objects**: Separate loading states for different operations
3. **Entity-Specific**: Loading states tied to specific items
4. **Global Tracking**: App-wide loading indicators
5. **Optimistic Updates**: Immediate UI updates with loading feedback

**Best Practices:**
- Use consistent loading state structures
- Provide appropriate user feedback
- Clean up loading states properly
- Handle timeouts and error states
- Test loading state transitions

**Interview Tip:** "In Redux Toolkit, I handle loading states using the `pending`, `fulfilled`, and `rejected` actions from `createAsyncThunk`. For complex apps, I use granular loading objects to track different operations independently, ensuring smooth user experience without blocking the entire UI."