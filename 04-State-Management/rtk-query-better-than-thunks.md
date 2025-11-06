# What is RTK Query? When is it better than Thunks?

## Question
What is RTK Query? When is it better than Thunks?

## Answer

RTK Query is a powerful data fetching and caching library built on top of Redux Toolkit. It provides a declarative way to manage server state, with automatic caching, background refetching, and optimistic updates. While thunks are great for one-off async logic, RTK Query excels at managing API interactions and server state.

## What is RTK Query?

RTK Query is a data fetching and state management library that:

- **Automatically manages server state** in Redux
- **Provides caching and background updates**
- **Generates React hooks** for data fetching
- **Handles loading/error states** automatically
- **Supports optimistic updates** and cache invalidation
- **Integrates seamlessly** with Redux Toolkit

## RTK Query vs Thunks Comparison

### 1. **Basic Data Fetching**

**With Thunks:**
```typescript
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchUsers = createAsyncThunk(
    'users/fetchUsers',
    async (_, { rejectWithValue }) => {
        try {
            const response = await fetch('/api/users');
            if (!response.ok) throw new Error('Failed to fetch');
            return await response.json();
        } catch (error) {
            return rejectWithValue(error.message);
        }
    }
);

const usersSlice = createSlice({
    name: 'users',
    initialState: { users: [], loading: false, error: null },
    reducers: {},
    extraReducers: (builder) => {
        builder
            .addCase(fetchUsers.pending, (state) => {
                state.loading = true;
            })
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.loading = false;
                state.users = action.payload;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload;
            });
    }
});

// Component usage
function UsersList() {
    const dispatch = useDispatch();
    const { users, loading, error } = useSelector(state => state.users);

    useEffect(() => {
        dispatch(fetchUsers());
    }, [dispatch]);

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <ul>
            {users.map(user => <li key={user.id}>{user.name}</li>)}
        </ul>
    );
}
```

**With RTK Query:**
```typescript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getUsers: builder.query({
            query: () => 'users',
        }),
    }),
});

export const { useGetUsersQuery } = api;

// Component usage
function UsersList() {
    const { data: users, isLoading, error } = useGetUsersQuery();

    if (isLoading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.data?.message || 'Error'}</div>;

    return (
        <ul>
            {users?.map(user => <li key={user.id}>{user.name}</li>)}
        </ul>
    );
}
```

### 2. **Caching and Background Updates**

**Thunks - No automatic caching:**
```typescript
// Each component needs to manage its own loading state
function UserProfile({ userId }) {
    const dispatch = useDispatch();
    const user = useSelector(state => state.users.entities[userId]);
    const loading = useSelector(state => state.users.loadingById[userId]);

    useEffect(() => {
        if (!user && !loading) {
            dispatch(fetchUserById(userId));
        }
    }, [userId, user, loading, dispatch]);

    // Manual cache management...
}
```

**RTK Query - Automatic caching:**
```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getUser: builder.query({
            query: (id) => `users/${id}`,
        }),
    }),
});

// Cache automatically managed
function UserProfile({ userId }) {
    const { data: user, isLoading } = api.useGetUserQuery(userId);
    // Data is cached and shared across components
}

function UserList() {
    const { data: users } = api.useGetUsersQuery();
    // Same cached data, no duplicate requests
}
```

### 3. **Mutations and Optimistic Updates**

**Thunks - Manual optimistic updates:**
```typescript
export const updateUser = createAsyncThunk(
    'users/updateUser',
    async ({ id, updates }, { rejectWithValue }) => {
        // Manual optimistic update logic
        const response = await fetch(`/api/users/${id}`, {
            method: 'PATCH',
            body: JSON.stringify(updates),
        });
        return response.json();
    }
);

// In component
const [optimisticUser, setOptimisticUser] = useState(null);

const handleUpdate = async (updates) => {
    setOptimisticUser({ ...user, ...updates });
    try {
        await dispatch(updateUser({ id: user.id, updates })).unwrap();
        setOptimisticUser(null);
    } catch (error) {
        setOptimisticUser(null);
        // Handle error
    }
};
```

**RTK Query - Built-in optimistic updates:**
```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        updateUser: builder.mutation({
            query: ({ id, ...updates }) => ({
                url: `users/${id}`,
                method: 'PATCH',
                body: updates,
            }),
            // Automatic optimistic updates
            optimisticUpdate: {
                users: (draft, { id, ...updates }) => {
                    const user = draft.find(u => u.id === id);
                    if (user) Object.assign(user, updates);
                },
            },
            // Automatic cache invalidation
            invalidatesTags: ['User'],
        }),
    }),
});

// Component usage
function UserEditor({ user }) {
    const [updateUser, { isLoading }] = api.useUpdateUserMutation();

    const handleSave = () => {
        updateUser({ id: user.id, name: 'New Name' });
        // Optimistic update happens automatically
    };
}
```

## When RTK Query is Better Than Thunks

### 1. **Server State Management**

**Use RTK Query when:**
- Managing data from REST APIs or GraphQL
- Need automatic caching and background refetching
- Want to share data across multiple components
- Require optimistic updates for mutations
- Need automatic cache invalidation

**Example - Automatic cache management:**
```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    tagTypes: ['Users', 'Posts'],
    endpoints: (builder) => ({
        getUsers: builder.query({
            query: () => 'users',
            providesTags: ['Users'], // Cache tag
        }),
        addUser: builder.mutation({
            query: (user) => ({
                url: 'users',
                method: 'POST',
                body: user,
            }),
            invalidatesTags: ['Users'], // Invalidate cache
        }),
        updateUser: builder.mutation({
            query: ({ id, ...user }) => ({
                url: `users/${id}`,
                method: 'PATCH',
                body: user,
            }),
            invalidatesTags: (result, error, { id }) => [
                { type: 'Users', id }
            ],
        }),
    }),
});

// Cache automatically updated when mutations occur
function App() {
    const { data: users } = api.useGetUsersQuery();

    return (
        <div>
            <UserList users={users} />
            <AddUserForm /> {/* Triggers cache invalidation */}
        </div>
    );
}
```

### 2. **Background Refetching and Polling**

**RTK Query automatic refetching:**
```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getStockPrice: builder.query({
            query: (symbol) => `stocks/${symbol}`,
            // Automatic refetching
            refetchOnMountOrArgChange: true,
            // Polling
            pollingInterval: 5000, // Refetch every 5 seconds
        }),
        getWeather: builder.query({
            query: (location) => `weather/${location}`,
            // Refetch on window focus
            refetchOnFocus: true,
            // Refetch on reconnect
            refetchOnReconnect: true,
        }),
    }),
});

// Data stays fresh automatically
function StockTicker({ symbol }) {
    const { data, isLoading } = api.useGetStockPriceQuery(symbol);
    // No manual polling logic needed
}
```

### 3. **Complex Data Relationships**

**RTK Query with normalized caching:**
```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getPosts: builder.query({
            query: () => 'posts',
            providesTags: (result) =>
                result
                    ? [
                        ...result.map(({ id }) => ({ type: 'Posts', id })),
                        { type: 'Posts', id: 'LIST' },
                    ]
                    : [{ type: 'Posts', id: 'LIST' }],
        }),
        getPost: builder.query({
            query: (id) => `posts/${id}`,
            providesTags: (result, error, id) => [{ type: 'Posts', id }],
        }),
        addComment: builder.mutation({
            query: ({ postId, comment }) => ({
                url: `posts/${postId}/comments`,
                method: 'POST',
                body: comment,
            }),
            // Invalidate specific post and comment list
            invalidatesTags: (result, error, { postId }) => [
                { type: 'Posts', id: postId },
                { type: 'Comments', id: 'LIST' },
            ],
        }),
    }),
});

// Efficient caching with relationships
function PostDetail({ postId }) {
    const { data: post } = api.useGetPostQuery(postId);
    // Cached data reused across components
}
```

### 4. **Real-time Updates with WebSockets**

**RTK Query with streaming updates:**
```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getMessages: builder.query({
            query: (channelId) => `channels/${channelId}/messages`,
            async onCacheEntryAdded(
                arg,
                { updateCachedData, cacheDataLoaded, cacheEntryRemoved }
            ) {
                // WebSocket connection for real-time updates
                const ws = new WebSocket(`ws://localhost:8080/channel/${arg}`);

                try {
                    await cacheDataLoaded;

                    ws.onmessage = (event) => {
                        const message = JSON.parse(event.data);
                        // Update cache with new message
                        updateCachedData((draft) => {
                            draft.push(message);
                        });
                    };
                } catch {
                    // Cache entry was removed
                }

                await cacheEntryRemoved;
                ws.close();
            },
        }),
    }),
});

// Real-time chat with automatic cache updates
function ChatRoom({ channelId }) {
    const { data: messages } = api.useGetMessagesQuery(channelId);
    // Messages update in real-time via WebSocket
}
```

## When Thunks are Better Than RTK Query

### 1. **One-off Async Logic**

**Use thunks for:**
- Form submissions
- Navigation after async operations
- Complex multi-step workflows
- Operations that don't need caching

```typescript
// Thunk for complex workflow
export const submitOrder = createAsyncThunk(
    'orders/submit',
    async (orderData, { dispatch, rejectWithValue }) => {
        try {
            // Validate order
            const validation = await dispatch(validateOrder(orderData)).unwrap();

            // Process payment
            const payment = await dispatch(processPayment(validation)).unwrap();

            // Create order
            const order = await dispatch(createOrder({ ...orderData, paymentId: payment.id })).unwrap();

            // Send confirmation email
            await dispatch(sendConfirmationEmail(order.id));

            // Navigate to success page
            history.push(`/orders/${order.id}/success`);

            return order;
        } catch (error) {
            return rejectWithValue(error.message);
        }
    }
);
```

### 2. **Complex Business Logic**

**Thunks excel at:**
- Coordinating multiple actions
- Complex error handling with retries
- Integration with external services
- Side effects beyond data fetching

```typescript
// Complex thunk with retry logic and side effects
export const processRefund = createAsyncThunk(
    'payments/processRefund',
    async ({ paymentId, amount }, { dispatch, rejectWithValue }) => {
        let attempts = 0;
        const maxAttempts = 3;

        while (attempts < maxAttempts) {
            try {
                // Process refund
                const refund = await api.processRefund(paymentId, amount);

                // Update payment status
                await dispatch(updatePaymentStatus({
                    id: paymentId,
                    status: 'refunded',
                    refundId: refund.id
                }));

                // Log audit trail
                await dispatch(logAuditEvent({
                    action: 'refund_processed',
                    paymentId,
                    amount,
                    refundId: refund.id
                }));

                // Send notification
                await dispatch(sendRefundNotification(refund));

                return refund;
            } catch (error) {
                attempts++;

                if (attempts === maxAttempts) {
                    // Log failure
                    await dispatch(logAuditEvent({
                        action: 'refund_failed',
                        paymentId,
                        amount,
                        error: error.message
                    }));

                    return rejectWithValue('Refund processing failed');
                }

                // Wait before retry
                await new Promise(resolve => setTimeout(resolve, 1000 * attempts));
            }
        }
    }
);
```

### 3. **Non-API Async Operations**

**Thunks for:**
- File uploads with progress tracking
- WebSocket connections
- Local storage operations
- Timer-based operations

```typescript
// File upload with progress
export const uploadFile = createAsyncThunk(
    'files/upload',
    async ({ file, onProgress }, { dispatch, rejectWithValue }) => {
        const formData = new FormData();
        formData.append('file', file);

        return new Promise((resolve, reject) => {
            const xhr = new XMLHttpRequest();

            xhr.upload.onprogress = (event) => {
                if (event.lengthComputable) {
                    const progress = (event.loaded / event.total) * 100;
                    onProgress(progress);
                }
            };

            xhr.onload = () => {
                if (xhr.status === 200) {
                    resolve(JSON.parse(xhr.response));
                } else {
                    reject(new Error('Upload failed'));
                }
            };

            xhr.onerror = () => reject(new Error('Network error'));

            xhr.open('POST', '/api/upload');
            xhr.send(formData);
        });
    }
);
```

## RTK Query Advanced Features

### 1. **Query Lifecycle Management**

```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getData: builder.query({
            query: (params) => ({
                url: 'data',
                params,
            }),
            // Lifecycle hooks
            onQueryStarted: async (arg, { queryFulfilled, dispatch }) => {
                // Called when query starts
                dispatch(showLoadingIndicator());
                try {
                    const result = await queryFulfilled;
                    dispatch(hideLoadingIndicator());
                    // Query succeeded
                } catch (error) {
                    dispatch(hideLoadingIndicator());
                    dispatch(showErrorNotification(error));
                }
            },
            // Transform response
            transformResponse: (response) => {
                return response.data.map(item => ({
                    ...item,
                    createdAt: new Date(item.createdAt),
                }));
            },
            // Transform error
            transformErrorResponse: (error) => {
                return {
                    message: error.data?.message || 'An error occurred',
                    status: error.status,
                };
            },
        }),
    }),
});
```

### 2. **Advanced Caching Strategies**

```typescript
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getPosts: builder.query({
            query: ({ page, limit }) => ({
                url: 'posts',
                params: { page, limit },
            }),
            // Serialize query args for caching
            serializeQueryArgs: ({ queryArgs }) => {
                // Cache based on page only, ignore limit
                return { page: queryArgs.page };
            },
            // Merge incoming data with existing cache
            merge: (currentCache, newItems) => {
                currentCache.posts.push(...newItems.posts);
            },
            // Determine if we have all data
            forceRefetch: ({ currentArg, previousArg }) => {
                return currentArg?.page !== previousArg?.page;
            },
        }),
        // Prefetch related data
        getPostWithComments: builder.query({
            query: (id) => `posts/${id}`,
            transformResponse: async (post, meta, arg) => {
                // Prefetch comments
                const comments = await meta.dispatch(
                    api.endpoints.getComments.initiate(arg)
                ).unwrap();

                return { ...post, comments };
            },
        }),
    }),
});
```

### 3. **Custom Base Query**

```typescript
// Custom base query with authentication and error handling
const baseQuery = fetchBaseQuery({
    baseUrl: '/api',
    prepareHeaders: (headers, { getState }) => {
        const token = (getState() as RootState).auth.token;
        if (token) {
            headers.set('authorization', `Bearer ${token}`);
        }
        return headers;
    },
});

// Enhanced base query with retry logic
const baseQueryWithRetry = async (args, api, extraOptions) => {
    let result = await baseQuery(args, api, extraOptions);

    if (result.error?.status === 401) {
        // Token expired, try to refresh
        const refreshResult = await baseQuery(
            { url: 'auth/refresh', method: 'POST' },
            api,
            extraOptions
        );

        if (refreshResult.data) {
            // Retry original request with new token
            result = await baseQuery(args, api, extraOptions);
        }
    }

    return result;
};

const api = createApi({
    baseQuery: baseQueryWithRetry,
    endpoints: (builder) => ({
        // endpoints
    }),
});
```

## Performance Comparison

| Feature | RTK Query | Thunks |
|---------|-----------|--------|
| **Caching** | Automatic, sophisticated | Manual or none |
| **Background Updates** | Built-in | Manual implementation |
| **Optimistic Updates** | Declarative configuration | Manual state management |
| **Loading States** | Auto-generated per query | Manual per operation |
| **TypeScript Support** | Excellent auto-generated types | Manual typing required |
| **Boilerplate** | Minimal | Moderate to high |
| **Flexibility** | High for data fetching | Unlimited for any async logic |
| **Bundle Size** | Larger (~15kb gzipped) | Smaller (included in RTK) |
| **Learning Curve** | Moderate | Low |

## Migration Strategy

### 1. **Gradual Migration**

```typescript
// Start with RTK Query for new features
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        // New API endpoints
        getUsers: builder.query({ query: () => 'users' }),
    }),
});

// Keep existing thunks for complex logic
export const complexWorkflow = createAsyncThunk(
    'app/complexWorkflow',
    async (data, { dispatch }) => {
        // Complex business logic that doesn't fit RTK Query
        const result1 = await dispatch(api.endpoints.getUsers.initiate()).unwrap();
        // ... more complex logic
    }
);
```

### 2. **Hybrid Approach**

```typescript
// Use RTK Query for data fetching, thunks for orchestration
const api = createApi({
    baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
    endpoints: (builder) => ({
        getCart: builder.query({ query: () => 'cart' }),
        checkout: builder.mutation({
            query: (order) => ({ url: 'checkout', method: 'POST', body: order }),
        }),
    }),
});

// Thunk coordinates multiple operations
export const completePurchase = createAsyncThunk(
    'cart/completePurchase',
    async (_, { dispatch }) => {
        const cart = await dispatch(api.endpoints.getCart.initiate()).unwrap();

        // Validate cart
        if (cart.items.length === 0) {
            throw new Error('Cart is empty');
        }

        // Process payment (external service)
        const paymentResult = await processPayment(cart.total);

        // Complete checkout
        const order = await dispatch(api.endpoints.checkout.initiate({
            ...cart,
            paymentId: paymentResult.id,
        })).unwrap();

        // Clear cart and redirect
        dispatch(clearCart());
        history.push(`/orders/${order.id}`);
    }
);
```

## Common Interview Questions

### Q: When should you use RTK Query over thunks?

**A:** Use RTK Query for server state management when you need automatic caching, background refetching, and optimistic updates. Use thunks for one-off async operations, complex business logic, or operations that don't involve API data fetching.

### Q: How does RTK Query caching work?

**A:** RTK Query uses a normalized cache with tags. Queries "provide" tags for the data they fetch, and mutations can "invalidate" those tags to automatically refetch stale data. It also supports cache merging and background updates.

### Q: Can RTK Query replace all thunks?

**A:** No, RTK Query is specifically for data fetching and caching. Thunks are still needed for complex workflows, side effects, navigation, or any async logic that doesn't fit the query/mutation pattern.

### Q: What's the main benefit of RTK Query?

**A:** RTK Query eliminates the need to manually manage loading states, caching, and synchronization between server and client state. It provides hooks that handle all the complexity automatically while giving you fine-grained control when needed.

## Summary

**RTK Query is Better When:**
- Managing server state and API data
- Need automatic caching and synchronization
- Want optimistic updates and cache invalidation
- Sharing data across multiple components
- Require background refetching or polling
- Building data-heavy applications

**Thunks are Better When:**
- Complex multi-step business logic
- One-off async operations
- Coordinating multiple actions
- Side effects beyond data fetching
- File uploads or WebSocket connections
- Navigation or external service integration

**Key RTK Query Features:**
- Automatic caching with invalidation
- Background refetching and polling
- Optimistic updates
- Auto-generated React hooks
- TypeScript support
- Normalized state management
- Real-time updates

**Interview Tip:** "RTK Query is a data fetching layer built on Redux Toolkit that automatically handles caching, background updates, and optimistic updates. It's better than thunks when managing server state because it eliminates manual state management, but thunks are still valuable for complex business logic and side effects."