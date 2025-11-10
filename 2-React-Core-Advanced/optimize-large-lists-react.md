# How do you optimize large lists in React?

## Question
How do you optimize large lists in React?

## Answer

Optimizing large lists in React involves several techniques to prevent performance issues when rendering hundreds or thousands of items. The key is to minimize DOM manipulation, reduce unnecessary re-renders, and only render visible items.

## The Problem with Large Lists

### Performance Issues:
- **DOM manipulation**: Creating thousands of DOM nodes is expensive
- **Memory usage**: Large virtual DOM trees consume memory
- **Render time**: Every item re-renders on any state change
- **Scroll performance**: Poor scrolling experience with many items

### Example of Inefficient List:

```javascript
function LargeList({ items }) {
    return (
        <ul>
            {items.map(item => (
                <li key={item.id}>
                    <ItemComponent item={item} />
                </li>
            ))}
        </ul>
    );
}

// If items has 10,000 elements, this creates 10,000+ DOM nodes
```

## Optimization Techniques

### 1. **React.memo for Item Components**

Prevent unnecessary re-renders of individual items:

```javascript
const ItemComponent = React.memo(function ItemComponent({ item, onClick }) {
    console.log('Item rendered:', item.id);

    return (
        <div onClick={() => onClick(item.id)}>
            <h3>{item.title}</h3>
            <p>{item.description}</p>
        </div>
    );
});
```

### 2. **useMemo for Computed Lists**

Memoize filtered/sorted lists to prevent recalculation:

```javascript
function FilteredList({ items, filter }) {
    const filteredItems = useMemo(() => {
        console.log('Filtering items...');
        return items.filter(item =>
            item.title.toLowerCase().includes(filter.toLowerCase())
        );
    }, [items, filter]);

    const sortedItems = useMemo(() => {
        console.log('Sorting items...');
        return [...filteredItems].sort((a, b) => a.title.localeCompare(b.title));
    }, [filteredItems]);

    return (
        <ul>
            {sortedItems.map(item => (
                <li key={item.id}>
                    <ItemComponent item={item} />
                </li>
            ))}
        </ul>
    );
}
```

### 3. **Virtual Scrolling (Windowing)**

Only render visible items in the viewport:

```javascript
function VirtualizedList({ items, itemHeight = 50, containerHeight = 400 }) {
    const [scrollTop, setScrollTop] = useState(0);

    // Calculate visible range
    const visibleRange = useMemo(() => {
        const startIndex = Math.floor(scrollTop / itemHeight);
        const endIndex = Math.min(
            startIndex + Math.ceil(containerHeight / itemHeight) + 1,
            items.length - 1
        );
        return { startIndex, endIndex };
    }, [scrollTop, itemHeight, containerHeight, items.length]);

    // Get visible items
    const visibleItems = useMemo(() => {
        return items.slice(visibleRange.startIndex, visibleRange.endIndex + 1);
    }, [items, visibleRange]);

    // Total height of all items
    const totalHeight = items.length * itemHeight;

    // Offset for visible items
    const offsetY = visibleRange.startIndex * itemHeight;

    return (
        <div
            style={{
                height: containerHeight,
                overflow: 'auto',
                border: '1px solid #ccc'
            }}
            onScroll={(e) => setScrollTop(e.target.scrollTop)}
        >
            <div style={{ height: totalHeight, position: 'relative' }}>
                <div style={{ transform: `translateY(${offsetY}px)` }}>
                    {visibleItems.map((item, index) => (
                        <div
                            key={item.id}
                            style={{
                                height: itemHeight,
                                display: 'flex',
                                alignItems: 'center',
                                borderBottom: '1px solid #eee'
                            }}
                        >
                            {item.title}
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
}
```

### 4. **Pagination**

Split large datasets into pages:

```javascript
function PaginatedList({ items, itemsPerPage = 20 }) {
    const [currentPage, setCurrentPage] = useState(1);

    const totalPages = Math.ceil(items.length / itemsPerPage);

    const currentItems = useMemo(() => {
        const startIndex = (currentPage - 1) * itemsPerPage;
        const endIndex = startIndex + itemsPerPage;
        return items.slice(startIndex, endIndex);
    }, [items, currentPage, itemsPerPage]);

    return (
        <div>
            <ul>
                {currentItems.map(item => (
                    <li key={item.id}>
                        <ItemComponent item={item} />
                    </li>
                ))}
            </ul>

            <div className="pagination">
                <button
                    onClick={() => setCurrentPage(p => Math.max(1, p - 1))}
                    disabled={currentPage === 1}
                >
                    Previous
                </button>

                <span>Page {currentPage} of {totalPages}</span>

                <button
                    onClick={() => setCurrentPage(p => Math.min(totalPages, p + 1))}
                    disabled={currentPage === totalPages}
                >
                    Next
                </button>
            </div>
        </div>
    );
}
```

### 5. **Infinite Scrolling**

Load more items as user scrolls:

```javascript
function InfiniteList({ fetchMoreItems }) {
    const [items, setItems] = useState([]);
    const [loading, setLoading] = useState(false);
    const [hasMore, setHasMore] = useState(true);
    const loaderRef = useRef();

    const loadMore = useCallback(async () => {
        if (loading || !hasMore) return;

        setLoading(true);
        try {
            const newItems = await fetchMoreItems(items.length);
            if (newItems.length === 0) {
                setHasMore(false);
            } else {
                setItems(prev => [...prev, ...newItems]);
            }
        } catch (error) {
            console.error('Failed to load more items:', error);
        } finally {
            setLoading(false);
        }
    }, [items.length, loading, hasMore, fetchMoreItems]);

    useEffect(() => {
        const observer = new IntersectionObserver(
            (entries) => {
                if (entries[0].isIntersecting) {
                    loadMore();
                }
            },
            { threshold: 0.1 }
        );

        if (loaderRef.current) {
            observer.observe(loaderRef.current);
        }

        return () => observer.disconnect();
    }, [loadMore]);

    return (
        <div>
            {items.map(item => (
                <div key={item.id} className="item">
                    <ItemComponent item={item} />
                </div>
            ))}

            {hasMore && (
                <div ref={loaderRef} className="loading-indicator">
                    {loading ? 'Loading more...' : 'Scroll for more'}
                </div>
            )}
        </div>
    );
}
```

## Advanced Optimization Libraries

### 1. **react-window**

High-performance virtual scrolling:

```javascript
import { FixedSizeList as List } from 'react-window';

function VirtualizedList({ items }) {
    const Row = ({ index, style }) => (
        <div style={style}>
            <ItemComponent item={items[index]} />
        </div>
    );

    return (
        <List
            height={400}
            itemCount={items.length}
            itemSize={50}
            width={300}
        >
            {Row}
        </List>
    );
}
```

### 2. **react-virtualized**

More features than react-window:

```javascript
import { List, AutoSizer } from 'react-virtualized';

function VirtualizedTable({ items }) {
    const rowRenderer = ({ index, key, style }) => (
        <div key={key} style={style}>
            <ItemComponent item={items[index]} />
        </div>
    );

    return (
        <AutoSizer>
            {({ height, width }) => (
                <List
                    height={height}
                    rowCount={items.length}
                    rowHeight={50}
                    rowRenderer={rowRenderer}
                    width={width}
                />
            )}
        </AutoSizer>
    );
}
```

### 3. **react-window-infinite-loader**

Infinite scrolling with virtualization:

```javascript
import { InfiniteLoader, List } from 'react-window-infinite-loader';

function InfiniteVirtualizedList({ items, loadMoreItems, hasNextPage }) {
    const itemCount = hasNextPage ? items.length + 1 : items.length;

    const isItemLoaded = (index) => !hasNextPage || index < items.length;

    const loadMoreItemsCallback = (startIndex, stopIndex) => {
        return loadMoreItems(startIndex, stopIndex);
    };

    const Item = ({ index, style }) => {
        if (!isItemLoaded(index)) {
            return <div style={style}>Loading...</div>;
        }

        return (
            <div style={style}>
                <ItemComponent item={items[index]} />
            </div>
        );
    };

    return (
        <InfiniteLoader
            isItemLoaded={isItemLoaded}
            itemCount={itemCount}
            loadMoreItems={loadMoreItemsCallback}
        >
            {({ onItemsRendered, ref }) => (
                <List
                    ref={ref}
                    height={400}
                    itemCount={itemCount}
                    itemSize={50}
                    onItemsRendered={onItemsRendered}
                    width={300}
                >
                    {Item}
                </List>
            )}
        </InfiniteLoader>
    );
}
```

## Real React Examples

### 1. **Optimized Contact List**

```javascript
function ContactList({ contacts, searchTerm }) {
    // Memoize filtered contacts
    const filteredContacts = useMemo(() => {
        if (!searchTerm) return contacts;

        return contacts.filter(contact =>
            contact.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
            contact.email.toLowerCase().includes(searchTerm.toLowerCase())
        );
    }, [contacts, searchTerm]);

    // Memoize sorted contacts
    const sortedContacts = useMemo(() => {
        return [...filteredContacts].sort((a, b) =>
            a.name.localeCompare(b.name)
        );
    }, [filteredContacts]);

    return (
        <div className="contact-list">
            <div className="search-bar">
                <input
                    type="text"
                    placeholder="Search contacts..."
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                />
            </div>

            <div className="contacts-container">
                {sortedContacts.length === 0 ? (
                    <div className="no-results">No contacts found</div>
                ) : (
                    sortedContacts.map(contact => (
                        <ContactItem
                            key={contact.id}
                            contact={contact}
                            onSelect={handleContactSelect}
                        />
                    ))
                )}
            </div>
        </div>
    );
}

const ContactItem = React.memo(function ContactItem({ contact, onSelect }) {
    return (
        <div
            className="contact-item"
            onClick={() => onSelect(contact)}
        >
            <div className="avatar">
                {contact.name.charAt(0).toUpperCase()}
            </div>
            <div className="contact-info">
                <div className="name">{contact.name}</div>
                <div className="email">{contact.email}</div>
            </div>
        </div>
    );
});
```

### 2. **Virtualized Data Table**

```javascript
function DataTable({ data, columns }) {
    const [sortConfig, setSortConfig] = useState({
        key: null,
        direction: 'asc'
    });

    // Memoize sorted data
    const sortedData = useMemo(() => {
        if (!sortConfig.key) return data;

        return [...data].sort((a, b) => {
            const aValue = a[sortConfig.key];
            const bValue = b[sortConfig.key];

            if (aValue < bValue) {
                return sortConfig.direction === 'asc' ? -1 : 1;
            }
            if (aValue > bValue) {
                return sortConfig.direction === 'asc' ? 1 : -1;
            }
            return 0;
        });
    }, [data, sortConfig]);

    const handleSort = (key) => {
        setSortConfig(prev => ({
            key,
            direction: prev.key === key && prev.direction === 'asc' ? 'desc' : 'asc'
        }));
    };

    return (
        <div className="data-table">
            <table>
                <thead>
                    <tr>
                        {columns.map(column => (
                            <th
                                key={column.key}
                                onClick={() => handleSort(column.key)}
                                className="sortable"
                            >
                                {column.label}
                                {sortConfig.key === column.key && (
                                    <span>{sortConfig.direction === 'asc' ? ' ↑' : ' ↓'}</span>
                                )}
                            </th>
                        ))}
                    </tr>
                </thead>
                <tbody>
                    {sortedData.map((row, index) => (
                        <tr key={row.id || index}>
                            {columns.map(column => (
                                <td key={column.key}>
                                    {column.render ? column.render(row) : row[column.key]}
                                </td>
                            ))}
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
}

// Usage with large dataset
const columns = [
    { key: 'name', label: 'Name' },
    { key: 'email', label: 'Email' },
    { key: 'status', label: 'Status' },
    {
        key: 'actions',
        label: 'Actions',
        render: (row) => (
            <button onClick={() => handleEdit(row)}>Edit</button>
        )
    }
];

<DataTable data={largeDataset} columns={columns} />
```

### 3. **Optimized Chat Messages**

```javascript
function ChatMessages({ messages, currentUserId }) {
    const messagesEndRef = useRef();

    // Auto-scroll to bottom when new messages arrive
    const scrollToBottom = () => {
        messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
    };

    useEffect(() => {
        scrollToBottom();
    }, [messages]);

    return (
        <div className="chat-messages">
            {messages.map((message, index) => {
                const isCurrentUser = message.userId === currentUserId;
                const showAvatar = index === 0 ||
                    messages[index - 1].userId !== message.userId;

                return (
                    <MessageItem
                        key={message.id}
                        message={message}
                        isCurrentUser={isCurrentUser}
                        showAvatar={showAvatar}
                    />
                );
            })}
            <div ref={messagesEndRef} />
        </div>
    );
}

const MessageItem = React.memo(function MessageItem({
    message,
    isCurrentUser,
    showAvatar
}) {
    return (
        <div className={`message ${isCurrentUser ? 'own' : 'other'}`}>
            {showAvatar && (
                <div className="avatar">
                    {message.userName.charAt(0)}
                </div>
            )}
            <div className="message-content">
                <div className="message-text">{message.text}</div>
                <div className="message-time">
                    {new Date(message.timestamp).toLocaleTimeString()}
                </div>
            </div>
        </div>
    );
});
```

### 4. **Search with Debounced Filtering**

```javascript
function SearchableList({ items }) {
    const [searchTerm, setSearchTerm] = useState('');
    const [debouncedSearchTerm, setDebouncedSearchTerm] = useState('');

    // Debounce search term to avoid excessive filtering
    useEffect(() => {
        const timer = setTimeout(() => {
            setDebouncedSearchTerm(searchTerm);
        }, 300);

        return () => clearTimeout(timer);
    }, [searchTerm]);

    // Memoize filtered results
    const filteredItems = useMemo(() => {
        if (!debouncedSearchTerm) return items;

        const term = debouncedSearchTerm.toLowerCase();
        return items.filter(item =>
            item.title.toLowerCase().includes(term) ||
            item.description.toLowerCase().includes(term) ||
            item.tags.some(tag => tag.toLowerCase().includes(term))
        );
    }, [items, debouncedSearchTerm]);

    return (
        <div className="searchable-list">
            <div className="search-input">
                <input
                    type="text"
                    placeholder="Search items..."
                    value={searchTerm}
                    onChange={(e) => setSearchTerm(e.target.value)}
                />
                {searchTerm !== debouncedSearchTerm && (
                    <span className="searching">Searching...</span>
                )}
            </div>

            <div className="results">
                <div className="results-count">
                    {filteredItems.length} results
                </div>

                {filteredItems.map(item => (
                    <SearchResultItem key={item.id} item={item} />
                ))}
            </div>
        </div>
    );
}

const SearchResultItem = React.memo(function SearchResultItem({ item }) {
    return (
        <div className="search-result">
            <h3>{item.title}</h3>
            <p>{item.description}</p>
            <div className="tags">
                {item.tags.map(tag => (
                    <span key={tag} className="tag">{tag}</span>
                ))}
            </div>
        </div>
    );
});
```

## Performance Monitoring

### 1. **React DevTools Profiler**

Use React DevTools to identify performance bottlenecks:

```javascript
// Wrap components with Profiler to measure performance
import { Profiler } from 'react';

function onRenderCallback(
    id,
    phase,
    actualDuration,
    baseDuration,
    startTime,
    commitTime
) {
    console.log(`${id} took ${actualDuration}ms to render`);
}

<Profiler id="LargeList" onRender={onRenderCallback}>
    <LargeList items={items} />
</Profiler>
```

### 2. **Performance Metrics**

```javascript
// Measure render time
function useRenderTime() {
    const renderTimeRef = useRef();
    const [renderTime, setRenderTime] = useState(0);

    useEffect(() => {
        renderTimeRef.current = performance.now();
    });

    useEffect(() => {
        const startTime = renderTimeRef.current;
        const endTime = performance.now();
        setRenderTime(endTime - startTime);
    });

    return renderTime;
}
```

## Best Practices

### 1. **When to Optimize**
- Lists with 100+ items
- Frequent updates to list data
- Complex item components
- Mobile applications

### 2. **Optimization Strategy**
- Start with React.memo and useMemo
- Add virtualization for 1000+ items
- Consider pagination for very large datasets
- Use infinite scrolling for continuous data

### 3. **Common Pitfalls**
- **Over-optimization**: Don't optimize prematurely
- **Missing keys**: Always provide stable keys
- **Inline functions**: Avoid creating functions in render
- **Unnecessary memoization**: Only memoize when it helps

### 4. **Testing Performance**
```javascript
// Test with large datasets
const largeDataset = Array.from({ length: 10000 }, (_, i) => ({
    id: i,
    title: `Item ${i}`,
    description: `Description for item ${i}`
}));

// Use React Testing Library with large data
test('renders large list efficiently', () => {
    const { container } = render(<VirtualizedList items={largeDataset} />);
    // Test that only visible items are rendered
    expect(container.querySelectorAll('.item')).toHaveLength(10); // visible items
});
```

### Interview Tip:
*"For large lists, use React.memo for items, useMemo for computed data, and virtualization libraries like react-window for 1000+ items. Pagination and infinite scrolling help with very large datasets. Always profile first to identify actual bottlenecks."*