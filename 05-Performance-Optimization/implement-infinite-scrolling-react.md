# Infinite Scroll in React

## Custom Hook: useInfiniteScroll

```typescript
import { useState, useEffect, useCallback } from 'react';

function useInfiniteScroll(callback: () => void, hasMore: boolean) {
    const [isFetching, setIsFetching] = useState(false);
    const [targetRef, setTargetRef] = useState<Element | null>(null);

    const observerCallback = useCallback(
        (entries: IntersectionObserverEntry[]) => {
            const [entry] = entries;
            if (entry.isIntersecting && !isFetching && hasMore) {
                setIsFetching(true);
                callback();
            }
        },
        [callback, isFetching, hasMore]
    );

    useEffect(() => {
        if (!targetRef) return;

        const observer = new IntersectionObserver(observerCallback, {
            threshold: 0.1
        });

        observer.observe(targetRef);
        return () => observer.disconnect();
    }, [targetRef, observerCallback]);

    const resetFetching = useCallback(() => setIsFetching(false), []);

    return { setRef: setTargetRef, isFetching, resetFetching };
}
```

## How to Use It

```typescript
function ProductList() {
    const [products, setProducts] = useState([]);
    const [page, setPage] = useState(1);
    const [hasMore, setHasMore] = useState(true);

    const loadMore = useCallback(async () => {
        const response = await fetch(`/api/products?page=${page}`);
        const data = await response.json();

        setProducts(prev => [...prev, ...data.products]);
        setHasMore(data.hasMore);
        setPage(prev => prev + 1);

        resetFetching(); // Reset loading state
    }, [page, resetFetching]);

    const { setRef, isFetching, resetFetching } = useInfiniteScroll(loadMore, hasMore);

    return (
        <div>
            {products.map(product => (
                <div key={product.id} className="product-card">
                    <h3>{product.name}</h3>
                    <p>${product.price}</p>
                </div>
            ))}

            {/* Loading trigger element */}
            {hasMore && (
                <div ref={setRef} className="loading-trigger">
                    {isFetching && <div>Loading more products...</div>}
                </div>
            )}

            {!hasMore && <div>No more products to load</div>}
        </div>
    );
}
```

## Key Points

- **IntersectionObserver**: Detects when user scrolls near bottom
- **hasMore flag**: Prevents unnecessary API calls when no more data
- **Loading state**: Shows feedback while fetching
- **Reset function**: Call this after your API call completes

## When to Use Infinite Scroll

✅ **Good for:**
- Social media feeds
- Product catalogs
- Search results
- Long lists of content

❌ **Avoid for:**
- Content that needs pagination numbers
- SEO-critical pages (search engines can't scroll)
- Short lists (< 50 items)

**Interview Tip:** "Infinite scroll uses IntersectionObserver to detect when a user reaches the bottom of a list, then loads more content automatically. It's great for performance because it only loads what the user needs to see."