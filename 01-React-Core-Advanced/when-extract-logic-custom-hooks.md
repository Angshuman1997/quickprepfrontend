# When to extract logic into custom hooks?

## Question
When to extract logic into custom hooks?

## Answer

Custom hooks are one of React's most powerful features for code reuse and organization. They allow you to extract stateful logic from components, making components cleaner and logic more reusable. Knowing when and how to create custom hooks is crucial for writing maintainable React applications.

## What are Custom Hooks?

Custom hooks are JavaScript functions that use React hooks internally and follow the naming convention of starting with "use":

```javascript
// Custom hook
function useLocalStorage(key, initialValue) {
    const [storedValue, setStoredValue] = useState(() => {
        try {
            const item = window.localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch (error) {
            console.error(error);
            return initialValue;
        }
    });

    const setValue = (value) => {
        try {
            setStoredValue(value);
            window.localStorage.setItem(key, JSON.stringify(value));
        } catch (error) {
            console.error(error);
        }
    };

    return [storedValue, setValue];
}

// Usage in component
function App() {
    const [name, setName] = useLocalStorage('name', 'John');

    return (
        <div>
            <input
                value={name}
                onChange={(e) => setName(e.target.value)}
            />
        </div>
    );
}
```

## When to Extract Logic into Custom Hooks

### 1. **Reusability Across Components**

When the same logic is used in multiple components:

```javascript
// ❌ Before: Duplicated logic
function ProfilePage() {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        fetchUser().then(setUser).finally(() => setLoading(false));
    }, []);

    if (loading) return <div>Loading...</div>;
    return <div>Welcome {user.name}!</div>;
}

function SettingsPage() {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        fetchUser().then(setUser).finally(() => setLoading(false));
    }, []);

    if (loading) return <div>Loading...</div>;
    return <div>Settings for {user.name}</div>;
}

// ✅ After: Extracted to custom hook
function useUser() {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        fetchUser()
            .then(setUser)
            .catch(setError)
            .finally(() => setLoading(false));
    }, []);

    return { user, loading, error };
}

function ProfilePage() {
    const { user, loading } = useUser();
    if (loading) return <div>Loading...</div>;
    return <div>Welcome {user.name}!</div>;
}

function SettingsPage() {
    const { user, loading } = useUser();
    if (loading) return <div>Loading...</div>;
    return <div>Settings for {user.name}</div>;
}
```

### 2. **Complex State Logic**

When component state logic becomes complex:

```javascript
// ❌ Before: Complex state logic in component
function TodoApp() {
    const [todos, setTodos] = useState([]);
    const [filter, setFilter] = useState('all');
    const [newTodo, setNewTodo] = useState('');

    const addTodo = () => {
        setTodos(prev => [...prev, {
            id: Date.now(),
            text: newTodo,
            completed: false
        }]);
        setNewTodo('');
    };

    const toggleTodo = (id) => {
        setTodos(prev => prev.map(todo =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo
        ));
    };

    const deleteTodo = (id) => {
        setTodos(prev => prev.filter(todo => todo.id !== id));
    };

    const filteredTodos = todos.filter(todo => {
        if (filter === 'active') return !todo.completed;
        if (filter === 'completed') return todo.completed;
        return true;
    });

    // Component becomes cluttered...
}

// ✅ After: Extracted to custom hook
function useTodos() {
    const [todos, setTodos] = useState([]);
    const [filter, setFilter] = useState('all');

    const addTodo = (text) => {
        setTodos(prev => [...prev, {
            id: Date.now(),
            text,
            completed: false
        }]);
    };

    const toggleTodo = (id) => {
        setTodos(prev => prev.map(todo =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo
        ));
    };

    const deleteTodo = (id) => {
        setTodos(prev => prev.filter(todo => todo.id !== id));
    };

    const filteredTodos = useMemo(() => {
        return todos.filter(todo => {
            if (filter === 'active') return !todo.completed;
            if (filter === 'completed') return todo.completed;
            return true;
        });
    }, [todos, filter]);

    return {
        todos: filteredTodos,
        addTodo,
        toggleTodo,
        deleteTodo,
        filter,
        setFilter
    };
}

function TodoApp() {
    const { todos, addTodo, toggleTodo, deleteTodo, filter, setFilter } = useTodos();
    const [newTodo, setNewTodo] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        addTodo(newTodo);
        setNewTodo('');
    };

    return (
        <div>
            <form onSubmit={handleSubmit}>
                <input
                    value={newTodo}
                    onChange={(e) => setNewTodo(e.target.value)}
                    placeholder="Add todo"
                />
                <button type="submit">Add</button>
            </form>

            <select value={filter} onChange={(e) => setFilter(e.target.value)}>
                <option value="all">All</option>
                <option value="active">Active</option>
                <option value="completed">Completed</option>
            </select>

            <ul>
                {todos.map(todo => (
                    <li key={todo.id}>
                        <input
                            type="checkbox"
                            checked={todo.completed}
                            onChange={() => toggleTodo(todo.id)}
                        />
                        <span style={{
                            textDecoration: todo.completed ? 'line-through' : 'none'
                        }}>
                            {todo.text}
                        </span>
                        <button onClick={() => deleteTodo(todo.id)}>Delete</button>
                    </li>
                ))}
            </ul>
        </div>
    );
}
```

### 3. **Side Effects Management**

When components have complex side effects:

```javascript
// ❌ Before: Side effects mixed with UI logic
function ChatRoom({ roomId }) {
    const [messages, setMessages] = useState([]);
    const [isConnected, setIsConnected] = useState(false);
    const [error, setError] = useState(null);

    useEffect(() => {
        let ws;
        let reconnectTimeout;

        const connect = () => {
            ws = new WebSocket(`ws://localhost:8080/rooms/${roomId}`);

            ws.onopen = () => {
                setIsConnected(true);
                setError(null);
            };

            ws.onmessage = (event) => {
                const message = JSON.parse(event.data);
                setMessages(prev => [...prev, message]);
            };

            ws.onclose = () => {
                setIsConnected(false);
                // Reconnect after 5 seconds
                reconnectTimeout = setTimeout(connect, 5000);
            };

            ws.onerror = (err) => {
                setError('Connection failed');
                console.error('WebSocket error:', err);
            };
        };

        connect();

        return () => {
            if (ws) ws.close();
            if (reconnectTimeout) clearTimeout(reconnectTimeout);
        };
    }, [roomId]);

    // Component is cluttered with connection logic...
}

// ✅ After: Extracted to custom hook
function useWebSocket(url) {
    const [messages, setMessages] = useState([]);
    const [isConnected, setIsConnected] = useState(false);
    const [error, setError] = useState(null);

    useEffect(() => {
        let ws;
        let reconnectTimeout;

        const connect = () => {
            ws = new WebSocket(url);

            ws.onopen = () => {
                setIsConnected(true);
                setError(null);
            };

            ws.onmessage = (event) => {
                const message = JSON.parse(event.data);
                setMessages(prev => [...prev, message]);
            };

            ws.onclose = () => {
                setIsConnected(false);
                reconnectTimeout = setTimeout(connect, 5000);
            };

            ws.onerror = (err) => {
                setError('Connection failed');
                console.error('WebSocket error:', err);
            };
        };

        connect();

        return () => {
            if (ws) ws.close();
            if (reconnectTimeout) clearTimeout(reconnectTimeout);
        };
    }, [url]);

    const sendMessage = (message) => {
        if (ws && ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify(message));
        }
    };

    return { messages, isConnected, error, sendMessage };
}

function ChatRoom({ roomId }) {
    const { messages, isConnected, error, sendMessage } = useWebSocket(
        `ws://localhost:8080/rooms/${roomId}`
    );
    const [newMessage, setNewMessage] = useState('');

    const handleSend = () => {
        sendMessage({ text: newMessage, timestamp: Date.now() });
        setNewMessage('');
    };

    return (
        <div>
            <div>Status: {isConnected ? 'Connected' : 'Disconnected'}</div>
            {error && <div>Error: {error}</div>}

            <div className="messages">
                {messages.map((msg, i) => (
                    <div key={i}>{msg.text}</div>
                ))}
            </div>

            <input
                value={newMessage}
                onChange={(e) => setNewMessage(e.target.value)}
                onKeyPress={(e) => e.key === 'Enter' && handleSend()}
            />
            <button onClick={handleSend}>Send</button>
        </div>
    );
}
```

### 4. **Form Handling Logic**

When forms have complex validation and submission logic:

```javascript
// ❌ Before: Form logic clutters component
function ContactForm() {
    const [values, setValues] = useState({ name: '', email: '', message: '' });
    const [errors, setErrors] = useState({});
    const [isSubmitting, setIsSubmitting] = useState(false);
    const [isSubmitted, setIsSubmitted] = useState(false);

    const validate = () => {
        const newErrors = {};
        if (!values.name) newErrors.name = 'Name is required';
        if (!values.email) newErrors.email = 'Email is required';
        if (values.email && !/\S+@\S+\.\S+/.test(values.email)) {
            newErrors.email = 'Email is invalid';
        }
        if (!values.message) newErrors.message = 'Message is required';
        return newErrors;
    };

    const handleChange = (field) => (e) => {
        setValues(prev => ({ ...prev, [field]: e.target.value }));
        if (errors[field]) {
            setErrors(prev => ({ ...prev, [field]: '' }));
        }
    };

    const handleSubmit = async (e) => {
        e.preventDefault();
        const validationErrors = validate();
        if (Object.keys(validationErrors).length > 0) {
            setErrors(validationErrors);
            return;
        }

        setIsSubmitting(true);
        try {
            await submitForm(values);
            setIsSubmitted(true);
        } catch (error) {
            setErrors({ submit: 'Failed to submit form' });
        } finally {
            setIsSubmitting(false);
        }
    };

    if (isSubmitted) return <div>Thank you for your message!</div>;

    // Component is overwhelmed with form logic...
}

// ✅ After: Extracted to custom hook
function useForm(initialValues, validate, onSubmit) {
    const [values, setValues] = useState(initialValues);
    const [errors, setErrors] = useState({});
    const [isSubmitting, setIsSubmitting] = useState(false);
    const [isSubmitted, setIsSubmitted] = useState(false);

    const handleChange = (field) => (e) => {
        const value = e.target.value;
        setValues(prev => ({ ...prev, [field]: value }));

        // Clear field error when user starts typing
        if (errors[field]) {
            setErrors(prev => ({ ...prev, [field]: '' }));
        }
    };

    const handleSubmit = async (e) => {
        e.preventDefault();

        const validationErrors = validate(values);
        if (Object.keys(validationErrors).length > 0) {
            setErrors(validationErrors);
            return;
        }

        setIsSubmitting(true);
        setErrors({});

        try {
            await onSubmit(values);
            setIsSubmitted(true);
        } catch (error) {
            setErrors({ submit: error.message || 'Submission failed' });
        } finally {
            setIsSubmitting(false);
        }
    };

    const reset = () => {
        setValues(initialValues);
        setErrors({});
        setIsSubmitted(false);
    };

    return {
        values,
        errors,
        isSubmitting,
        isSubmitted,
        handleChange,
        handleSubmit,
        reset
    };
}

function ContactForm() {
    const validate = (values) => {
        const errors = {};
        if (!values.name) errors.name = 'Name is required';
        if (!values.email) errors.email = 'Email is required';
        if (values.email && !/\S+@\S+\.\S+/.test(values.email)) {
            errors.email = 'Email is invalid';
        }
        if (!values.message) errors.message = 'Message is required';
        return errors;
    };

    const handleSubmit = async (values) => {
        await submitForm(values);
    };

    const {
        values,
        errors,
        isSubmitting,
        isSubmitted,
        handleChange,
        handleSubmit
    } = useForm(
        { name: '', email: '', message: '' },
        validate,
        handleSubmit
    );

    if (isSubmitted) return <div>Thank you for your message!</div>;

    return (
        <form onSubmit={handleSubmit}>
            <div>
                <input
                    type="text"
                    value={values.name}
                    onChange={handleChange('name')}
                    placeholder="Name"
                />
                {errors.name && <span className="error">{errors.name}</span>}
            </div>

            <div>
                <input
                    type="email"
                    value={values.email}
                    onChange={handleChange('email')}
                    placeholder="Email"
                />
                {errors.email && <span className="error">{errors.email}</span>}
            </div>

            <div>
                <textarea
                    value={values.message}
                    onChange={handleChange('message')}
                    placeholder="Message"
                />
                {errors.message && <span className="error">{errors.message}</span>}
            </div>

            {errors.submit && <div className="error">{errors.submit}</div>}

            <button type="submit" disabled={isSubmitting}>
                {isSubmitting ? 'Submitting...' : 'Submit'}
            </button>
        </form>
    );
}
```

### 5. **Animation and Transition Logic**

When components have complex animation logic:

```javascript
// ❌ Before: Animation logic in component
function CollapsiblePanel({ title, children }) {
    const [isOpen, setIsOpen] = useState(false);
    const [height, setHeight] = useState(0);
    const contentRef = useRef();

    useEffect(() => {
        if (isOpen) {
            setHeight(contentRef.current.scrollHeight);
        } else {
            setHeight(0);
        }
    }, [isOpen]);

    const toggle = () => setIsOpen(!isOpen);

    return (
        <div>
            <button onClick={toggle}>{title}</button>
            <div
                style={{
                    height: `${height}px`,
                    overflow: 'hidden',
                    transition: 'height 0.3s ease'
                }}
            >
                <div ref={contentRef}>
                    {children}
                </div>
            </div>
        </div>
    );
}

// ✅ After: Extracted to custom hook
function useCollapsible(initialOpen = false) {
    const [isOpen, setIsOpen] = useState(initialOpen);
    const [height, setHeight] = useState(initialOpen ? 'auto' : 0);
    const contentRef = useRef();

    const toggle = useCallback(() => {
        if (isOpen) {
            // Closing animation
            setHeight(contentRef.current.scrollHeight);
            requestAnimationFrame(() => {
                setHeight(0);
            });
        } else {
            // Opening animation
            setHeight(contentRef.current.scrollHeight);
        }
        setIsOpen(!isOpen);
    }, [isOpen]);

    const handleTransitionEnd = useCallback(() => {
        if (isOpen) {
            setHeight('auto'); // Allow content to grow naturally
        }
    }, [isOpen]);

    return {
        isOpen,
        height,
        contentRef,
        toggle,
        handleTransitionEnd
    };
}

function CollapsiblePanel({ title, children }) {
    const {
        isOpen,
        height,
        contentRef,
        toggle,
        handleTransitionEnd
    } = useCollapsible();

    return (
        <div>
            <button onClick={toggle}>
                {title} {isOpen ? '▼' : '▶'}
            </button>
            <div
                style={{
                    height: typeof height === 'number' ? `${height}px` : height,
                    overflow: 'hidden',
                    transition: 'height 0.3s ease'
                }}
                onTransitionEnd={handleTransitionEnd}
            >
                <div ref={contentRef}>
                    {children}
                </div>
            </div>
        </div>
    );
}
```

### 6. **API Calls and Data Fetching**

When components have complex data fetching logic:

```javascript
// ❌ Before: Data fetching logic in component
function UserList({ department }) {
    const [users, setUsers] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [page, setPage] = useState(1);
    const [hasMore, setHasMore] = useState(true);

    useEffect(() => {
        const fetchUsers = async () => {
            setLoading(true);
            setError(null);

            try {
                const response = await fetch(
                    `/api/users?department=${department}&page=${page}`
                );
                const data = await response.json();

                setUsers(prev => page === 1 ? data.users : [...prev, ...data.users]);
                setHasMore(data.hasMore);
            } catch (err) {
                setError(err.message);
            } finally {
                setLoading(false);
            }
        };

        fetchUsers();
    }, [department, page]);

    const loadMore = () => {
        if (!loading && hasMore) {
            setPage(prev => prev + 1);
        }
    };

    // Component is cluttered...
}

// ✅ After: Extracted to custom hook
function usePaginatedData(url, dependencies = []) {
    const [data, setData] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    const [page, setPage] = useState(1);
    const [hasMore, setHasMore] = useState(true);

    const fetchData = useCallback(async (pageNum = 1, reset = false) => {
        setLoading(true);
        setError(null);

        try {
            const response = await fetch(`${url}&page=${pageNum}`);
            const result = await response.json();

            setData(prev => reset ? result.data : [...prev, ...result.data]);
            setHasMore(result.hasMore);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }, [url]);

    useEffect(() => {
        fetchData(1, true); // Reset data when dependencies change
    }, dependencies);

    const loadMore = useCallback(() => {
        if (!loading && hasMore) {
            const nextPage = page + 1;
            setPage(nextPage);
            fetchData(nextPage);
        }
    }, [loading, hasMore, page, fetchData]);

    const reset = useCallback(() => {
        setPage(1);
        setData([]);
        setHasMore(true);
        setError(null);
        fetchData(1, true);
    }, [fetchData]);

    return {
        data,
        loading,
        error,
        hasMore,
        loadMore,
        reset
    };
}

function UserList({ department }) {
    const {
        data: users,
        loading,
        error,
        hasMore,
        loadMore
    } = usePaginatedData(
        `/api/users?department=${department}`,
        [department]
    );

    if (error) return <div>Error: {error}</div>;

    return (
        <div>
            <ul>
                {users.map(user => (
                    <li key={user.id}>{user.name}</li>
                ))}
            </ul>

            {loading && <div>Loading...</div>}

            {hasMore && !loading && (
                <button onClick={loadMore}>Load More</button>
            )}
        </div>
    );
}
```

## Advanced Custom Hook Patterns

### 1. **Hook Composition**

Combining multiple hooks:

```javascript
function useSearchableList(items, searchFields = ['name']) {
    const [searchTerm, setSearchTerm] = useState('');
    const [sortBy, setSortBy] = useState(null);
    const [sortOrder, setSortOrder] = useState('asc');

    const filteredItems = useMemo(() => {
        let result = items;

        // Apply search
        if (searchTerm) {
            result = result.filter(item =>
                searchFields.some(field =>
                    item[field]?.toLowerCase().includes(searchTerm.toLowerCase())
                )
            );
        }

        // Apply sorting
        if (sortBy) {
            result = [...result].sort((a, b) => {
                const aValue = a[sortBy];
                const bValue = b[sortBy];

                if (aValue < bValue) return sortOrder === 'asc' ? -1 : 1;
                if (aValue > bValue) return sortOrder === 'asc' ? 1 : -1;
                return 0;
            });
        }

        return result;
    }, [items, searchTerm, searchFields, sortBy, sortOrder]);

    return {
        items: filteredItems,
        searchTerm,
        setSearchTerm,
        sortBy,
        setSortBy,
        sortOrder,
        setSortOrder
    };
}
```

### 2. **Conditional Hook Execution**

```javascript
function useConditionalEffect(condition, effect, dependencies) {
    useEffect(() => {
        if (condition) {
            return effect();
        }
    }, [condition, ...dependencies]);
}

// Usage
function MyComponent({ shouldFetch, userId }) {
    useConditionalEffect(
        shouldFetch,
        () => {
            // Fetch logic
            console.log('Fetching user:', userId);
        },
        [userId]
    );

    return <div>Component</div>;
}
```

### 3. **Hook with Reducer**

```javascript
function useAsyncReducer(reducer, initialState, asyncActions) {
    const [state, dispatch] = useReducer(reducer, initialState);

    const asyncDispatch = useCallback(async (action) => {
        if (typeof action === 'function') {
            // Async action creator
            const asyncAction = action;
            const result = await asyncAction(dispatch, () => state);
            return result;
        }

        dispatch(action);
    }, [state]);

    return [state, asyncDispatch];
}

// Usage
const initialState = { data: null, loading: false, error: null };

function reducer(state, action) {
    switch (action.type) {
        case 'FETCH_START':
            return { ...state, loading: true, error: null };
        case 'FETCH_SUCCESS':
            return { data: action.payload, loading: false, error: null };
        case 'FETCH_ERROR':
            return { ...state, loading: false, error: action.payload };
        default:
            return state;
    }
}

function useApi(endpoint) {
    const [state, dispatch] = useAsyncReducer(reducer, initialState);

    const fetchData = useCallback(() => {
        return async (dispatch, getState) => {
            dispatch({ type: 'FETCH_START' });
            try {
                const response = await fetch(endpoint);
                const data = await response.json();
                dispatch({ type: 'FETCH_SUCCESS', payload: data });
            } catch (error) {
                dispatch({ type: 'FETCH_ERROR', payload: error.message });
            }
        };
    }, [endpoint]);

    useEffect(() => {
        dispatch(fetchData());
    }, [dispatch, fetchData]);

    return state;
}
```

## Best Practices for Custom Hooks

### 1. **Naming Convention**

```javascript
// ✅ Good: Starts with "use"
function useLocalStorage() { /* ... */ }
function useDebounce() { /* ... */ }
function useAsync() { /* ... */ }

// ❌ Bad: Doesn't start with "use"
function localStorage() { /* ... */ }
function getUserData() { /* ... */ }
```

### 2. **Single Responsibility**

```javascript
// ✅ Good: One hook, one responsibility
function useLocalStorage(key, initialValue) { /* ... */ }
function useDebounce(value, delay) { /* ... */ }

// ❌ Bad: Multiple responsibilities
function useStorageAndDebounce(key, initialValue, delay) { /* ... */ }
```

### 3. **Return Consistent Interface**

```javascript
// ✅ Good: Consistent return pattern
function useCounter(initialValue = 0) {
    const [count, setCount] = useState(initialValue);

    const increment = () => setCount(c => c + 1);
    const decrement = () => setCount(c => c - 1);
    const reset = () => setCount(initialValue);

    return { count, increment, decrement, reset };
}

// Usage
const { count, increment } = useCounter(5);
```

### 4. **Handle Cleanup Properly**

```javascript
// ✅ Good: Proper cleanup
function useInterval(callback, delay) {
    useEffect(() => {
        if (delay !== null) {
            const id = setInterval(callback, delay);
            return () => clearInterval(id);
        }
    }, [callback, delay]);
}

// ❌ Bad: Missing cleanup
function useInterval(callback, delay) {
    useEffect(() => {
        const id = setInterval(callback, delay);
        // No cleanup - memory leak!
    }, [callback, delay]);
}
```

### 5. **Dependencies Management**

```javascript
// ✅ Good: Include all dependencies
function useDebounce(value, delay) {
    const [debouncedValue, setDebouncedValue] = useState(value);

    useEffect(() => {
        const handler = setTimeout(() => {
            setDebouncedValue(value);
        }, delay);

        return () => clearTimeout(handler);
    }, [value, delay]); // All dependencies included

    return debouncedValue;
}
```

### 6. **Error Handling**

```javascript
// ✅ Good: Handle errors gracefully
function useApi(endpoint) {
    const [state, setState] = useState({
        data: null,
        loading: true,
        error: null
    });

    useEffect(() => {
        fetch(endpoint)
            .then(response => response.json())
            .then(data => setState({ data, loading: false, error: null }))
            .catch(error => setState({
                data: null,
                loading: false,
                error: error.message
            }));
    }, [endpoint]);

    return state;
}
```

## Testing Custom Hooks

```javascript
import { renderHook, act } from '@testing-library/react';

// Test hook logic
test('useCounter increments correctly', () => {
    const { result } = renderHook(() => useCounter(0));

    expect(result.current.count).toBe(0);

    act(() => {
        result.current.increment();
    });

    expect(result.current.count).toBe(1);
});

// Test async hooks
test('useApi fetches data', async () => {
    const mockData = { name: 'John' };
    global.fetch = jest.fn(() =>
        Promise.resolve({
            json: () => Promise.resolve(mockData)
        })
    );

    const { result, waitForNextUpdate } = renderHook(() =>
        useApi('/api/user')
    );

    expect(result.current.loading).toBe(true);

    await waitForNextUpdate();

    expect(result.current.loading).toBe(false);
    expect(result.current.data).toEqual(mockData);
});
```

## Common Anti-Patterns

### 1. **Conditional Hook Calls**

```javascript
// ❌ Bad: Conditional hook calls
function MyComponent({ condition }) {
    if (condition) {
        const [state, setState] = useState(0); // Violates rules of hooks
    }

    return <div />;
}

// ✅ Good: Move condition inside hook
function useConditionalState(condition, initialValue) {
    const [state, setState] = useState(condition ? initialValue : null);
    return [state, setState];
}
```

### 2. **Hooks in Loops or Nested Functions**

```javascript
// ❌ Bad: Hooks in loop
function MyComponent() {
    const items = [1, 2, 3];

    return items.map(item => {
        const [count, setCount] = useState(0); // Violates rules of hooks
        return <div key={item}>{count}</div>;
    });
}

// ✅ Good: Extract to component
function Item({ item }) {
    const [count, setCount] = useState(0);
    return <div>{count}</div>;
}

function MyComponent() {
    const items = [1, 2, 3];
    return items.map(item => <Item key={item} item={item} />);
}
```

### 3. **Over-Engineering**

```javascript
// ❌ Bad: Hook for simple logic
function useAdd(a, b) {
    return a + b;
}

// ✅ Good: Keep simple logic inline
function MyComponent() {
    const sum = a + b; // No need for a hook
}
```

## Real-World Examples

### 1. **Authentication Hook**

```javascript
function useAuth() {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        // Check if user is logged in
        const checkAuth = async () => {
            try {
                const token = localStorage.getItem('token');
                if (token) {
                    const response = await fetch('/api/auth/verify', {
                        headers: { Authorization: `Bearer ${token}` }
                    });
                    const userData = await response.json();
                    setUser(userData);
                }
            } catch (error) {
                console.error('Auth check failed:', error);
            } finally {
                setLoading(false);
            }
        };

        checkAuth();
    }, []);

    const login = async (credentials) => {
        const response = await fetch('/api/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(credentials)
        });
        const { token, user } = await response.json();
        localStorage.setItem('token', token);
        setUser(user);
    };

    const logout = () => {
        localStorage.removeItem('token');
        setUser(null);
    };

    return { user, loading, login, logout };
}
```

### 2. **Window Size Hook**

```javascript
function useWindowSize() {
    const [windowSize, setWindowSize] = useState({
        width: window.innerWidth,
        height: window.innerHeight
    });

    useEffect(() => {
        const handleResize = () => {
            setWindowSize({
                width: window.innerWidth,
                height: window.innerHeight
            });
        };

        window.addEventListener('resize', handleResize);
        return () => window.removeEventListener('resize', handleResize);
    }, []);

    return windowSize;
}
```

### 3. **Debounce Hook**

```javascript
function useDebounce(value, delay) {
    const [debouncedValue, setDebouncedValue] = useState(value);

    useEffect(() => {
        const handler = setTimeout(() => {
            setDebouncedValue(value);
        }, delay);

        return () => clearTimeout(handler);
    }, [value, delay]);

    return debouncedValue;
}
```

### Interview Tip:
*"Extract logic into custom hooks when you have reusable stateful logic, complex side effects, or when component logic becomes too cluttered. Custom hooks should start with 'use', have a single responsibility, and return a consistent interface."*