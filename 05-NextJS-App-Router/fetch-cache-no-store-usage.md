# What is fetch({ cache: 'no-store' }) used for?

## Question
What is fetch({ cache: 'no-store' }) used for?

## Answer

In Next.js App Router, the `fetch` API has enhanced caching capabilities that control how responses are cached during server-side rendering. The `cache: 'no-store'` option disables caching entirely, ensuring fresh data on every request.

## Understanding Fetch Cache Options

### Available Cache Options

```javascript
// Default caching (force-cache)
fetch('https://api.example.com/data');

// Explicit force-cache
fetch('https://api.example.com/data', { cache: 'force-cache' });

// No caching
fetch('https://api.example.com/data', { cache: 'no-store' });

// Revalidate after specified time
fetch('https://api.example.com/data', {
    next: { revalidate: 60 }
});

// Cache with tags for selective invalidation
fetch('https://api.example.com/data', {
    next: { tags: ['data'] }
});
```

## When to Use cache: 'no-store'

### 1. **Real-time Data Requirements**

```javascript
// Stock prices that change constantly
async function getStockPrice(symbol) {
    const response = await fetch(
        `https://api.financial.com/stocks/${symbol}`,
        { cache: 'no-store' }
    );

    return response.json();
}

export default async function StockTicker({ params }) {
    const price = await getStockPrice(params.symbol);

    return (
        <div>
            <h1>{params.symbol.toUpperCase()}</h1>
            <p>Current Price: ${price.current}</p>
            <p>Last Updated: {new Date(price.timestamp).toLocaleTimeString()}</p>
        </div>
    );
}
```

### 2. **User-Specific Data**

```javascript
// User profile data that should always be fresh
async function getUserProfile(userId) {
    const response = await fetch(
        `https://api.example.com/users/${userId}`,
        { cache: 'no-store' }
    );

    if (!response.ok) {
        throw new Error('Failed to fetch user profile');
    }

    return response.json();
}

export default async function UserProfile({ params }) {
    const user = await getUserProfile(params.id);

    return (
        <div>
            <h1>{user.name}</h1>
            <p>Email: {user.email}</p>
            <p>Last Login: {new Date(user.lastLogin).toLocaleString()}</p>
        </div>
    );
}
```

### 3. **Authentication Status**

```javascript
// Check authentication status on every request
async function checkAuthStatus() {
    try {
        const response = await fetch(
            'https://api.example.com/auth/status',
            { cache: 'no-store' }
        );

        if (response.ok) {
            return await response.json();
        }

        return null;
    } catch (error) {
        console.error('Auth check failed:', error);
        return null;
    }
}

export default async function ProtectedPage() {
    const authStatus = await checkAuthStatus();

    if (!authStatus?.authenticated) {
        redirect('/login');
    }

    return (
        <div>
            <h1>Welcome, {authStatus.user.name}!</h1>
            <p>Session expires: {new Date(authStatus.expires).toLocaleString()}</p>
        </div>
    );
}
```

### 4. **Dynamic Search Results**

```javascript
// Search results that depend on user input
async function searchProducts(query, filters) {
    const params = new URLSearchParams({
        q: query,
        ...filters
    });

    const response = await fetch(
        `https://api.example.com/search?${params}`,
        { cache: 'no-store' }
    );

    return response.json();
}

export default async function SearchPage({ searchParams }) {
    const { q: query, category, priceRange } = searchParams;
    const results = await searchProducts(query, { category, priceRange });

    return (
        <div>
            <h1>Search Results for "{query}"</h1>
            <p>Found {results.total} products</p>
            <ProductGrid products={results.products} />
        </div>
    );
}
```

### 5. **Shopping Cart Data**

```javascript
// Shopping cart that changes frequently
async function getCart(userId) {
    const response = await fetch(
        `https://api.example.com/cart/${userId}`,
        { cache: 'no-store' }
    );

    return response.json();
}

export default async function CartPage() {
    const cart = await getCart(getCurrentUserId());

    return (
        <div>
            <h1>Shopping Cart</h1>
            <CartItems items={cart.items} />
            <CartTotal total={cart.total} />
        </div>
    );
}
```

### 6. **Live Notifications**

```javascript
// Unread notifications count
async function getNotificationCount(userId) {
    const response = await fetch(
        `https://api.example.com/notifications/count/${userId}`,
        { cache: 'no-store' }
    );

    return response.json();
}

export default async function Header() {
    const { count } = await getNotificationCount(getCurrentUserId());

    return (
        <header>
            <nav>
                <Link href="/dashboard">Dashboard</Link>
                <Link href="/notifications">
                    Notifications {count > 0 && `(${count})`}
                </Link>
            </nav>
        </header>
    );
}
```

## Cache Options Comparison

### force-cache (Default)

```javascript
// Cached for the duration of the request
async function getStaticData() {
    const response = await fetch('https://api.example.com/static-data');
    // Same as { cache: 'force-cache' }
    return response.json();
}

// Use case: Data that doesn't change during a request
export default async function Page() {
    const [posts, categories] = await Promise.all([
        getStaticData(), // Cached
        getStaticData()  // Uses cached response
    ]);

    return <BlogPage posts={posts} categories={categories} />;
}
```

### no-store

```javascript
// Never cached, always fresh
async function getDynamicData() {
    const response = await fetch('https://api.example.com/dynamic-data', {
        cache: 'no-store'
    });
    return response.json();
}

// Use case: Data that must be fresh on every request
export default async function LivePage() {
    const data = await getDynamicData(); // Always fetches fresh
    return <LiveDataDisplay data={data} />;
}
```

### revalidate

```javascript
// Cache with time-based revalidation
async function getPeriodicData() {
    const response = await fetch('https://api.example.com/periodic-data', {
        next: { revalidate: 300 } // 5 minutes
    });
    return response.json();
}

// Use case: Data that updates periodically
export default async function Dashboard() {
    const data = await getPeriodicData(); // Cached for 5 minutes
    return <DashboardContent data={data} />;
}
```

### tags

```javascript
// Cache with tags for selective invalidation
async function getTaggedData() {
    const response = await fetch('https://api.example.com/tagged-data', {
        next: { tags: ['important-data'] }
    });
    return response.json();
}

// Invalidate by tag
import { revalidateTag } from 'next/cache';

function updateData() {
    // Update database
    revalidateTag('important-data'); // Invalidate all tagged requests
}
```

## Advanced Patterns

### 1. **Conditional Caching**

```javascript
async function getData(options = {}) {
    const { forceFresh = false, userId } = options;

    const cacheOption = forceFresh ? 'no-store' : 'force-cache';

    const response = await fetch(
        `https://api.example.com/data?userId=${userId}`,
        { cache: cacheOption }
    );

    return response.json();
}

export default async function Page({ searchParams }) {
    const forceFresh = searchParams.refresh === 'true';
    const data = await getData({ forceFresh, userId: getCurrentUserId() });

    return <DataDisplay data={data} />;
}
```

### 2. **Hybrid Caching Strategy**

```javascript
async function getHybridData() {
    // Get cached base data
    const baseData = await fetch('https://api.example.com/base-data');

    // Always get fresh user-specific data
    const userData = await fetch(
        `https://api.example.com/user-data/${getCurrentUserId()}`,
        { cache: 'no-store' }
    );

    return { ...baseData, ...userData };
}

export default async function PersonalizedPage() {
    const data = await getHybridData();

    return (
        <div>
            <StaticContent data={data.base} />
            <PersonalizedContent data={data.user} />
        </div>
    );
}
```

### 3. **Cache Busting with Query Parameters**

```javascript
async function getVersionedData(version) {
    const response = await fetch(
        `https://api.example.com/data?v=${version}`,
        { cache: 'no-store' }
    );

    return response.json();
}

export default async function VersionedPage({ params }) {
    const data = await getVersionedData(params.version);

    return <VersionedContent data={data} version={params.version} />;
}
```

## Performance Implications

### Cache Hit Scenarios

```javascript
// ✅ Good: Cache static data
async function getCategories() {
    return await fetch('https://api.example.com/categories');
    // Cached by default, fast subsequent requests
}

// ❌ Bad: Cache dynamic data unnecessarily
async function getUserBalance() {
    return await fetch(`https://api.example.com/balance/${userId}`);
    // Should use no-store for real-time balance
}

// ✅ Good: Use appropriate caching
async function getUserBalance() {
    return await fetch(
        `https://api.example.com/balance/${userId}`,
        { cache: 'no-store' }
    );
    // Always fresh, appropriate for financial data
}
```

### Request Deduplication

```javascript
// Next.js automatically deduplicates identical requests in the same render
export default async function Page() {
    const [data1, data2, data3] = await Promise.all([
        fetch('https://api.example.com/data'), // Makes 1 request
        fetch('https://api.example.com/data'), // Uses cached response
        fetch('https://api.example.com/data')  // Uses cached response
    ]);

    return <div>{data1.value}</div>;
}
```

## Error Handling with Cache Options

```javascript
async function safeFetch(url, options = {}) {
    try {
        const response = await fetch(url, options);

        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        return await response.json();
    } catch (error) {
        console.error(`Fetch failed for ${url}:`, error);

        // Fallback for cached requests
        if (options.cache !== 'no-store') {
            console.log('Attempting fallback...');
            // Try fallback API or cached data
        }

        throw error;
    }
}

export default async function RobustPage() {
    let data;

    try {
        data = await safeFetch('https://api.example.com/data', {
            cache: 'no-store'
        });
    } catch (error) {
        data = await getFallbackData();
    }

    return <PageContent data={data} />;
}
```

## Real-World Examples

### E-commerce Product Availability

```javascript
async function checkProductAvailability(productId) {
    const response = await fetch(
        `https://api.store.com/products/${productId}/availability`,
        { cache: 'no-store' }
    );

    const availability = await response.json();

    return {
        inStock: availability.quantity > 0,
        quantity: availability.quantity,
        lastChecked: new Date().toISOString()
    };
}

export default async function ProductPage({ params }) {
    const availability = await checkProductAvailability(params.id);

    return (
        <div>
            <ProductDetails productId={params.id} />
            <AvailabilityStatus
                inStock={availability.inStock}
                quantity={availability.quantity}
                lastChecked={availability.lastChecked}
            />
        </div>
    );
}
```

### Social Media Feed

```javascript
async function getFeed(userId, cursor) {
    const params = new URLSearchParams();
    if (cursor) params.set('cursor', cursor);

    const response = await fetch(
        `https://api.social.com/feed/${userId}?${params}`,
        { cache: 'no-store' }
    );

    return response.json();
}

export default async function FeedPage({ searchParams }) {
    const feed = await getFeed(getCurrentUserId(), searchParams.cursor);

    return (
        <div>
            <PostList posts={feed.posts} />
            {feed.hasMore && (
                <LoadMoreButton cursor={feed.nextCursor} />
            )}
        </div>
    );
}
```

### Live Auction Bids

```javascript
async function getAuctionStatus(auctionId) {
    const response = await fetch(
        `https://api.auction.com/auctions/${auctionId}/status`,
        { cache: 'no-store' }
    );

    return response.json();
}

export default async function AuctionPage({ params }) {
    const status = await getAuctionStatus(params.id);

    return (
        <div>
            <AuctionHeader auction={status.auction} />
            <CurrentBid bid={status.currentBid} />
            <BidHistory bids={status.recentBids} />
            <PlaceBidForm auctionId={params.id} />
        </div>
    );
}
```

## Best Practices

### 1. **Choose Appropriate Caching**

```javascript
// Static reference data - cache aggressively
const getCountries = () => fetch('/api/countries');

// User preferences - cache per user
const getUserPrefs = (userId) =>
    fetch(`/api/users/${userId}/preferences`, { next: { revalidate: 300 } });

// Real-time data - no cache
const getLiveStats = () =>
    fetch('/api/live-stats', { cache: 'no-store' });
```

### 2. **Handle Cache Invalidation**

```javascript
// Server Action with cache invalidation
'use server';

export async function updateUserProfile(userId, data) {
    await updateUserInDB(userId, data);

    // Invalidate related caches
    revalidatePath(`/users/${userId}`);
    revalidateTag(`user-${userId}`);
}
```

### 3. **Monitor Cache Performance**

```javascript
// Add logging for cache analysis
async function loggedFetch(url, options = {}) {
    const startTime = Date.now();
    const response = await fetch(url, options);
    const duration = Date.now() - startTime;

    console.log(`Fetch ${url}: ${duration}ms, cache: ${options.cache || 'default'}`);

    return response;
}
```

### 4. **Fallback Strategies**

```javascript
async function fetchWithFallback(primaryUrl, fallbackUrl) {
    try {
        return await fetch(primaryUrl, { cache: 'no-store' });
    } catch (error) {
        console.warn('Primary API failed, using fallback');
        return await fetch(fallbackUrl, { cache: 'force-cache' });
    }
}
```

## Common Mistakes

### 1. **Over-caching Dynamic Data**

```javascript
// ❌ Bad: Caching real-time data
async function getCurrentWeather() {
    return await fetch('https://api.weather.com/current');
    // Weather changes constantly, should use no-store
}

// ✅ Good: No caching for real-time data
async function getCurrentWeather() {
    return await fetch('https://api.weather.com/current', {
        cache: 'no-store'
    });
}
```

### 2. **Under-caching Static Data**

```javascript
// ❌ Bad: No caching for static data
async function getSiteConfig() {
    return await fetch('/api/config', { cache: 'no-store' });
    // Config rarely changes, wasteful to always fetch
}

// ✅ Good: Cache static data
async function getSiteConfig() {
    return await fetch('/api/config', { next: { revalidate: 3600 } });
}
```

### 3. **Ignoring Request Deduplication**

```javascript
// ❌ Bad: Multiple calls to same endpoint
export default async function Page() {
    const data1 = await fetch('/api/data');
    const data2 = await fetch('/api/data'); // Duplicate request
    const data3 = await fetch('/api/data'); // Another duplicate

    return <div>{data1.value}</div>;
}

// ✅ Good: Single request, automatic deduplication
export default async function Page() {
    const [data1, data2, data3] = await Promise.all([
        fetch('/api/data'), // 1 request made
        fetch('/api/data'), // Uses cached response
        fetch('/api/data')  // Uses cached response
    ]);

    return <div>{data1.value}</div>;
}
```

### Interview Tip:
*"`cache: 'no-store'` disables all caching and ensures fresh data on every request, making it essential for real-time data like stock prices, user authentication status, search results, and any content that must be current. Use it when data freshness is more important than performance."*