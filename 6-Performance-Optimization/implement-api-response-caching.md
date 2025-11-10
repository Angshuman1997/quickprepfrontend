# Custom Cache Hook

**Simple React hook for caching:**
```typescript
function useApiCache(url, ttl = 5 * 60 * 1000) { // 5 minutes default
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const cache = useRef(new Map());

    useEffect(() => {
        const cached = cache.current.get(url);

        if (cached && Date.now() - cached.time < ttl) {
            setData(cached.data);
            setLoading(false);
            return;
        }

        // Fetch new data
        fetch(url)
            .then(res => res.json())
            .then(newData => {
                setData(newData);
                cache.current.set(url, {
                    data: newData,
                    time: Date.now()
                });
                setLoading(false);
            });
    }, [url, ttl]);

    return { data, loading };
}

// Usage
function MyComponent() {
    const { data, loading } = useApiCache('/api/users');

    if (loading) return <div>Loading...</div>;

    return <div>{data.map(user => <p>{user.name}</p>)}</div>;
}
```

**How it works:**
1. **Check cache first** - Looks for existing data in the Map
2. **Check if expired** - Compares current time with stored time
3. **Return cached data** - If fresh, use it immediately
4. **Fetch new data** - If expired or missing, make API call
5. **Store in cache** - Save new data with timestamp

**Benefits:**
- **Simple to use** - Just call the hook with a URL
- **Automatic caching** - No manual cache management
- **TTL support** - Data expires automatically
- **Loading states** - Built-in loading indicator