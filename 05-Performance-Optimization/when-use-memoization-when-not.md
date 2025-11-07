# Simple Memoization Hook

**Basic custom hook for expensive computations:**
```typescript
import { useMemo } from 'react';

function useExpensiveCalculation<T>(
    computeFn: () => T,
    dependencies: React.DependencyList
): T {
    return useMemo(computeFn, dependencies);
}

// Usage - When to use memoization
function DataProcessor({ data, filter, sortBy }) {
    // ✅ Good: Memoize expensive data filtering and sorting
    const processedData = useExpensiveCalculation(() => {
        console.log('🔄 Processing data...'); // Only runs when dependencies change

        let result = [...data];

        // Expensive filtering
        if (filter) {
            result = result.filter(item =>
                item.name.toLowerCase().includes(filter.toLowerCase())
            );
        }

        // Expensive sorting
        result.sort((a, b) => {
            if (sortBy === 'name') return a.name.localeCompare(b.name);
            if (sortBy === 'price') return a.price - b.price;
            return 0;
        });

        return result;
    }, [data, filter, sortBy]); // Only recompute when these change

    return (
        <div>
            <h3>Processed {processedData.length} items</h3>
            {processedData.map(item => (
                <div key={item.id}>
                    {item.name} - ${item.price}
                </div>
            ))}
        </div>
    );
}

// Usage - When NOT to use memoization
function SimpleCalculator({ a, b }) {
    // ❌ Bad: Simple arithmetic doesn't need memoization
    // const sum = useMemo(() => a + b, [a, b]);

    // ✅ Better: Just compute directly
    const sum = a + b;

    return <div>Sum: {sum}</div>;
}

function UserCard({ user }) {
    // ❌ Bad: Simple object creation
    // const displayName = useMemo(() => `${user.firstName} ${user.lastName}`, [user]);

    // ✅ Better: Compute inline
    const displayName = `${user.firstName} ${user.lastName}`;

    return (
        <div className="user-card">
            <h3>{displayName}</h3>
            <p>{user.email}</p>
        </div>
    );
}
```

**When to use:**
- ✅ Expensive computations (> 1ms)
- ✅ Complex data transformations
- ✅ API response processing
- ✅ Heavy calculations in loops

**When NOT to use:**
- ❌ Simple arithmetic (`a + b`)
- ❌ String concatenation (`${first} ${last}`)
- ❌ Primitive values
- ❌ One-time computations

**Key Benefits:**
- **Prevents unnecessary re-computation** - Only runs when dependencies change
- **Improves performance** - Skips expensive operations on re-renders
- **Stable references** - Same input = same output
- **Easy to implement** - Just wrap expensive code in useMemo
            return Object.entries(filters).every(([key, value]) => {
                return item[key].toLowerCase().includes(value.toLowerCase());
            });
        });

        // Sort data (expensive with large datasets)
        result.sort((a, b) => {
            const aVal = a[sortBy];
            const bVal = b[sortBy];
            return aVal.localeCompare(bVal);
        });

        return result;
    }, [data, filters, sortBy]); // Only recalculate when dependencies change

    return <Table data={processedData} />;
}
```

### 2. **Complex Object/Array Creation**

**Use when:** Creating large objects or arrays that are passed as props.

```typescript
// ✅ Good: Complex object creation
const chartConfig = useMemo(() => ({
    type: 'line',
    data: processedData,
    options: {
        responsive: true,
        plugins: {
            legend: { position: 'top' },
            title: { display: true, text: 'Sales Over Time' }
        },
        scales: {
            x: { type: 'time' },
            y: { beginAtZero: true }
        }
    }
}), [processedData]);

// ❌ Bad: Simple object
const styles = useMemo(() => ({
    color: props.color,
    fontSize: '14px'
}), [props.color]); // Unnecessary memoization
```

### 3. **Preventing Unnecessary Re-renders**

**Use when:** Child components re-render unnecessarily due to reference inequality.

```typescript
// ✅ Good: Stable function reference
const handleSubmit = useCallback((data) => {
    submitForm(data);
}, []); // Dependencies array - empty means never recreate

// ✅ Good: Stable object reference
const tableProps = useMemo(() => ({
    data: processedData,
    onRowClick: handleRowClick,
    columns: tableColumns
}), [processedData, handleRowClick, tableColumns]);

// Pass to memoized child
<MemoizedTable {...tableProps} />
```

### 4. **API Response Transformation**

**Use when:** Transforming API responses that don't change frequently.

```typescript
function UserProfile({ userId }) {
    const { data: user } = useQuery(['user', userId], fetchUser);

    // ✅ Good: Transform API response
    const displayName = useMemo(() => {
        if (!user) return '';
        return `${user.firstName} ${user.lastName}`.trim();
    }, [user?.firstName, user?.lastName]);

    // ✅ Good: Normalize data structure
    const userStats = useMemo(() => {
        if (!user?.stats) return null;
        return {
            totalPosts: user.stats.posts || 0,
            totalFollowers: user.stats.followers || 0,
            totalFollowing: user.stats.following || 0,
            joinDate: new Date(user.createdAt).toLocaleDateString()
        };
    }, [user?.stats, user?.createdAt]);

    return <ProfileDisplay name={displayName} stats={userStats} />;
}
```

### 5. **Heavy DOM Computations**

**Use when:** Computing values used in rendering that are expensive.

```typescript
function VirtualizedGrid({ items, searchTerm }) {
    // ✅ Good: Filter large dataset
    const filteredItems = useMemo(() => {
        if (!searchTerm) return items;
        return items.filter(item =>
            item.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
            item.description.toLowerCase().includes(searchTerm.toLowerCase())
        );
    }, [items, searchTerm]);

    // ✅ Good: Calculate grid layout
    const gridLayout = useMemo(() => {
        const columns = Math.floor(containerWidth / itemWidth);
        const rows = Math.ceil(filteredItems.length / columns);

        return {
            columns,
            rows,
            layout: filteredItems.map((item, index) => ({
                item,
                row: Math.floor(index / columns),
                col: index % columns
            }))
        };
    }, [filteredItems, containerWidth, itemWidth]);

    return <Grid layout={gridLayout} />;
}
```

## When NOT to Use Memoization

### 1. **Simple Calculations**

**Avoid when:** The computation is faster than memoization overhead.

```typescript
// ❌ Bad: Simple arithmetic
const total = useMemo(() => a + b + c, [a, b, c]);

// ❌ Bad: Simple string operations
const fullName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);

// ❌ Bad: Simple boolean logic
const isDisabled = useMemo(() => !enabled || loading, [enabled, loading]);

// ✅ Better: Just compute inline
const total = a + b + c;
const fullName = `${firstName} ${lastName}`;
const isDisabled = !enabled || loading;
```

### 2. **Primitive Values**

**Avoid when:** The value is already a primitive (string, number, boolean).

```typescript
// ❌ Bad: Primitive values don't need memoization
const count = useMemo(() => items.length, [items]);
const isVisible = useMemo(() => show && !hidden, [show, hidden]);
const label = useMemo(() => `Item ${index + 1}`, [index]);

// ✅ Better: Compute inline
const count = items.length;
const isVisible = show && !hidden;
const label = `Item ${index + 1}`;
```

### 3. **Unstable Dependencies**

**Avoid when:** Dependencies change frequently, defeating memoization benefits.

```typescript
// ❌ Bad: Unstable dependencies
const currentTime = useMemo(() => new Date().toISOString(), [Date.now()]);

// ❌ Bad: Function that changes every render
const handleClick = useCallback(() => {
    console.log('Clicked', Math.random()); // Different every time
}, [Math.random()]); // This will change every render!

// ✅ Better: Move unstable logic out
const handleClick = useCallback(() => {
    console.log('Clicked');
}, []); // No dependencies needed
```

### 4. **One-time Computations**

**Avoid when:** The value is computed once and never changes.

```typescript
// ❌ Bad: Computed once in useEffect
const [processedData, setProcessedData] = useState(null);

useEffect(() => {
    const data = expensiveComputation(props.data);
    setProcessedData(data);
}, [props.data]); // Only runs when props.data changes

// ❌ Unnecessary memoization
const processedData = useMemo(() => expensiveComputation(props.data), [props.data]);

// ✅ Better: Just use useMemo if you need synchronous access
const processedData = useMemo(() => expensiveComputation(props.data), [props.data]);
```

### 5. **In Event Handlers**

**Avoid when:** The function is recreated unnecessarily but the cost is minimal.

```typescript
// ❌ Bad: Over-memoization
const handleChange = useCallback((value) => {
    setState(value);
}, []);

// ❌ Bad: Inline function is fine here
<input onChange={useCallback((e) => setValue(e.target.value), [])} />

// ✅ Good: Only memoize if passed to optimized children
const handleSubmit = useCallback((data) => {
    submitForm(data).catch(handleError);
}, [handleError]); // Include dependencies
```

## React.memo Usage Guidelines

### **When to Use React.memo**

```typescript
// ✅ Good: Expensive component with stable props
const ExpensiveChart = React.memo(function ExpensiveChart({ data, config }) {
    // Heavy rendering logic
    return <Chart data={data} config={config} />;
});

// ✅ Good: Component re-rendering frequently with same props
const UserCard = React.memo(function UserCard({ user, onSelect }) {
    return (
        <div onClick={() => onSelect(user.id)}>
            <img src={user.avatar} alt={user.name} />
            <h3>{user.name}</h3>
        </div>
    );
});
```

### **When NOT to Use React.memo**

```typescript
// ❌ Bad: Simple component
const SimpleButton = React.memo(function SimpleButton({ children, onClick }) {
    return <button onClick={onClick}>{children}</button>;
});

// ❌ Bad: Component with unstable props
const UnstableComponent = React.memo(function UnstableComponent({ data }) {
    return <div>{JSON.stringify(data)}</div>; // data changes every render
});

// ✅ Better: Let it re-render, it's cheap
function SimpleButton({ children, onClick }) {
    return <button onClick={onClick}>{children}</button>;
}
```

### **Custom Comparison Function**

```typescript
// Use when you need custom prop comparison
const CustomMemoComponent = React.memo(
    function CustomMemoComponent({ items, filter }) {
        return <List items={items} filter={filter} />;
    },
    (prevProps, nextProps) => {
        // Custom comparison logic
        return (
            prevProps.items.length === nextProps.items.length &&
            prevProps.filter === nextProps.filter
        );
    }
);
```

## useCallback vs useMemo

### **useCallback**
- **Purpose:** Memoize function references
- **Use when:** Passing functions to optimized children
- **Returns:** The same function instance

```typescript
// ✅ Good: Stable function reference for memoized children
const handleSelect = useCallback((id) => {
    setSelectedId(id);
    logAnalytics('item_selected', { id });
}, []); // Empty deps - function never changes

// ✅ Good: Include necessary dependencies
const handleUpdate = useCallback((updates) => {
    updateUser(userId, updates);
}, [userId]); // Include userId dependency
```

### **useMemo**
- **Purpose:** Memoize computation results
- **Use when:** Expensive calculations or object creation
- **Returns:** The result of the computation

```typescript
// ✅ Good: Expensive computation
const sortedData = useMemo(() => {
    return [...data].sort((a, b) => a.name.localeCompare(b.name));
}, [data]);

// ✅ Good: Stable object reference
const styles = useMemo(() => ({
    backgroundColor: theme.primary,
    padding: '1rem',
    borderRadius: '4px'
}), [theme.primary]);
```

## Performance Measurement

### **When to Measure**

**Always measure before optimizing:**

```typescript
// Measure component render time
function useRenderTime() {
    const renderTime = useRef(0);
    const startTime = useRef(0);

    useEffect(() => {
        startTime.current = performance.now();
    });

    useEffect(() => {
        renderTime.current = performance.now() - startTime.current;
        console.log('Render time:', renderTime.current, 'ms');
    });

    return renderTime.current;
}

// Measure function execution time
function measureExecutionTime(fn, label) {
    const start = performance.now();
    const result = fn();
    const end = performance.now();
    console.log(`${label} took ${end - start} ms`);
    return result;
}
```

### **React DevTools Profiler**

**Use React DevTools to identify real performance issues:**

```typescript
// In development, check for unnecessary re-renders
function ProblematicComponent({ data }) {
    // This will cause re-renders even if data hasn't changed meaningfully
    const processedData = useMemo(() => {
        return data.map(item => ({
            ...item,
            displayName: item.name.toUpperCase() // Expensive operation
        }));
    }, [data]); // Only depends on data reference

    return <List data={processedData} />;
}
```

## Common Anti-Patterns

### 1. **Premature Memoization**

```typescript
// ❌ Bad: Memoizing everything "just in case"
function Component({ a, b, c, d, e }) {
    const value1 = useMemo(() => a + b, [a, b]);
    const value2 = useMemo(() => c * d, [c, d]);
    const value3 = useMemo(() => e.toString(), [e]);
    const value4 = useMemo(() => [value1, value2], [value1, value2]);

    return <div>{value1 + value2 + value3 + value4.length}</div>;
}

// ✅ Better: Only memoize if proven necessary
function Component({ a, b, c, d, e }) {
    const result = a + b + c * d + e.toString().length;
    return <div>{result}</div>;
}
```

### 2. **Over-Memoization**

```typescript
// ❌ Bad: Memoizing simple values
const config = useMemo(() => ({
    apiUrl: 'https://api.example.com',
    timeout: 5000,
    retries: 3
}), []); // Never changes anyway

// ✅ Better: Just export a constant
export const API_CONFIG = {
    apiUrl: 'https://api.example.com',
    timeout: 5000,
    retries: 3
};
```

### 3. **Incorrect Dependencies**

```typescript
// ❌ Bad: Missing dependencies
const fullName = useMemo(() => `${user.firstName} ${user.lastName}`, []);
// Will use stale values if user object changes

// ❌ Bad: Unnecessary dependencies
const greeting = useMemo(() => `Hello ${name}!`, [name, age, email]);
// age and email not used in computation

// ✅ Good: Correct dependencies
const fullName = useMemo(() => `${user.firstName} ${user.lastName}`, [user.firstName, user.lastName]);
const greeting = useMemo(() => `Hello ${name}!`, [name]);
```

## Testing Memoization

### **Unit Testing**

```typescript
describe('Memoization', () => {
    it('should memoize expensive computation', () => {
        const expensiveFn = jest.fn(() => {
            // Simulate expensive operation
            for (let i = 0; i < 1000000; i++) {}
            return 'result';
        });

        const { result, rerender } = renderHook(
            ({ dep }) => useMemo(() => expensiveFn(), [dep]),
            { initialProps: { dep: 1 } }
        );

        expect(expensiveFn).toHaveBeenCalledTimes(1);

        // Re-render with same dependency
        rerender({ dep: 1 });
        expect(expensiveFn).toHaveBeenCalledTimes(1); // Should not call again

        // Re-render with different dependency
        rerender({ dep: 2 });
        expect(expensiveFn).toHaveBeenCalledTimes(2); // Should call again
    });

    it('should not memoize simple operations', () => {
        // Test that simple operations are not unnecessarily memoized
        const SimpleComponent = ({ a, b }) => {
            const sum = a + b; // No memoization
            return <div>{sum}</div>;
        };

        const { rerender } = render(<SimpleComponent a={1} b={2} />);
        expect(screen.getByText('3')).toBeInTheDocument();

        rerender(<SimpleComponent a={2} b={3} />);
        expect(screen.getByText('5')).toBeInTheDocument();
    });
});
```

### **Performance Testing**

```typescript
describe('Memoization Performance', () => {
    it('should improve performance for expensive operations', () => {
        const expensiveOperation = () => {
            const data = [];
            for (let i = 0; i < 10000; i++) {
                data.push(Math.random() * i);
            }
            return data.reduce((sum, val) => sum + val, 0);
        };

        // Without memoization
        const start1 = performance.now();
        expensiveOperation();
        expensiveOperation(); // Run again
        const time1 = performance.now() - start1;

        // With memoization (simulate)
        const start2 = performance.now();
        const memoized = useMemo(() => expensiveOperation(), []);
        const sameResult = useMemo(() => expensiveOperation(), []); // Same deps
        const time2 = performance.now() - start2;

        expect(time2).toBeLessThan(time1); // Memoized should be faster
    });
});
```

## Decision Framework

### **Questions to Ask Before Memoizing**

1. **Is this computation expensive?** (> 1ms consistently)
2. **Does it run frequently?** (> 10 times per second)
3. **Are the inputs stable?** (Do dependencies change often?)
4. **Is the result used in rendering?** (Affects re-render frequency)
5. **Have I measured the performance impact?** (Use React DevTools Profiler)

### **Quick Checklist**

```typescript
function shouldMemoize(computation, dependencies) {
    // Is it an expensive computation?
    const isExpensive = measureExecutionTime(computation) > 1; // > 1ms

    // Are dependencies stable?
    const depsStable = dependencies.every(dep => {
        // Check if dependency changes frequently
        return !isFrequentlyChanging(dep);
    });

    // Is it called frequently?
    const calledFrequently = callFrequency(computation) > 10; // > 10 calls/second

    return isExpensive && depsStable && calledFrequently;
}
```

## Common Interview Questions

### Q: When should you use React.memo?

**A:** "I use React.memo when a component re-renders frequently with the same props and the component is expensive to render. It's most useful for components that perform heavy calculations or render large lists. However, I avoid it for simple components where the memoization overhead outweighs the benefits."

### Q: What's the difference between useMemo and useCallback?

**A:** "useMemo memoizes the result of a computation, while useCallback memoizes the function itself. I use useMemo for expensive calculations that return values, and useCallback for functions that need stable references, especially when passing them to memoized child components."

### Q: Why is premature memoization harmful?

**A:** "Premature memoization adds unnecessary complexity and memory overhead without proven benefits. It can also hide performance issues by masking the real bottlenecks. I always measure performance first using React DevTools Profiler before adding memoization."

### Q: How do you decide what to memoize?

**A:** "I look for expensive computations that run frequently with stable inputs. I measure the performance impact first, then add memoization only where it provides clear benefits. I also consider the memory cost and ensure proper dependency management."

### Q: When should you NOT use memoization?

**A:** "I avoid memoization for simple calculations, primitive values, and computations with unstable dependencies. Memoization has overhead, so it should only be used when the performance benefits clearly outweigh the costs."

## Summary

**Use memoization when:**
- **Expensive computations** (> 1ms) that run frequently
- **Complex object/array creation** passed as props
- **Preventing unnecessary re-renders** in child components
- **Stable dependencies** that don't change often
- **Proven performance issues** (measured, not assumed)

**Avoid memoization when:**
- **Simple calculations** (arithmetic, string operations)
- **Primitive values** (strings, numbers, booleans)
- **Unstable dependencies** that change frequently
- **One-time computations** that never change
- **Without performance measurement** (premature optimization)

**Key principles:**
- **Measure first:** Use React DevTools Profiler to identify real bottlenecks
- **Start simple:** Add memoization only when necessary
- **Correct dependencies:** Include all dependencies, exclude unnecessary ones
- **Memory aware:** Consider memory overhead of caching
- **Test thoroughly:** Ensure memoization works correctly and improves performance

**Interview Tip:** "I use memoization for expensive computations with stable dependencies that run frequently. I always measure performance first and avoid premature memoization. The key is finding the right balance between performance benefits and complexity costs."