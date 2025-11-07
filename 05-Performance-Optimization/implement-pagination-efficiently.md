# Custom Pagination Hook

**Simple React hook for pagination:**
```typescript
import { useState, useCallback } from 'react';

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

            <div className="pagination-controls">
                <button onClick={prevPage} disabled={!hasPrevPage}>
                    Previous
                </button>
                <span>Page {currentPage} of {totalPages}</span>
                <button onClick={nextPage} disabled={!hasNextPage}>
                    Next
                </button>
            </div>
        </div>
    );
}
```

**How it works:**
1. **State management** - Tracks current page, items per page, and total items
2. **Navigation methods** - setPage, nextPage, prevPage functions
3. **Dynamic updates** - setItemsPerPage and setTotalItems for flexibility
4. **Boundary checks** - Prevents navigation beyond valid page ranges
5. **Computed properties** - hasNextPage and hasPrevPage for UI state

**Key Features:**
- **Flexible configuration** - Set initial page, items per page, total items
- **Safe navigation** - Automatic boundary checking
- **Dynamic updates** - Change items per page or total items
- **Computed state** - hasNext/hasPrev for button states
- **Reset behavior** - Changes items per page resets to page 1

**Benefits:**
- **Reusable** - Works with any data source
- **Type-safe** - Full TypeScript support
- **Flexible** - Configurable for different use cases
- **Safe** - Prevents invalid page navigation
- **Simple** - Easy to integrate and use