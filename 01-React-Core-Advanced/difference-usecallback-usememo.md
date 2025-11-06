# Difference between useCallback and useMemo

## Question
Difference between useCallback and useMemo.

## Answer

`useCallback` and `useMemo` are both React hooks designed for performance optimization, but they serve different purposes and optimize different aspects of your components.

## useMemo

`useMemo` memoizes the result of a computation. It returns a memoized value that only changes when one of its dependencies changes.

### Purpose:
- **Memoize expensive calculations**: Prevent re-computation on every render
- **Memoize complex objects**: Prevent recreation of objects/arrays on every render
- **Optimize derived state**: Cache computed values from props/state

### Syntax:
```javascript
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

### When to use:
- Expensive calculations (sorting, filtering, complex math)
- Creating objects/arrays that are passed as props
- Computing derived state from props
- Preventing unnecessary re-renders of child components

### Example:

```javascript
import { useMemo } from 'react';

function TodoList({ todos, filter }) {
    // Memoize filtered todos to avoid re-filtering on every render
    const filteredTodos = useMemo(() => {
        console.log('Filtering todos...');
        return todos.filter(todo => {
            switch (filter) {
                case 'completed':
                    return todo.completed;
                case 'active':
                    return !todo.completed;
                default:
                    return true;
            }
        });
    }, [todos, filter]);

    // Memoize sorted todos
    const sortedTodos = useMemo(() => {
        console.log('Sorting todos...');
        return [...filteredTodos].sort((a, b) => a.text.localeCompare(b.text));
    }, [filteredTodos]);

    return (
        <ul>
            {sortedTodos.map(todo => (
                <li key={todo.id}>{todo.text}</li>
            ))}
        </ul>
    );
}
```

## useCallback

`useCallback` memoizes a function definition. It returns a memoized callback that only changes when one of its dependencies changes.

### Purpose:
- **Memoize function references**: Prevent function recreation on every render
- **Stable function references**: Ensure child components don't re-render unnecessarily
- **Optimize event handlers**: Prevent passing new function references to optimized children

### Syntax:
```javascript
const memoizedCallback = useCallback(() => doSomething(a, b), [a, b]);
```

### When to use:
- Passing callbacks to optimized child components (with React.memo)
- Event handlers that are dependencies of other hooks
- Functions used in useEffect dependencies
- Preventing infinite re-renders in custom hooks

### Example:

```javascript
import { useCallback, useState } from 'react';

function ParentComponent() {
    const [count, setCount] = useState(0);
    const [text, setText] = useState('');

    // Without useCallback, this function is recreated on every render
    // causing ChildComponent to re-render even when count hasn't changed
    const increment = useCallback(() => {
        setCount(c => c + 1);
    }, []); // No dependencies - function never changes

    // This function depends on 'text', so it changes when text changes
    const handleTextChange = useCallback((newText) => {
        setText(newText);
    }, []);

    return (
        <div>
            <input
                value={text}
                onChange={(e) => handleTextChange(e.target.value)}
            />
            <ChildComponent onIncrement={increment} count={count} />
        </div>
    );
}

// Child component wrapped with React.memo
const ChildComponent = React.memo(({ onIncrement, count }) => {
    console.log('ChildComponent rendered');
    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={onIncrement}>Increment</button>
        </div>
    );
});
```

## Key Differences

| Aspect | useMemo | useCallback |
|--------|---------|-------------|
| **Returns** | Memoized value | Memoized function |
| **Purpose** | Cache computation results | Cache function references |
| **Use case** | Expensive calculations, derived state | Stable function references |
| **Dependencies** | Values used in computation | Values used in function |
| **Performance impact** | Reduces computation time | Reduces re-renders |

## Real React Examples

### 1. **Data Processing with useMemo**

```javascript
function DataDashboard({ rawData, searchTerm, sortBy }) {
    // Memoize expensive data processing
    const processedData = useMemo(() => {
        console.log('Processing data...');

        let data = rawData;

        // Filter data
        if (searchTerm) {
            data = data.filter(item =>
                item.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
                item.description.toLowerCase().includes(searchTerm.toLowerCase())
            );
        }

        // Sort data
        data = [...data].sort((a, b) => {
            switch (sortBy) {
                case 'name':
                    return a.name.localeCompare(b.name);
                case 'price':
                    return a.price - b.price;
                case 'rating':
                    return b.rating - a.rating;
                default:
                    return 0;
            }
        });

        // Calculate statistics
        const stats = {
            total: data.length,
            averagePrice: data.reduce((sum, item) => sum + item.price, 0) / data.length,
            topRated: Math.max(...data.map(item => item.rating))
        };

        return { data, stats };
    }, [rawData, searchTerm, sortBy]);

    return (
        <div>
            <StatsDisplay stats={processedData.stats} />
            <DataTable data={processedData.data} />
        </div>
    );
}
```

### 2. **Form Handling with useCallback**

```javascript
function MultiStepForm({ onSubmit }) {
    const [currentStep, setCurrentStep] = useState(1);
    const [formData, setFormData] = useState({});

    // Memoize form handlers to prevent unnecessary re-renders
    const handleNext = useCallback(() => {
        setCurrentStep(step => step + 1);
    }, []);

    const handlePrev = useCallback(() => {
        setCurrentStep(step => step - 1);
    }, []);

    const updateFormData = useCallback((stepData) => {
        setFormData(prev => ({ ...prev, ...stepData }));
    }, []);

    const handleSubmit = useCallback((finalData) => {
        const completeData = { ...formData, ...finalData };
        onSubmit(completeData);
    }, [formData, onSubmit]);

    return (
        <div>
            {currentStep === 1 && (
                <Step1
                    onNext={handleNext}
                    onUpdate={updateFormData}
                    initialData={formData}
                />
            )}
            {currentStep === 2 && (
                <Step2
                    onNext={handleSubmit}
                    onPrev={handlePrev}
                    onUpdate={updateFormData}
                    initialData={formData}
                />
            )}
        </div>
    );
}

const Step1 = React.memo(({ onNext, onUpdate, initialData }) => {
    const [name, setName] = useState(initialData.name || '');
    const [email, setEmail] = useState(initialData.email || '');

    const handleSubmit = useCallback(() => {
        onUpdate({ name, email });
        onNext();
    }, [name, email, onUpdate, onNext]);

    return (
        <div>
            <input
                value={name}
                onChange={(e) => setName(e.target.value)}
                placeholder="Name"
            />
            <input
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                placeholder="Email"
            />
            <button onClick={handleSubmit}>Next</button>
        </div>
    );
});
```

### 3. **Custom Hook with Both Hooks**

```javascript
function useOptimizedDataFetch(url, options = {}) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    // Memoize fetch options to prevent unnecessary re-fetches
    const fetchOptions = useMemo(() => ({
        method: 'GET',
        headers: {
            'Content-Type': 'application/json',
            ...options.headers
        },
        ...options
    }), [options]);

    // Memoize the fetch function to prevent infinite loops in useEffect
    const fetchData = useCallback(async () => {
        if (!url) return;

        setLoading(true);
        setError(null);

        try {
            const response = await fetch(url, fetchOptions);
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }
            const result = await response.json();
            setData(result);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }, [url, fetchOptions]);

    // Memoize the refetch function
    const refetch = useCallback(() => {
        fetchData();
    }, [fetchData]);

    useEffect(() => {
        fetchData();
    }, [fetchData]);

    // Memoize the return object to prevent unnecessary re-renders
    return useMemo(() => ({
        data,
        loading,
        error,
        refetch
    }), [data, loading, error, refetch]);
}

// Usage
function UserProfile({ userId }) {
    const { data: user, loading, error, refetch } = useOptimizedDataFetch(
        `https://api.example.com/users/${userId}`
    );

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <div>
            <h1>{user?.name}</h1>
            <p>{user?.email}</p>
            <button onClick={refetch}>Refresh</button>
        </div>
    );
}
```

### 4. **List Virtualization with Performance Optimization**

```javascript
function VirtualizedList({ items, itemHeight, containerHeight }) {
    const [scrollTop, setScrollTop] = useState(0);

    // Memoize visible range calculation
    const visibleRange = useMemo(() => {
        const startIndex = Math.floor(scrollTop / itemHeight);
        const endIndex = Math.min(
            startIndex + Math.ceil(containerHeight / itemHeight),
            items.length - 1
        );
        return { startIndex, endIndex };
    }, [scrollTop, itemHeight, containerHeight, items.length]);

    // Memoize visible items
    const visibleItems = useMemo(() => {
        return items.slice(visibleRange.startIndex, visibleRange.endIndex + 1);
    }, [items, visibleRange]);

    // Memoize total height calculation
    const totalHeight = useMemo(() => {
        return items.length * itemHeight;
    }, [items.length, itemHeight]);

    // Memoize scroll handler
    const handleScroll = useCallback((event) => {
        setScrollTop(event.target.scrollTop);
    }, []);

    return (
        <div
            style={{ height: containerHeight, overflow: 'auto' }}
            onScroll={handleScroll}
        >
            <div style={{ height: totalHeight, position: 'relative' }}>
                <div
                    style={{
                        transform: `translateY(${visibleRange.startIndex * itemHeight}px)`
                    }}
                >
                    {visibleItems.map((item, index) => (
                        <div
                            key={visibleRange.startIndex + index}
                            style={{ height: itemHeight }}
                        >
                            {item.name}
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
}
```

## Performance Considerations

### When to Use Each Hook:

**Use `useMemo` when:**
- You have expensive computations that run on every render
- You're creating objects/arrays that are passed as props
- You need to compute derived state from props
- You're doing complex calculations in render

**Use `useCallback` when:**
- You're passing functions as props to memoized components
- Functions are dependencies of other hooks (useEffect, useMemo)
- You want to prevent infinite re-render loops
- You're optimizing event handlers

### Common Mistakes:

1. **Over-memoization**: Don't memoize everything - only expensive operations
2. **Missing dependencies**: Include all dependencies in the dependency array
3. **Unnecessary memoization**: Simple calculations don't need memoization
4. **Stale closures**: Be careful with dependencies that might become stale

### Best Practices:

- **Profile first**: Use React DevTools Profiler to identify performance bottlenecks
- **Start simple**: Add memoization only when you see performance issues
- **Test thoroughly**: Memoization can hide bugs if dependencies are incorrect
- **Use ESLint rules**: Enable `react-hooks/exhaustive-deps` for dependency warnings

### Interview Tip:
*"`useMemo` memoizes values from expensive computations, while `useCallback` memoizes function references. Use `useMemo` for computed values and `useCallback` for stable function references passed to optimized children."*