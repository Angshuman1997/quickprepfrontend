# How do you avoid unnecessary re-renders?

## Question
How do you avoid unnecessary re-renders?

## Answer

Unnecessary re-renders are one of the most common performance issues in React applications. React re-renders components when their state or props change, but this can cascade through the component tree unnecessarily. The key is to prevent re-renders when the component's output wouldn't actually change.

## Understanding Re-renders

### 1. **When React Re-renders**

React re-renders a component when:
- **State changes** (useState, useReducer)
- **Props change** (parent re-renders)
- **Context value changes** (useContext)
- **Parent component re-renders**
- **Force update** is called

### 2. **The Performance Cost**

Each re-render involves:
- **Virtual DOM reconciliation** (diffing algorithm)
- **Component function execution** (potentially expensive)
- **Child component re-renders** (cascading effect)
- **DOM updates** (actual browser rendering)

## Core Optimization Techniques

### 1. **React.memo for Component Memoization**

**Basic Usage:**
```typescript
import React from 'react';

interface UserCardProps {
    user: { id: number; name: string; email: string };
    onSelect: (userId: number) => void;
}

// Without memo - re-renders whenever parent re-renders
function UserCard({ user, onSelect }: UserCardProps) {
    console.log('UserCard re-rendered for:', user.name);
    return (
        <div onClick={() => onSelect(user.id)}>
            <h3>{user.name}</h3>
            <p>{user.email}</p>
        </div>
    );
}

// With memo - only re-renders when props actually change
const UserCardMemo = React.memo(UserCard);

// Or with custom comparison
const UserCardCustom = React.memo(UserCard, (prevProps, nextProps) => {
    // Only re-render if user id or name changed
    return prevProps.user.id === nextProps.user.id &&
           prevProps.user.name === nextProps.user.name;
});
```

**Advanced Example:**
```typescript
// Component that receives complex objects
interface DataTableProps {
    data: Array<{ id: number; value: string }>;
    selectedIds: Set<number>;
    onRowClick: (id: number) => void;
    theme: 'light' | 'dark';
}

const DataTable = React.memo<DataTableProps>(({
    data,
    selectedIds,
    onRowClick,
    theme
}) => {
    console.log('DataTable re-rendered');

    return (
        <table className={theme}>
            <tbody>
                {data.map(item => (
                    <tr
                        key={item.id}
                        className={selectedIds.has(item.id) ? 'selected' : ''}
                        onClick={() => onRowClick(item.id)}
                    >
                        <td>{item.value}</td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}, (prevProps, nextProps) => {
    // Custom comparison for performance
    return (
        prevProps.data === nextProps.data &&
        prevProps.selectedIds === nextProps.selectedIds &&
        prevProps.theme === nextProps.theme
    );
});
```

### 2. **useCallback for Stable Function References**

**Problem - New function on every render:**
```typescript
function ParentComponent() {
    const [count, setCount] = useState(0);

    // New function reference on every render
    const handleIncrement = () => {
        setCount(c => c + 1);
    };

    return (
        <>
            <CounterButton onClick={handleIncrement} />
            <OtherComponent /> {/* This will cause CounterButton to re-render */}
        </>
    );
}

const CounterButton = React.memo(({ onClick }) => {
    console.log('CounterButton re-rendered');
    return <button onClick={onClick}>Increment</button>;
});
```

**Solution - Stable function with useCallback:**
```typescript
function ParentComponent() {
    const [count, setCount] = useState(0);

    // Stable function reference
    const handleIncrement = useCallback(() => {
        setCount(c => c + 1);
    }, []); // Empty deps - never changes

    return (
        <>
            <CounterButton onClick={handleIncrement} />
            <OtherComponent />
        </>
    );
}
```

**Advanced useCallback patterns:**
```typescript
function DataFetcher({ userId, onData }) {
    const [data, setData] = useState(null);

    // Callback that depends on props
    const handleDataUpdate = useCallback((newData) => {
        setData(newData);
        onData(newData); // Parent callback
    }, [onData]); // Include dependencies

    // API call function
    const fetchData = useCallback(async () => {
        const result = await api.fetchUserData(userId);
        handleDataUpdate(result);
    }, [userId, handleDataUpdate]); // Include all dependencies

    useEffect(() => {
        fetchData();
    }, [fetchData]);

    return <div>{data ? JSON.stringify(data) : 'Loading...'}</div>;
}
```

### 3. **useMemo for Expensive Computations**

**Basic useMemo:**
```typescript
function UserList({ users, filter }) {
    // Expensive computation - runs on every render
    const filteredUsers = users.filter(user =>
        user.name.toLowerCase().includes(filter.toLowerCase())
    );

    // Even more expensive - sorting
    const sortedUsers = [...filteredUsers].sort((a, b) =>
        a.name.localeCompare(b.name)
    );

    return (
        <ul>
            {sortedUsers.map(user => (
                <li key={user.id}>{user.name}</li>
            ))}
        </ul>
    );
}
```

**Optimized with useMemo:**
```typescript
function UserList({ users, filter }) {
    // Memoize filtered results
    const filteredUsers = useMemo(() => {
        console.log('Filtering users...');
        return users.filter(user =>
            user.name.toLowerCase().includes(filter.toLowerCase())
        );
    }, [users, filter]); // Only re-compute when users or filter change

    // Memoize sorted results
    const sortedUsers = useMemo(() => {
        console.log('Sorting users...');
        return [...filteredUsers].sort((a, b) =>
            a.name.localeCompare(b.name)
        );
    }, [filteredUsers]); // Only re-sort when filtered list changes

    return (
        <ul>
            {sortedUsers.map(user => (
                <li key={user.id}>{user.name}</li>
            ))}
        </ul>
    );
}
```

**Complex computation example:**
```typescript
function AnalyticsDashboard({ rawData, dateRange, metrics }) {
    // Expensive data processing
    const processedData = useMemo(() => {
        console.log('Processing analytics data...');

        const filtered = rawData.filter(item =>
            item.timestamp >= dateRange.start &&
            item.timestamp <= dateRange.end
        );

        const aggregated = metrics.map(metric => ({
            name: metric,
            value: filtered.reduce((sum, item) => sum + item[metric], 0),
            average: filtered.reduce((sum, item) => sum + item[metric], 0) / filtered.length,
            trend: calculateTrend(filtered, metric)
        }));

        return aggregated;
    }, [rawData, dateRange, metrics]);

    // Memoize chart data transformation
    const chartData = useMemo(() => {
        return processedData.map(item => ({
            x: item.name,
            y: item.value,
            trend: item.trend
        }));
    }, [processedData]);

    return (
        <div>
            <Chart data={chartData} />
            <MetricsTable data={processedData} />
        </div>
    );
}
```

### 4. **Optimizing Context Consumers**

**Problem - Context causes unnecessary re-renders:**
```typescript
const ThemeContext = createContext();

function App() {
    const [theme, setTheme] = useState('light');
    const [user, setUser] = useState({ name: 'John' });

    // Every context value change re-renders ALL consumers
    const contextValue = {
        theme,
        user,
        setTheme,
        setUser
    };

    return (
        <ThemeContext.Provider value={contextValue}>
            <Header /> {/* Re-renders when user changes */}
            <Sidebar /> {/* Re-renders when user changes */}
            <Content /> {/* Re-renders when user changes */}
        </ThemeContext.Provider>
    );
}

function Header() {
    const { theme } = useContext(ThemeContext); // Only needs theme
    return <header className={theme}>Header</header>;
}

function Sidebar() {
    const { theme } = useContext(ThemeContext); // Only needs theme
    return <sidebar className={theme}>Sidebar</sidebar>;
}

function Content() {
    const { user } = useContext(ThemeContext); // Only needs user
    return <main>User: {user.name}</main>;
}
```

**Solution - Split contexts or use selectors:**
```typescript
// Split into multiple contexts
const ThemeContext = createContext();
const UserContext = createContext();

function App() {
    const [theme, setTheme] = useState('light');
    const [user, setUser] = useState({ name: 'John' });

    return (
        <ThemeContext.Provider value={{ theme, setTheme }}>
            <UserContext.Provider value={{ user, setUser }}>
                <Header />
                <Sidebar />
                <Content />
            </UserContext.Provider>
        </ThemeContext.Provider>
    );
}

function Header() {
    const { theme } = useContext(ThemeContext);
    return <header className={theme}>Header</header>;
}

function Content() {
    const { user } = useContext(UserContext);
    return <main>User: {user.name}</main>;
}
```

**Advanced - Context with useMemo:**
```typescript
function App() {
    const [theme, setTheme] = useState('light');
    const [user, setUser] = useState({ name: 'John' });

    // Memoize context values separately
    const themeValue = useMemo(() => ({
        theme,
        setTheme
    }), [theme]);

    const userValue = useMemo(() => ({
        user,
        setUser
    }), [user]);

    return (
        <ThemeContext.Provider value={themeValue}>
            <UserContext.Provider value={userValue}>
                <Header />
                <Content />
            </UserContext.Provider>
        </ThemeContext.Provider>
    );
}
```

### 5. **Component Splitting and Lazy Loading**

**Split large components:**
```typescript
// Before - Large component that re-renders entirely
function Dashboard({ data, filters, user }) {
    return (
        <div>
            <Header user={user} />
            <Filters filters={filters} onChange={handleFilterChange} />
            <DataTable data={data} />
            <Charts data={data} />
            <Summary data={data} />
        </div>
    );
}

// After - Split into smaller components
const Header = React.memo(({ user }) => <header>Welcome {user.name}</header>);
const Filters = React.memo(({ filters, onChange }) => { /* ... */ });
const DataTable = React.memo(({ data }) => { /* ... */ });
const Charts = React.memo(({ data }) => { /* ... */ });
const Summary = React.memo(({ data }) => { /* ... */ });

function Dashboard({ data, filters, user }) {
    return (
        <div>
            <Header user={user} />
            <Filters filters={filters} onChange={handleFilterChange} />
            <DataTable data={data} />
            <Charts data={data} />
            <Summary data={data} />
        </div>
    );
}
```

### 6. **List Optimization with Keys**

**Proper key usage:**
```typescript
function UserList({ users }) {
    return (
        <ul>
            {users.map(user => (
                // Stable keys prevent unnecessary re-renders
                <UserItem key={user.id} user={user} />
            ))}
        </ul>
    );
}

const UserItem = React.memo(({ user }) => {
    console.log(`Rendering ${user.name}`);
    return <li>{user.name}</li>;
});
```

**Avoid index as key when order changes:**
```typescript
// Bad - index as key causes unnecessary re-renders when order changes
{items.map((item, index) => (
    <Item key={index} item={item} />
))}

// Good - stable unique identifier
{items.map(item => (
    <Item key={item.id} item={item} />
))}
```

### 7. **State Colocation**

**Move state down to where it's used:**
```typescript
// Before - State at top level causes unnecessary re-renders
function App() {
    const [searchTerm, setSearchTerm] = useState('');
    const [users, setUsers] = useState([]);

    return (
        <div>
            <SearchBar value={searchTerm} onChange={setSearchTerm} />
            <UserList users={users} searchTerm={searchTerm} />
            <UserStats users={users} /> {/* Re-renders when searchTerm changes */}
        </div>
    );
}

// After - State where it's needed
function App() {
    const [users, setUsers] = useState([]);

    return (
        <div>
            <SearchableUserList users={users} />
            <UserStats users={users} />
        </div>
    );
}

function SearchableUserList({ users }) {
    const [searchTerm, setSearchTerm] = useState('');

    const filteredUsers = useMemo(() =>
        users.filter(user =>
            user.name.toLowerCase().includes(searchTerm.toLowerCase())
        ), [users, searchTerm]);

    return (
        <>
            <SearchBar value={searchTerm} onChange={setSearchTerm} />
            <UserList users={filteredUsers} />
        </>
    );
}
```

## Advanced Patterns

### 1. **Custom Hooks for State Logic**

```typescript
// Custom hook to encapsulate stateful logic
function useSearchableList(items) {
    const [searchTerm, setSearchTerm] = useState('');
    const [sortBy, setSortBy] = useState('name');

    const filteredAndSortedItems = useMemo(() => {
        let result = items.filter(item =>
            item.name.toLowerCase().includes(searchTerm.toLowerCase())
        );

        result = [...result].sort((a, b) => {
            if (sortBy === 'name') return a.name.localeCompare(b.name);
            if (sortBy === 'date') return new Date(b.date) - new Date(a.date);
            return 0;
        });

        return result;
    }, [items, searchTerm, sortBy]);

    return {
        searchTerm,
        setSearchTerm,
        sortBy,
        setSortBy,
        items: filteredAndSortedItems
    };
}

function ItemList({ items }) {
    const {
        searchTerm,
        setSearchTerm,
        sortBy,
        setSortBy,
        items: displayItems
    } = useSearchableList(items);

    return (
        <div>
            <input
                value={searchTerm}
                onChange={e => setSearchTerm(e.target.value)}
                placeholder="Search..."
            />
            <select value={sortBy} onChange={e => setSortBy(e.target.value)}>
                <option value="name">Name</option>
                <option value="date">Date</option>
            </select>
            <ul>
                {displayItems.map(item => (
                    <li key={item.id}>{item.name}</li>
                ))}
            </ul>
        </div>
    );
}
```

### 2. **Render Props for Performance**

```typescript
// Render prop pattern for performance
function DataProvider({ children, data }) {
    const processedData = useMemo(() => expensiveProcess(data), [data]);

    return children(processedData);
}

function App() {
    const [rawData, setRawData] = useState([]);

    return (
        <DataProvider data={rawData}>
            {(processedData) => (
                <div>
                    <DataTable data={processedData} />
                    <DataChart data={processedData} />
                    <DataSummary data={processedData} />
                </div>
            )}
        </DataProvider>
    );
}
```

### 3. **Compound Components with Context**

```typescript
const ListContext = createContext();

function List({ children, items }) {
    const processedItems = useMemo(() => processItems(items), [items]);

    return (
        <ListContext.Provider value={processedItems}>
            <ul>{children}</ul>
        </ListContext.Provider>
    );
}

const ListItem = React.memo(() => {
    const items = useContext(ListContext);
    // Only re-renders when items actually change
    return <li>{/* render item */}</li>;
});

function App() {
    return (
        <List items={data}>
            {data.map((_, index) => (
                <ListItem key={index} index={index} />
            ))}
        </List>
    );
}
```

## Debugging Re-renders

### 1. **React DevTools Profiler**

```typescript
// Use React DevTools to identify re-render causes
// Look for:
- Components re-rendering without prop/state changes
- Cascading re-renders through component tree
- Expensive computations running too frequently
```

### 2. **Custom useWhyDidYouUpdate Hook**

```typescript
import { useEffect, useRef } from 'react';

function useWhyDidYouUpdate(name, props) {
    const previousProps = useRef();

    useEffect(() => {
        if (previousProps.current) {
            const allKeys = Object.keys({ ...previousProps.current, ...props });
            const changesObj = {};

            allKeys.forEach(key => {
                if (previousProps.current[key] !== props[key]) {
                    changesObj[key] = {
                        from: previousProps.current[key],
                        to: props[key]
                    };
                }
            });

            if (Object.keys(changesObj).length) {
                console.log('[why-did-you-update]', name, changesObj);
            }
        }

        previousProps.current = props;
    });
}

// Usage
function MyComponent(props) {
    useWhyDidYouUpdate('MyComponent', props);
    // ... component logic
}
```

### 3. **Performance Monitoring**

```typescript
// Monitor render frequency
function useRenderCount(componentName) {
    const count = useRef(0);
    count.current += 1;

    useEffect(() => {
        console.log(`${componentName} rendered ${count.current} times`);
    });

    return count.current;
}

function ExpensiveComponent() {
    const renderCount = useRenderCount('ExpensiveComponent');
    // ... expensive component
}
```

## Common Interview Questions

### Q: What's the difference between useMemo and useCallback?

**A:** `useMemo` memoizes the result of a computation, while `useCallback` memoizes a function reference. Use `useMemo` for expensive calculations that return values, use `useCallback` for functions passed as props to prevent unnecessary re-renders of child components.

### Q: When should you NOT use React.memo?

**A:** Don't use `React.memo` for simple components that re-render cheaply, or when the props comparison is more expensive than the render itself. Also avoid it if the component needs to re-render on every parent render for functionality reasons.

### Q: How do you optimize list rendering?

**A:** Use stable keys (not array indices), memoize expensive computations with `useMemo`, split large lists with virtualization (react-window), use `React.memo` on list items, and consider pagination or infinite scrolling for large datasets.

### Q: What's the most common cause of unnecessary re-renders?

**A:** Creating new function references on every render (not using `useCallback`) and passing new objects/arrays as props instead of memoizing them. Context value changes that affect all consumers is another common issue.

## Summary

**Key Techniques:**
1. **React.memo** - Prevent re-renders when props haven't changed
2. **useCallback** - Stable function references for event handlers
3. **useMemo** - Cache expensive computations
4. **Component splitting** - Smaller components re-render less
5. **State colocation** - Move state to components that use it
6. **Context optimization** - Split contexts or memoize values

**When to Optimize:**
- Components re-rendering frequently with same output
- Expensive computations running unnecessarily
- Cascading re-renders through large component trees
- Performance issues identified by React DevTools Profiler

**Interview Tip:** "I avoid unnecessary re-renders by using React.memo for components, useCallback for stable function references, useMemo for expensive computations, and ensuring proper component splitting. The key is identifying when a component's output actually changes versus when it's receiving new references to the same data."