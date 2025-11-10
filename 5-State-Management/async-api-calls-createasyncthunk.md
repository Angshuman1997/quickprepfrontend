# How to use createAsyncThunk for API calls?

## Question
How to use createAsyncThunk for API calls?

## Answer
createAsyncThunk automatically creates actions for async operations. It generates pending, fulfilled, and rejected actions.

## Basic Usage
```javascript
// Create async thunk
const fetchUsers = createAsyncThunk(
  'users/fetchUsers',
  async () => {
    const response = await fetch('/api/users');
    if (!response.ok) throw new Error('Failed to fetch');
    return response.json(); // This becomes the payload
  }
);
```

## In Slice
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

**Q: What does createAsyncThunk do?**

A: Creates three actions automatically: pending (started), fulfilled (success), rejected (error) for async operations.

**Q: How do you handle errors in createAsyncThunk?**

A: Throw error in async function, then handle in rejected case in extraReducers.

**Q: What's the benefit over manual action creators?**

A: Less boilerplate code, consistent patterns, automatic action type generation.

const initialState: UsersState = {
    data: [],
    loading: false,
    error: null,
    pagination: null
};

const usersSlice = createSlice({
    name: 'users',
    initialState,
    reducers: {
        // Regular synchronous reducers
        clearUsers: (state) => {
            state.data = [];
            state.error = null;
        },
        resetUsersState: () => initialState
    },
    extraReducers: (builder) => {
        builder
            // Pending state
            .addCase(fetchUsers.pending, (state) => {
                state.loading = true;
                state.error = null;
            })
            // Fulfilled state
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.loading = false;
                state.data = action.payload.users;
                state.pagination = action.payload.pagination;
            })
            // Rejected state
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload as string || 'Failed to fetch users';
            });
    }
});

export const { clearUsers, resetUsersState } = usersSlice.actions;
export default usersSlice.reducer;
```

### 3. **Using in Components**

```typescript
import React, { useEffect } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { fetchUsers, clearUsers } from './usersSlice';
import type { RootState } from './store';

function UsersList() {
    const dispatch = useDispatch();
    const { data: users, loading, error, pagination } = useSelector(
        (state: RootState) => state.users
    );

    useEffect(() => {
        dispatch(fetchUsers({ page: 1, limit: 10 }));
    }, [dispatch]);

    const handleLoadMore = () => {
        if (pagination?.hasMore) {
            dispatch(fetchUsers({
                page: (pagination.page || 1) + 1,
                limit: 10
            }));
        }
    };

    const handleRefresh = () => {
        dispatch(fetchUsers({ page: 1, limit: 10 }));
    };

    if (loading && users.length === 0) {
        return <div>Loading users...</div>;
    }

    if (error) {
        return (
            <div>
                <p>Error: {error}</p>
                <button onClick={handleRefresh}>Try Again</button>
            </div>
        );
    }

    return (
        <div>
            <button onClick={handleRefresh}>Refresh</button>
            <button onClick={() => dispatch(clearUsers())}>Clear</button>

            <ul>
                {users.map(user => (
                    <li key={user.id}>{user.name} - {user.email}</li>
                ))}
            </ul>

            {loading && <div>Loading more...</div>}

            {pagination?.hasMore && !loading && (
                <button onClick={handleLoadMore}>Load More</button>
            )}
        </div>
    );
}
```

## Advanced Patterns

### 1. **Conditional Dispatch and Thunk Arguments**

```typescript
// Thunk with conditional logic
export const updateUser = createAsyncThunk(
    'users/updateUser',
    async ({ id, updates }: { id: number; updates: Partial<User> }, { getState, rejectWithValue }) => {
        const state = getState() as RootState;
        const currentUser = state.users.data.find(user => user.id === id);

        // Optimistic check
        if (!currentUser) {
            return rejectWithValue('User not found');
        }

        try {
            const response = await fetch(`/api/users/${id}`, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(updates)
            });

            if (!response.ok) {
                throw new Error('Failed to update user');
            }

            return await response.json();
        } catch (error) {
            return rejectWithValue(error instanceof Error ? error.message : 'Update failed');
        }
    }
);

// Thunk with dependencies
export const createUserWithValidation = createAsyncThunk(
    'users/createUser',
    async (userData: Omit<User, 'id'>, { dispatch, rejectWithValue }) => {
        try {
            // First validate
            const validationResult = await dispatch(validateUser(userData));
            if (validationResult.error) {
                return rejectWithValue(validationResult.error);
            }

            // Then create
            const response = await fetch('/api/users', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(userData)
            });

            if (!response.ok) {
                throw new Error('Failed to create user');
            }

            const newUser = await response.json();

            // Optionally dispatch another action
            dispatch(fetchUsers()); // Refresh the list

            return newUser;
        } catch (error) {
            return rejectWithValue(error instanceof Error ? error.message : 'Creation failed');
        }
    }
);
```

### 2. **Error Handling and Retry Logic**

```typescript
// Thunk with retry logic
export const fetchUserWithRetry = createAsyncThunk(
    'users/fetchUserWithRetry',
    async (userId: number, { rejectWithValue }) => {
        const maxRetries = 3;
        let lastError: Error | null = null;

        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                const response = await fetch(`/api/users/${userId}`);

                if (!response.ok) {
                    if (response.status >= 500 && attempt < maxRetries) {
                        // Retry on server errors
                        await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
                        continue;
                    }
                    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
                }

                return await response.json();
            } catch (error) {
                lastError = error instanceof Error ? error : new Error('Network error');

                if (attempt === maxRetries) {
                    break;
                }

                // Exponential backoff
                await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, attempt - 1)));
            }
        }

        return rejectWithValue(lastError?.message || 'Failed after retries');
    }
);

// Enhanced slice with retry state
interface UsersState {
    data: User[];
    loading: boolean;
    error: string | null;
    retryCount: number;
    lastFetchAttempt: number | null;
}

const usersSlice = createSlice({
    name: 'users',
    initialState: {
        data: [],
        loading: false,
        error: null,
        retryCount: 0,
        lastFetchAttempt: null
    } as UsersState,
    reducers: {
        resetRetryCount: (state) => {
            state.retryCount = 0;
        }
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchUserWithRetry.pending, (state) => {
                state.loading = true;
                state.error = null;
                state.lastFetchAttempt = Date.now();
            })
            .addCase(fetchUserWithRetry.fulfilled, (state, action) => {
                state.loading = false;
                state.data = [action.payload];
                state.retryCount = 0;
            })
            .addCase(fetchUserWithRetry.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload as string;
                state.retryCount += 1;
            });
    }
});
```

### 3. **Optimistic Updates**

```typescript
// Optimistic update thunk
export const deleteUserOptimistic = createAsyncThunk(
    'users/deleteUserOptimistic',
    async (userId: number, { dispatch, getState, rejectWithValue }) => {
        const state = getState() as RootState;
        const userToDelete = state.users.data.find(user => user.id === userId);

        if (!userToDelete) {
            return rejectWithValue('User not found');
        }

        // Optimistically remove from state
        dispatch(removeUserOptimistic(userId));

        try {
            const response = await fetch(`/api/users/${userId}`, {
                method: 'DELETE'
            });

            if (!response.ok) {
                // Revert optimistic update on failure
                dispatch(addUserBack(userToDelete));
                throw new Error('Failed to delete user');
            }

            return userId; // Return the deleted ID for confirmation
        } catch (error) {
            // Revert optimistic update
            dispatch(addUserBack(userToDelete));
            return rejectWithValue(error instanceof Error ? error.message : 'Delete failed');
        }
    }
);

// Slice with optimistic update actions
const usersSlice = createSlice({
    name: 'users',
    initialState,
    reducers: {
        removeUserOptimistic: (state, action) => {
            state.data = state.data.filter(user => user.id !== action.payload);
        },
        addUserBack: (state, action) => {
            state.data.push(action.payload);
        }
    },
    extraReducers: (builder) => {
        builder
            .addCase(deleteUserOptimistic.fulfilled, (state, action) => {
                // Optimistic update succeeded, no need to change state
                console.log(`User ${action.payload} deleted successfully`);
            })
            .addCase(deleteUserOptimistic.rejected, (state, action) => {
                // State already reverted in the thunk, just update error
                state.error = action.payload as string;
            });
    }
});
```

## Real-World Examples

### 1. **Authentication Flow**

```typescript
export const loginUser = createAsyncThunk(
    'auth/login',
    async (credentials: { email: string; password: string }, { rejectWithValue }) => {
        try {
            const response = await fetch('/api/auth/login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(credentials)
            });

            const data = await response.json();

            if (!response.ok) {
                return rejectWithValue(data.message || 'Login failed');
            }

            // Store token in localStorage
            localStorage.setItem('token', data.token);

            return data.user;
        } catch (error) {
            return rejectWithValue('Network error during login');
        }
    }
);

export const logoutUser = createAsyncThunk(
    'auth/logout',
    async (_, { dispatch }) => {
        try {
            await fetch('/api/auth/logout', {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${localStorage.getItem('token')}`
                }
            });
        } catch (error) {
            // Even if logout fails, we should clear local state
            console.warn('Logout API call failed:', error);
        } finally {
            // Always clear local storage and reset state
            localStorage.removeItem('token');
            dispatch(clearUserData());
        }
    }
);
```

### 2. **File Upload with Progress**

```typescript
export const uploadFile = createAsyncThunk(
    'files/upload',
    async ({ file, onProgress }: { file: File; onProgress?: (progress: number) => void }, { rejectWithValue }) => {
        try {
            const formData = new FormData();
            formData.append('file', file);

            const xhr = new XMLHttpRequest();

            // Set up progress tracking
            if (onProgress) {
                xhr.upload.addEventListener('progress', (event) => {
                    if (event.lengthComputable) {
                        const progress = (event.loaded / event.total) * 100;
                        onProgress(progress);
                    }
                });
            }

            // Wrap XHR in a Promise
            const response = await new Promise<{ success: boolean; data?: any; error?: string }>((resolve) => {
                xhr.open('POST', '/api/files/upload');

                xhr.onload = () => {
                    try {
                        const data = JSON.parse(xhr.responseText);
                        resolve({
                            success: xhr.status >= 200 && xhr.status < 300,
                            data,
                            error: xhr.status >= 400 ? data.message : undefined
                        });
                    } catch (error) {
                        resolve({
                            success: false,
                            error: 'Invalid response format'
                        });
                    }
                };

                xhr.onerror = () => {
                    resolve({
                        success: false,
                        error: 'Network error'
                    });
                };

                xhr.send(formData);
            });

            if (!response.success) {
                return rejectWithValue(response.error);
            }

            return response.data;
        } catch (error) {
            return rejectWithValue(error instanceof Error ? error.message : 'Upload failed');
        }
    }
);
```

### 3. **Batch Operations**

```typescript
export const batchUpdateUsers = createAsyncThunk(
    'users/batchUpdate',
    async (updates: Array<{ id: number; changes: Partial<User> }>, { rejectWithValue }) => {
        try {
            const response = await fetch('/api/users/batch', {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ updates })
            });

            if (!response.ok) {
                throw new Error('Batch update failed');
            }

            const result = await response.json();

            // Return both successful updates and failures
            return {
                successful: result.successful || [],
                failed: result.failed || []
            };
        } catch (error) {
            return rejectWithValue(error instanceof Error ? error.message : 'Batch update failed');
        }
    }
);

// Handle partial failures in slice
const usersSlice = createSlice({
    name: 'users',
    initialState: {
        ...initialState,
        batchResults: null as { successful: User[]; failed: any[] } | null
    },
    reducers: {
        clearBatchResults: (state) => {
            state.batchResults = null;
        }
    },
    extraReducers: (builder) => {
        builder
            .addCase(batchUpdateUsers.fulfilled, (state, action) => {
                state.loading = false;

                // Update successful changes
                action.payload.successful.forEach(updatedUser => {
                    const index = state.data.findIndex(user => user.id === updatedUser.id);
                    if (index !== -1) {
                        state.data[index] = { ...state.data[index], ...updatedUser };
                    }
                });

                // Store batch results for UI feedback
                state.batchResults = action.payload;
            });
    }
});
```

## Best Practices

### 1. **Consistent Error Handling**

```typescript
// ✅ Good: Consistent error structure
.addCase(fetchUsers.rejected, (state, action) => {
    state.loading = false;
    state.error = action.payload || action.error.message || 'An error occurred';
});

// ❌ Avoid: Inconsistent error handling
.addCase(fetchUsers.rejected, (state, action) => {
    if (action.payload) {
        state.error = action.payload;
    } else if (action.error) {
        state.error = action.error.message;
    } else {
        state.error = 'Something went wrong';
    }
});
```

### 2. **Loading States**

```typescript
// ✅ Good: Handle different loading states
interface UsersState {
    loading: {
        fetch: boolean;
        create: boolean;
        update: boolean;
        delete: boolean;
    };
    // ... other state
}

// ✅ Good: Use loading state in components
const isLoading = useSelector(state => state.users.loading.fetch);
```

### 3. **Thunk Naming Convention**

```typescript
// ✅ Good: Descriptive names with feature prefix
export const fetchUsers = createAsyncThunk('users/fetchUsers', ...);
export const createUser = createAsyncThunk('users/createUser', ...);
export const updateUser = createAsyncThunk('users/updateUser', ...);

// ❌ Avoid: Generic names
export const getData = createAsyncThunk('data/get', ...);
export const postData = createAsyncThunk('data/post', ...);
```

### 4. **Type Safety**

```typescript
// ✅ Good: Properly typed thunks
export const fetchUsers = createAsyncThunk<
    User[], // Return type
    { page?: number; limit?: number }, // First argument type
    { rejectValue: string } // Thunk config
>('users/fetchUsers', async (params, { rejectWithValue }) => {
    // Implementation
});
```

## Common Interview Questions

### Q: What's the difference between createAsyncThunk and regular async actions?

**A:** `createAsyncThunk` automatically generates three action types (pending, fulfilled, rejected) and handles the async logic, error handling, and dispatch. Regular async actions require manual creation of all action types and handling of the async flow.

### Q: How do you handle loading states with multiple async operations?

**A:** Use an object for loading states instead of a boolean: `loading: { fetch: boolean, create: boolean, update: boolean }`. This allows tracking multiple concurrent operations.

### Q: What happens if a component unmounts during an async operation?

**A:** The thunk will still complete, but you should check if the component is still mounted before updating state. You can use `AbortController` or cleanup functions in useEffect to cancel operations.

### Q: How do you test createAsyncThunk?

**A:** Test the thunk function directly by mocking the API calls, and test the slice reducers by dispatching the generated actions. Use `jest.mock()` for API calls and test the state changes.

## Summary

**Key Benefits of createAsyncThunk:**
- Automatic action type generation
- Built-in error handling with `rejectWithValue`
- Consistent loading/error state management
- Type-safe with TypeScript
- Supports complex async patterns (retry, optimistic updates)

**Basic Pattern:**
1. Create thunk with `createAsyncThunk(type, asyncFunction)`
2. Handle the three states in `extraReducers`
3. Dispatch from components using `useDispatch`
4. Select state using `useSelector`

**Advanced Patterns:**
- Conditional dispatch with thunk arguments
- Retry logic with exponential backoff
- Optimistic updates with rollback
- Progress tracking for uploads
- Batch operations with partial failures

**Interview Tip:** "`createAsyncThunk` simplifies async Redux logic by automatically generating pending/fulfilled/rejected actions. I use it for all API calls, handle errors with `rejectWithValue`, and implement patterns like optimistic updates and retry logic for robust user experiences."