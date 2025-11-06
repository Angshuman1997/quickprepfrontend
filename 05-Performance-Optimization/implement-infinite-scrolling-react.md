# How do you implement infinite scrolling in React?

## Question
How do you implement infinite scrolling in React?

## Answer

Infinite scrolling provides a seamless user experience by loading content progressively as the user scrolls, rather than using pagination. It's commonly used in social media feeds, search results, and content-heavy applications. The key challenges are performance optimization, loading states, and preventing duplicate requests.

## Basic Implementation

### 1. **Intersection Observer API**

**Modern approach using IntersectionObserver:**
```typescript
import { useState, useEffect, useCallback, useRef } from 'react';

interface UseInfiniteScrollOptions {
    threshold?: number;
    rootMargin?: string;
}

function useInfiniteScroll(
    callback: () => void,
    options: UseInfiniteScrollOptions = {}
) {
    const [isFetching, setIsFetching] = useState(false);
    const [targetRef, setTargetRef] = useState<Element | null>(null);

    const { threshold = 0.1, rootMargin = '100px' } = options;

    const observerCallback = useCallback(
        (entries: IntersectionObserverEntry[]) => {
            const [entry] = entries;
            if (entry.isIntersecting && !isFetching) {
                setIsFetching(true);
                callback();
            }
        },
        [callback, isFetching]
    );

    useEffect(() => {
        if (!targetRef) return;

        const observer = new IntersectionObserver(observerCallback, {
            threshold,
            rootMargin,
        });

        observer.observe(targetRef);

        return () => observer.disconnect();
    }, [targetRef, observerCallback, threshold, rootMargin]);

    const setRef = useCallback((node: Element | null) => {
        setTargetRef(node);
    }, []);

    const resetFetching = useCallback(() => {
        setIsFetching(false);
    }, []);

    return { setRef, isFetching, resetFetching };
}

// Usage
function InfiniteList() {
    const [items, setItems] = useState<Item[]>([]);
    const [hasMore, setHasMore] = useState(true);
    const [page, setPage] = useState(1);

    const loadMore = useCallback(async () => {
        try {
            const response = await fetch(`/api/items?page=${page}&limit=20`);
            const newItems = await response.json();

            if (newItems.length === 0) {
                setHasMore(false);
            } else {
                setItems(prev => [...prev, ...newItems]);
                setPage(prev => prev + 1);
            }
        } catch (error) {
            console.error('Failed to load more items:', error);
        } finally {
            resetFetching();
        }
    }, [page, resetFetching]);

    const { setRef, isFetching, resetFetching } = useInfiniteScroll(loadMore, {
        threshold: 0.5,
        rootMargin: '200px'
    });

    return (
        <div>
            {items.map((item, index) => (
                <div key={item.id} className="item">
                    {item.content}
                </div>
            ))}

            {hasMore && (
                <div ref={setRef} className="loading-trigger">
                    {isFetching && <div>Loading more...</div>}
                </div>
            )}

            {!hasMore && <div>No more items to load</div>}
        </div>
    );
}
```

### 2. **Scroll Event-Based Implementation**

**Traditional scroll event approach:**
```typescript
function useInfiniteScroll(callback: () => void, hasMore: boolean) {
    const [isFetching, setIsFetching] = useState(false);

    useEffect(() => {
        const handleScroll = () => {
            if (
                window.innerHeight + document.documentElement.scrollTop
                >= document.documentElement.offsetHeight - 1000 // 1000px before bottom
                && !isFetching
                && hasMore
            ) {
                setIsFetching(true);
                callback();
            }
        };

        window.addEventListener('scroll', handleScroll);
        return () => window.removeEventListener('scroll', handleScroll);
    }, [callback, isFetching, hasMore]);

    const resetFetching = useCallback(() => {
        setIsFetching(false);
    }, []);

    return { isFetching, resetFetching };
}

// Usage
function ScrollBasedInfiniteList() {
    const [items, setItems] = useState([]);
    const [hasMore, setHasMore] = useState(true);
    const [page, setPage] = useState(1);

    const loadMore = useCallback(async () => {
        const newItems = await fetchItems(page, 20);

        if (newItems.length === 0) {
            setHasMore(false);
        } else {
            setItems(prev => [...prev, ...newItems]);
            setPage(prev => prev + 1);
        }

        resetFetching();
    }, [page, resetFetching]);

    const { isFetching, resetFetching } = useInfiniteScroll(loadMore, hasMore);

    return (
        <div>
            {items.map(item => (
                <Item key={item.id} item={item} />
            ))}
            {isFetching && <LoadingSpinner />}
            {!hasMore && <div>End of list</div>}
        </div>
    );
}
```

## Advanced Implementation Patterns

### 1. **React Query/SWR Integration**

**Using React Query for infinite scrolling:**
```typescript
import { useInfiniteQuery } from 'react-query';

function useInfiniteItems() {
    return useInfiniteQuery(
        'items',
        async ({ pageParam = 1 }) => {
            const response = await fetch(`/api/items?page=${pageParam}&limit=20`);
            return response.json();
        },
        {
            getNextPageParam: (lastPage, pages) => {
                // Return next page number if there are more items
                return lastPage.length === 20 ? pages.length + 1 : undefined;
            },
            staleTime: 5 * 60 * 1000, // 5 minutes
        }
    );
}

function InfiniteQueryList() {
    const {
        data,
        fetchNextPage,
        hasNextPage,
        isFetchingNextPage,
        isLoading,
        error
    } = useInfiniteItems();

    const { setRef } = useInfiniteScroll(() => {
        if (hasNextPage && !isFetchingNextPage) {
            fetchNextPage();
        }
    }, { threshold: 0.1 });

    if (isLoading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;

    const allItems = data?.pages.flat() || [];

    return (
        <div>
            {allItems.map((item) => (
                <Item key={item.id} item={item} />
            ))}

            {hasNextPage && (
                <div ref={setRef}>
                    {isFetchingNextPage && <div>Loading more...</div>}
                </div>
            )}

            {!hasNextPage && <div>No more items</div>}
        </div>
    );
}
```

**SWR implementation:**
```typescript
import useSWRInfinite from 'swr/infinite';

const fetcher = (url) => fetch(url).then(res => res.json());

function useInfiniteItems() {
    const getKey = (pageIndex, previousPageData) => {
        if (previousPageData && !previousPageData.length) return null; // Reached end
        return `/api/items?page=${pageIndex + 1}&limit=20`;
    };

    const { data, error, size, setSize, isValidating } = useSWRInfinite(
        getKey,
        fetcher,
        {
            revalidateFirstPage: false,
            revalidateOnFocus: false,
        }
    );

    const items = data ? [].concat(...data) : [];
    const isLoadingInitialData = !data && !error;
    const isLoadingMore = isLoadingInitialData ||
        (size > 0 && data && typeof data[size - 1] === 'undefined');
    const isEmpty = data?.[0]?.length === 0;
    const isReachingEnd = isEmpty ||
        (data && data[data.length - 1]?.length < 20);

    return {
        items,
        error,
        isLoadingMore,
        size,
        setSize,
        isReachingEnd
    };
}
```

### 2. **Virtual Scrolling for Large Lists**

**Using react-window for virtualization:**
```typescript
import { FixedSizeList as List } from 'react-window';
import { useInfiniteScroll } from './useInfiniteScroll';

interface VirtualizedInfiniteListProps {
    items: Item[];
    itemHeight: number;
    containerHeight: number;
    hasMore: boolean;
    loadMore: () => void;
    isLoading: boolean;
}

function VirtualizedInfiniteList({
    items,
    itemHeight,
    containerHeight,
    hasMore,
    loadMore,
    isLoading
}: VirtualizedInfiniteListProps) {
    const { setRef } = useInfiniteScroll(loadMore, {
        threshold: 0.8 // Load more when 80% through current items
    });

    const ItemRenderer = ({ index, style }) => {
        const item = items[index];

        // Load more when approaching the end
        if (index === items.length - 5 && hasMore && !isLoading) {
            setRef(document.createElement('div')); // Trigger load
        }

        return (
            <div style={style}>
                <Item item={item} />
            </div>
        );
    };

    return (
        <div>
            <List
                height={containerHeight}
                itemCount={items.length}
                itemSize={itemHeight}
                width="100%"
            >
                {ItemRenderer}
            </List>

            {isLoading && <div>Loading more...</div>}
            {!hasMore && <div>No more items</div>}
        </div>
    );
}

// Usage
function App() {
    const [items, setItems] = useState([]);
    const [hasMore, setHasMore] = useState(true);
    const [page, setPage] = useState(1);
    const [isLoading, setIsLoading] = useState(false);

    const loadMore = async () => {
        setIsLoading(true);
        try {
            const newItems = await fetchItems(page, 50);
            if (newItems.length === 0) {
                setHasMore(false);
            } else {
                setItems(prev => [...prev, ...newItems]);
                setPage(prev => prev + 1);
            }
        } finally {
            setIsLoading(false);
        }
    };

    return (
        <VirtualizedInfiniteList
            items={items}
            itemHeight={100}
            containerHeight={600}
            hasMore={hasMore}
            loadMore={loadMore}
            isLoading={isLoading}
        />
    );
}
```

### 3. **Grid Layout Infinite Scrolling**

**Masonry/grid layouts:**
```typescript
function InfiniteGrid() {
    const [items, setItems] = useState<Item[]>([]);
    const [columns, setColumns] = useState(3);
    const [hasMore, setHasMore] = useState(true);

    // Organize items into columns
    const columnItems = useMemo(() => {
        const cols = Array.from({ length: columns }, () => []);
        items.forEach((item, index) => {
            cols[index % columns].push(item);
        });
        return cols;
    }, [items, columns]);

    const loadMore = async () => {
        const newItems = await fetchItems();
        if (newItems.length === 0) {
            setHasMore(false);
        } else {
            setItems(prev => [...prev, ...newItems]);
        }
    };

    const { setRef, isFetching } = useInfiniteScroll(loadMore);

    // Update columns on resize
    useEffect(() => {
        const updateColumns = () => {
            const width = window.innerWidth;
            if (width < 768) setColumns(1);
            else if (width < 1024) setColumns(2);
            else setColumns(3);
        };

        updateColumns();
        window.addEventListener('resize', updateColumns);
        return () => window.removeEventListener('resize', updateColumns);
    }, []);

    return (
        <div className="grid-container">
            {columnItems.map((column, columnIndex) => (
                <div key={columnIndex} className="grid-column">
                    {column.map((item) => (
                        <Item key={item.id} item={item} />
                    ))}
                </div>
            ))}

            {hasMore && (
                <div ref={setRef} className="load-more-trigger">
                    {isFetching && <LoadingSpinner />}
                </div>
            )}
        </div>
    );
}
```

## Performance Optimizations

### 1. **Debounced Loading**

**Prevent rapid successive loads:**
```typescript
function useDebouncedInfiniteScroll(callback: () => void, delay = 200) {
    const [isFetching, setIsFetching] = useState(false);
    const timeoutRef = useRef<NodeJS.Timeout>();

    const debouncedCallback = useCallback(() => {
        if (timeoutRef.current) {
            clearTimeout(timeoutRef.current);
        }

        timeoutRef.current = setTimeout(() => {
            if (!isFetching) {
                setIsFetching(true);
                callback();
            }
        }, delay);
    }, [callback, delay, isFetching]);

    const resetFetching = useCallback(() => {
        setIsFetching(false);
    }, []);

    useEffect(() => {
        return () => {
            if (timeoutRef.current) {
                clearTimeout(timeoutRef.current);
            }
        };
    }, []);

    return { isFetching, resetFetching, triggerLoad: debouncedCallback };
}
```

### 2. **Request Deduplication**

**Prevent duplicate requests:**
```typescript
function useDeduplicatedInfiniteScroll() {
    const pendingRequests = useRef(new Set<string>());
    const [isFetching, setIsFetching] = useState(false);

    const makeRequest = useCallback(async (page: number) => {
        const requestKey = `page-${page}`;

        if (pendingRequests.current.has(requestKey)) {
            return null; // Request already in progress
        }

        pendingRequests.current.add(requestKey);
        setIsFetching(true);

        try {
            const data = await fetchItems(page);
            return data;
        } finally {
            pendingRequests.current.delete(requestKey);
            setIsFetching(false);
        }
    }, []);

    return { makeRequest, isFetching };
}
```

### 3. **Memory Management**

**Clean up old items for memory efficiency:**
```typescript
function useMemoryEfficientInfiniteScroll(maxItems = 1000) {
    const [items, setItems] = useState<Item[]>([]);

    const addItems = useCallback((newItems: Item[]) => {
        setItems(prev => {
            const combined = [...prev, ...newItems];

            // Keep only the most recent items
            if (combined.length > maxItems) {
                return combined.slice(-maxItems);
            }

            return combined;
        });
    }, [maxItems]);

    const clearOldItems = useCallback((keepCount = 500) => {
        setItems(prev => prev.slice(-keepCount));
    }, []);

    return { items, addItems, clearOldItems };
}
```

## Error Handling and UX

### 1. **Error Boundaries and Retry**

```typescript
function InfiniteScrollErrorBoundary({ children }) {
    const [hasError, setHasError] = useState(false);
    const [retryCount, setRetryCount] = useState(0);

    const retry = useCallback(() => {
        setHasError(false);
        setRetryCount(prev => prev + 1);
    }, []);

    if (hasError) {
        return (
            <div className="error-state">
                <p>Failed to load more items</p>
                {retryCount < 3 && (
                    <button onClick={retry}>Try Again</button>
                )}
            </div>
        );
    }

    return children;
}

function RobustInfiniteList() {
    const [loadError, setLoadError] = useState<Error | null>(null);

    const loadMore = async () => {
        try {
            setLoadError(null);
            await fetchMoreItems();
        } catch (error) {
            setLoadError(error);
            throw error; // Re-throw for error boundary
        }
    };

    return (
        <InfiniteScrollErrorBoundary>
            <InfiniteList
                loadMore={loadMore}
                error={loadError}
            />
        </InfiniteScrollErrorBoundary>
    );
}
```

### 2. **Loading States and Skeletons**

```typescript
function LoadingSkeleton() {
    return (
        <div className="skeleton-item">
            <div className="skeleton-avatar"></div>
            <div className="skeleton-text">
                <div className="skeleton-line"></div>
                <div className="skeleton-line short"></div>
            </div>
        </div>
    );
}

function InfiniteListWithSkeleton() {
    const { items, isFetching, hasMore } = useInfiniteItems();

    return (
        <div>
            {items.map(item => (
                <Item key={item.id} item={item} />
            ))}

            {isFetching && (
                <div className="loading-section">
                    <LoadingSkeleton />
                    <LoadingSkeleton />
                    <LoadingSkeleton />
                </div>
            )}

            {!hasMore && items.length > 0 && (
                <div className="end-message">
                    You've reached the end!
                </div>
            )}
        </div>
    );
}
```

### 3. **Pull to Refresh**

**Mobile-friendly pull to refresh:**
```typescript
function usePullToRefresh(callback: () => void, threshold = 80) {
    const [pullDistance, setPullDistance] = useState(0);
    const [isRefreshing, setIsRefreshing] = useState(false);
    const startY = useRef(0);
    const isPulling = useRef(false);

    const handleTouchStart = useCallback((e: TouchEvent) => {
        if (window.scrollY === 0) {
            startY.current = e.touches[0].clientY;
            isPulling.current = true;
        }
    }, []);

    const handleTouchMove = useCallback((e: TouchEvent) => {
        if (!isPulling.current || isRefreshing) return;

        const currentY = e.touches[0].clientY;
        const distance = Math.max(0, currentY - startY.current);

        if (distance > 0) {
            e.preventDefault();
            setPullDistance(distance);
        }
    }, [isRefreshing]);

    const handleTouchEnd = useCallback(async () => {
        if (!isPulling.current) return;

        isPulling.current = false;

        if (pullDistance >= threshold && !isRefreshing) {
            setIsRefreshing(true);
            setPullDistance(0);

            try {
                await callback();
            } finally {
                setIsRefreshing(false);
            }
        } else {
            setPullDistance(0);
        }
    }, [pullDistance, threshold, callback, isRefreshing]);

    useEffect(() => {
        document.addEventListener('touchstart', handleTouchStart, { passive: false });
        document.addEventListener('touchmove', handleTouchMove, { passive: false });
        document.addEventListener('touchend', handleTouchEnd);

        return () => {
            document.removeEventListener('touchstart', handleTouchStart);
            document.removeEventListener('touchmove', handleTouchMove);
            document.removeEventListener('touchend', handleTouchEnd);
        };
    }, [handleTouchStart, handleTouchMove, handleTouchEnd]);

    return { pullDistance, isRefreshing };
}
```

## Testing Infinite Scroll

### 1. **Unit Testing**

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { InfiniteList } from './InfiniteList';

// Mock IntersectionObserver
global.IntersectionObserver = class IntersectionObserver {
    constructor(callback) {
        this.callback = callback;
    }
    observe() {
        // Trigger intersection immediately for testing
        this.callback([{ isIntersecting: true }]);
    }
    disconnect() {}
};

describe('InfiniteList', () => {
    it('loads initial items', async () => {
        render(<InfiniteList />);

        await waitFor(() => {
            expect(screen.getByText('Item 1')).toBeInTheDocument();
        });
    });

    it('loads more items on scroll', async () => {
        render(<InfiniteList />);

        // Wait for initial load
        await waitFor(() => {
            expect(screen.getByText('Item 20')).toBeInTheDocument();
        });

        // Simulate scroll to bottom (IntersectionObserver will trigger)
        // Additional items should load
        await waitFor(() => {
            expect(screen.getByText('Item 40')).toBeInTheDocument();
        });
    });

    it('shows loading state', async () => {
        render(<InfiniteList />);

        expect(screen.getByText('Loading more...')).toBeInTheDocument();
    });

    it('shows end message when no more items', async () => {
        // Mock API to return empty array
        render(<InfiniteList />);

        await waitFor(() => {
            expect(screen.getByText('No more items to load')).toBeInTheDocument();
        });
    });
});
```

### 2. **Integration Testing**

```typescript
describe('Infinite Scroll Integration', () => {
    it('handles rapid scroll events', async () => {
        const mockFetch = jest.fn();
        render(<InfiniteList fetchItems={mockFetch} />);

        // Simulate rapid scroll events
        const scrollContainer = screen.getByRole('list');

        // Fire multiple scroll events quickly
        for (let i = 0; i < 10; i++) {
            fireEvent.scroll(scrollContainer, { target: { scrollTop: 1000 } });
        }

        // Should only trigger one fetch due to debouncing
        await waitFor(() => {
            expect(mockFetch).toHaveBeenCalledTimes(1);
        });
    });

    it('handles network errors gracefully', async () => {
        const mockFetch = jest.fn().mockRejectedValue(new Error('Network error'));
        render(<InfiniteList fetchItems={mockFetch} />);

        await waitFor(() => {
            expect(screen.getByText('Failed to load more items')).toBeInTheDocument();
        });

        // Should show retry button
        expect(screen.getByText('Try Again')).toBeInTheDocument();
    });
});
```

## Common Interview Questions

### Q: How do you implement infinite scrolling in React?

**A:** I use the Intersection Observer API with a custom hook that triggers loading when a sentinel element comes into view. I manage state for items, loading status, and whether more items are available. For performance, I implement debouncing to prevent rapid successive loads and use virtualization for large lists.

### Q: What's the difference between Intersection Observer and scroll events?

**A:** Intersection Observer is more performant and reliable than scroll events. It uses the browser's native intersection detection and doesn't require continuous event listeners. Scroll events can fire hundreds of times per second and may cause performance issues, while Intersection Observer only fires when elements actually intersect with the viewport.

### Q: How do you handle loading states in infinite scroll?

**A:** I show loading indicators at the bottom of the list while fetching, use skeleton loaders for better UX, and implement error states with retry functionality. I also prevent multiple simultaneous requests and show appropriate messages when reaching the end of the list.

### Q: What are the performance considerations for infinite scroll?

**A:** Key considerations include debouncing rapid scroll events, implementing virtualization for large lists, cleaning up old items to manage memory, preventing duplicate requests, and using efficient data structures. I also consider when to stop loading (no more data) and provide ways to refresh or reset the list.

### Q: How do you test infinite scrolling?

**A:** I mock the Intersection Observer API in unit tests, test loading states and error handling, verify that items are appended correctly, and ensure no duplicate requests occur. For integration tests, I simulate scroll behavior and verify the correct number of API calls and UI updates.

## Summary

**Core Implementation:**
1. **Intersection Observer** - Modern, performant approach
2. **State management** - Items, loading, hasMore flags
3. **Error handling** - Retry logic and user feedback
4. **Performance optimization** - Debouncing, virtualization

**Key Patterns:**
- **Sentinel element** - Triggers loading when visible
- **State management** - Track loading, errors, and pagination
- **Memory management** - Clean up old items for large lists
- **Error boundaries** - Graceful error handling

**Best Practices:**
- **Debounce rapid events** to prevent excessive API calls
- **Use virtualization** for lists with 1000+ items
- **Implement proper loading states** and error handling
- **Consider accessibility** with keyboard navigation
- **Test thoroughly** for edge cases and performance

**Interview Tip:** "I implement infinite scrolling using the Intersection Observer API with a sentinel element that triggers loading when it comes into view. I manage state for the items array, loading status, and whether more data is available, and implement debouncing to prevent rapid successive API calls. For performance, I use virtualization for large lists and clean up old items to manage memory."