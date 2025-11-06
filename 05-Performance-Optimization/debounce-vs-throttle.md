# Explain Debounce vs Throttle

## Question
Explain Debounce vs Throttle.

## Answer

Debounce and throttle are performance optimization techniques used to control how often functions are executed when events fire frequently. They help prevent excessive function calls that can degrade performance, especially for events like scrolling, typing, mouse movement, and window resizing.

## Understanding the Problem

### 1. **Frequent Event Triggers**

Without optimization, events can fire hundreds of times per second:

```javascript
// Problem: Function called on every keystroke
document.getElementById('search').addEventListener('input', function(e) {
    searchAPI(e.target.value); // Called for every character typed
});

// Problem: Function called on every scroll pixel
window.addEventListener('scroll', function() {
    updateScrollPosition(); // Called dozens of times per second
});
```

### 2. **Performance Impact**

- **API calls** for every keystroke waste bandwidth
- **DOM manipulations** on every scroll cause jank
- **Heavy computations** run unnecessarily frequently
- **Battery drain** on mobile devices

## Debounce

### 1. **How Debounce Works**

Debounce delays function execution until a pause occurs in the event stream. It resets the timer every time the event fires.

**Visual representation:**
```
Event fires:     | | | |   | | |     |
Debounce (300ms): |-------|   |-------|
Function calls:         X       X
```

**Basic implementation:**
```javascript
function debounce(func, delay) {
    let timeoutId;

    return function(...args) {
        const context = this;

        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => {
            func.apply(context, args);
        }, delay);
    };
}

// Usage
const debouncedSearch = debounce(searchAPI, 300);

document.getElementById('search').addEventListener('input', function(e) {
    debouncedSearch(e.target.value);
});
```

### 2. **Debounce Use Cases**

**Search/Autocomplete:**
```typescript
import { useState, useCallback } from 'react';

function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);

    // Debounced search function
    const debouncedSearch = useCallback(
        debounce(async (searchTerm: string) => {
            if (searchTerm.length > 2) {
                const response = await fetch(`/api/search?q=${searchTerm}`);
                const data = await response.json();
                setResults(data);
            }
        }, 300),
        []
    );

    const handleInputChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        const value = e.target.value;
        setQuery(value);
        debouncedSearch(value);
    };

    return (
        <div>
            <input
                type="text"
                value={query}
                onChange={handleInputChange}
                placeholder="Search..."
            />
            <ul>
                {results.map(item => (
                    <li key={item.id}>{item.name}</li>
                ))}
            </ul>
        </div>
    );
}

function debounce<T extends (...args: any[]) => any>(
    func: T,
    delay: number
): (...args: Parameters<T>) => void {
    let timeoutId: NodeJS.Timeout;

    return (...args: Parameters<T>) => {
        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => func(...args), delay);
    };
}
```

**Form Validation:**
```typescript
function FormValidation() {
    const [email, setEmail] = useState('');
    const [isValid, setIsValid] = useState(true);

    const validateEmail = useCallback(
        debounce((emailValue: string) => {
            const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
            setIsValid(emailRegex.test(emailValue));
        }, 500),
        []
    );

    const handleEmailChange = (e) => {
        const value = e.target.value;
        setEmail(value);
        validateEmail(value);
    };

    return (
        <div>
            <input
                type="email"
                value={email}
                onChange={handleEmailChange}
                className={isValid ? '' : 'invalid'}
            />
            {!isValid && <span>Please enter a valid email</span>}
        </div>
    );
}
```

**Window Resize Handling:**
```typescript
function useWindowSize() {
    const [size, setSize] = useState({
        width: window.innerWidth,
        height: window.innerHeight
    });

    const handleResize = useCallback(
        debounce(() => {
            setSize({
                width: window.innerWidth,
                height: window.innerHeight
            });
        }, 250),
        []
    );

    useEffect(() => {
        window.addEventListener('resize', handleResize);
        return () => window.removeEventListener('resize', handleResize);
    }, [handleResize]);

    return size;
}
```

### 3. **Debounce Leading Edge**

Some implementations allow immediate execution on the first call:

```javascript
function debounce(func, delay, { leading = false } = {}) {
    let timeoutId;
    let lastExecTime = 0;

    return function(...args) {
        const context = this;
        const now = Date.now();

        if (leading && now - lastExecTime >= delay) {
            func.apply(context, args);
            lastExecTime = now;
        } else {
            clearTimeout(timeoutId);
            timeoutId = setTimeout(() => {
                if (!leading) {
                    func.apply(context, args);
                }
                lastExecTime = now;
            }, delay);
        }
    };
}

// Usage - execute immediately, then debounce subsequent calls
const debouncedSave = debounce(saveToServer, 1000, { leading: true });
```

## Throttle

### 1. **How Throttle Works**

Throttle limits function execution to once per time interval, regardless of how many times the event fires.

**Visual representation:**
```
Event fires:     | | | | | | | | | | |
Throttle (300ms): |-----|-----|-----|
Function calls:   X     X     X
```

**Basic implementation:**
```javascript
function throttle(func, limit) {
    let inThrottle;

    return function(...args) {
        const context = this;

        if (!inThrottle) {
            func.apply(context, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

// Usage
const throttledScroll = throttle(updateScrollPosition, 100);

window.addEventListener('scroll', throttledScroll);
```

### 2. **Throttle Use Cases**

**Scroll-based Animations:**
```typescript
function ScrollProgress() {
    const [scrollProgress, setScrollProgress] = useState(0);

    const handleScroll = useCallback(
        throttle(() => {
            const scrolled = (window.scrollY / (document.documentElement.scrollHeight - window.innerHeight)) * 100;
            setScrollProgress(scrolled);
        }, 16), // ~60fps
        []
    );

    useEffect(() => {
        window.addEventListener('scroll', handleScroll);
        return () => window.removeEventListener('scroll', handleScroll);
    }, [handleScroll]);

    return (
        <div className="progress-bar">
            <div
                className="progress-fill"
                style={{ width: `${scrollProgress}%` }}
            />
        </div>
    );
}

function throttle<T extends (...args: any[]) => any>(
    func: T,
    limit: number
): (...args: Parameters<T>) => void {
    let inThrottle: boolean;

    return (...args: Parameters<T>) => {
        if (!inThrottle) {
            func.apply(...args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}
```

**Mouse Movement Tracking:**
```typescript
function MouseTracker() {
    const [position, setPosition] = useState({ x: 0, y: 0 });

    const handleMouseMove = useCallback(
        throttle((e: MouseEvent) => {
            setPosition({ x: e.clientX, y: e.clientY });
        }, 50), // Update every 50ms
        []
    );

    useEffect(() => {
        window.addEventListener('mousemove', handleMouseMove);
        return () => window.removeEventListener('mousemove', handleMouseMove);
    }, [handleMouseMove]);

    return (
        <div>
            Mouse position: {position.x}, {position.y}
        </div>
    );
}
```

**Infinite Scroll:**
```typescript
function InfiniteScroll({ loadMore }) {
    const [isLoading, setIsLoading] = useState(false);

    const handleScroll = useCallback(
        throttle(() => {
            const { scrollTop, scrollHeight, clientHeight } = document.documentElement;

            if (scrollTop + clientHeight >= scrollHeight - 100 && !isLoading) {
                setIsLoading(true);
                loadMore().finally(() => setIsLoading(false));
            }
        }, 200),
        [isLoading, loadMore]
    );

    useEffect(() => {
        window.addEventListener('scroll', handleScroll);
        return () => window.removeEventListener('scroll', handleScroll);
    }, [handleScroll]);

    return (
        <div>
            {/* Content */}
            {isLoading && <div>Loading more...</div>}
        </div>
    );
}
```

### 3. **Throttle with Leading/Trailing Options**

More sophisticated throttle implementations:

```javascript
function throttle(func, limit, { leading = true, trailing = true } = {}) {
    let timeoutId;
    let lastExecTime = 0;

    return function(...args) {
        const context = this;
        const now = Date.now();

        if (!leading && !timeoutId) {
            // Set up trailing call
            timeoutId = setTimeout(() => {
                lastExecTime = Date.now();
                timeoutId = null;
                func.apply(context, args);
            }, limit);
        }

        if (leading && now - lastExecTime >= limit) {
            func.apply(context, args);
            lastExecTime = now;

            if (trailing) {
                timeoutId = setTimeout(() => {
                    lastExecTime = Date.now();
                    timeoutId = null;
                    func.apply(context, args);
                }, limit);
            }
        }
    };
}
```

## Debounce vs Throttle Comparison

### 1. **When Events Stop**

```
Debounce: Waits for pause, then executes once
Throttle: Executes immediately, then waits for interval
```

**Debounce behavior:**
- User types: "h" "he" "hel" "hell" "hello"
- Function calls: None until 300ms pause, then executes once with "hello"

**Throttle behavior:**
- User types: "h" "he" "hel" "hell" "hello"
- Function calls: Executes immediately with "h", then throttles subsequent calls

### 2. **Performance Characteristics**

| Aspect | Debounce | Throttle |
|--------|----------|----------|
| **Execution timing** | After event pause | At regular intervals |
| **First execution** | Delayed | Immediate |
| **Subsequent calls** | Reset timer | Ignored during limit |
| **Use case** | Completion-based | Rate-limiting |
| **API calls** | Good for search | Good for scroll |
| **Responsiveness** | Less immediate | More immediate |

### 3. **Choosing Between Them**

**Use Debounce when:**
- You want to wait for user to finish (search, form validation)
- The action should happen after a pause
- You want to batch rapid changes into one action
- API calls that should only happen when input stabilizes

**Use Throttle when:**
- You need consistent updates (scroll position, mouse tracking)
- The action should happen at regular intervals
- You want immediate response to first event
- Real-time feedback is important

## Advanced Patterns

### 1. **Combining Both**

Sometimes you need both behaviors:

```typescript
function useSmartSearch() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);

    // Throttle for immediate feedback (first few chars)
    const throttledSearch = useCallback(
        throttle(async (searchTerm: string) => {
            const quickResults = await quickSearchAPI(searchTerm);
            setResults(quickResults);
        }, 100),
        []
    );

    // Debounce for full search (after user pauses)
    const debouncedFullSearch = useCallback(
        debounce(async (searchTerm: string) => {
            const fullResults = await fullSearchAPI(searchTerm);
            setResults(fullResults);
        }, 500),
        []
    );

    const handleSearch = useCallback((searchTerm: string) => {
        setQuery(searchTerm);
        throttledSearch(searchTerm);
        debouncedFullSearch(searchTerm);
    }, [throttledSearch, debouncedFullSearch]);

    return { query, results, handleSearch };
}
```

### 2. **Dynamic Delays**

Adjust delay based on context:

```typescript
function useAdaptiveDebounce() {
    const [delay, setDelay] = useState(300);

    const adaptiveDebounce = useCallback((func, context) => {
        // Shorter delay for fast typers, longer for slow typers
        const newDelay = context === 'search' ? 150 : 500;
        setDelay(newDelay);

        return debounce(func, newDelay);
    }, []);

    return adaptiveDebounce;
}
```

### 3. **Canceling Debounced Functions**

```typescript
function useCancellableDebounce() {
    const debounceRef = useRef();

    const cancellableDebounce = useCallback((func, delay) => {
        if (debounceRef.current) {
            clearTimeout(debounceRef.current);
        }

        debounceRef.current = setTimeout(func, delay);
    }, []);

    const cancel = useCallback(() => {
        if (debounceRef.current) {
            clearTimeout(debounceRef.current);
        }
    }, []);

    useEffect(() => {
        return cancel; // Cleanup on unmount
    }, [cancel]);

    return { debounce: cancellableDebounce, cancel };
}
```

## Libraries and Utilities

### 1. **Lodash Implementations**

```javascript
import { debounce, throttle } from 'lodash';

// Debounce with options
const debouncedSearch = debounce(searchAPI, 300, {
    leading: false,  // Don't execute immediately
    trailing: true   // Execute after delay
});

// Throttle with options
const throttledScroll = throttle(updatePosition, 100, {
    leading: true,   // Execute on first call
    trailing: true   // Execute after interval
});
```

### 2. **React-Specific Hooks**

```typescript
// Custom hooks for React
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

## Testing Debounce and Throttle

### 1. **Testing Debounce**

```typescript
describe('debounce', () => {
    beforeEach(() => {
        jest.useFakeTimers();
    });

    it('should delay function execution', () => {
        const mockFn = jest.fn();
        const debouncedFn = debounce(mockFn, 300);

        debouncedFn();
        expect(mockFn).not.toHaveBeenCalled();

        jest.advanceTimersByTime(300);
        expect(mockFn).toHaveBeenCalledTimes(1);
    });

    it('should reset timer on multiple calls', () => {
        const mockFn = jest.fn();
        const debouncedFn = debounce(mockFn, 300);

        debouncedFn();
        jest.advanceTimersByTime(200);
        debouncedFn(); // Reset timer
        jest.advanceTimersByTime(200);
        expect(mockFn).not.toHaveBeenCalled();

        jest.advanceTimersByTime(100);
        expect(mockFn).toHaveBeenCalledTimes(1);
    });
});
```

### 2. **Testing Throttle**

```typescript
describe('throttle', () => {
    beforeEach(() => {
        jest.useFakeTimers();
    });

    it('should execute immediately on first call', () => {
        const mockFn = jest.fn();
        const throttledFn = throttle(mockFn, 300);

        throttledFn();
        expect(mockFn).toHaveBeenCalledTimes(1);
    });

    it('should prevent execution during throttle period', () => {
        const mockFn = jest.fn();
        const throttledFn = throttle(mockFn, 300);

        throttledFn();
        throttledFn();
        throttledFn();

        expect(mockFn).toHaveBeenCalledTimes(1);

        jest.advanceTimersByTime(300);
        throttledFn();
        expect(mockFn).toHaveBeenCalledTimes(2);
    });
});
```

## Common Interview Questions

### Q: What's the difference between debounce and throttle?

**A:** Debounce delays execution until events stop firing for a specified time, then executes once. Throttle executes immediately and then prevents further execution for the specified time interval. Use debounce for search inputs (wait for user to finish typing), use throttle for scroll events (regular updates).

### Q: When would you use debounce over throttle?

**A:** Use debounce when you want to wait for a pause in events before executing, like search autocomplete (don't want to search on every keystroke), form validation (validate after user stops typing), or API calls that should only happen when input stabilizes.

### Q: When would you use throttle over debounce?

**A:** Use throttle when you need immediate response but want to limit frequency, like scroll position tracking, mouse movement, drag operations, or infinite scroll loading. Throttle gives you consistent updates at regular intervals.

### Q: Can you combine debounce and throttle?

**A:** Yes, you can use both in the same application for different purposes. For example, use throttle for immediate feedback during typing and debounce for the final API call when the user pauses.

### Q: How do you handle cleanup for debounced/throttled functions?

**A:** Always clean up timers in useEffect cleanup functions or component unmount handlers. For debounced functions, you might also want to implement cancellation logic to prevent stale executions.

## Summary

**Debounce:**
- **Waits for pause** in event stream
- **Executes once** after delay when events stop
- **Good for:** Search, validation, API calls
- **Example:** Search autocomplete

**Throttle:**
- **Limits execution rate** to once per interval
- **Executes immediately**, then throttles
- **Good for:** Scroll, mouse tracking, real-time updates
- **Example:** Scroll position updates

**Key Differences:**
- Debounce: "Wait for quiet, then act"
- Throttle: "Act now, but not too often"

**Interview Tip:** "Debounce is like waiting for someone to finish speaking before responding, while throttle is like taking a sip of water every few minutes during a long speech. Choose debounce when you want to react to the end of activity, choose throttle when you want regular updates during ongoing activity."