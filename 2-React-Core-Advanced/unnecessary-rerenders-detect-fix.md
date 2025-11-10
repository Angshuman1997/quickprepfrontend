# What causes unnecessary re-renders? How do you detect & fix them?

## Question
What causes unnecessary re-renders? How do you detect & fix them?

## Answer

Unnecessary re-renders are a major performance bottleneck in React applications. Understanding their causes and knowing how to detect and fix them is crucial for building performant apps. React re-renders components when their state or props change, but often this happens more frequently than needed.

## Common Causes of Unnecessary Re-renders

### 1. **Object/Array Creation in Render**

Creating new objects or arrays in every render causes child components to re-render:

```javascript
// ❌ Bad: New array created on every render
function TodoList({ todos }) {
    const processedTodos = todos.map(todo => ({
        ...todo,
        completed: todo.status === 'done'
    }));

    return (
        <div>
            {processedTodos.map(todo => (
                <TodoItem key={todo.id} todo={todo} />
            ))}
        </div>
    );
}

// ✅ Good: Memoize the computation
function TodoList({ todos }) {
    const processedTodos = useMemo(() =>
        todos.map(todo => ({
            ...todo,
            completed: todo.status === 'done'
        })), [todos]
    );

    return (
        <div>
            {processedTodos.map(todo => (
                <TodoItem key={todo.id} todo={todo} />
            ))}
        </div>
    );
}
```

### 2. **Inline Function Creation**

Functions created inline are new references each render:

```javascript
// ❌ Bad: New function on every render
function UserList({ users, onUserSelect }) {
    return (
        <div>
            {users.map(user => (
                <UserItem
                    key={user.id}
                    user={user}
                    onSelect={() => onUserSelect(user.id)} // New function each render
                />
            ))}
        </div>
    );
}

// ✅ Good: Use useCallback
function UserList({ users, onUserSelect }) {
    const handleUserSelect = useCallback((userId) => {
        onUserSelect(userId);
    }, [onUserSelect]);

    return (
        <div>
            {users.map(user => (
                <UserItem
                    key={user.id}
                    user={user}
                    onSelect={() => handleUserSelect(user.id)}
                />
            ))}
        </div>
    );
}
```

### 3. **Passing Unstable References**

Parent components passing new object/array references:

```javascript
// ❌ Bad: Parent creates new objects
function App() {
    const [count, setCount] = useState(0);

    return (
        <div>
            <CounterDisplay count={count} />
            <ExpensiveComponent
                config={{ theme: 'dark', lang: 'en' }} // New object each render
                data={[]} // New array each render
                onChange={() => setCount(c => c + 1)} // New function each render
            />
        </div>
    );
}

// ✅ Good: Move objects outside component or memoize
const CONFIG = { theme: 'dark', lang: 'en' };
const EMPTY_ARRAY = [];

function App() {
    const [count, setCount] = useState(0);

    const handleChange = useCallback(() => {
        setCount(c => c + 1);
    }, []);

    return (
        <div>
            <CounterDisplay count={count} />
            <ExpensiveComponent
                config={CONFIG}
                data={EMPTY_ARRAY}
                onChange={handleChange}
            />
        </div>
    );
}
```

### 4. **Context Value Changes**

Context providers that create new values each render:

```javascript
// ❌ Bad: New context value each render
const ThemeContext = createContext();

function ThemeProvider({ children }) {
    const [theme, setTheme] = useState('light');

    return (
        <ThemeContext.Provider value={{
            theme,
            setTheme,
            colors: { primary: '#007bff', secondary: '#6c757d' } // New object each render
        }}>
            {children}
        </ThemeContext.Provider>
    );
}

// ✅ Good: Memoize context value
function ThemeProvider({ children }) {
    const [theme, setTheme] = useState('light');

    const contextValue = useMemo(() => ({
        theme,
        setTheme,
        colors: { primary: '#007bff', secondary: '#6c757d' }
    }), [theme]);

    return (
        <ThemeContext.Provider value={contextValue}>
            {children}
        </ThemeContext.Provider>
    );
}
```

### 5. **State Updates That Don't Change Values**

Setting state to the same value triggers re-render:

```javascript
// ❌ Bad: Always triggers re-render
function Counter() {
    const [count, setCount] = useState(0);

    const increment = () => {
        setCount(count + 1);
    };

    const resetToZero = () => {
        setCount(0); // Always triggers re-render even if already 0
    };

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={increment}>Increment</button>
            <button onClick={resetToZero}>Reset to Zero</button>
        </div>
    );
}

// ✅ Good: Check before updating
function Counter() {
    const [count, setCount] = useState(0);

    const increment = () => {
        setCount(count + 1);
    };

    const resetToZero = () => {
        setCount(prevCount => prevCount === 0 ? prevCount : 0);
    };

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={increment}>Increment</button>
            <button onClick={resetToZero}>Reset to Zero</button>
        </div>
    );
}
```

## Detection Methods

### 1. **React DevTools Profiler**

```javascript
// Enable React DevTools Profiler in development
// Look for components with high render counts

// In code, add whyDidYouRender for debugging
import whyDidYouRender from '@welldidyou/render';

if (process.env.NODE_ENV === 'development') {
    whyDidYouRender(React);
}

// Decorate components to track renders
Component.whyDidYouRender = true;
```

### 2. **Console Logging**

```javascript
// Add render logging to components
function MyComponent({ prop }) {
    console.log('MyComponent rendered', { prop });

    return <div>{prop}</div>;
}

// Or use useEffect to track renders
function MyComponent({ prop }) {
    useEffect(() => {
        console.log('MyComponent rendered with prop:', prop);
    });

    return <div>{prop}</div>;
}
```

### 3. **Custom Hook for Render Tracking**

```javascript
function useRenderTracker(name) {
    const renders = useRef(0);
    renders.current += 1;

    useEffect(() => {
        console.log(`${name} rendered ${renders.current} times`);
    });

    return renders.current;
}

// Usage
function ExpensiveComponent() {
    const renderCount = useRenderTracker('ExpensiveComponent');

    return (
        <div>
            <p>Renders: {renderCount}</p>
            {/* Expensive content */}
        </div>
    );
}
```

### 4. **React.memo with Logging**

```javascript
const LoggedComponent = React.memo(function MyComponent({ prop }) {
    console.log('MyComponent rendered with:', prop);
    return <div>{prop}</div>;
});

// Or with custom comparison
const LoggedComponent = React.memo(
    function MyComponent({ prop }) {
        console.log('MyComponent rendered with:', prop);
        return <div>{prop}</div>;
    },
    (prevProps, nextProps) => {
        const areEqual = prevProps.prop === nextProps.prop;
        if (!areEqual) {
            console.log('Props changed:', { prev: prevProps.prop, next: nextProps.prop });
        }
        return areEqual;
    }
);
```

## Fixing Strategies

### 1. **React.memo for Component Memoization**

```javascript
// Basic memoization
const TodoItem = React.memo(function TodoItem({ todo, onToggle }) {
    console.log('TodoItem rendered:', todo.id);
    return (
        <div>
            <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => onToggle(todo.id)}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
                {todo.text}
            </span>
        </div>
    );
});

// With custom comparison function
const UserCard = React.memo(
    function UserCard({ user, onSelect }) {
        return (
            <div onClick={() => onSelect(user.id)}>
                <h3>{user.name}</h3>
                <p>{user.email}</p>
            </div>
        );
    },
    (prevProps, nextProps) => {
        // Only re-render if user data actually changed
        return (
            prevProps.user.id === nextProps.user.id &&
            prevProps.user.name === nextProps.user.name &&
            prevProps.user.email === nextProps.user.email
        );
    }
);
```

### 2. **useMemo for Expensive Computations**

```javascript
function ProductList({ products, filter, sortBy }) {
    // Memoize filtered and sorted products
    const processedProducts = useMemo(() => {
        let result = products;

        // Apply filter
        if (filter) {
            result = result.filter(product =>
                product.name.toLowerCase().includes(filter.toLowerCase())
            );
        }

        // Apply sorting
        result = [...result].sort((a, b) => {
            switch (sortBy) {
                case 'name':
                    return a.name.localeCompare(b.name);
                case 'price':
                    return a.price - b.price;
                default:
                    return 0;
            }
        });

        return result;
    }, [products, filter, sortBy]);

    return (
        <div>
            {processedProducts.map(product => (
                <ProductCard key={product.id} product={product} />
            ))}
        </div>
    );
}
```

### 3. **useCallback for Stable Function References**

```javascript
function DataTable({ data, onRowSelect }) {
    // Memoize the row selection handler
    const handleRowSelect = useCallback((rowId) => {
        onRowSelect(rowId);
    }, [onRowSelect]);

    // Memoize the sorting function
    const handleSort = useCallback((column) => {
        // Sorting logic
    }, []);

    return (
        <table>
            <thead>
                <tr>
                    {columns.map(column => (
                        <th key={column} onClick={() => handleSort(column)}>
                            {column}
                        </th>
                    ))}
                </tr>
            </thead>
            <tbody>
                {data.map(row => (
                    <tr key={row.id} onClick={() => handleRowSelect(row.id)}>
                        {columns.map(column => (
                            <td key={column}>{row[column]}</td>
                        ))}
                    </tr>
                ))}
            </tbody>
        </table>
    );
}
```

### 4. **Component Splitting**

```javascript
// Before: One component doing everything
function Dashboard({ user, stats, activities }) {
    const [selectedTab, setSelectedTab] = useState('overview');

    // All logic in one component - re-renders everything when tab changes
    return (
        <div>
            <Tabs selected={selectedTab} onSelect={setSelectedTab} />
            {selectedTab === 'overview' && <Overview stats={stats} />}
            {selectedTab === 'activities' && <Activities activities={activities} />}
            <UserInfo user={user} /> {/* Re-renders even when user doesn't change */}
        </div>
    );
}

// After: Split into focused components
function Dashboard({ user, stats, activities }) {
    return (
        <div>
            <DashboardTabs>
                <TabPanel name="overview">
                    <Overview stats={stats} />
                </TabPanel>
                <TabPanel name="activities">
                    <Activities activities={activities} />
                </TabPanel>
            </DashboardTabs>
            <UserInfo user={user} />
        </div>
    );
}

// Memoized components
const UserInfo = React.memo(function UserInfo({ user }) {
    return (
        <div className="user-info">
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
});

const Overview = React.memo(function Overview({ stats }) {
    return (
        <div>
            <StatCard title="Users" value={stats.users} />
            <StatCard title="Revenue" value={stats.revenue} />
        </div>
    );
});
```

### 5. **Context Optimization**

```javascript
// Create separate contexts for different concerns
const UserContext = createContext();
const ThemeContext = createContext();
const NotificationContext = createContext();

// Provider component with selective updates
function AppProviders({ children }) {
    const [user, setUser] = useState(null);
    const [theme, setTheme] = useState('light');
    const [notifications, setNotifications] = useState([]);

    // Memoize each context value separately
    const userValue = useMemo(() => ({ user, setUser }), [user]);
    const themeValue = useMemo(() => ({ theme, setTheme }), [theme]);
    const notificationValue = useMemo(() => ({
        notifications,
        setNotifications
    }), [notifications]);

    return (
        <UserContext.Provider value={userValue}>
            <ThemeContext.Provider value={themeValue}>
                <NotificationContext.Provider value={notificationValue}>
                    {children}
                </NotificationContext.Provider>
            </ThemeContext.Provider>
        </UserContext.Provider>
    );
}
```

## Advanced Optimization Techniques

### 1. **Windowing for Large Lists**

```javascript
import { FixedSizeList as List } from 'react-window';

// Before: Renders all items
function LargeList({ items }) {
    return (
        <div style={{ height: 400, overflow: 'auto' }}>
            {items.map(item => (
                <ListItem key={item.id} item={item} />
            ))}
        </div>
    );
}

// After: Only renders visible items
function LargeList({ items }) {
    return (
        <List
            height={400}
            itemCount={items.length}
            itemSize={50}
            width="100%"
        >
            {({ index, style }) => (
                <div style={style}>
                    <ListItem item={items[index]} />
                </div>
            )}
        </List>
    );
}
```

### 2. **Debounced Updates**

```javascript
function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);
    const [loading, setLoading] = useState(false);

    // Debounce search to avoid excessive API calls and re-renders
    const debouncedSearch = useMemo(
        () => debounce(async (searchQuery) => {
            setLoading(true);
            try {
                const response = await fetch(`/api/search?q=${searchQuery}`);
                const data = await response.json();
                setResults(data);
            } finally {
                setLoading(false);
            }
        }, 300),
        []
    );

    useEffect(() => {
        if (query) {
            debouncedSearch(query);
        } else {
            setResults([]);
        }
    }, [query, debouncedSearch]);

    return (
        <div>
            <input
                value={query}
                onChange={(e) => setQuery(e.target.value)}
                placeholder="Search..."
            />
            {loading && <div>Searching...</div>}
            <ResultsList results={results} />
        </div>
    );
}

function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}
```

### 3. **Selective Context Consumers**

```javascript
// Instead of consuming entire context
function ConsumerComponent() {
    const { user, theme, notifications, settings } = useContext(AppContext);

    // Component re-renders when ANY context value changes
    return <div>{user.name}</div>;
}

// Create selective consumers
const UserConsumer = ({ children }) => {
    const { user } = useContext(AppContext);
    return children(user);
};

const ThemeConsumer = ({ children }) => {
    const { theme } = useContext(AppContext);
    return children(theme);
};

// Usage
function OptimizedComponent() {
    return (
        <UserConsumer>
            {(user) => (
                <div>{user.name}</div> // Only re-renders when user changes
            )}
        </UserConsumer>
    );
}
```

## Real React Examples

### 1. **E-commerce Product Grid**

```javascript
// Optimized product grid with memoization
const ProductGrid = React.memo(function ProductGrid({ products, filters, sortBy }) {
    const filteredAndSortedProducts = useMemo(() => {
        let result = products;

        // Apply filters
        if (filters.category) {
            result = result.filter(p => p.category === filters.category);
        }
        if (filters.priceRange) {
            result = result.filter(p =>
                p.price >= filters.priceRange.min &&
                p.price <= filters.priceRange.max
            );
        }

        // Apply sorting
        result = [...result].sort((a, b) => {
            switch (sortBy) {
                case 'price-asc': return a.price - b.price;
                case 'price-desc': return b.price - a.price;
                case 'name': return a.name.localeCompare(b.name);
                default: return 0;
            }
        });

        return result;
    }, [products, filters, sortBy]);

    return (
        <div className="product-grid">
            {filteredAndSortedProducts.map(product => (
                <ProductCard key={product.id} product={product} />
            ))}
        </div>
    );
});

const ProductCard = React.memo(function ProductCard({ product }) {
    const handleAddToCart = useCallback(() => {
        // Add to cart logic
    }, [product.id]);

    return (
        <div className="product-card">
            <img src={product.image} alt={product.name} />
            <h3>{product.name}</h3>
            <p>${product.price}</p>
            <button onClick={handleAddToCart}>Add to Cart</button>
        </div>
    );
});
```

### 2. **Real-time Chat Application**

```javascript
// Optimized chat component
function ChatRoom({ roomId }) {
    const [messages, setMessages] = useState([]);
    const [onlineUsers, setOnlineUsers] = useState([]);

    // Memoize message rendering to avoid re-rendering all messages
    const messageElements = useMemo(() =>
        messages.map(message => (
            <MessageItem key={message.id} message={message} />
        )), [messages]
    );

    // Separate online users list to prevent chat re-renders
    const onlineUsersElement = useMemo(() =>
        <OnlineUsers users={onlineUsers} />, [onlineUsers]
    );

    return (
        <div className="chat-room">
            <div className="messages">
                {messageElements}
            </div>
            <div className="sidebar">
                {onlineUsersElement}
                <ChatInput roomId={roomId} />
            </div>
        </div>
    );
}

const MessageItem = React.memo(function MessageItem({ message }) {
    return (
        <div className={`message ${message.isMine ? 'mine' : 'theirs'}`}>
            <span className="sender">{message.sender}</span>
            <p>{message.text}</p>
            <time>{formatTime(message.timestamp)}</time>
        </div>
    );
});

const OnlineUsers = React.memo(function OnlineUsers({ users }) {
    return (
        <div className="online-users">
            <h3>Online ({users.length})</h3>
            <ul>
                {users.map(user => (
                    <li key={user.id}>{user.name}</li>
                ))}
            </ul>
        </div>
    );
});
```

### 3. **Data Table with Sorting and Filtering**

```javascript
function DataTable({ data, columns }) {
    const [sortConfig, setSortConfig] = useState({ key: null, direction: 'asc' });
    const [filters, setFilters] = useState({});

    // Memoize sorted and filtered data
    const processedData = useMemo(() => {
        let result = data;

        // Apply filters
        Object.entries(filters).forEach(([key, value]) => {
            if (value) {
                result = result.filter(item =>
                    item[key].toString().toLowerCase().includes(value.toLowerCase())
                );
            }
        });

        // Apply sorting
        if (sortConfig.key) {
            result = [...result].sort((a, b) => {
                const aValue = a[sortConfig.key];
                const bValue = b[sortConfig.key];

                if (aValue < bValue) return sortConfig.direction === 'asc' ? -1 : 1;
                if (aValue > bValue) return sortConfig.direction === 'asc' ? 1 : -1;
                return 0;
            });
        }

        return result;
    }, [data, sortConfig, filters]);

    const handleSort = useCallback((key) => {
        setSortConfig(prev => ({
            key,
            direction: prev.key === key && prev.direction === 'asc' ? 'desc' : 'asc'
        }));
    }, []);

    const handleFilterChange = useCallback((key, value) => {
        setFilters(prev => ({ ...prev, [key]: value }));
    }, []);

    return (
        <div className="data-table">
            <TableFilters
                columns={columns}
                filters={filters}
                onFilterChange={handleFilterChange}
            />
            <table>
                <thead>
                    <tr>
                        {columns.map(column => (
                            <SortableHeader
                                key={column.key}
                                column={column}
                                sortConfig={sortConfig}
                                onSort={handleSort}
                            />
                        ))}
                    </tr>
                </thead>
                <tbody>
                    {processedData.map(item => (
                        <TableRow key={item.id} item={item} columns={columns} />
                    ))}
                </tbody>
            </table>
        </div>
    );
}

const SortableHeader = React.memo(function SortableHeader({ column, sortConfig, onSort }) {
    const handleClick = useCallback(() => {
        onSort(column.key);
    }, [column.key, onSort]);

    return (
        <th onClick={handleClick} style={{ cursor: 'pointer' }}>
            {column.label}
            {sortConfig.key === column.key && (
                <span>{sortConfig.direction === 'asc' ? '↑' : '↓'}</span>
            )}
        </th>
    );
});
```

## Performance Monitoring

### 1. **React Performance DevTools**

```javascript
// In development, use React DevTools Profiler
// - Record performance traces
// - Identify components with high render counts
// - Analyze render causes and timings
```

### 2. **Custom Performance Hook**

```javascript
function usePerformanceMonitor(componentName) {
    const renderCount = useRef(0);
    const lastRenderTime = useRef(performance.now());

    renderCount.current += 1;

    useEffect(() => {
        const now = performance.now();
        const renderTime = now - lastRenderTime.current;

        if (renderTime > 16.67) { // More than one frame
            console.warn(`${componentName} slow render: ${renderTime.toFixed(2)}ms`);
        }

        lastRenderTime.current = now;
    });

    // Report to analytics in production
    useEffect(() => {
        if (process.env.NODE_ENV === 'production' && renderCount.current > 10) {
            // Send to analytics
            console.log(`${componentName} rendered ${renderCount.current} times`);
        }
    }, []);

    return renderCount.current;
}
```

### 3. **Bundle Analyzer**

```javascript
// Use webpack-bundle-analyzer to identify large components
// Optimize bundle splitting for better loading performance
```

## Best Practices

### 1. **Start with React.memo**

```javascript
// Always wrap expensive components with React.memo
const ExpensiveComponent = React.memo(function ExpensiveComponent({ data }) {
    // Expensive rendering logic
    return <div>{/* Complex JSX */}</div>;
});
```

### 2. **Use Proper Dependencies**

```javascript
// Include ALL dependencies in useMemo/useCallback
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
const memoizedCallback = useCallback(() => doSomething(a, b), [a, b]);
```

### 3. **Profile Before Optimizing**

```javascript
// Don't optimize prematurely - measure first
// Use React DevTools Profiler to identify actual bottlenecks
```

### 4. **Consider Component Size**

```javascript
// Small components may not benefit from memoization
// The memoization overhead might be worse than the re-render
```

### 5. **Use Key Props Wisely**

```javascript
// Stable keys prevent unnecessary re-mounts
{items.map(item => (
    <Item key={item.id} item={item} /> // Use stable ID
))}
```

### Interview Tip:
*"Unnecessary re-renders occur when components re-render without actual changes. Use React.memo, useMemo, and useCallback to prevent them. Always profile first with React DevTools before optimizing."*