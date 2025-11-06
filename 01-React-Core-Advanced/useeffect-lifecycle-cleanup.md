# Explain the lifecycle of useEffect, including cleanup

## Question
Explain the lifecycle of useEffect, including cleanup.

## Answer

`useEffect` is React's primary hook for handling side effects in functional components. It combines the functionality of `componentDidMount`, `componentDidUpdate`, and `componentWillUnmount` from class components. Understanding its lifecycle is crucial for managing subscriptions, timers, API calls, and other side effects properly.

## useEffect Lifecycle Phases

### 1. **Mount Phase**
When a component mounts, `useEffect` runs after the first render:

```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);

    useEffect(() => {
        console.log('Component mounted');
        // This runs after the component renders for the first time
    }, []); // Empty dependency array = runs only on mount

    return <div>{user?.name}</div>;
}
```

### 2. **Update Phase**
`useEffect` runs after every render when dependencies change:

```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);

    useEffect(() => {
        console.log('userId changed, fetching user data');
        fetchUser(userId).then(setUser);
    }, [userId]); // Runs when userId changes

    return <div>{user?.name}</div>;
}
```

### 3. **Cleanup Phase**
Cleanup functions run before the effect runs again or when the component unmounts:

```javascript
function TimerComponent() {
    const [count, setCount] = useState(0);

    useEffect(() => {
        console.log('Setting up timer');
        const interval = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);

        // Cleanup function - runs before effect runs again or on unmount
        return () => {
            console.log('Cleaning up timer');
            clearInterval(interval);
        };
    }, []); // Empty array = runs only on mount/unmount

    return <div>Count: {count}</div>;
}
```

## Dependency Array Behavior

### 1. **No Dependency Array**
Runs after every render:

```javascript
useEffect(() => {
    console.log('Runs after every render');
    // This can cause infinite loops if it updates state!
});
```

### 2. **Empty Dependency Array**
Runs only on mount and unmount:

```javascript
useEffect(() => {
    console.log('Runs only on mount');

    return () => {
        console.log('Runs only on unmount');
    };
}, []); // Empty array
```

### 3. **With Dependencies**
Runs when any dependency changes:

```javascript
useEffect(() => {
    console.log('Runs when count or step changes');

    return () => {
        console.log('Cleanup when count or step changes');
    };
}, [count, step]); // Runs when count or step changes
```

## Cleanup Function Timing

### 1. **Before Re-running Effect**

```javascript
function SearchComponent({ query }) {
    useEffect(() => {
        console.log('Setting up search for:', query);

        const timeoutId = setTimeout(() => {
            performSearch(query);
        }, 300);

        // Cleanup runs BEFORE the next effect (when query changes)
        return () => {
            console.log('Cancelling previous search');
            clearTimeout(timeoutId);
        };
    }, [query]); // Effect re-runs when query changes

    return <div>Searching for: {query}</div>;
}
```

### 2. **On Component Unmount**

```javascript
function ChatRoom({ roomId }) {
    useEffect(() => {
        console.log('Joining room:', roomId);

        // Subscribe to chat messages
        const unsubscribe = subscribeToRoom(roomId, handleNewMessage);

        // Cleanup runs when component unmounts
        return () => {
            console.log('Leaving room:', roomId);
            unsubscribe();
        };
    }, [roomId]);

    return <ChatInterface />;
}
```

### 3. **Multiple Effects with Different Lifecycles**

```javascript
function ComplexComponent({ userId, theme }) {
    // Effect 1: Runs on mount/unmount only
    useEffect(() => {
        console.log('Analytics: Component mounted');
        trackPageView();

        return () => {
            console.log('Analytics: Component unmounted');
            trackPageLeave();
        };
    }, []); // Empty dependencies

    // Effect 2: Runs when userId changes
    useEffect(() => {
        console.log('Fetching user data for:', userId);
        fetchUser(userId);

        return () => {
            console.log('Cleaning up user data for:', userId);
            // Cleanup previous user data
        };
    }, [userId]); // Runs when userId changes

    // Effect 3: Runs when theme changes
    useEffect(() => {
        console.log('Applying theme:', theme);
        applyTheme(theme);

        return () => {
            console.log('Removing theme:', theme);
            removeTheme(theme);
        };
    }, [theme]); // Runs when theme changes

    return <div>Complex component</div>;
}
```

## Common Patterns and Examples

### 1. **Data Fetching with Cleanup**

```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        let isMounted = true; // Track if component is still mounted

        const fetchUser = async () => {
            setLoading(true);
            try {
                const response = await fetch(`/api/users/${userId}`);
                const userData = await response.json();

                if (isMounted) { // Only update state if component is still mounted
                    setUser(userData);
                    setLoading(false);
                }
            } catch (error) {
                if (isMounted) {
                    console.error('Failed to fetch user:', error);
                    setLoading(false);
                }
            }
        };

        fetchUser();

        // Cleanup function sets isMounted to false
        return () => {
            isMounted = false;
        };
    }, [userId]);

    if (loading) return <div>Loading...</div>;
    return <div>{user?.name}</div>;
}
```

### 2. **Event Listeners**

```javascript
function WindowSizeDisplay() {
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

        // Add event listener
        window.addEventListener('resize', handleResize);

        // Cleanup: remove event listener
        return () => {
            window.removeEventListener('resize', handleResize);
        };
    }, []); // Empty array = only on mount/unmount

    return (
        <div>
            Window size: {windowSize.width} x {windowSize.height}
        </div>
    );
}
```

### 3. **WebSocket Connections**

```javascript
function ChatRoom({ roomId }) {
    const [messages, setMessages] = useState([]);
    const [connectionStatus, setConnectionStatus] = useState('connecting');

    useEffect(() => {
        const ws = new WebSocket(`ws://localhost:8080/rooms/${roomId}`);

        ws.onopen = () => {
            console.log('Connected to room:', roomId);
            setConnectionStatus('connected');
        };

        ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            setMessages(prev => [...prev, message]);
        };

        ws.onclose = () => {
            console.log('Disconnected from room:', roomId);
            setConnectionStatus('disconnected');
        };

        ws.onerror = (error) => {
            console.error('WebSocket error:', error);
            setConnectionStatus('error');
        };

        // Cleanup: close WebSocket connection
        return () => {
            console.log('Closing WebSocket connection');
            ws.close();
        };
    }, [roomId]); // Reconnect when roomId changes

    return (
        <div>
            <div>Status: {connectionStatus}</div>
            <div>
                {messages.map((msg, i) => (
                    <div key={i}>{msg.text}</div>
                ))}
            </div>
        </div>
    );
}
```

### 4. **Timer Management**

```javascript
function CountdownTimer({ initialTime, onComplete }) {
    const [timeLeft, setTimeLeft] = useState(initialTime);

    useEffect(() => {
        if (timeLeft <= 0) {
            onComplete?.();
            return;
        }

        const timer = setTimeout(() => {
            setTimeLeft(prev => prev - 1);
        }, 1000);

        // Cleanup: clear timeout
        return () => {
            clearTimeout(timer);
        };
    }, [timeLeft, onComplete]);

    return <div>Time left: {timeLeft} seconds</div>;
}
```

### 5. **Intersection Observer**

```javascript
function LazyImage({ src, alt, placeholder }) {
    const [isVisible, setIsVisible] = useState(false);
    const [isLoaded, setIsLoaded] = useState(false);
    const imgRef = useRef();

    useEffect(() => {
        const observer = new IntersectionObserver(
            ([entry]) => {
                if (entry.isIntersecting) {
                    setIsVisible(true);
                    observer.disconnect(); // Stop observing once visible
                }
            },
            { threshold: 0.1 }
        );

        if (imgRef.current) {
            observer.observe(imgRef.current);
        }

        // Cleanup: disconnect observer
        return () => {
            observer.disconnect();
        };
    }, []);

    const handleLoad = () => {
        setIsLoaded(true);
    };

    return (
        <div ref={imgRef} style={{ minHeight: '200px' }}>
            {isVisible ? (
                <img
                    src={src}
                    alt={alt}
                    onLoad={handleLoad}
                    style={{ opacity: isLoaded ? 1 : 0.5 }}
                />
            ) : (
                <div>{placeholder}</div>
            )}
        </div>
    );
}
```

## Advanced Patterns

### 1. **Conditional Effects**

```javascript
function DataFetcher({ url, shouldFetch }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        if (!shouldFetch) return; // Early return - no effect runs

        const fetchData = async () => {
            const response = await fetch(url);
            const result = await response.json();
            setData(result);
        };

        fetchData();
    }, [url, shouldFetch]); // Effect runs when shouldFetch becomes true

    return <div>{data ? JSON.stringify(data) : 'No data'}</div>;
}
```

### 2. **Effect Dependencies with Functions**

```javascript
function DataComponent({ fetchFunction, id }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        const loadData = async () => {
            const result = await fetchFunction(id);
            setData(result);
        };

        loadData();
    }, [fetchFunction, id]); // Be careful with function dependencies!

    return <div>{data?.name}</div>;
}

// Better approach: use useCallback in parent
function ParentComponent() {
    const fetchUser = useCallback(async (id) => {
        return await api.getUser(id);
    }, []);

    return <DataComponent fetchFunction={fetchUser} id={userId} />;
}
```

### 3. **Multiple Effects for Different Concerns**

```javascript
function VideoPlayer({ src, autoplay }) {
    const videoRef = useRef();

    // Effect 1: Load video source
    useEffect(() => {
        if (videoRef.current) {
            videoRef.current.src = src;
        }
    }, [src]);

    // Effect 2: Handle autoplay
    useEffect(() => {
        if (autoplay && videoRef.current) {
            videoRef.current.play();
        }
    }, [autoplay]);

    // Effect 3: Set up event listeners
    useEffect(() => {
        const video = videoRef.current;
        if (!video) return;

        const handlePlay = () => console.log('Video started');
        const handlePause = () => console.log('Video paused');

        video.addEventListener('play', handlePlay);
        video.addEventListener('pause', handlePause);

        return () => {
            video.removeEventListener('play', handlePlay);
            video.removeEventListener('pause', handlePause);
        };
    }, []); // Empty array since we only set up listeners once

    return <video ref={videoRef} controls />;
}
```

### 4. **Effect with Async Cleanup**

```javascript
function AsyncCleanupComponent() {
    const [data, setData] = useState(null);

    useEffect(() => {
        let isCancelled = false;

        const fetchData = async () => {
            try {
                const response = await fetch('/api/data');
                const result = await response.json();

                if (!isCancelled) {
                    setData(result);
                }
            } catch (error) {
                if (!isCancelled) {
                    console.error('Fetch error:', error);
                }
            }
        };

        fetchData();

        // Cleanup function can be async in some cases
        return () => {
            isCancelled = true;
            // Could also cancel fetch requests here
        };
    }, []);

    return <div>{data ? data.message : 'Loading...'}</div>;
}
```

## Common Mistakes and Solutions

### 1. **Infinite Loops**

```javascript
// ❌ Bad: Updates state in effect without dependencies
useEffect(() => {
    setCount(count + 1); // This causes infinite loop!
});

// ✅ Good: Include dependencies or use functional update
useEffect(() => {
    setCount(prevCount => prevCount + 1);
}, []); // Empty array = runs once
```

### 2. **Stale Closures**

```javascript
// ❌ Bad: Captures stale value
function DelayedCounter() {
    const [count, setCount] = useState(0);

    useEffect(() => {
        const timeout = setTimeout(() => {
            console.log(count); // Always logs initial value (0)
        }, 1000);

        return () => clearTimeout(timeout);
    }, []); // Empty array = never updates

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={() => setCount(c => c + 1)}>Increment</button>
        </div>
    );
}

// ✅ Good: Include dependencies
useEffect(() => {
    const timeout = setTimeout(() => {
        console.log(count); // Logs current value
    }, 1000);

    return () => clearTimeout(timeout);
}, [count]); // Include count in dependencies
```

### 3. **Memory Leaks**

```javascript
// ❌ Bad: No cleanup for subscriptions
function NewsFeed() {
    const [articles, setArticles] = useState([]);

    useEffect(() => {
        const subscription = newsAPI.subscribe(setArticles);
        // No cleanup - subscription never cancelled!
    }, []);

    return <ArticleList articles={articles} />;
}

// ✅ Good: Proper cleanup
useEffect(() => {
    const subscription = newsAPI.subscribe(setArticles);

    return () => {
        subscription.unsubscribe(); // Cleanup prevents memory leaks
    };
}, []);
```

## Real React Examples

### 1. **API Polling Component**

```javascript
function StockPriceTicker({ symbol }) {
    const [price, setPrice] = useState(null);
    const [isPolling, setIsPolling] = useState(true);

    useEffect(() => {
        if (!isPolling) return;

        const pollPrice = async () => {
            try {
                const response = await fetch(`/api/stocks/${symbol}/price`);
                const data = await response.json();
                setPrice(data.price);
            } catch (error) {
                console.error('Failed to fetch price:', error);
            }
        };

        // Initial fetch
        pollPrice();

        // Set up polling
        const interval = setInterval(pollPrice, 5000);

        // Cleanup: stop polling
        return () => {
            clearInterval(interval);
        };
    }, [symbol, isPolling]);

    return (
        <div>
            <h2>{symbol} Price</h2>
            <p>${price}</p>
            <button onClick={() => setIsPolling(!isPolling)}>
                {isPolling ? 'Stop' : 'Start'} Polling
            </button>
        </div>
    );
}
```

### 2. **Form with Auto-save**

```javascript
function AutoSaveForm({ initialData, onSave }) {
    const [formData, setFormData] = useState(initialData);
    const [saveStatus, setSaveStatus] = useState('saved');

    // Auto-save effect
    useEffect(() => {
        setSaveStatus('saving');

        const saveTimeout = setTimeout(async () => {
            try {
                await onSave(formData);
                setSaveStatus('saved');
            } catch (error) {
                setSaveStatus('error');
                console.error('Auto-save failed:', error);
            }
        }, 1000); // Debounce saves by 1 second

        // Cleanup: cancel pending save
        return () => {
            clearTimeout(saveTimeout);
            setSaveStatus('pending');
        };
    }, [formData, onSave]);

    const handleChange = (field, value) => {
        setFormData(prev => ({ ...prev, [field]: value }));
    };

    return (
        <form>
            <input
                value={formData.title}
                onChange={(e) => handleChange('title', e.target.value)}
                placeholder="Title"
            />
            <textarea
                value={formData.content}
                onChange={(e) => handleChange('content', e.target.value)}
                placeholder="Content"
            />
            <div>Status: {saveStatus}</div>
        </form>
    );
}
```

### 3. **Modal with Focus Management**

```javascript
function Modal({ isOpen, onClose, children }) {
    const modalRef = useRef();

    // Focus management effect
    useEffect(() => {
        if (!isOpen) return;

        const previouslyFocusedElement = document.activeElement;

        // Focus the modal when it opens
        if (modalRef.current) {
            modalRef.current.focus();
        }

        // Cleanup: restore focus when modal closes
        return () => {
            if (previouslyFocusedElement) {
                previouslyFocusedElement.focus();
            }
        };
    }, [isOpen]);

    // Escape key handler
    useEffect(() => {
        if (!isOpen) return;

        const handleEscape = (e) => {
            if (e.key === 'Escape') {
                onClose();
            }
        };

        document.addEventListener('keydown', handleEscape);

        // Cleanup: remove event listener
        return () => {
            document.removeEventListener('keydown', handleEscape);
        };
    }, [isOpen, onClose]);

    if (!isOpen) return null;

    return (
        <div className="modal-overlay" onClick={onClose}>
            <div
                className="modal-content"
                ref={modalRef}
                tabIndex={-1}
                onClick={(e) => e.stopPropagation()}
            >
                {children}
                <button onClick={onClose}>Close</button>
            </div>
        </div>
    );
}
```

## Performance Considerations

### 1. **Dependency Array Optimization**

```javascript
// Use useCallback for stable function references
const handleSubmit = useCallback((data) => {
    submitForm(data);
}, []); // Stable reference

useEffect(() => {
    // This effect won't re-run unnecessarily
}, [handleSubmit]);
```

### 2. **Conditional Effect Execution**

```javascript
useEffect(() => {
    if (!isEnabled) return; // Skip effect execution

    expensiveOperation();

    return () => {
        cleanupOperation();
    };
}, [isEnabled]); // Only runs when isEnabled changes
```

### 3. **Effect Splitting**

```javascript
// Split unrelated effects for better performance
useEffect(() => {
    setupAnalytics();
    return () => cleanupAnalytics();
}, []); // Analytics setup

useEffect(() => {
    fetchUserData();
    return () => cancelFetch();
}, [userId]); // Data fetching

useEffect(() => {
    applyTheme();
    return () => removeTheme();
}, [theme]); // Theme application
```

## Testing useEffect

### 1. **Testing Cleanup**

```javascript
import { render, waitFor } from '@testing-library/react';

test('cleanup function is called on unmount', () => {
    const mockCleanup = jest.fn();
    const mockEffect = jest.fn(() => mockCleanup);

    function TestComponent() {
        useEffect(mockEffect, []);
        return <div />;
    }

    const { unmount } = render(<TestComponent />);
    expect(mockEffect).toHaveBeenCalledTimes(1);

    unmount();
    expect(mockCleanup).toHaveBeenCalledTimes(1);
});
```

### 2. **Testing Async Effects**

```javascript
test('fetches data on mount', async () => {
    const mockData = { name: 'John' };
    global.fetch = jest.fn(() =>
        Promise.resolve({
            json: () => Promise.resolve(mockData)
        })
    );

    function TestComponent() {
        const [data, setData] = useState(null);

        useEffect(() => {
            fetch('/api/user').then(res => res.json()).then(setData);
        }, []);

        return <div>{data?.name}</div>;
    }

    render(<TestComponent />);

    await waitFor(() => {
        expect(screen.getByText('John')).toBeInTheDocument();
    });
});
```

### Interview Tip:
*"useEffect runs after render. Cleanup functions run before the effect runs again or on unmount. Always include all dependencies in the dependency array to prevent stale closures and bugs."*