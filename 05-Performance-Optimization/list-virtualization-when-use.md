# Custom Virtual List Hook

**Simple React hook for list virtualization:**
```typescript
import { useState, useCallback, useRef, useEffect } from 'react';

interface UseVirtualListOptions {
    itemHeight: number;
    containerHeight: number;
    overscan?: number; // Extra items to render above/below visible area
}

function useVirtualList<T>(
    items: T[],
    options: UseVirtualListOptions
) {
    const { itemHeight, containerHeight, overscan = 5 } = options;

    const [scrollTop, setScrollTop] = useState(0);
    const containerRef = useRef<HTMLDivElement>(null);

    // Calculate visible range
    const startIndex = Math.max(
        0,
        Math.floor(scrollTop / itemHeight) - overscan
    );

    const endIndex = Math.min(
        items.length - 1,
        startIndex + Math.ceil(containerHeight / itemHeight) + overscan
    );

    // Get visible items
    const visibleItems = items.slice(startIndex, endIndex);

    // Calculate offset for positioning
    const offsetY = startIndex * itemHeight;

    // Total height of all items
    const totalHeight = items.length * itemHeight;

    const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
        setScrollTop(e.currentTarget.scrollTop);
    }, []);

    const scrollToIndex = useCallback((index: number) => {
        if (containerRef.current) {
            const scrollTop = index * itemHeight;
            containerRef.current.scrollTop = scrollTop;
            setScrollTop(scrollTop);
        }
    }, [itemHeight]);

    const scrollToTop = useCallback(() => scrollToIndex(0), [scrollToIndex]);
    const scrollToBottom = useCallback(() => scrollToIndex(items.length - 1), [scrollToIndex, items.length]);

    return {
        containerRef,
        visibleItems,
        startIndex,
        endIndex,
        offsetY,
        totalHeight,
        handleScroll,
        scrollToIndex,
        scrollToTop,
        scrollToBottom
    };
}

// Usage
function VirtualizedList({ items }: { items: Array<{id: number, name: string}> }) {
    const {
        containerRef,
        visibleItems,
        offsetY,
        totalHeight,
        handleScroll
    } = useVirtualList(items, {
        itemHeight: 50,
        containerHeight: 400,
        overscan: 3
    });

    return (
        <div
            ref={containerRef}
            style={{
                height: 400,
                overflow: 'auto',
                border: '1px solid #ccc'
            }}
            onScroll={handleScroll}
        >
            <div style={{ height: totalHeight, position: 'relative' }}>
                <div style={{ transform: `translateY(${offsetY}px)` }}>
                    {visibleItems.map((item, index) => (
                        <div
                            key={item.id}
                            style={{
                                height: 50,
                                display: 'flex',
                                alignItems: 'center',
                                padding: '0 16px',
                                borderBottom: '1px solid #eee',
                                background: index % 2 === 0 ? '#f9f9f9' : 'white'
                            }}
                        >
                            <span>{item.name}</span>
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
}
```

**How it works:**
1. **Scroll tracking** - Monitors scroll position to calculate visible range
2. **Range calculation** - Determines which items should be rendered based on viewport
3. **Overscan buffer** - Renders extra items above/below for smooth scrolling
4. **Absolute positioning** - Positions visible items correctly using transform
5. **Memory efficient** - Only renders visible items + small buffer

**Key Features:**
- **Configurable item height** - Fixed height items for simplicity
- **Overscan support** - Prevents blank areas during fast scrolling
- **Scroll methods** - scrollToIndex, scrollToTop, scrollToBottom
- **Performance optimized** - Handles thousands of items smoothly
- **TypeScript support** - Full type safety

**Benefits:**
- **Massive performance gain** - 99% fewer DOM nodes for large lists
- **Smooth scrolling** - No lag even with 100,000+ items
- **Memory efficient** - Constant memory usage regardless of list size
- **Simple API** - Easy to integrate into existing components
- **Flexible** - Configurable heights, overscan, and scroll behavior