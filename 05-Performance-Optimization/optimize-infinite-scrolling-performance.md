# How to optimize infinite scrolling performance?

## Question
How to optimize infinite scrolling performance?

## Answer

Optimizing infinite scrolling performance requires careful management of memory usage, rendering efficiency, and user experience. The key challenges are preventing memory leaks, maintaining smooth scrolling, and handling large datasets without performance degradation. Here are comprehensive strategies to optimize infinite scrolling implementations.

## Core Performance Challenges

### **Memory Accumulation**
- **Problem:** Items accumulate in DOM and memory
- **Impact:** Browser slowdown, crashes with large datasets
- **Solution:** Implement item virtualization and cleanup

### **Scroll Performance**
- **Problem:** Large DOM trees cause janky scrolling
- **Impact:** Poor user experience, high CPU usage
- **Solution:** Virtual scrolling and efficient rendering

### **Network Efficiency**
- **Problem:** Unnecessary API calls, duplicate requests
- **Impact:** Slow loading, wasted bandwidth
- **Solution:** Smart prefetching and caching

## Memory Management Strategies

### 1. **Item Cleanup and Recycling**

**Remove old items to prevent memory leaks:**

```typescript
function useMemoryEfficientInfiniteScroll(maxItems = 1000) {
    const [items, setItems] = useState([]);
    const [startIndex, setStartIndex] = useState(0);

    const addItems = useCallback((newItems) => {
        setItems(prev => {
            const combined = [...prev, ...newItems];

            // Keep only the most recent items
            if (combined.length > maxItems) {
                const excess = combined.length - maxItems;
                setStartIndex(prev => prev + excess);
                return combined.slice(excess);
            }

            return combined;
        });
    }, [maxItems]);

    const reset = useCallback(() => {
        setItems([]);
        setStartIndex(0);
    }, []);

    return {
        visibleItems: items,
        addItems,
        reset,
        totalLoaded: startIndex + items.length
    };
}

// Usage
function OptimizedInfiniteList() {
    const { visibleItems, addItems, totalLoaded } = useMemoryEfficientInfiniteScroll(500);

    const loadMore = async () => {
        const newItems = await fetchItems(totalLoaded, 50);
        addItems(newItems);
    };

    return (
        <div>
            <div className="total-loaded">Loaded {totalLoaded} items</div>
            <VirtualizedList items={visibleItems} loadMore={loadMore} />
        </div>
    );
}
```

### 2. **Sliding Window Technique**

**Maintain a fixed-size window of visible items:**

```typescript
function useSlidingWindow(windowSize = 100) {
    const [items, setItems] = useState([]);
    const [windowStart, setWindowStart] = useState(0);
    const [totalItems, setTotalItems] = useState(0);

    const visibleItems = items.slice(windowStart, windowStart + windowSize);

    const shiftWindow = useCallback((direction, amount = windowSize / 2) => {
        if (direction === 'up' && windowStart > 0) {
            setWindowStart(prev => Math.max(0, prev - amount));
        } else if (direction === 'down' && windowStart + windowSize < totalItems) {
            setWindowStart(prev => Math.min(totalItems - windowSize, prev + amount));
        }
    }, [windowSize, totalItems]);

    const loadItems = useCallback(async (start, count) => {
        const newItems = await fetchItems(start, count);
        setItems(prev => {
            const updated = [...prev];
            // Replace items in the range
            for (let i = 0; i < newItems.length; i++) {
                updated[start + i] = newItems[i];
            }
            return updated;
        });
    }, []);

    return {
        visibleItems,
        shiftWindow,
        loadItems,
        windowStart,
        totalItems
    };
}
```

### 3. **Garbage Collection Optimization**

**Help the browser clean up unused resources:**

```typescript
function useGarbageCollection() {
    const gcTimer = useRef();

    const scheduleGC = useCallback(() => {
        clearTimeout(gcTimer.current);
        gcTimer.current = setTimeout(() => {
            // Force garbage collection hint (if available)
            if (window.gc) {
                window.gc();
            }

            // Clear cached images/videos
            const images = document.querySelectorAll('img[data-lazy]');
            images.forEach(img => {
                if (img.src && !img.isIntersecting) {
                    img.src = ''; // Clear source to free memory
                }
            });
        }, 5000); // Delay to avoid interfering with user interaction
    }, []);

    useEffect(() => {
        return () => clearTimeout(gcTimer.current);
    }, []);

    return { scheduleGC };
}
```

## Virtual Scrolling Implementation

### 1. **Basic Virtual Scrolling**

**Render only visible items with absolute positioning:**

```typescript
function VirtualizedInfiniteScroll({
    items,
    itemHeight = 50,
    containerHeight = 400,
    buffer = 5
}) {
    const [scrollTop, setScrollTop] = useState(0);

    // Calculate visible range
    const startIndex = Math.max(0,
        Math.floor(scrollTop / itemHeight) - buffer
    );
    const endIndex = Math.min(items.length - 1,
        startIndex + Math.ceil(containerHeight / itemHeight) + buffer * 2
    );

    const visibleItems = items.slice(startIndex, endIndex);
    const offsetY = startIndex * itemHeight;

    return (
        <div
            className="virtual-container"
            style={{
                height: containerHeight,
                overflow: 'auto',
                position: 'relative'
            }}
            onScroll={(e) => setScrollTop(e.target.scrollTop)}
        >
            {/* Total height placeholder */}
            <div style={{ height: items.length * itemHeight, position: 'relative' }}>
                {/* Visible items container */}
                <div style={{ transform: `translateY(${offsetY}px)` }}>
                    {visibleItems.map((item, index) => (
                        <div
                            key={startIndex + index}
                            style={{
                                height: itemHeight,
                                display: 'flex',
                                alignItems: 'center',
                                padding: '0 16px'
                            }}
                        >
                            <ItemComponent item={item} />
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
}
```

### 2. **Advanced Virtual Scrolling with react-window**

**Using react-window for optimized performance:**

```typescript
import { FixedSizeList as List } from 'react-window';
import InfiniteLoader from 'react-window-infinite-loader';

function ReactWindowInfiniteScroll({ itemCount, loadMoreItems, itemHeight }) {
    // Check if item is loaded
    const isItemLoaded = (index) => index < itemCount;

    // Load more items when approaching the end
    const loadMore = (startIndex, stopIndex) => {
        return loadMoreItems(startIndex, stopIndex);
    };

    // Item renderer
    const Item = ({ index, style }) => (
        <div style={style}>
            {isItemLoaded(index) ? (
                <ItemComponent item={items[index]} />
            ) : (
                <SkeletonItem />
            )}
        </div>
    );

    return (
        <InfiniteLoader
            isItemLoaded={isItemLoaded}
            itemCount={itemCount + 1} // +1 for loading indicator
            loadMoreItems={loadMore}
            threshold={10} // Load more when 10 items from end
        >
            {({ onItemsRendered, ref }) => (
                <List
                    ref={ref}
                    height={400}
                    itemCount={itemCount}
                    itemSize={itemHeight}
                    onItemsRendered={onItemsRendered}
                    width="100%"
                >
                    {Item}
                </List>
            )}
        </InfiniteLoader>
    );
}
```

### 3. **Variable Height Virtual Scrolling**

**Handle items with different heights:**

```typescript
function VariableHeightVirtualScroll({ items }) {
    const [heights, setHeights] = useState(new Map());
    const [scrollTop, setScrollTop] = useState(0);
    const containerHeight = 400;

    // Calculate total height
    const getTotalHeight = () => {
        let total = 0;
        for (let i = 0; i < items.length; i++) {
            total += heights.get(i) || 50; // Default height
        }
        return total;
    };

    // Find visible range
    const getVisibleRange = () => {
        let currentHeight = 0;
        let startIndex = 0;

        // Find start index
        while (currentHeight < scrollTop && startIndex < items.length) {
            currentHeight += heights.get(startIndex) || 50;
            startIndex++;
        }
        startIndex = Math.max(0, startIndex - 2);

        // Find end index
        let endIndex = startIndex;
        let visibleHeight = 0;
        while (visibleHeight < containerHeight && endIndex < items.length) {
            visibleHeight += heights.get(endIndex) || 50;
            endIndex++;
        }
        endIndex = Math.min(items.length, endIndex + 2);

        return { startIndex, endIndex };
    };

    const { startIndex, endIndex } = getVisibleRange();
    const visibleItems = items.slice(startIndex, endIndex);

    // Calculate offset
    let offsetY = 0;
    for (let i = 0; i < startIndex; i++) {
        offsetY += heights.get(i) || 50;
    }

    // Measure item heights
    const measureItem = (index, height) => {
        setHeights(prev => new Map(prev).set(index, height));
    };

    return (
        <div
            style={{ height: containerHeight, overflow: 'auto' }}
            onScroll={(e) => setScrollTop(e.target.scrollTop)}
        >
            <div style={{ height: getTotalHeight(), position: 'relative' }}>
                <div style={{ transform: `translateY(${offsetY}px)` }}>
                    {visibleItems.map((item, index) => (
                        <MeasuredItem
                            key={startIndex + index}
                            item={item}
                            index={startIndex + index}
                            onMeasure={measureItem}
                        />
                    ))}
                </div>
            </div>
        </div>
    );
}
```

## Network Optimization

### 1. **Smart Prefetching**

**Load data before it's needed:**

```typescript
function useSmartPrefetch(prefetchDistance = 5) {
    const [prefetchedPages, setPrefetchedPages] = useState(new Set());

    const prefetchPage = useCallback(async (pageNumber) => {
        if (prefetchedPages.has(pageNumber)) return;

        try {
            await fetchItems(pageNumber * pageSize, pageSize);
            setPrefetchedPages(prev => new Set([...prev, pageNumber]));
        } catch (error) {
            console.warn('Prefetch failed:', error);
        }
    }, [prefetchedPages]);

    const prefetchNearbyPages = useCallback((currentPage) => {
        // Prefetch next few pages
        for (let i = 1; i <= prefetchDistance; i++) {
            prefetchPage(currentPage + i);
        }

        // Prefetch previous pages (less aggressively)
        for (let i = 1; i <= Math.floor(prefetchDistance / 2); i++) {
            if (currentPage - i >= 0) {
                prefetchPage(currentPage - i);
            }
        }
    }, [prefetchPage, prefetchDistance]);

    return { prefetchNearbyPages };
}
```

### 2. **Request Deduplication**

**Prevent duplicate API calls:**

```typescript
function useRequestDedupe() {
    const pendingRequests = useRef(new Map());

    const dedupedFetch = useCallback(async (key, fetchFn) => {
        if (pendingRequests.current.has(key)) {
            return pendingRequests.current.get(key);
        }

        const promise = fetchFn().finally(() => {
            pendingRequests.current.delete(key);
        });

        pendingRequests.current.set(key, promise);
        return promise;
    }, []);

    const clearPending = useCallback((key) => {
        pendingRequests.current.delete(key);
    }, []);

    return { dedupedFetch, clearPending };
}

// Usage
function OptimizedInfiniteList() {
    const { dedupedFetch } = useRequestDedupe();

    const loadMore = async (page) => {
        const cacheKey = `page-${page}`;
        return dedupedFetch(cacheKey, () => fetchItems(page * 50, 50));
    };

    // ...
}
```

### 3. **Adaptive Loading**

**Adjust loading strategy based on connection speed:**

```typescript
function useAdaptiveLoading() {
    const [connectionSpeed, setConnectionSpeed] = useState('unknown');

    useEffect(() => {
        // Estimate connection speed
        const connection = navigator.connection ||
                          navigator.mozConnection ||
                          navigator.webkitConnection;

        if (connection) {
            const updateSpeed = () => {
                if (connection.effectiveType === '4g') {
                    setConnectionSpeed('fast');
                } else if (connection.effectiveType === '3g') {
                    setConnectionSpeed('slow');
                } else {
                    setConnectionSpeed('very-slow');
                }
            };

            updateSpeed();
            connection.addEventListener('change', updateSpeed);

            return () => connection.removeEventListener('change', updateSpeed);
        }
    }, []);

    const getBatchSize = () => {
        switch (connectionSpeed) {
            case 'fast': return 50;
            case 'slow': return 20;
            case 'very-slow': return 10;
            default: return 25;
        }
    };

    const getPrefetchDistance = () => {
        switch (connectionSpeed) {
            case 'fast': return 10;
            case 'slow': return 3;
            case 'very-slow': return 1;
            default: return 5;
        }
    };

    return { batchSize: getBatchSize(), prefetchDistance: getPrefetchDistance() };
}
```

## Scroll Performance Optimization

### 1. **Throttle Scroll Events**

**Prevent excessive event firing:**

```typescript
function useThrottledScroll(callback, delay = 16) {
    const throttledCallback = useRef();
    const lastExecuted = useRef(Date.now());

    useEffect(() => {
        throttledCallback.current = callback;
    }, [callback]);

    return useCallback((event) => {
        if (Date.now() - lastExecuted.current >= delay) {
            lastExecuted.current = Date.now();
            throttledCallback.current(event);
        }
    }, [delay]);
}

// Usage
function SmoothInfiniteScroll() {
    const handleScroll = useThrottledScroll((e) => {
        const scrollTop = e.target.scrollTop;
        // Update virtual scroll position
        setScrollTop(scrollTop);
    }, 16); // ~60fps

    return (
        <div onScroll={handleScroll}>
            {/* Virtualized content */}
        </div>
    );
}
```

### 2. **Intersection Observer for Loading**

**More efficient than scroll event listeners:**

```typescript
function useIntersectionObserver(callback, options = {}) {
    const [target, setTarget] = useState(null);
    const observer = useRef();

    useEffect(() => {
        if (!target) return;

        observer.current = new IntersectionObserver(
            (entries) => {
                entries.forEach((entry) => {
                    if (entry.isIntersecting) {
                        callback(entry);
                    }
                });
            },
            {
                root: null,
                rootMargin: '100px', // Load 100px before reaching
                threshold: 0.1,
                ...options
            }
        );

        observer.current.observe(target);

        return () => {
            if (observer.current) {
                observer.current.disconnect();
            }
        };
    }, [target, callback, options]);

    return setTarget;
}

// Usage
function IntersectionInfiniteScroll() {
    const loadMoreTrigger = useIntersectionObserver(() => {
        loadMoreItems();
    });

    return (
        <div>
            <VirtualizedList items={items} />
            <div ref={loadMoreTrigger} className="load-more-trigger" />
        </div>
    );
}
```

### 3. **Passive Event Listeners**

**Improve scroll performance:**

```typescript
function usePassiveScroll(callback) {
    return useCallback((event) => {
        // Use passive listeners for better performance
        callback(event);
    }, [callback]);
}

// In component
useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const handleScroll = usePassiveScroll((e) => {
        // Handle scroll logic
    });

    container.addEventListener('scroll', handleScroll, { passive: true });

    return () => {
        container.removeEventListener('scroll', handleScroll);
    };
}, []);
```

## Caching and State Management

### 1. **Intelligent Caching**

**Cache items and manage cache size:**

```typescript
function useIntelligentCache(maxCacheSize = 1000) {
    const [cache, setCache] = useState(new Map());
    const [accessOrder, setAccessOrder] = useState([]);

    const get = useCallback((key) => {
        if (cache.has(key)) {
            // Move to end (most recently used)
            setAccessOrder(prev => [...prev.filter(k => k !== key), key]);
            return cache.get(key);
        }
        return null;
    }, [cache]);

    const set = useCallback((key, value) => {
        setCache(prev => {
            const newCache = new Map(prev);

            // Update access order
            setAccessOrder(current => [...current.filter(k => k !== key), key]);

            // Add new item
            newCache.set(key, value);

            // Evict least recently used items if cache is full
            if (newCache.size > maxCacheSize) {
                const toEvict = accessOrder[0];
                newCache.delete(toEvict);
                setAccessOrder(current => current.slice(1));
            }

            return newCache;
        });
    }, [maxCacheSize]);

    return { get, set, clear: () => setCache(new Map()) };
}
```

### 2. **React Query Integration**

**Leverage React Query's caching:**

```typescript
import { useInfiniteQuery } from 'react-query';

function useOptimizedInfiniteQuery(queryKey, fetchFn, options = {}) {
    return useInfiniteQuery(
        queryKey,
        ({ pageParam = 0 }) => fetchFn(pageParam),
        {
            getNextPageParam: (lastPage, pages) => {
                return lastPage.length === 0 ? undefined : pages.length * 50;
            },
            staleTime: 5 * 60 * 1000, // 5 minutes
            cacheTime: 10 * 60 * 1000, // 10 minutes
            refetchOnWindowFocus: false,
            refetchOnMount: false,
            ...options
        }
    );
}

// Usage
function ReactQueryInfiniteList() {
    const {
        data,
        fetchNextPage,
        hasNextPage,
        isFetchingNextPage
    } = useOptimizedInfiniteQuery(
        ['items'],
        ({ pageParam }) => fetchItems(pageParam, 50)
    );

    const items = data?.pages.flat() || [];

    return (
        <VirtualizedList
            items={items}
            loadMore={fetchNextPage}
            hasMore={hasNextPage}
            loading={isFetchingNextPage}
        />
    );
}
```

## Performance Monitoring

### 1. **Scroll Performance Metrics**

**Monitor scroll smoothness:**

```typescript
function useScrollPerformance() {
    const [metrics, setMetrics] = useState({
        averageFrameTime: 0,
        droppedFrames: 0,
        scrollEventsPerSecond: 0
    });

    useEffect(() => {
        let frameCount = 0;
        let totalFrameTime = 0;
        let droppedFrames = 0;
        let scrollEvents = 0;
        let lastTime = performance.now();

        const measureFrame = (timestamp) => {
            const frameTime = timestamp - lastTime;
            totalFrameTime += frameTime;
            frameCount++;

            // Detect dropped frames (frame time > 16.67ms at 60fps)
            if (frameTime > 16.67) {
                droppedFrames++;
            }

            lastTime = timestamp;
            requestAnimationFrame(measureFrame);
        };

        const measureScroll = () => {
            scrollEvents++;
        };

        // Start measuring
        requestAnimationFrame(measureFrame);
        window.addEventListener('scroll', measureScroll, { passive: true });

        // Update metrics every second
        const interval = setInterval(() => {
            setMetrics({
                averageFrameTime: totalFrameTime / frameCount,
                droppedFrames,
                scrollEventsPerSecond: scrollEvents
            });

            // Reset counters
            frameCount = 0;
            totalFrameTime = 0;
            droppedFrames = 0;
            scrollEvents = 0;
        }, 1000);

        return () => {
            clearInterval(interval);
            window.removeEventListener('scroll', measureScroll);
        };
    }, []);

    return metrics;
}
```

### 2. **Memory Usage Tracking**

**Monitor memory consumption:**

```typescript
function useMemoryMonitor() {
    const [memoryUsage, setMemoryUsage] = useState({
        used: 0,
        total: 0,
        limit: 0
    });

    useEffect(() => {
        const updateMemory = () => {
            if (performance.memory) {
                setMemoryUsage({
                    used: performance.memory.usedJSHeapSize,
                    total: performance.memory.totalJSHeapSize,
                    limit: performance.memory.jsHeapSizeLimit
                });
            }
        };

        updateMemory();
        const interval = setInterval(updateMemory, 5000);

        return () => clearInterval(interval);
    }, []);

    const memoryPercentage = memoryUsage.limit ?
        (memoryUsage.used / memoryUsage.limit) * 100 : 0;

    return { ...memoryUsage, percentage: memoryPercentage };
}
```

## Error Handling and Recovery

### 1. **Graceful Degradation**

**Handle loading failures:**

```typescript
function useResilientInfiniteScroll() {
    const [error, setError] = useState(null);
    const [retryCount, setRetryCount] = useState(0);

    const loadMore = async () => {
        try {
            setError(null);
            await fetchItems();
            setRetryCount(0); // Reset on success
        } catch (err) {
            setError(err);

            // Exponential backoff retry
            if (retryCount < 3) {
                setTimeout(() => {
                    setRetryCount(prev => prev + 1);
                    loadMore();
                }, Math.pow(2, retryCount) * 1000);
            }
        }
    };

    const retry = () => {
        setError(null);
        setRetryCount(0);
        loadMore();
    };

    return { loadMore, error, retry };
}
```

### 2. **Loading States**

**Provide clear feedback:**

```typescript
function LoadingStates({ loading, error, retry }) {
    if (error) {
        return (
            <div className="error-state">
                <p>Failed to load more items</p>
                <button onClick={retry}>Retry</button>
            </div>
        );
    }

    if (loading) {
        return (
            <div className="loading-state">
                <div className="spinner"></div>
                <p>Loading more items...</p>
            </div>
        );
    }

    return null;
}
```

## Testing Optimization

### **Performance Testing**

```typescript
describe('Infinite Scroll Performance', () => {
    it('should maintain smooth scrolling with large datasets', async () => {
        const largeDataset = Array.from({ length: 10000 }, (_, i) => ({ id: i }));

        render(<OptimizedInfiniteList items={largeDataset} />);

        const container = screen.getByRole('list');

        // Simulate rapid scrolling
        for (let i = 0; i < 100; i++) {
            fireEvent.scroll(container, { target: { scrollTop: i * 10 } });
        }

        // Should still be responsive
        expect(container).toBeInTheDocument();

        // Check memory usage (if testable)
        if (performance.memory) {
            const memoryUsage = performance.memory.usedJSHeapSize;
            expect(memoryUsage).toBeLessThan(50 * 1024 * 1024); // < 50MB
        }
    });

    it('should cleanup old items', () => {
        const { rerender } = render(<MemoryEfficientList maxItems={100} />);

        // Add many items
        for (let i = 0; i < 200; i++) {
            // Simulate adding items
        }

        rerender(<MemoryEfficientList maxItems={100} />);

        // Should only keep recent items
        const items = screen.getAllByRole('listitem');
        expect(items.length).toBeLessThanOrEqual(100);
    });
});
```

## Common Interview Questions

### Q: How do you optimize infinite scrolling performance?

**A:** "I optimize infinite scrolling by implementing virtual scrolling to render only visible items, using memory management techniques to clean up old items, and implementing smart prefetching. I also use Intersection Observer for efficient loading triggers and React Query for intelligent caching and request deduplication."

### Q: What are the main performance issues with infinite scrolling?

**A:** "The main issues are memory accumulation from loading unlimited items, poor scroll performance with large DOM trees, and inefficient network usage. Without optimization, infinite scrolling can cause browser slowdown and crashes with large datasets."

### Q: How do you prevent memory leaks in infinite scrolling?

**A:** "I implement item cleanup by maintaining a sliding window of visible items and removing old items that are no longer needed. I also use virtualization to limit DOM nodes and implement proper cleanup in useEffect hooks to prevent memory leaks."

### Q: What's the difference between virtual scrolling and regular infinite scrolling?

**A:** "Regular infinite scrolling loads all items into the DOM, causing performance issues with large datasets. Virtual scrolling only renders visible items plus a small buffer, positioning them absolutely to maintain the illusion of a complete list. This dramatically reduces memory usage and improves performance."

### Q: How do you handle variable-height items in virtual scrolling?

**A:** "For variable-height items, I measure each item's actual height using ResizeObserver or after rendering, then calculate the total scroll height and visible range dynamically. This requires more complex calculations but provides accurate scrolling for content with different heights."

## Summary

**Key optimization strategies:**
- **Memory management:** Item cleanup, sliding windows, garbage collection hints
- **Virtual scrolling:** Render only visible items with absolute positioning
- **Network optimization:** Smart prefetching, request deduplication, adaptive loading
- **Scroll performance:** Throttled events, Intersection Observer, passive listeners
- **Caching:** Intelligent caching with LRU eviction, React Query integration

**Performance benefits:**
- **Memory usage:** 95%+ reduction in DOM nodes and memory consumption
- **Scroll smoothness:** 60fps scrolling even with 100k+ items
- **Loading performance:** Faster initial loads with progressive enhancement
- **Network efficiency:** Reduced redundant requests and bandwidth usage

**Best practices:**
- **Use virtualization** for lists >1000 items
- **Implement memory cleanup** to prevent leaks
- **Monitor performance** with custom metrics
- **Handle errors gracefully** with retry logic
- **Test thoroughly** for memory usage and scroll performance

**Interview Tip:** "I optimize infinite scrolling performance through virtual scrolling (rendering only visible items), memory management (cleaning up old items), and smart prefetching. The key is maintaining smooth 60fps scrolling while preventing memory leaks, even with very large datasets."