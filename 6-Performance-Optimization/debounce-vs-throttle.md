# Debounce vs Throttle in React

## Question
Explain Debounce vs Throttle in React.

## Answer

Debounce and throttle are React hooks that control how often functions run during frequent events. They prevent too many re-renders and API calls.

## The Problem

Without these hooks, events can trigger functions hundreds of times per second:

```javascript
// Bad: Component re-renders on every keystroke
function SearchBox() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);

    useEffect(() => {
        // Runs on every character typed
        searchAPI(query).then(setResults);
    }, [query]);

    return <input value={query} onChange={e => setQuery(e.target.value)} />;
}
```

This causes:
- Too many API calls
- Unnecessary re-renders
- Poor performance

## useDebounce Hook

**Debounce waits for events to stop, then runs the function once.**

Like waiting for someone to finish typing before searching.

```javascript
function useDebounce(value, delay) {
    const [debouncedValue, setDebouncedValue] = useState(value);

    useEffect(() => {
        const handler = setTimeout(() => {
            setDebouncedValue(value);
        }, delay);

        return () => {
            clearTimeout(handler);
        };
    }, [value, delay]);

    return debouncedValue;
}
```

**Usage:**
```javascript
function SearchBox() {
    const [query, setQuery] = useState('');
    const debouncedQuery = useDebounce(query, 300);

    useEffect(() => {
        if (debouncedQuery) {
            // API call only after user stops typing
            searchAPI(debouncedQuery).then(setResults);
        }
    }, [debouncedQuery]);

    return (
        <input
            value={query}
            onChange={e => setQuery(e.target.value)}
            placeholder="Search..."
        />
    );
}
```

## useThrottle Hook

**Throttle runs the function at regular intervals, no matter how many events happen.**

Like updating scroll position every 100ms during scrolling.

```javascript
function useThrottle(callback, delay) {
    const lastRan = useRef(Date.now());

    return useCallback((...args) => {
        if (Date.now() - lastRan.current >= delay) {
            callback(...args);
            lastRan.current = Date.now();
        }
    }, [callback, delay]);
}
```

**Usage:**
```javascript
function ScrollTracker() {
    const [scrollY, setScrollY] = useState(0);

    const throttledUpdate = useThrottle(() => {
        setScrollY(window.scrollY);
    }, 100);

    useEffect(() => {
        const handleScroll = () => throttledUpdate();
        window.addEventListener('scroll', handleScroll);
        return () => window.removeEventListener('scroll', handleScroll);
    }, [throttledUpdate]);

    return <div>Scroll position: {scrollY}</div>;
}
```

## useDebouncedCallback Hook

For debouncing functions instead of values:

```javascript
function useDebouncedCallback(callback, delay) {
    const timeoutRef = useRef();

    return useCallback((...args) => {
        if (timeoutRef.current) {
            clearTimeout(timeoutRef.current);
        }

        timeoutRef.current = setTimeout(() => {
            callback(...args);
        }, delay);
    }, [callback, delay]);
}
```

**Usage:**
```javascript
function FormValidation() {
    const [email, setEmail] = useState('');
    const [isValid, setIsValid] = useState(true);

    const debouncedValidate = useDebouncedCallback((value) => {
        const valid = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
        setIsValid(valid);
    }, 500);

    const handleChange = (e) => {
        const value = e.target.value;
        setEmail(value);
        debouncedValidate(value);
    };

    return (
        <div>
            <input
                type="email"
                value={email}
                onChange={handleChange}
                className={isValid ? '' : 'error'}
            />
            {!isValid && <span>Invalid email</span>}
        </div>
    );
}
```

## useThrottledCallback Hook

For throttling functions:

```javascript
function useThrottledCallback(callback, delay) {
    const lastRan = useRef(Date.now());
    const timeoutRef = useRef();

    return useCallback((...args) => {
        if (Date.now() - lastRan.current >= delay) {
            callback(...args);
            lastRan.current = Date.now();
        } else {
            if (timeoutRef.current) {
                clearTimeout(timeoutRef.current);
            }
            timeoutRef.current = setTimeout(() => {
                callback(...args);
                lastRan.current = Date.now();
            }, delay - (Date.now() - lastRan.current));
        }
    }, [callback, delay]);
}
```

**Usage:**
```javascript
function MouseTracker() {
    const [position, setPosition] = useState({ x: 0, y: 0 });

    const throttledUpdate = useThrottledCallback((e) => {
        setPosition({ x: e.clientX, y: e.clientY });
    }, 50);

    useEffect(() => {
        window.addEventListener('mousemove', throttledUpdate);
        return () => window.removeEventListener('mousemove', throttledUpdate);
    }, [throttledUpdate]);

    return (
        <div>
            Mouse: {position.x}, {position.y}
        </div>
    );
}
```

## Key Differences

| Hook | When it runs | Best for |
|------|-------------|----------|
| **useDebounce** | After events stop | Search, validation, API calls |
| **useThrottle** | At regular intervals | Scroll, mouse tracking, resize |

## Real-World Examples

**Search with Debounce:**
```javascript
function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);
    const debouncedQuery = useDebounce(query, 300);

    useEffect(() => {
        if (debouncedQuery.length > 2) {
            fetch(`/api/search?q=${debouncedQuery}`)
                .then(res => res.json())
                .then(setResults);
        }
    }, [debouncedQuery]);

    return (
        <div>
            <input
                value={query}
                onChange={e => setQuery(e.target.value)}
                placeholder="Search products..."
            />
            <ul>
                {results.map(item => (
                    <li key={item.id}>{item.name}</li>
                ))}
            </ul>
        </div>
    );
}
```

**Infinite Scroll with Throttle:**
```javascript
function InfiniteList({ loadMore }) {
    const [items, setItems] = useState([]);
    const [loading, setLoading] = useState(false);

    const throttledLoad = useThrottledCallback(async () => {
        if (!loading) {
            setLoading(true);
            const newItems = await loadMore();
            setItems(prev => [...prev, ...newItems]);
            setLoading(false);
        }
    }, 200);

    useEffect(() => {
        const handleScroll = () => {
            const { scrollTop, scrollHeight, clientHeight } = document.documentElement;
            if (scrollTop + clientHeight >= scrollHeight - 100) {
                throttledLoad();
            }
        };

        window.addEventListener('scroll', handleScroll);
        return () => window.removeEventListener('scroll', handleScroll);
    }, [throttledLoad]);

    return (
        <div>
            {items.map(item => <div key={item.id}>{item.title}</div>)}
            {loading && <div>Loading...</div>}
        </div>
    );
}
```

## When to Use Which?

**Use Debounce when:**
- Waiting for user to finish input
- Making expensive API calls
- Validating forms after typing stops

**Use Throttle when:**
- Tracking continuous events (scroll, mouse)
- Need immediate feedback
- Updating UI during ongoing activity

## Summary

- **useDebounce:** Waits for quiet, then acts once
- **useThrottle:** Acts regularly during activity
- Both prevent performance issues
- Choose based on your specific use case