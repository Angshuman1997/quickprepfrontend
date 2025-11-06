# How do you implement pagination efficiently?

## Question
How do you implement pagination efficiently?

## Answer

Efficient pagination is crucial for handling large datasets without compromising performance. The goal is to minimize server load, reduce client-side memory usage, and provide a smooth user experience. Different pagination strategies work better for different use cases based on data size, access patterns, and user requirements.

## Pagination Strategies

### 1. **Offset-Based Pagination**

**Traditional approach using LIMIT and OFFSET:**
```typescript
// API endpoint
app.get('/api/items', (req, res) => {
    const { page = 1, limit = 10 } = req.query;
    const offset = (page - 1) * limit;

    // Inefficient for large datasets
    const query = `
        SELECT * FROM items
        ORDER BY created_at DESC
        LIMIT ${limit} OFFSET ${offset}
    `;

    db.query(query, (err, results) => {
        res.json({
            items: results,
            page: parseInt(page),
            limit: parseInt(limit),
            // Total count for pagination UI
            total: results.totalCount
        });
    });
});

// Client-side implementation
function PaginatedList() {
    const [items, setItems] = useState([]);
    const [currentPage, setCurrentPage] = useState(1);
    const [totalPages, setTotalPages] = useState(0);
    const [loading, setLoading] = useState(false);

    const fetchPage = async (page) => {
        setLoading(true);
        try {
            const response = await fetch(`/api/items?page=${page}&limit=10`);
            const data = await response.json();

            setItems(data.items);
            setCurrentPage(data.page);
            setTotalPages(Math.ceil(data.total / data.limit));
        } finally {
            setLoading(false);
        }
    };

    useEffect(() => {
        fetchPage(1);
    }, []);

    return (
        <div>
            <div className="items-grid">
                {items.map(item => (
                    <Item key={item.id} item={item} />
                ))}
            </div>

            <PaginationControls
                currentPage={currentPage}
                totalPages={totalPages}
                onPageChange={fetchPage}
                loading={loading}
            />
        </div>
    );
}
```

**Problems with offset-based pagination:**
- **Performance degradation** - Database must skip rows for large offsets
- **Inconsistent results** - Data changes between requests can cause duplicates/missing items
- **Memory intensive** - Requires counting total records

### 2. **Cursor-Based Pagination**

**More efficient for large datasets:**
```typescript
// API with cursor pagination
app.get('/api/items', (req, res) => {
    const { cursor, limit = 10, direction = 'next' } = req.query;

    let query = 'SELECT * FROM items WHERE ';
    let params = [];

    if (cursor) {
        const operator = direction === 'next' ? '>' : '<';
        query += 'id ' + operator + ' ? ';
        params.push(cursor);
    }

    query += 'ORDER BY id ' + (direction === 'next' ? 'ASC' : 'DESC');
    query += ' LIMIT ?';
    params.push(parseInt(limit) + 1); // +1 to check if there are more

    db.query(query, params, (err, results) => {
        const hasNextPage = results.length > limit;
        const items = hasNextPage ? results.slice(0, -1) : results;

        res.json({
            items,
            nextCursor: hasNextPage ? items[items.length - 1].id : null,
            prevCursor: cursor && direction === 'prev' ? items[0].id : null,
            hasNextPage,
            hasPrevPage: !!cursor
        });
    });
});

// Client implementation
function CursorPaginatedList() {
    const [items, setItems] = useState([]);
    const [cursors, setCursors] = useState({
        next: null,
        prev: null
    });
    const [loading, setLoading] = useState(false);

    const fetchPage = async (cursor = null, direction = 'next') => {
        setLoading(true);
        try {
            const params = new URLSearchParams({
                limit: '10',
                direction
            });

            if (cursor) {
                params.set('cursor', cursor);
            }

            const response = await fetch(`/api/items?${params}`);
            const data = await response.json();

            setItems(data.items);
            setCursors({
                next: data.nextCursor,
                prev: data.prevCursor
            });
        } finally {
            setLoading(false);
        }
    };

    useEffect(() => {
        fetchPage();
    }, []);

    return (
        <div>
            <div className="items-grid">
                {items.map(item => (
                    <Item key={item.id} item={item} />
                ))}
            </div>

            <div className="pagination-controls">
                <button
                    onClick={() => fetchPage(cursors.prev, 'prev')}
                    disabled={!cursors.prev || loading}
                >
                    Previous
                </button>

                <button
                    onClick={() => fetchPage(cursors.next, 'next')}
                    disabled={!cursors.next || loading}
                >
                    Next
                </button>
            </div>
        </div>
    );
}
```

**Advantages of cursor pagination:**
- **Consistent performance** - No degradation with large datasets
- **Stable results** - No duplicates or missing items
- **Better for real-time data** - Handles insertions/deletions gracefully

### 3. **Keyset Pagination**

**Similar to cursor but uses multiple columns:**
```typescript
// API with keyset pagination
app.get('/api/posts', (req, res) => {
    const { after, before, limit = 10 } = req.query;

    let query = 'SELECT * FROM posts WHERE ';
    let params = [];

    if (after) {
        // Parse cursor: "created_at:id"
        const [afterDate, afterId] = after.split(':');
        query += '(created_at, id) > (?, ?) ';
        params.push(afterDate, afterId);
    } else if (before) {
        const [beforeDate, beforeId] = before.split(':');
        query += '(created_at, id) < (?, ?) ';
        params.push(beforeDate, beforeId);
    }

    query += 'ORDER BY created_at DESC, id DESC LIMIT ?';
    params.push(parseInt(limit) + 1);

    db.query(query, params, (err, results) => {
        const hasMore = results.length > limit;
        const posts = hasMore ? results.slice(0, -1) : results;

        const nextCursor = hasMore ?
            `${posts[posts.length - 1].created_at}:${posts[posts.length - 1].id}` :
            null;

        const prevCursor = posts.length > 0 ?
            `${posts[0].created_at}:${posts[0].id}` :
            null;

        res.json({
            posts,
            nextCursor,
            prevCursor,
            hasNextPage: hasMore,
            hasPrevPage: !!before
        });
    });
});
```

## React Implementation Patterns

### 1. **Custom Pagination Hook**

```typescript
interface PaginationOptions {
    initialPage?: number;
    itemsPerPage?: number;
    totalItems?: number;
}

interface PaginationState {
    currentPage: number;
    itemsPerPage: number;
    totalItems: number;
    totalPages: number;
}

function usePagination(options: PaginationOptions = {}) {
    const {
        initialPage = 1,
        itemsPerPage = 10,
        totalItems = 0
    } = options;

    const [state, setState] = useState<PaginationState>({
        currentPage: initialPage,
        itemsPerPage,
        totalItems,
        totalPages: Math.ceil(totalItems / itemsPerPage)
    });

    const setPage = useCallback((page: number) => {
        setState(prev => ({
            ...prev,
            currentPage: Math.max(1, Math.min(page, prev.totalPages))
        }));
    }, []);

    const nextPage = useCallback(() => {
        setState(prev => ({
            ...prev,
            currentPage: Math.min(prev.currentPage + 1, prev.totalPages)
        }));
    }, []);

    const prevPage = useCallback(() => {
        setState(prev => ({
            ...prev,
            currentPage: Math.max(prev.currentPage - 1, 1)
        }));
    }, []);

    const setItemsPerPage = useCallback((newItemsPerPage: number) => {
        setState(prev => ({
            ...prev,
            itemsPerPage: newItemsPerPage,
            totalPages: Math.ceil(prev.totalItems / newItemsPerPage),
            currentPage: 1 // Reset to first page
        }));
    }, []);

    const setTotalItems = useCallback((total: number) => {
        setState(prev => ({
            ...prev,
            totalItems: total,
            totalPages: Math.ceil(total / prev.itemsPerPage)
        }));
    }, []);

    return {
        ...state,
        setPage,
        nextPage,
        prevPage,
        setItemsPerPage,
        setTotalItems,
        hasNextPage: state.currentPage < state.totalPages,
        hasPrevPage: state.currentPage > 1
    };
}

// Usage
function PaginatedTable({ data }) {
    const {
        currentPage,
        itemsPerPage,
        totalPages,
        setPage,
        nextPage,
        prevPage,
        hasNextPage,
        hasPrevPage
    } = usePagination({
        totalItems: data.length,
        itemsPerPage: 20
    });

    const startIndex = (currentPage - 1) * itemsPerPage;
    const endIndex = startIndex + itemsPerPage;
    const currentItems = data.slice(startIndex, endIndex);

    return (
        <div>
            <table>
                <tbody>
                    {currentItems.map(item => (
                        <tr key={item.id}>
                            <td>{item.name}</td>
                            <td>{item.value}</td>
                        </tr>
                    ))}
                </tbody>
            </table>

            <PaginationControls
                currentPage={currentPage}
                totalPages={totalPages}
                onPageChange={setPage}
                onNext={nextPage}
                onPrev={prevPage}
                hasNext={hasNextPage}
                hasPrev={hasPrevPage}
            />
        </div>
    );
}
```

### 2. **Server-State Pagination with React Query**

```typescript
import { useQuery } from 'react-query';

function usePaginatedData(page: number, limit: number) {
    return useQuery(
        ['data', page, limit],
        () => fetchData(page, limit),
        {
            keepPreviousData: true, // Keep old data while loading new
            staleTime: 5 * 60 * 1000, // 5 minutes
        }
    );
}

function ServerPaginatedList() {
    const [page, setPage] = useState(1);
    const limit = 10;

    const { data, isLoading, isPreviousData, error } = usePaginatedData(page, limit);

    const totalPages = data ? Math.ceil(data.total / limit) : 0;

    return (
        <div>
            {isLoading ? (
                <div>Loading...</div>
            ) : error ? (
                <div>Error: {error.message}</div>
            ) : (
                <div>
                    <div className="items-grid">
                        {data.items.map(item => (
                            <Item key={item.id} item={item} />
                        ))}
                    </div>

                    <PaginationControls
                        currentPage={page}
                        totalPages={totalPages}
                        onPageChange={setPage}
                        disabled={isPreviousData} // Disable while loading
                    />
                </div>
            )}
        </div>
    );
}
```

### 3. **Infinite Scroll with Pagination Fallback**

**Hybrid approach for better UX:**
```typescript
function HybridList() {
    const [mode, setMode] = useState<'infinite' | 'paginated'>('infinite');
    const [items, setItems] = useState([]);
    const [page, setPage] = useState(1);
    const [hasMore, setHasMore] = useState(true);

    // Infinite scroll logic
    const loadMore = useCallback(async () => {
        const newItems = await fetchItems(page, 20);
        if (newItems.length === 0) {
            setHasMore(false);
            setMode('paginated'); // Switch to pagination when no more items
        } else {
            setItems(prev => [...prev, ...newItems]);
            setPage(prev => prev + 1);
        }
    }, [page]);

    // Pagination logic
    const loadPage = useCallback(async (targetPage) => {
        const pageItems = await fetchItems(targetPage, 20);
        setItems(pageItems);
        setPage(targetPage);
    }, []);

    return (
        <div>
            <div className="controls">
                <button onClick={() => setMode('infinite')}>Infinite Scroll</button>
                <button onClick={() => setMode('paginated')}>Pagination</button>
            </div>

            {mode === 'infinite' ? (
                <InfiniteScrollList
                    items={items}
                    loadMore={loadMore}
                    hasMore={hasMore}
                />
            ) : (
                <PaginatedList
                    items={items}
                    currentPage={page}
                    onPageChange={loadPage}
                />
            )}
        </div>
    );
}
```

## Performance Optimizations

### 1. **Prefetching**

**Preload adjacent pages:**
```typescript
function usePrefetchPagination() {
    const queryClient = useQueryClient();

    const prefetchPage = useCallback((page: number) => {
        queryClient.prefetchQuery(
            ['data', page],
            () => fetchData(page),
            {
                staleTime: 10 * 60 * 1000, // 10 minutes
            }
        );
    }, [queryClient]);

    const prefetchAdjacentPages = useCallback((currentPage: number, totalPages: number) => {
        // Prefetch next page
        if (currentPage < totalPages) {
            prefetchPage(currentPage + 1);
        }

        // Prefetch previous page
        if (currentPage > 1) {
            prefetchPage(currentPage - 1);
        }
    }, [prefetchPage]);

    return { prefetchAdjacentPages };
}

// Usage
function OptimizedPaginatedList() {
    const { prefetchAdjacentPages } = usePrefetchPagination();
    const [currentPage, setCurrentPage] = useState(1);

    const handlePageChange = (newPage) => {
        setCurrentPage(newPage);
        prefetchAdjacentPages(newPage, totalPages);
    };

    return (
        <PaginationControls
            currentPage={currentPage}
            onPageChange={handlePageChange}
        />
    );
}
```

### 2. **Virtual Scrolling for Large Pages**

**Render only visible items:**
```typescript
import { FixedSizeList as List } from 'react-window';

function VirtualizedPagination({ items, itemHeight = 50 }) {
    const [currentPage, setCurrentPage] = useState(1);
    const itemsPerPage = 100; // Large page size with virtualization

    const startIndex = (currentPage - 1) * itemsPerPage;
    const endIndex = startIndex + itemsPerPage;
    const currentItems = items.slice(startIndex, endIndex);

    const ItemRenderer = ({ index, style }) => {
        const item = currentItems[index];
        return (
            <div style={style}>
                <Item item={item} />
            </div>
        );
    };

    return (
        <div>
            <List
                height={400}
                itemCount={currentItems.length}
                itemSize={itemHeight}
                width="100%"
            >
                {ItemRenderer}
            </List>

            <PaginationControls
                currentPage={currentPage}
                totalPages={Math.ceil(items.length / itemsPerPage)}
                onPageChange={setCurrentPage}
            />
        </div>
    );
}
```

### 3. **Memory Management**

**Clean up unused pages:**
```typescript
function useMemoryEfficientPagination(maxCachedPages = 5) {
    const [pageCache, setPageCache] = useState(new Map());
    const [accessOrder, setAccessOrder] = useState([]);

    const getPage = useCallback((pageNumber) => {
        return pageCache.get(pageNumber);
    }, [pageCache]);

    const setPage = useCallback((pageNumber, data) => {
        setPageCache(prev => {
            const newCache = new Map(prev);

            // Update access order
            setAccessOrder(current => {
                const filtered = current.filter(p => p !== pageNumber);
                return [...filtered, pageNumber];
            });

            // Add new page
            newCache.set(pageNumber, data);

            // Remove old pages if cache is full
            if (newCache.size > maxCachedPages) {
                const toRemove = accessOrder[0];
                newCache.delete(toRemove);
            }

            return newCache;
        });
    }, [maxCachedPages]);

    const clearCache = useCallback(() => {
        setPageCache(new Map());
        setAccessOrder([]);
    }, []);

    return { getPage, setPage, clearCache };
}
```

## UI/UX Considerations

### 1. **Pagination Controls**

```typescript
interface PaginationControlsProps {
    currentPage: number;
    totalPages: number;
    onPageChange: (page: number) => void;
    showFirstLast?: boolean;
    maxVisiblePages?: number;
    loading?: boolean;
}

function PaginationControls({
    currentPage,
    totalPages,
    onPageChange,
    showFirstLast = true,
    maxVisiblePages = 5,
    loading = false
}: PaginationControlsProps) {
    const getVisiblePages = () => {
        const half = Math.floor(maxVisiblePages / 2);
        let start = Math.max(1, currentPage - half);
        let end = Math.min(totalPages, start + maxVisiblePages - 1);

        if (end - start + 1 < maxVisiblePages) {
            start = Math.max(1, end - maxVisiblePages + 1);
        }

        return Array.from({ length: end - start + 1 }, (_, i) => start + i);
    };

    const visiblePages = getVisiblePages();

    return (
        <nav className="pagination" aria-label="Pagination">
            {showFirstLast && (
                <button
                    onClick={() => onPageChange(1)}
                    disabled={currentPage === 1 || loading}
                    aria-label="First page"
                >
                    ««
                </button>
            )}

            <button
                onClick={() => onPageChange(currentPage - 1)}
                disabled={currentPage === 1 || loading}
                aria-label="Previous page"
            >
                ‹
            </button>

            {visiblePages.map(page => (
                <button
                    key={page}
                    onClick={() => onPageChange(page)}
                    className={page === currentPage ? 'active' : ''}
                    disabled={loading}
                    aria-current={page === currentPage ? 'page' : undefined}
                >
                    {page}
                </button>
            ))}

            <button
                onClick={() => onPageChange(currentPage + 1)}
                disabled={currentPage === totalPages || loading}
                aria-label="Next page"
            >
                ›
            </button>

            {showFirstLast && (
                <button
                    onClick={() => onPageChange(totalPages)}
                    disabled={currentPage === totalPages || loading}
                    aria-label="Last page"
                >
                    »»
                </button>
            )}
        </nav>
    );
}
```

### 2. **Loading States and Skeletons**

```typescript
function PaginationSkeleton() {
    return (
        <div className="pagination-skeleton">
            <div className="skeleton-button"></div>
            <div className="skeleton-button"></div>
            <div className="skeleton-button active"></div>
            <div className="skeleton-button"></div>
            <div className="skeleton-button"></div>
        </div>
    );
}

function TableSkeleton() {
    return (
        <div className="table-skeleton">
            {Array.from({ length: 10 }, (_, i) => (
                <div key={i} className="skeleton-row">
                    <div className="skeleton-cell"></div>
                    <div className="skeleton-cell"></div>
                    <div className="skeleton-cell"></div>
                </div>
            ))}
        </div>
    );
}

function LoadingPaginatedTable() {
    const [loading, setLoading] = useState(true);

    return loading ? (
        <div>
            <TableSkeleton />
            <PaginationSkeleton />
        </div>
    ) : (
        <PaginatedTable />
    );
}
```

### 3. **Items Per Page Selector**

```typescript
function ItemsPerPageSelector({ value, options, onChange }) {
    return (
        <div className="items-per-page">
            <label htmlFor="items-per-page">Items per page:</label>
            <select
                id="items-per-page"
                value={value}
                onChange={(e) => onChange(Number(e.target.value))}
            >
                {options.map(option => (
                    <option key={option} value={option}>
                        {option}
                    </option>
                ))}
            </select>
        </div>
    );
}

// Usage
function FlexiblePagination() {
    const [itemsPerPage, setItemsPerPage] = useState(10);
    const options = [5, 10, 25, 50, 100];

    const pagination = usePagination({
        itemsPerPage,
        totalItems: 1000
    });

    return (
        <div>
            <ItemsPerPageSelector
                value={itemsPerPage}
                options={options}
                onChange={(newValue) => {
                    setItemsPerPage(newValue);
                    pagination.setItemsPerPage(newValue);
                }}
            />

            <PaginatedList
                pagination={pagination}
                // ... other props
            />
        </div>
    );
}
```

## Database Optimization

### 1. **Indexing for Pagination**

```sql
-- Create indexes for efficient pagination
CREATE INDEX idx_items_created_at_id ON items (created_at DESC, id DESC);
CREATE INDEX idx_posts_cursor ON posts (created_at DESC, id DESC);

-- For cursor pagination with multiple columns
CREATE INDEX idx_orders_compound ON orders (status, created_at DESC, id);
```

### 2. **Query Optimization**

```typescript
// Optimized query with covered index
const optimizedQuery = `
    SELECT id, title, created_at
    FROM posts
    WHERE (created_at, id) < (?, ?)
    ORDER BY created_at DESC, id DESC
    LIMIT ?
`;

// Avoid SELECT * for better performance
const coveredQuery = `
    SELECT id, title, created_at, author_id
    FROM posts
    WHERE id > ?
    ORDER BY id
    LIMIT ?
`;
```

### 3. **Connection Pooling**

```typescript
// Database connection pooling for better performance
const pool = mysql.createPool({
    connectionLimit: 10,
    host: 'localhost',
    user: 'user',
    password: 'password',
    database: 'mydb'
});

// Use connection pooling instead of single connections
app.get('/api/items', (req, res) => {
    pool.query('SELECT * FROM items LIMIT ?', [limit], (error, results) => {
        res.json(results);
    });
});
```

## Testing Pagination

### 1. **Unit Testing**

```typescript
describe('usePagination', () => {
    it('should initialize with correct state', () => {
        const { result } = renderHook(() =>
            usePagination({ totalItems: 100, itemsPerPage: 10 })
        );

        expect(result.current.currentPage).toBe(1);
        expect(result.current.totalPages).toBe(10);
        expect(result.current.hasNextPage).toBe(true);
        expect(result.current.hasPrevPage).toBe(false);
    });

    it('should navigate to next page', () => {
        const { result } = renderHook(() =>
            usePagination({ totalItems: 100, itemsPerPage: 10 })
        );

        act(() => {
            result.current.nextPage();
        });

        expect(result.current.currentPage).toBe(2);
        expect(result.current.hasPrevPage).toBe(true);
    });

    it('should not go beyond total pages', () => {
        const { result } = renderHook(() =>
            usePagination({ totalItems: 50, itemsPerPage: 10 })
        );

        act(() => {
            result.current.setPage(10); // Beyond total pages
        });

        expect(result.current.currentPage).toBe(5); // Clamped to max
    });
});
```

### 2. **Integration Testing**

```typescript
describe('Pagination Integration', () => {
    it('should load correct page data', async () => {
        const mockApi = jest.fn();
        render(<PaginatedList fetchPage={mockApi} />);

        await waitFor(() => {
            expect(mockApi).toHaveBeenCalledWith(1);
        });

        // Click next page
        const nextButton = screen.getByText('Next');
        fireEvent.click(nextButton);

        await waitFor(() => {
            expect(mockApi).toHaveBeenCalledWith(2);
        });
    });

    it('should handle loading states', async () => {
        const mockApi = jest.fn().mockImplementation(
            () => new Promise(resolve => setTimeout(() => resolve([]), 100))
        );

        render(<PaginatedList fetchPage={mockApi} />);

        expect(screen.getByText('Loading...')).toBeInTheDocument();

        await waitFor(() => {
            expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
        });
    });
});
```

## Common Interview Questions

### Q: How do you implement pagination efficiently?

**A:** For small to medium datasets, I use offset-based pagination with LIMIT and OFFSET. For large datasets, I prefer cursor-based pagination using database indexes on sortable columns. I implement client-side pagination for already-loaded data and server-side pagination for large datasets, always considering the trade-offs between performance and user experience.

### Q: What's the difference between offset and cursor pagination?

**A:** Offset pagination uses LIMIT and OFFSET, which causes performance issues with large datasets as the database must skip rows. Cursor pagination uses a cursor (like an ID) to fetch the next set of records, providing consistent performance regardless of dataset size and avoiding duplicates when data changes.

### Q: How do you handle pagination state in React?

**A:** I create a custom usePagination hook that manages current page, items per page, and total pages. For server state, I use React Query with keepPreviousData to maintain smooth transitions. I implement prefetching for adjacent pages and handle loading states properly to prevent UI jumps.

### Q: What are the performance considerations for pagination?

**A:** Key considerations include database indexing for efficient queries, avoiding SELECT * queries, implementing proper caching, using connection pooling, and choosing the right pagination strategy. I also consider memory usage on the client side and implement virtualization for large page sizes.

### Q: How do you test pagination functionality?

**A:** I test pagination hooks with various scenarios like navigation, boundary conditions, and state management. For integration tests, I verify correct API calls, loading states, and UI updates. I also test error handling and edge cases like empty datasets or network failures.

## Summary

**Pagination Strategies:**
1. **Offset-based** - Simple but degrades with large datasets
2. **Cursor-based** - Efficient for large, changing datasets
3. **Keyset-based** - Advanced cursor with multiple columns

**React Patterns:**
- **Custom hooks** for pagination logic
- **React Query** for server state management
- **Prefetching** for better performance
- **Virtual scrolling** for large pages

**Performance Optimizations:**
- **Database indexing** for efficient queries
- **Query optimization** (avoid SELECT *, use covered indexes)
- **Caching** at multiple levels
- **Connection pooling** for database efficiency

**Best Practices:**
- **Choose strategy** based on dataset size and requirements
- **Implement proper loading states** and error handling
- **Consider accessibility** with keyboard navigation
- **Test thoroughly** for edge cases and performance
- **Monitor performance** and optimize bottlenecks

**Interview Tip:** "I implement pagination efficiently by choosing the right strategy based on dataset size - offset-based for small datasets, cursor-based for large ones. In React, I use custom hooks for state management and React Query for server state, implementing prefetching and proper loading states for the best user experience."