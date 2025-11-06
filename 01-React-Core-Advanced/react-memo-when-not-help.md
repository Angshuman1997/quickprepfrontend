# What is React.memo? When does it NOT help?

## Question
What is React.memo? When does it NOT help?

## Answer

`React.memo` is a higher-order component that memoizes a component, preventing unnecessary re-renders when the component's props haven't changed. However, it's not a silver bullet and can sometimes hurt performance or create bugs when used incorrectly.

## What is React.memo?

`React.memo` is a performance optimization that shallowly compares a component's props and skips re-rendering if they haven't changed.

### Basic Usage:

```javascript
const MyComponent = React.memo(function MyComponent(props) {
    return <div>{props.value}</div>;
});

// Or with arrow function
const MyComponent = React.memo(({ value }) => <div>{value}</div>);
```

### How it Works:

1. **Shallow comparison**: Compares primitive values and object references
2. **Memoization**: Remembers the last rendered result
3. **Conditional rendering**: Only re-renders when props actually change

## When React.memo Helps

### 1. **Expensive Components**

```javascript
const ExpensiveChart = React.memo(function ExpensiveChart({ data, width, height }) {
    // Complex calculations and rendering
    const processedData = useMemo(() => {
        // Expensive data processing
        return processChartData(data);
    }, [data]);

    return (
        <svg width={width} height={height}>
            {/* Complex SVG rendering */}
        </svg>
    );
});

// Parent component
function Dashboard() {
    const [filter, setFilter] = useState('all');
    const [viewMode, setViewMode] = useState('chart');

    // Chart only re-renders when data changes, not when filter/viewMode change
    return (
        <div>
            <Controls filter={filter} onFilterChange={setFilter} />
            <ViewToggle mode={viewMode} onModeChange={setViewMode} />
            <ExpensiveChart data={chartData} width={800} height={400} />
        </div>
    );
}
```

### 2. **Pure Functional Components**

```javascript
const UserCard = React.memo(function UserCard({ user, onEdit }) {
    return (
        <div className="user-card">
            <img src={user.avatar} alt={user.name} />
            <h3>{user.name}</h3>
            <p>{user.email}</p>
            <button onClick={() => onEdit(user.id)}>Edit</button>
        </div>
    );
});

// In a list where only some users change
function UserList({ users, onEditUser }) {
    return (
        <div>
            {users.map(user => (
                <UserCard
                    key={user.id}
                    user={user}
                    onEdit={onEditUser}
                />
            ))}
        </div>
    );
}
```

### 3. **Components with Stable Props**

```javascript
const Navigation = React.memo(function Navigation({ currentUser, menuItems }) {
    return (
        <nav>
            <div className="user-info">
                Welcome, {currentUser.name}
            </div>
            <ul>
                {menuItems.map(item => (
                    <li key={item.id}>
                        <Link to={item.path}>{item.label}</Link>
                    </li>
                ))}
            </ul>
        </nav>
    );
});

// Props rarely change
<Navigation currentUser={user} menuItems={menuConfig} />
```

## When React.memo Does NOT Help

### 1. **Frequent Prop Changes**

```javascript
// DON'T do this - props change on every render
const TimeDisplay = React.memo(function TimeDisplay({ currentTime }) {
    return <div>{currentTime.toLocaleTimeString()}</div>;
});

function App() {
    const [time, setTime] = useState(new Date());

    useEffect(() => {
        const interval = setInterval(() => {
            setTime(new Date()); // New Date object every second
        }, 1000);
        return () => clearInterval(interval);
    }, []);

    return <TimeDisplay currentTime={time} />; // Re-renders every second anyway
}
```

### 2. **Inline Functions as Props**

```javascript
// DON'T do this - new function reference on every render
const ItemList = React.memo(function ItemList({ items }) {
    return (
        <ul>
            {items.map(item => (
                <li key={item.id}>
                    {/* New function on every render */}
                    <button onClick={() => handleItemClick(item.id)}>
                        {item.name}
                    </button>
                </li>
            ))}
        </ul>
    );
});

// Better - use useCallback
function ParentComponent() {
    const handleItemClick = useCallback((id) => {
        console.log('Clicked item:', id);
    }, []);

    return <ItemList items={items} onItemClick={handleItemClick} />;
}

const ItemList = React.memo(function ItemList({ items, onItemClick }) {
    return (
        <ul>
            {items.map(item => (
                <li key={item.id}>
                    <button onClick={() => onItemClick(item.id)}>
                        {item.name}
                    </button>
                </li>
            ))}
        </ul>
    );
});
```

### 3. **Complex Object Props**

```javascript
// DON'T do this - new object on every render
const DataDisplay = React.memo(function DataDisplay({ config }) {
    return <div>{config.title}</div>;
});

function App() {
    const [count, setCount] = useState(0);

    // New config object on every render
    const config = {
        title: 'Dashboard',
        theme: 'light',
        layout: 'grid'
    };

    return (
        <div>
            <button onClick={() => setCount(c => c + 1)}>
                Count: {count}
            </button>
            <DataDisplay config={config} /> {/* Re-renders on every count change */}
        </div>
    );
}

// Better - memoize the object
function App() {
    const [count, setCount] = useState(0);

    const config = useMemo(() => ({
        title: 'Dashboard',
        theme: 'light',
        layout: 'grid'
    }), []); // Empty deps - never changes

    return (
        <div>
            <button onClick={() => setCount(c => c + 1)}>
                Count: {count}
            </button>
            <DataDisplay config={config} /> {/* Only renders once */}
        </div>
    );
}
```

### 4. **Lightweight Components**

```javascript
// DON'T memoize simple components
const SimpleButton = React.memo(function SimpleButton({ children, onClick }) {
    return <button onClick={onClick}>{children}</button>;
});

// This is overkill - the component is very cheap to render
// The memo comparison might cost more than just re-rendering
```

### 5. **Components That Always Need to Re-render**

```javascript
// DON'T memoize components that need fresh data
const LiveData = React.memo(function LiveData({ timestamp }) {
    return <div>Updated at: {timestamp}</div>;
});

// This component should re-render when timestamp changes
// React.memo won't help here
```

## Custom Comparison Function

You can provide a custom comparison function as the second argument:

```javascript
const CustomMemoComponent = React.memo(
    function CustomMemoComponent({ data, filter }) {
        return <div>{/* render logic */}</div>;
    },
    (prevProps, nextProps) => {
        // Custom comparison logic
        return (
            prevProps.data.id === nextProps.data.id &&
            prevProps.filter === nextProps.filter
        );
    }
);
```

## Real React Examples

### 1. **E-commerce Product List**

```javascript
// Good use of React.memo
const ProductCard = React.memo(function ProductCard({
    product,
    onAddToCart,
    isInCart
}) {
    return (
        <div className="product-card">
            <img src={product.image} alt={product.name} />
            <h3>{product.name}</h3>
            <p>${product.price}</p>
            <button
                onClick={() => onAddToCart(product.id)}
                disabled={isInCart}
            >
                {isInCart ? 'In Cart' : 'Add to Cart'}
            </button>
        </div>
    );
});

function ProductList({ products, cartItems, onAddToCart }) {
    return (
        <div className="product-grid">
            {products.map(product => (
                <ProductCard
                    key={product.id}
                    product={product}
                    onAddToCart={onAddToCart}
                    isInCart={cartItems.includes(product.id)}
                />
            ))}
        </div>
    );
}

// Only ProductCard re-renders when its specific product changes
```

### 2. **Form with Validation**

```javascript
// Bad example - over-memoization
const FormField = React.memo(function FormField({
    label,
    value,
    error,
    onChange,
    onBlur
}) {
    return (
        <div className="form-field">
            <label>{label}</label>
            <input
                value={value}
                onChange={(e) => onChange(e.target.value)}
                onBlur={onBlur}
            />
            {error && <span className="error">{error}</span>}
        </div>
    );
});

// This creates new functions on every render, defeating memoization
function ContactForm() {
    const [formData, setFormData] = useState({ name: '', email: '' });
    const [errors, setErrors] = useState({});

    return (
        <form>
            <FormField
                label="Name"
                value={formData.name}
                error={errors.name}
                onChange={(value) => setFormData(prev => ({ ...prev, name: value }))}
                onBlur={() => validateField('name')}
            />
            {/* Same issue with email field */}
        </form>
    );
}

// Better approach - use useCallback
function ContactForm() {
    const [formData, setFormData] = useState({ name: '', email: '' });
    const [errors, setErrors] = useState({});

    const handleNameChange = useCallback((value) => {
        setFormData(prev => ({ ...prev, name: value }));
    }, []);

    const handleEmailChange = useCallback((value) => {
        setFormData(prev => ({ ...prev, email: value }));
    }, []);

    const handleNameBlur = useCallback(() => {
        validateField('name');
    }, []);

    const handleEmailBlur = useCallback(() => {
        validateField('email');
    }, []);

    return (
        <form>
            <FormField
                label="Name"
                value={formData.name}
                error={errors.name}
                onChange={handleNameChange}
                onBlur={handleNameBlur}
            />
            <FormField
                label="Email"
                value={formData.email}
                error={errors.email}
                onChange={handleEmailChange}
                onBlur={handleEmailBlur}
            />
        </form>
    );
}
```

### 3. **Data Table with Sorting**

```javascript
// Good use case for React.memo
const TableRow = React.memo(function TableRow({ item, columns, onRowClick }) {
    return (
        <tr onClick={() => onRowClick(item.id)}>
            {columns.map(column => (
                <td key={column.key}>
                    {column.render ? column.render(item) : item[column.key]}
                </td>
            ))}
        </tr>
    );
});

function DataTable({ data, columns, sortConfig, onSort, onRowClick }) {
    const sortedData = useMemo(() => {
        if (!sortConfig.key) return data;

        return [...data].sort((a, b) => {
            const aValue = a[sortConfig.key];
            const bValue = b[sortConfig.key];

            if (aValue < bValue) return sortConfig.direction === 'asc' ? -1 : 1;
            if (aValue > bValue) return sortConfig.direction === 'asc' ? 1 : -1;
            return 0;
        });
    }, [data, sortConfig]);

    return (
        <table>
            <thead>
                <tr>
                    {columns.map(column => (
                        <th
                            key={column.key}
                            onClick={() => onSort(column.key)}
                            className="sortable"
                        >
                            {column.label}
                            {sortConfig.key === column.key && (
                                <span>{sortConfig.direction === 'asc' ? ' ↑' : ' ↓'}</span>
                            )}
                        </th>
                    ))}
                </tr>
            </thead>
            <tbody>
                {sortedData.map(item => (
                    <TableRow
                        key={item.id}
                        item={item}
                        columns={columns}
                        onRowClick={onRowClick}
                    />
                ))}
            </tbody>
        </table>
    );
}
```

### 4. **Component with Internal State**

```javascript
// React.memo doesn't prevent re-renders due to internal state changes
const Counter = React.memo(function Counter({ initialValue }) {
    const [count, setCount] = useState(initialValue);

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={() => setCount(c => c + 1)}>Increment</button>
        </div>
    );
});

// This component will re-render when its internal state changes
// React.memo only affects re-renders triggered by parent props
function App() {
    const [multiplier, setMultiplier] = useState(1);

    return (
        <div>
            <button onClick={() => setMultiplier(m => m + 1)}>
                Change Multiplier: {multiplier}
            </button>
            <Counter initialValue={0} /> {/* Props never change, but internal state does */}
        </div>
    );
}
```

## Performance Considerations

### 1. **Shallow Comparison Cost**

```javascript
// For many props, shallow comparison can be expensive
const ComponentWithManyProps = React.memo(function ComponentWithManyProps({
    prop1, prop2, prop3, prop4, prop5, prop6, prop7, prop8, prop9, prop10
}) {
    return <div>{prop1}</div>;
});

// Use custom comparison for better performance
const OptimizedComponent = React.memo(
    function OptimizedComponent(props) {
        return <div>{props.prop1}</div>;
    },
    (prevProps, nextProps) => {
        // Only compare the props we care about
        return prevProps.prop1 === nextProps.prop1;
    }
);
```

### 2. **Memory Usage**

```javascript
// React.memo stores previous props and rendered result
// This can increase memory usage for components with large props
const ComponentWithLargeProps = React.memo(function ComponentWithLargeProps({
    largeDataObject,
    bigArray,
    complexNestedStructure
}) {
    return <div>Component content</div>;
});
```

### 3. **When to Use Custom Comparison**

```javascript
// Use custom comparison when:
// 1. Component has many props but only cares about some
// 2. Props contain non-primitive values that change reference but not value
// 3. You want to ignore certain props

const SelectiveMemoComponent = React.memo(
    function SelectiveMemoComponent({ data, theme, user, timestamp }) {
        return <div>{data.title}</div>;
    },
    (prevProps, nextProps) => {
        // Only re-render when data.title changes
        return prevProps.data.title === nextProps.data.title;
    }
);
```

## Best Practices

### 1. **Profile First**
```javascript
// Use React DevTools Profiler to identify slow components
// Don't memoize everything - only optimize bottlenecks
```

### 2. **Combine with Other Optimizations**
```javascript
// Use React.memo with useMemo and useCallback
const OptimizedComponent = React.memo(function OptimizedComponent({ data }) {
    const processedData = useMemo(() => expensiveOperation(data), [data]);

    const handleClick = useCallback(() => {
        doSomething(processedData);
    }, [processedData]);

    return <button onClick={handleClick}>Click me</button>;
});
```

### 3. **Avoid Premature Optimization**
```javascript
// Start without React.memo
// Add it only when you identify performance issues
// Measure the impact of your optimization
```

### 4. **Test Memoized Components**
```javascript
// Test that memoization works correctly
test('component does not re-render with same props', () => {
    const mockRender = jest.fn();
    const MemoizedComponent = React.memo(function TestComponent(props) {
        mockRender();
        return <div>{props.value}</div>;
    });

    const { rerender } = render(<MemoizedComponent value="test" />);
    expect(mockRender).toHaveBeenCalledTimes(1);

    // Same props - should not re-render
    rerender(<MemoizedComponent value="test" />);
    expect(mockRender).toHaveBeenCalledTimes(1);

    // Different props - should re-render
    rerender(<MemoizedComponent value="changed" />);
    expect(mockRender).toHaveBeenCalledTimes(2);
});
```

### Interview Tip:
*"React.memo prevents re-renders when props haven't changed by doing shallow comparison. It doesn't help with internal state changes, inline functions, or frequently changing props. Use it for expensive components with stable props, but always profile first."*