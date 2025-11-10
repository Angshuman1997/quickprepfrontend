# How do you create custom hooks for data fetching?

## Question
How do you create custom hooks for data fetching?

## Answer

Custom hooks for data fetching are a powerful pattern in React that allows you to extract and reuse data fetching logic across components. They follow the same rules as regular hooks and can encapsulate complex async operations, error handling, and state management.

## Basic Custom Hook Structure

### Simple Data Fetching Hook

```javascript
import { useState, useEffect } from 'react';

function useFetch(url) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        let isMounted = true;

        const fetchData = async () => {
            try {
                setLoading(true);
                setError(null);

                const response = await fetch(url);

                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }

                const result = await response.json();

                if (isMounted) {
                    setData(result);
                }
            } catch (err) {
                if (isMounted) {
                    setError(err.message);
                }
            } finally {
                if (isMounted) {
                    setLoading(false);
                }
            }
        };

        if (url) {
            fetchData();
        }

        // Cleanup function to prevent state updates on unmounted components
        return () => {
            isMounted = false;
        };
    }, [url]);

    return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
    const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <div>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
}
```

## Advanced Custom Hook with Refetch Capability

### Hook with Manual Refetch

```javascript
function useFetchWithRefetch(url, options = {}) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const fetchData = useCallback(async (fetchUrl = url) => {
        try {
            setLoading(true);
            setError(null);

            const response = await fetch(fetchUrl, options);

            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            const result = await response.json();
            setData(result);
            return result;
        } catch (err) {
            setError(err.message);
            throw err; // Re-throw for manual handling
        } finally {
            setLoading(false);
        }
    }, [url, options]);

    useEffect(() => {
        if (url) {
            fetchData();
        }
    }, [url, fetchData]);

    const refetch = useCallback(() => {
        return fetchData();
    }, [fetchData]);

    return { data, loading, error, refetch };
}

// Usage with refetch
function RefreshableData() {
    const { data, loading, error, refetch } = useFetchWithRefetch('/api/data');

    return (
        <div>
            {loading && <div>Loading...</div>}
            {error && <div>Error: {error}</div>}
            {data && <pre>{JSON.stringify(data, null, 2)}</pre>}

            <button onClick={refetch} disabled={loading}>
                Refresh Data
            </button>
        </div>
    );
}
```

## Custom Hook with Dependencies

### Hook that depends on multiple parameters

```javascript
function useApi(endpoint, params = {}, dependencies = []) {
    const [state, setState] = useState({
        data: null,
        loading: true,
        error: null
    });

    useEffect(() => {
        let isMounted = true;

        const fetchData = async () => {
            try {
                setState(prev => ({ ...prev, loading: true, error: null }));

                // Build query string from params
                const queryString = new URLSearchParams(params).toString();
                const url = queryString ? `${endpoint}?${queryString}` : endpoint;

                const response = await fetch(url);

                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }

                const result = await response.json();

                if (isMounted) {
                    setState({ data: result, loading: false, error: null });
                }
            } catch (err) {
                if (isMounted) {
                    setState({ data: null, loading: false, error: err.message });
                }
            }
        };

        fetchData();

        return () => {
            isMounted = false;
        };
    }, [endpoint, ...dependencies]); // Include dependencies in effect

    return state;
}

// Usage
function SearchResults({ query, category }) {
    const { data, loading, error } = useApi('/api/search', {
        q: query,
        category: category
    }, [query, category]); // Refetch when query or category changes

    if (loading) return <div>Searching...</div>;
    if (error) return <div>Search failed: {error}</div>;

    return (
        <div>
            <h3>Results for "{query}" in {category}</h3>
            {data?.results?.map(item => (
                <div key={item.id}>{item.title}</div>
            ))}
        </div>
    );
}
```

## Custom Hook with Caching

### Hook with built-in caching

```javascript
// Simple in-memory cache
const cache = new Map();

function useCachedFetch(url, ttl = 5 * 60 * 1000) { // 5 minutes default TTL
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        if (!url) return;

        const cached = cache.get(url);
        const now = Date.now();

        // Check if we have valid cached data
        if (cached && (now - cached.timestamp) < ttl) {
            setData(cached.data);
            setLoading(false);
            setError(null);
            return;
        }

        // Fetch new data
        let isMounted = true;

        const fetchData = async () => {
            try {
                setLoading(true);
                setError(null);

                const response = await fetch(url);
                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`);
                }

                const result = await response.json();

                // Cache the result
                cache.set(url, {
                    data: result,
                    timestamp: now
                });

                if (isMounted) {
                    setData(result);
                }
            } catch (err) {
                if (isMounted) {
                    setError(err.message);
                }
            } finally {
                if (isMounted) {
                    setLoading(false);
                }
            }
        };

        fetchData();

        return () => {
            isMounted = false;
        };
    }, [url, ttl]);

    const invalidateCache = useCallback(() => {
        cache.delete(url);
    }, [url]);

    return { data, loading, error, invalidateCache };
}

// Usage
function CachedUserList() {
    const { data: users, loading, error, invalidateCache } = useCachedFetch('/api/users');

    return (
        <div>
            <h2>Users (Cached)</h2>
            <button onClick={invalidateCache}>Refresh Cache</button>

            {loading && <div>Loading...</div>}
            {error && <div>Error: {error}</div>}
            {users && (
                <ul>
                    {users.map(user => (
                        <li key={user.id}>{user.name}</li>
                    ))}
                </ul>
            )}
        </div>
    );
}
```

## Custom Hook with Mutations (POST/PUT/DELETE)

### Hook for data mutations

```javascript
function useMutation(url, method = 'POST') {
    const [state, setState] = useState({
        data: null,
        loading: false,
        error: null
    });

    const mutate = useCallback(async (payload) => {
        try {
            setState({ data: null, loading: true, error: null });

            const response = await fetch(url, {
                method,
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(payload)
            });

            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            const result = await response.json();
            setState({ data: result, loading: false, error: null });

            return result;
        } catch (err) {
            setState({ data: null, loading: false, error: err.message });
            throw err;
        }
    }, [url, method]);

    const reset = useCallback(() => {
        setState({ data: null, loading: false, error: null });
    }, []);

    return { ...state, mutate, reset };
}

// Usage
function CreateUserForm() {
    const [name, setName] = useState('');
    const [email, setEmail] = useState('');
    const { data, loading, error, mutate, reset } = useMutation('/api/users', 'POST');

    const handleSubmit = async (e) => {
        e.preventDefault();

        try {
            await mutate({ name, email });
            setName('');
            setEmail('');
            alert('User created successfully!');
        } catch (err) {
            // Error is already handled by the hook
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="text"
                value={name}
                onChange={(e) => setName(e.target.value)}
                placeholder="Name"
                required
            />
            <input
                type="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                placeholder="Email"
                required
            />

            <button type="submit" disabled={loading}>
                {loading ? 'Creating...' : 'Create User'}
            </button>

            {error && <div className="error">Error: {error}</div>}
            {data && <div className="success">Created: {data.name}</div>}

            <button type="button" onClick={reset}>Reset</button>
        </form>
    );
}
```

## Advanced Hook with Optimistic Updates

### Hook with optimistic updates for better UX

```javascript
function useOptimisticMutation(url, method = 'POST', options = {}) {
    const [state, setState] = useState({
        data: null,
        loading: false,
        error: null
    });

    const mutate = useCallback(async (payload, optimisticUpdate) => {
        // Store original state for rollback
        const originalState = { ...state };

        // Apply optimistic update immediately
        if (optimisticUpdate) {
            setState(prev => ({
                ...prev,
                data: optimisticUpdate(prev.data),
                loading: true
            }));
        } else {
            setState(prev => ({ ...prev, loading: true, error: null }));
        }

        try {
            const response = await fetch(url, {
                method,
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(payload)
            });

            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            const result = await response.json();
            setState({ data: result, loading: false, error: null });

            return result;
        } catch (err) {
            // Rollback optimistic update on error
            setState({
                data: originalState.data,
                loading: false,
                error: err.message
            });
            throw err;
        }
    }, [url, method, state]);

    return { ...state, mutate };
}

// Usage with optimistic updates
function TodoList() {
    const { data: todos, loading, mutate } = useOptimisticMutation('/api/todos');

    const addTodo = async (text) => {
        // Optimistic update: add todo immediately to UI
        const optimisticUpdate = (currentTodos) => [
            ...currentTodos,
            { id: Date.now(), text, completed: false }
        ];

        try {
            await mutate({ text }, optimisticUpdate);
        } catch (error) {
            // UI already rolled back by the hook
            alert('Failed to add todo');
        }
    };

    const toggleTodo = async (id) => {
        const optimisticUpdate = (currentTodos) =>
            currentTodos.map(todo =>
                todo.id === id ? { ...todo, completed: !todo.completed } : todo
            );

        try {
            await mutate({ id, action: 'toggle' }, optimisticUpdate);
        } catch (error) {
            alert('Failed to update todo');
        }
    };

    if (loading && !todos) return <div>Loading...</div>;

    return (
        <div>
            <button onClick={() => addTodo('New Todo')}>Add Todo</button>
            {todos?.map(todo => (
                <div key={todo.id}>
                    <input
                        type="checkbox"
                        checked={todo.completed}
                        onChange={() => toggleTodo(todo.id)}
                    />
                    <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
                        {todo.text}
                    </span>
                </div>
            ))}
        </div>
    );
}
```

## Real React Examples:

### 1. **User Management Dashboard**

```javascript
function useUsers() {
    const [users, setUsers] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    const fetchUsers = useCallback(async () => {
        try {
            setLoading(true);
            const response = await fetch('/api/users');
            const data = await response.json();
            setUsers(data);
            setError(null);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }, []);

    const createUser = useCallback(async (userData) => {
        try {
            const response = await fetch('/api/users', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(userData)
            });
            const newUser = await response.json();
            setUsers(prev => [...prev, newUser]);
            return newUser;
        } catch (err) {
            setError(err.message);
            throw err;
        }
    }, []);

    const updateUser = useCallback(async (id, userData) => {
        try {
            const response = await fetch(`/api/users/${id}`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(userData)
            });
            const updatedUser = await response.json();
            setUsers(prev => prev.map(user =>
                user.id === id ? updatedUser : user
            ));
            return updatedUser;
        } catch (err) {
            setError(err.message);
            throw err;
        }
    }, []);

    const deleteUser = useCallback(async (id) => {
        try {
            await fetch(`/api/users/${id}`, { method: 'DELETE' });
            setUsers(prev => prev.filter(user => user.id !== id));
        } catch (err) {
            setError(err.message);
            throw err;
        }
    }, []);

    useEffect(() => {
        fetchUsers();
    }, [fetchUsers]);

    return {
        users,
        loading,
        error,
        createUser,
        updateUser,
        deleteUser,
        refetch: fetchUsers
    };
}

function UserDashboard() {
    const {
        users,
        loading,
        error,
        createUser,
        updateUser,
        deleteUser,
        refetch
    } = useUsers();

    const [newUserName, setNewUserName] = useState('');

    const handleCreateUser = async () => {
        if (!newUserName.trim()) return;

        try {
            await createUser({ name: newUserName });
            setNewUserName('');
        } catch (err) {
            // Error handled by hook
        }
    };

    if (loading) return <div>Loading users...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <div className="user-dashboard">
            <h2>User Management</h2>

            <div className="create-user">
                <input
                    type="text"
                    value={newUserName}
                    onChange={(e) => setNewUserName(e.target.value)}
                    placeholder="New user name"
                />
                <button onClick={handleCreateUser}>Add User</button>
            </div>

            <button onClick={refetch}>Refresh</button>

            <div className="user-list">
                {users.map(user => (
                    <div key={user.id} className="user-item">
                        <span>{user.name}</span>
                        <button onClick={() => deleteUser(user.id)}>Delete</button>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

### 2. **Search with Debouncing**

```javascript
function useDebouncedSearch(initialQuery = '', delay = 300) {
    const [query, setQuery] = useState(initialQuery);
    const [debouncedQuery, setDebouncedQuery] = useState(initialQuery);
    const [results, setResults] = useState([]);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    // Debounce the query
    useEffect(() => {
        const timer = setTimeout(() => {
            setDebouncedQuery(query);
        }, delay);

        return () => clearTimeout(timer);
    }, [query, delay]);

    // Search when debounced query changes
    useEffect(() => {
        if (!debouncedQuery.trim()) {
            setResults([]);
            return;
        }

        let isMounted = true;

        const search = async () => {
            try {
                setLoading(true);
                setError(null);

                const response = await fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`);
                const data = await response.json();

                if (isMounted) {
                    setResults(data.results || []);
                }
            } catch (err) {
                if (isMounted) {
                    setError(err.message);
                    setResults([]);
                }
            } finally {
                if (isMounted) {
                    setLoading(false);
                }
            }
        };

        search();

        return () => {
            isMounted = false;
        };
    }, [debouncedQuery]);

    return {
        query,
        setQuery,
        results,
        loading,
        error,
        debouncedQuery
    };
}

function SearchComponent() {
    const { query, setQuery, results, loading, error } = useDebouncedSearch();

    return (
        <div className="search-component">
            <input
                type="text"
                value={query}
                onChange={(e) => setQuery(e.target.value)}
                placeholder="Search..."
                className="search-input"
            />

            {loading && <div className="loading">Searching...</div>}
            {error && <div className="error">Search error: {error}</div>}

            <div className="search-results">
                {results.map((result, index) => (
                    <div key={index} className="result-item">
                        <h4>{result.title}</h4>
                        <p>{result.description}</p>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

### 3. **Infinite Scroll Data Fetching**

```javascript
function useInfiniteScroll(endpoint, pageSize = 20) {
    const [data, setData] = useState([]);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);
    const [hasMore, setHasMore] = useState(true);
    const [page, setPage] = useState(1);

    const loadMore = useCallback(async () => {
        if (loading || !hasMore) return;

        try {
            setLoading(true);
            setError(null);

            const response = await fetch(`${endpoint}?page=${page}&limit=${pageSize}`);
            const newData = await response.json();

            if (newData.length < pageSize) {
                setHasMore(false);
            }

            setData(prev => [...prev, ...newData]);
            setPage(prev => prev + 1);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }, [endpoint, page, pageSize, loading, hasMore]);

    const reset = useCallback(() => {
        setData([]);
        setPage(1);
        setHasMore(true);
        setError(null);
    }, []);

    // Initial load
    useEffect(() => {
        loadMore();
    }, []); // Empty dependency array for initial load only

    return {
        data,
        loading,
        error,
        hasMore,
        loadMore,
        reset
    };
}

function InfiniteList() {
    const { data, loading, error, hasMore, loadMore } = useInfiniteScroll('/api/items');
    const observerRef = useRef();

    useEffect(() => {
        const observer = new IntersectionObserver(
            (entries) => {
                if (entries[0].isIntersecting && hasMore && !loading) {
                    loadMore();
                }
            },
            { threshold: 1.0 }
        );

        if (observerRef.current) {
            observer.observe(observerRef.current);
        }

        return () => observer.disconnect();
    }, [hasMore, loading, loadMore]);

    return (
        <div className="infinite-list">
            {data.map((item, index) => (
                <div key={item.id} className="list-item">
                    {item.title}
                </div>
            ))}

            {loading && <div className="loading">Loading more...</div>}
            {error && <div className="error">Error: {error}</div>}

            {/* Invisible element to trigger loading */}
            <div ref={observerRef} style={{ height: '20px' }} />
        </div>
    );
}
```

### 4. **Real-time Data with WebSocket**

```javascript
function useWebSocketData(url, eventType) {
    const [data, setData] = useState(null);
    const [connected, setConnected] = useState(false);
    const [error, setError] = useState(null);
    const wsRef = useRef(null);

    useEffect(() => {
        try {
            const ws = new WebSocket(url);
            wsRef.current = ws;

            ws.onopen = () => {
                setConnected(true);
                setError(null);
            };

            ws.onmessage = (event) => {
                try {
                    const message = JSON.parse(event.data);
                    if (message.type === eventType) {
                        setData(message.payload);
                    }
                } catch (err) {
                    console.error('Failed to parse WebSocket message:', err);
                }
            };

            ws.onerror = (error) => {
                setError('WebSocket error');
                setConnected(false);
            };

            ws.onclose = () => {
                setConnected(false);
            };

            return () => {
                ws.close();
            };
        } catch (err) {
            setError(err.message);
        }
    }, [url, eventType]);

    const sendMessage = useCallback((message) => {
        if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
            wsRef.current.send(JSON.stringify(message));
        }
    }, []);

    return { data, connected, error, sendMessage };
}

function LiveDataComponent() {
    const { data, connected, error, sendMessage } = useWebSocketData('ws://localhost:8080', 'update');

    const sendUpdate = () => {
        sendMessage({
            type: 'request_update',
            payload: { action: 'refresh' }
        });
    };

    return (
        <div className="live-data">
            <div className="connection-status">
                Status: {connected ? '🟢 Connected' : '🔴 Disconnected'}
            </div>

            {error && <div className="error">Error: {error}</div>}

            <button onClick={sendUpdate} disabled={!connected}>
                Request Update
            </button>

            {data && (
                <div className="data-display">
                    <h3>Live Data</h3>
                    <pre>{JSON.stringify(data, null, 2)}</pre>
                </div>
            )}
        </div>
    );
}
```

## Best Practices for Custom Data Fetching Hooks

### 1. **Error Handling**
- Always handle errors gracefully
- Provide meaningful error messages
- Consider retry logic for transient failures

### 2. **Loading States**
- Provide loading indicators
- Handle multiple loading states if needed
- Prevent multiple simultaneous requests

### 3. **Cleanup**
- Always clean up subscriptions and timers
- Use `isMounted` flags to prevent state updates on unmounted components
- Cancel ongoing requests when component unmounts

### 4. **Caching**
- Implement caching for frequently accessed data
- Provide cache invalidation methods
- Consider TTL (time-to-live) for cached data

### 5. **Performance**
- Use `useCallback` for functions passed as dependencies
- Memoize expensive computations
- Debounce rapid successive calls

### 6. **Testing**
- Test success scenarios
- Test error scenarios
- Test loading states
- Mock fetch/API calls

### Interview Tip:
*"Custom hooks for data fetching encapsulate async logic, error handling, and loading states. They should handle cleanup to prevent memory leaks, provide caching for performance, and follow React's rules of hooks. Always consider the component lifecycle and prevent state updates on unmounted components."*