# When should you use useRef instead of state?

## Question
When should you use useRef instead of state?

## Answer

`useRef` and `useState` are both React hooks for managing data, but they serve different purposes. `useRef` creates a mutable reference that persists across re-renders without triggering them, while `useState` manages state that triggers re-renders when updated. Knowing when to use each is crucial for performance and correct behavior.

## Key Differences Between useRef and useState

### useState
- **Triggers re-renders**: When state changes, component re-renders
- **Immutable updates**: State should be treated as immutable
- **For UI state**: Data that affects what the user sees

### useRef
- **No re-renders**: Changing ref value doesn't trigger re-render
- **Mutable**: Can be modified directly
- **For non-UI data**: DOM references, timers, previous values, etc.

```javascript
// useState: Triggers re-render
const [count, setCount] = useState(0);
setCount(count + 1); // Component re-renders

// useRef: No re-render
const countRef = useRef(0);
countRef.current += 1; // No re-render
```

## When to Use useRef

### 1. **DOM Element References**

When you need direct access to DOM elements:

```javascript
// ❌ Bad: Using state for DOM reference
function BadInput() {
    const [inputElement, setInputElement] = useState(null);

    return (
        <input
            ref={setInputElement} // This would cause re-renders!
            type="text"
        />
    );
}

// ✅ Good: Using ref for DOM reference
function GoodInput() {
    const inputRef = useRef(null);

    const focusInput = () => {
        inputRef.current.focus(); // Direct DOM access
    };

    return (
        <div>
            <input ref={inputRef} type="text" />
            <button onClick={focusInput}>Focus Input</button>
        </div>
    );
}
```

### 2. **Storing Previous Values**

When you need to compare current value with previous value:

```javascript
function usePrevious(value) {
    const ref = useRef();
    useEffect(() => {
        ref.current = value;
    });
    return ref.current;
}

function Counter() {
    const [count, setCount] = useState(0);
    const prevCount = usePrevious(count);

    useEffect(() => {
        if (prevCount !== undefined) {
            console.log(`Count changed from ${prevCount} to ${count}`);
        }
    }, [count, prevCount]);

    return (
        <div>
            <p>Current: {count}, Previous: {prevCount}</p>
            <button onClick={() => setCount(count + 1)}>Increment</button>
        </div>
    );
}
```

### 3. **Timer and Interval Management**

When managing timers that shouldn't trigger re-renders:

```javascript
// ❌ Bad: Timer ID in state causes unnecessary re-renders
function BadTimer() {
    const [timerId, setTimerId] = useState(null);
    const [count, setCount] = useState(0);

    const startTimer = () => {
        const id = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);
        setTimerId(id); // This causes a re-render!
    };

    const stopTimer = () => {
        if (timerId) {
            clearInterval(timerId);
            setTimerId(null); // Another re-render!
        }
    };

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={startTimer}>Start</button>
            <button onClick={stopTimer}>Stop</button>
        </div>
    );
}

// ✅ Good: Timer ID in ref
function GoodTimer() {
    const timerRef = useRef(null);
    const [count, setCount] = useState(0);

    const startTimer = () => {
        if (timerRef.current) return; // Already running

        timerRef.current = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);
    };

    const stopTimer = () => {
        if (timerRef.current) {
            clearInterval(timerRef.current);
            timerRef.current = null;
        }
    };

    // Cleanup on unmount
    useEffect(() => {
        return () => {
            if (timerRef.current) {
                clearInterval(timerRef.current);
            }
        };
    }, []);

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={startTimer}>Start</button>
            <button onClick={stopTimer}>Stop</button>
        </div>
    );
}
```

### 4. **Animation Values**

When storing animation state that changes frequently:

```javascript
function DraggableBox() {
    const [position, setPosition] = useState({ x: 0, y: 0 });
    const animationRef = useRef(null);
    const velocityRef = useRef({ x: 0, y: 0 });

    const startDrag = (e) => {
        const startX = e.clientX - position.x;
        const startY = e.clientY - position.y;

        const handleMouseMove = (e) => {
            const newX = e.clientX - startX;
            const newY = e.clientY - startY;

            // Update animation values without re-renders
            velocityRef.current = {
                x: newX - position.x,
                y: newY - position.y
            };

            setPosition({ x: newX, y: newY });
        };

        const handleMouseUp = () => {
            document.removeEventListener('mousemove', handleMouseMove);
            document.removeEventListener('mouseup', handleMouseUp);

            // Start momentum animation
            animateMomentum();
        };

        document.addEventListener('mousemove', handleMouseMove);
        document.addEventListener('mouseup', handleMouseUp);
    };

    const animateMomentum = () => {
        if (Math.abs(velocityRef.current.x) < 0.1 &&
            Math.abs(velocityRef.current.y) < 0.1) {
            return; // Stop animation
        }

        // Update position with momentum
        setPosition(prev => ({
            x: prev.x + velocityRef.current.x,
            y: prev.y + velocityRef.current.y
        }));

        // Apply friction
        velocityRef.current.x *= 0.95;
        velocityRef.current.y *= 0.95;

        animationRef.current = requestAnimationFrame(animateMomentum);
    };

    useEffect(() => {
        return () => {
            if (animationRef.current) {
                cancelAnimationFrame(animationRef.current);
            }
        };
    }, []);

    return (
        <div
            style={{
                position: 'absolute',
                left: position.x,
                top: position.y,
                width: '100px',
                height: '100px',
                background: 'blue',
                cursor: 'grab'
            }}
            onMouseDown={startDrag}
        >
            Drag me
        </div>
    );
}
```

### 5. **Caching Expensive Computations**

When caching values that don't need to trigger re-renders:

```javascript
function ExpensiveList({ items, filter }) {
    const cacheRef = useRef(new Map());

    const filteredItems = useMemo(() => {
        const cacheKey = `${filter}-${items.length}`;

        if (cacheRef.current.has(cacheKey)) {
            return cacheRef.current.get(cacheKey);
        }

        // Expensive filtering operation
        const result = items.filter(item =>
            item.name.toLowerCase().includes(filter.toLowerCase())
        );

        cacheRef.current.set(cacheKey, result);
        return result;
    }, [items, filter]);

    return (
        <ul>
            {filteredItems.map(item => (
                <li key={item.id}>{item.name}</li>
            ))}
        </ul>
    );
}
```

### 6. **WebSocket or Event Source References**

When managing WebSocket connections:

```javascript
function useWebSocket(url) {
    const [messages, setMessages] = useState([]);
    const [isConnected, setIsConnected] = useState(false);
    const wsRef = useRef(null);
    const reconnectTimeoutRef = useRef(null);

    useEffect(() => {
        const connect = () => {
            wsRef.current = new WebSocket(url);

            wsRef.current.onopen = () => {
                setIsConnected(true);
            };

            wsRef.current.onmessage = (event) => {
                const message = JSON.parse(event.data);
                setMessages(prev => [...prev, message]);
            };

            wsRef.current.onclose = () => {
                setIsConnected(false);
                // Reconnect logic
                reconnectTimeoutRef.current = setTimeout(connect, 5000);
            };
        };

        connect();

        return () => {
            if (wsRef.current) {
                wsRef.current.close();
            }
            if (reconnectTimeoutRef.current) {
                clearTimeout(reconnectTimeoutRef.current);
            }
        };
    }, [url]);

    const sendMessage = (message) => {
        if (wsRef.current && wsRef.current.readyState === WebSocket.OPEN) {
            wsRef.current.send(JSON.stringify(message));
        }
    };

    return { messages, isConnected, sendMessage };
}
```

### 7. **Form Validation State**

When managing form validation without triggering re-renders:

```javascript
function useFormValidation(initialValues) {
    const [values, setValues] = useState(initialValues);
    const [errors, setErrors] = useState({});
    const touchedRef = useRef(new Set()); // Track touched fields

    const handleChange = (field) => (e) => {
        const value = e.target.value;
        setValues(prev => ({ ...prev, [field]: value }));

        // Mark field as touched
        touchedRef.current.add(field);

        // Clear error for this field
        if (errors[field]) {
            setErrors(prev => ({ ...prev, [field]: '' }));
        }
    };

    const validate = () => {
        const newErrors = {};
        const touchedFields = Array.from(touchedRef.current);

        // Only validate touched fields
        touchedFields.forEach(field => {
            if (!values[field]) {
                newErrors[field] = `${field} is required`;
            }
        });

        setErrors(newErrors);
        return Object.keys(newErrors).length === 0;
    };

    const reset = () => {
        setValues(initialValues);
        setErrors({});
        touchedRef.current.clear();
    };

    return {
        values,
        errors,
        handleChange,
        validate,
        reset,
        touchedFields: Array.from(touchedRef.current)
    };
}
```

### 8. **Scroll Position Management**

When tracking scroll position without re-renders:

```javascript
function useScrollPosition() {
    const elementRef = useRef(null);
    const scrollPositionRef = useRef({ x: 0, y: 0 });

    const updateScrollPosition = useCallback(() => {
        if (elementRef.current) {
            scrollPositionRef.current = {
                x: elementRef.current.scrollLeft,
                y: elementRef.current.scrollTop
            };
        }
    }, []);

    useEffect(() => {
        const element = elementRef.current;
        if (element) {
            element.addEventListener('scroll', updateScrollPosition);
            return () => element.removeEventListener('scroll', updateScrollPosition);
        }
    }, [updateScrollPosition]);

    const scrollTo = useCallback((x, y) => {
        if (elementRef.current) {
            elementRef.current.scrollTo(x, y);
        }
    }, []);

    const getScrollPosition = useCallback(() => {
        return scrollPositionRef.current;
    }, []);

    return { elementRef, scrollTo, getScrollPosition };
}

function ScrollableList({ items }) {
    const { elementRef, scrollTo, getScrollPosition } = useScrollPosition();

    const scrollToTop = () => scrollTo(0, 0);
    const scrollToBottom = () => scrollTo(0, elementRef.current.scrollHeight);

    return (
        <div>
            <button onClick={scrollToTop}>Scroll to Top</button>
            <button onClick={scrollToBottom}>Scroll to Bottom</button>
            <div
                ref={elementRef}
                style={{ height: '300px', overflow: 'auto' }}
            >
                {items.map(item => (
                    <div key={item.id} style={{ height: '50px' }}>
                        {item.name}
                    </div>
                ))}
            </div>
        </div>
    );
}
```

## Advanced useRef Patterns

### 1. **useImperativeHandle with Refs**

```javascript
const FancyInput = forwardRef((props, ref) => {
    const inputRef = useRef();

    useImperativeHandle(ref, () => ({
        focus: () => inputRef.current.focus(),
        blur: () => inputRef.current.blur(),
        getValue: () => inputRef.current.value,
        setValue: (value) => inputRef.current.value = value
    }));

    return <input ref={inputRef} {...props} />;
});

// Usage
function Form() {
    const inputRef = useRef();

    const handleSubmit = () => {
        const value = inputRef.current.getValue();
        console.log('Form value:', value);
    };

    return (
        <div>
            <FancyInput ref={inputRef} />
            <button onClick={handleSubmit}>Submit</button>
        </div>
    );
}
```

### 2. **Ref as Instance Variable**

```javascript
function useInstanceVar(initialValue) {
    const ref = useRef(initialValue);
    return ref.current;
}

function Counter() {
    const count = useInstanceVar(0);

    const increment = () => {
        count += 1;
        console.log('Count:', count);
    };

    return (
        <div>
            <button onClick={increment}>Increment</button>
            <p>Check console for count value</p>
        </div>
    );
}
```

### 3. **Combining useRef with useState**

```javascript
function SmartInput({ value, onChange }) {
    const inputRef = useRef();
    const [localValue, setLocalValue] = useState(value);
    const timeoutRef = useRef();

    const handleChange = (e) => {
        const newValue = e.target.value;
        setLocalValue(newValue);

        // Debounce the onChange callback
        if (timeoutRef.current) {
            clearTimeout(timeoutRef.current);
        }

        timeoutRef.current = setTimeout(() => {
            onChange(newValue);
        }, 300);
    };

    // Sync with external value changes
    useEffect(() => {
        setLocalValue(value);
    }, [value]);

    return (
        <input
            ref={inputRef}
            value={localValue}
            onChange={handleChange}
        />
    );
}
```

## Common Mistakes and Solutions

### 1. **Using useRef for UI State**

```javascript
// ❌ Bad: Using ref for UI state
function BadCounter() {
    const countRef = useRef(0);

    const increment = () => {
        countRef.current += 1;
        // UI doesn't update!
    };

    return <div>Count: {countRef.current}</div>; // Always shows 0
}

// ✅ Good: Use state for UI state
function GoodCounter() {
    const [count, setCount] = useState(0);

    const increment = () => {
        setCount(count + 1); // Triggers re-render
    };

    return <div>Count: {count}</div>; // Updates correctly
}
```

### 2. **Forgetting .current**

```javascript
// ❌ Bad: Accessing ref without .current
const myRef = useRef(0);
console.log(myRef.value); // undefined

// ✅ Good: Use .current
console.log(myRef.current); // 0
```

### 3. **Using useRef in Conditional Logic**

```javascript
// ❌ Bad: Conditional ref usage
function BadComponent({ condition }) {
    if (condition) {
        const myRef = useRef(); // Violates rules of hooks
    }
    return <div />;
}

// ✅ Good: Always declare refs
function GoodComponent({ condition }) {
    const myRef = useRef();

    useEffect(() => {
        if (condition && myRef.current) {
            // Use ref here
        }
    }, [condition]);

    return <div ref={condition ? myRef : undefined} />;
}
```

### 4. **Not Cleaning Up Refs**

```javascript
// ❌ Bad: Refs not cleaned up
function BadTimer() {
    const intervalRef = useRef();

    useEffect(() => {
        intervalRef.current = setInterval(() => {
            console.log('tick');
        }, 1000);
        // No cleanup - memory leak!
    }, []);

    return <div>Timer running</div>;
}

// ✅ Good: Proper cleanup
function GoodTimer() {
    const intervalRef = useRef();

    useEffect(() => {
        intervalRef.current = setInterval(() => {
            console.log('tick');
        }, 1000);

        return () => {
            if (intervalRef.current) {
                clearInterval(intervalRef.current);
            }
        };
    }, []);

    return <div>Timer running</div>;
}
```

## Performance Considerations

### 1. **When useRef is Faster**

```javascript
// For frequent updates that don't affect UI
function FrequentUpdater() {
    const updateCountRef = useRef(0);

    useEffect(() => {
        const interval = setInterval(() => {
            updateCountRef.current += 1;
            // No re-render triggered
        }, 100);

        return () => clearInterval(interval);
    }, []);

    // Only show count every second
    const [displayCount, setDisplayCount] = useState(0);

    useEffect(() => {
        const interval = setInterval(() => {
            setDisplayCount(updateCountRef.current);
        }, 1000);

        return () => clearInterval(interval);
    }, []);

    return <div>Updates: {displayCount}</div>;
}
```

### 2. **Memory Management**

```javascript
// Refs can accumulate memory if not managed
function useCache() {
    const cacheRef = useRef(new Map());

    const get = useCallback((key) => {
        return cacheRef.current.get(key);
    }, []);

    const set = useCallback((key, value) => {
        // Limit cache size to prevent memory leaks
        if (cacheRef.current.size > 100) {
            const firstKey = cacheRef.current.keys().next().value;
            cacheRef.current.delete(firstKey);
        }
        cacheRef.current.set(key, value);
    }, []);

    const clear = useCallback(() => {
        cacheRef.current.clear();
    }, []);

    return { get, set, clear };
}
```

## Real React Examples

### 1. **Auto-focus Input**

```javascript
function AutoFocusInput() {
    const inputRef = useRef();

    useEffect(() => {
        inputRef.current.focus();
    }, []);

    return <input ref={inputRef} placeholder="Auto-focused input" />;
}
```

### 2. **Video Player Controls**

```javascript
function VideoPlayer({ src }) {
    const videoRef = useRef();

    const play = () => videoRef.current.play();
    const pause = () => videoRef.current.pause();
    const setVolume = (volume) => videoRef.current.volume = volume;

    return (
        <div>
            <video ref={videoRef} src={src} />
            <div>
                <button onClick={play}>Play</button>
                <button onClick={pause}>Pause</button>
                <input
                    type="range"
                    min="0"
                    max="1"
                    step="0.1"
                    onChange={(e) => setVolume(e.target.value)}
                />
            </div>
        </div>
    );
}
```

### 3. **Canvas Drawing**

```javascript
function DrawingCanvas() {
    const canvasRef = useRef();
    const isDrawingRef = useRef(false);
    const lastPosRef = useRef({ x: 0, y: 0 });

    useEffect(() => {
        const canvas = canvasRef.current;
        const ctx = canvas.getContext('2d');

        const startDrawing = (e) => {
            isDrawingRef.current = true;
            lastPosRef.current = { x: e.offsetX, y: e.offsetY };
        };

        const draw = (e) => {
            if (!isDrawingRef.current) return;

            ctx.beginPath();
            ctx.moveTo(lastPosRef.current.x, lastPosRef.current.y);
            ctx.lineTo(e.offsetX, e.offsetY);
            ctx.stroke();

            lastPosRef.current = { x: e.offsetX, y: e.offsetY };
        };

        const stopDrawing = () => {
            isDrawingRef.current = false;
        };

        canvas.addEventListener('mousedown', startDrawing);
        canvas.addEventListener('mousemove', draw);
        canvas.addEventListener('mouseup', stopDrawing);
        canvas.addEventListener('mouseout', stopDrawing);

        return () => {
            canvas.removeEventListener('mousedown', startDrawing);
            canvas.removeEventListener('mousemove', draw);
            canvas.removeEventListener('mouseup', stopDrawing);
            canvas.removeEventListener('mouseout', stopDrawing);
        };
    }, []);

    return <canvas ref={canvasRef} width={400} height={300} />;
}
```

### 4. **Intersection Observer**

```javascript
function useIntersectionObserver(callback, options = {}) {
    const elementRef = useRef();
    const observerRef = useRef();

    useEffect(() => {
        if (observerRef.current) {
            observerRef.current.disconnect();
        }

        observerRef.current = new IntersectionObserver(callback, options);

        if (elementRef.current) {
            observerRef.current.observe(elementRef.current);
        }

        return () => {
            if (observerRef.current) {
                observerRef.current.disconnect();
            }
        };
    }, [callback, options]);

    return elementRef;
}

function LazyImage({ src, alt }) {
    const [isLoaded, setIsLoaded] = useState(false);
    const [isInView, setIsInView] = useState(false);

    const elementRef = useIntersectionObserver(
        ([entry]) => {
            if (entry.isIntersecting) {
                setIsInView(true);
            }
        },
        { threshold: 0.1 }
    );

    return (
        <img
            ref={elementRef}
            src={isInView ? src : ''}
            alt={alt}
            onLoad={() => setIsLoaded(true)}
            style={{ opacity: isLoaded ? 1 : 0.5 }}
        />
    );
}
```

### Interview Tip:
*"Use useRef for DOM references, storing previous values, timers/intervals, animation values, and any mutable data that doesn't need to trigger re-renders. Use useState for UI state that affects what users see and triggers re-renders when changed."*