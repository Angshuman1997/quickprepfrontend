# How to stream API responses to UI in Next.js 13+/14?

## Question
How to stream API responses to UI in Next.js 13+/14?

## Answer

Streaming in Next.js App Router allows you to progressively send UI from the server to the client as it becomes ready, improving perceived performance and user experience. This is achieved through React Server Components, Suspense boundaries, and streaming responses.

## Basic Streaming with Suspense

### Server Component with Suspense

```javascript
// app/dashboard/page.js
import { Suspense } from 'react';
import { StatsCards, RecentActivity, AnalyticsChart } from './components';

export default function DashboardPage() {
    return (
        <div>
            <h1>Dashboard</h1>

            {/* Fast loading section */}
            <StatsCards />

            {/* Slow loading sections with streaming */}
            <Suspense fallback={<ActivitySkeleton />}>
                <RecentActivity />
            </Suspense>

            <Suspense fallback={<ChartSkeleton />}>
                <AnalyticsChart />
            </Suspense>
        </div>
    );
}

// components/RecentActivity.js
async function RecentActivity() {
    // Simulate slow API call
    await new Promise(resolve => setTimeout(resolve, 3000));

    const activities = await fetchRecentActivities();

    return (
        <div>
            <h2>Recent Activity</h2>
            {activities.map(activity => (
                <div key={activity.id}>{activity.description}</div>
            ))}
        </div>
    );
}
```

**How Streaming Works:**
1. Server renders `StatsCards` immediately
2. HTML for `StatsCards` streams to client
3. Client hydrates and shows `StatsCards`
4. Server continues rendering `RecentActivity`
5. When ready, `RecentActivity` HTML streams to client
6. Suspense boundary resolves, showing content

## Streaming API Responses

### Streaming from Route Handlers

```javascript
// app/api/stream-data/route.js
import { NextResponse } from 'next/server';

export async function GET() {
    const encoder = new TextEncoder();
    const stream = new ReadableStream({
        start(controller) {
            // Stream data in chunks
            const sendChunk = async (data) => {
                controller.enqueue(encoder.encode(`data: ${JSON.stringify(data)}\n\n`));
                await new Promise(resolve => setTimeout(resolve, 1000)); // Simulate delay
            };

            // Send initial data
            sendChunk({ status: 'processing', progress: 0 });

            // Simulate streaming chunks
            for (let i = 1; i <= 10; i++) {
                sendChunk({
                    status: 'processing',
                    progress: i * 10,
                    data: `Chunk ${i}`
                });
            }

            // Send final result
            sendChunk({ status: 'complete', result: 'All data processed' });

            controller.close();
        }
    });

    return new Response(stream, {
        headers: {
            'Content-Type': 'text/plain',
            'Cache-Control': 'no-cache'
        }
    });
}
```

### Consuming Streamed API Responses

```javascript
// components/StreamingData.js
'use client';

import { useEffect, useState } from 'react';

export function StreamingData() {
    const [data, setData] = useState([]);
    const [status, setStatus] = useState('loading');

    useEffect(() => {
        const eventSource = new EventSource('/api/stream-data');

        eventSource.onmessage = (event) => {
            const chunk = JSON.parse(event.data);

            if (chunk.status === 'complete') {
                setStatus('complete');
                eventSource.close();
            } else {
                setData(prev => [...prev, chunk]);
                setStatus('processing');
            }
        };

        eventSource.onerror = () => {
            setStatus('error');
            eventSource.close();
        };

        return () => eventSource.close();
    }, []);

    return (
        <div>
            <h2>Streaming Data</h2>
            <p>Status: {status}</p>
            <div>
                {data.map((chunk, i) => (
                    <div key={i}>
                        Progress: {chunk.progress}% - {chunk.data}
                    </div>
                ))}
            </div>
        </div>
    );
}
```

## Progressive Loading Patterns

### 1. **Skeleton Loading**

```javascript
// components/Skeleton.js
export function DataSkeleton() {
    return (
        <div className="skeleton">
            <div className="skeleton-header"></div>
            <div className="skeleton-content">
                <div className="skeleton-line"></div>
                <div className="skeleton-line short"></div>
                <div className="skeleton-line"></div>
            </div>
        </div>
    );
}

// app/products/page.js
import { Suspense } from 'react';
import { ProductList } from './ProductList';
import { DataSkeleton } from '../components/Skeleton';

export default function ProductsPage() {
    return (
        <div>
            <h1>Products</h1>
            <Suspense fallback={<DataSkeleton />}>
                <ProductList />
            </Suspense>
        </div>
    );
}
```

### 2. **Progressive Data Loading**

```javascript
// app/analytics/page.js
import { Suspense } from 'react';

function AnalyticsPage() {
    return (
        <div>
            <h1>Analytics Dashboard</h1>

            {/* Load immediately */}
            <SummaryStats />

            {/* Load with medium priority */}
            <Suspense fallback={<ChartSkeleton />}>
                <RevenueChart />
            </Suspense>

            {/* Load with low priority */}
            <Suspense fallback={<TableSkeleton />}>
                <DetailedTable />
            </Suspense>

            {/* Load last */}
            <Suspense fallback={<ReportSkeleton />}>
                <MonthlyReport />
            </Suspense>
        </div>
    );
}
```

### 3. **Conditional Streaming**

```javascript
// components/ConditionalStream.js
import { Suspense } from 'react';

export function ConditionalStream({ condition, children, fallback }) {
    if (!condition) {
        return children; // No streaming needed
    }

    return (
        <Suspense fallback={fallback}>
            {children}
        </Suspense>
    );
}

// Usage
<ConditionalStream
    condition={isSlowData}
    fallback={<LoadingSpinner />}
>
    <SlowComponent />
</ConditionalStream>
```

## Streaming with Server Actions

### Streaming Form Submissions

```javascript
// app/actions/streamAction.js
'use server';

export async function processLargeData(formData) {
    const file = formData.get('file');
    const chunks = [];

    // Process file in chunks
    for await (const chunk of processFileInChunks(file)) {
        chunks.push(chunk);

        // Simulate streaming progress
        await new Promise(resolve => setTimeout(resolve, 100));
    }

    return { result: chunks, totalProcessed: chunks.length };
}

// components/FileProcessor.js
'use client';

import { useTransition } from 'react';
import { processLargeData } from '../actions/streamAction';

export function FileProcessor() {
    const [isPending, startTransition] = useTransition();
    const [progress, setProgress] = useState(0);

    const handleSubmit = async (formData) => {
        startTransition(async () => {
            const result = await processLargeData(formData);
            setProgress(100);
        });
    };

    return (
        <form action={handleSubmit}>
            <input type="file" name="file" />
            <button disabled={isPending}>
                {isPending ? `Processing... ${progress}%` : 'Process File'}
            </button>
        </form>
    );
}
```

## Advanced Streaming Techniques

### 1. **Parallel Data Fetching**

```javascript
// app/dashboard/page.js
async function DashboardPage() {
    // Start all fetches in parallel
    const statsPromise = getStats();
    const usersPromise = getUsers();
    const ordersPromise = getOrders();

    return (
        <div>
            <h1>Dashboard</h1>

            {/* Fastest data loads first */}
            <Suspense fallback={<StatsSkeleton />}>
                <StatsSection promise={statsPromise} />
            </Suspense>

            {/* Medium priority */}
            <Suspense fallback={<UsersSkeleton />}>
                <UsersSection promise={usersPromise} />
            </Suspense>

            {/* Slowest data loads last */}
            <Suspense fallback={<OrdersSkeleton />}>
                <OrdersSection promise={ordersPromise} />
            </Suspense>
        </div>
    );
}

// components/StatsSection.js
async function StatsSection({ promise }) {
    const stats = await promise;

    return (
        <div>
            <h2>Statistics</h2>
            <p>Total: {stats.total}</p>
        </div>
    );
}
```

### 2. **Streaming with Error Boundaries**

```javascript
// components/ErrorBoundary.js
'use client';

import { Component } from 'react';

export class ErrorBoundary extends Component {
    constructor(props) {
        super(props);
        this.state = { hasError: false };
    }

    static getDerivedStateFromError(error) {
        return { hasError: true };
    }

    componentDidCatch(error, errorInfo) {
        console.error('Streaming error:', error, errorInfo);
    }

    render() {
        if (this.state.hasError) {
            return <div>Something went wrong while loading.</div>;
        }

        return this.props.children;
    }
}

// Usage with streaming
<ErrorBoundary>
    <Suspense fallback={<Loading />}>
        <StreamingComponent />
    </Suspense>
</ErrorBoundary>
```

### 3. **Streaming with Loading States**

```javascript
// components/StreamingLoader.js
'use client';

import { useEffect, useState } from 'react';

export function StreamingLoader({ children, fallback }) {
    const [isLoading, setIsLoading] = useState(true);

    useEffect(() => {
        // Simulate streaming detection
        const timer = setTimeout(() => setIsLoading(false), 100);
        return () => clearTimeout(timer);
    }, []);

    if (isLoading) {
        return fallback;
    }

    return children;
}

// Usage
<StreamingLoader fallback={<Skeleton />}>
    <HeavyComponent />
</StreamingLoader>
```

## Streaming Route Segments

### Loading UI for Route Changes

```javascript
// app/dashboard/loading.js
export default function DashboardLoading() {
    return (
        <div className="dashboard-loading">
            <div className="loading-header">
                <h1>Dashboard</h1>
                <div className="spinner"></div>
            </div>
            <div className="loading-content">
                <div className="loading-card"></div>
                <div className="loading-card"></div>
                <div className="loading-card"></div>
            </div>
        </div>
    );
}

// app/dashboard/analytics/page.js
export default async function AnalyticsPage() {
    // This page streams in when navigating to /dashboard/analytics
    const data = await fetchAnalytics();

    return (
        <div>
            <h1>Analytics</h1>
            <AnalyticsChart data={data} />
        </div>
    );
}
```

## Streaming Configuration

### Next.js Configuration

```javascript
// next.config.js
module.exports = {
    experimental: {
        // Enable streaming SSR
        serverComponentsExternalPackages: [],

        // Configure streaming behavior
        swcMinify: true,

        // Enable React 18 features
        reactRoot: true
    }
};
```

### Custom Streaming Implementation

```javascript
// lib/streaming.js
export function createStreamingResponse(generator) {
    const encoder = new TextEncoder();

    const stream = new ReadableStream({
        async start(controller) {
            try {
                for await (const chunk of generator()) {
                    const data = `data: ${JSON.stringify(chunk)}\n\n`;
                    controller.enqueue(encoder.encode(data));
                }
                controller.close();
            } catch (error) {
                controller.error(error);
            }
        }
    });

    return new Response(stream, {
        headers: {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive'
        }
    });
}

// Usage in route handler
export async function GET() {
    return createStreamingResponse(async function* () {
        for (let i = 0; i < 10; i++) {
            yield { progress: (i + 1) * 10, data: `Chunk ${i + 1}` };
            await new Promise(resolve => setTimeout(resolve, 500));
        }
    });
}
```

## Performance Optimization

### Streaming Best Practices

```javascript
// ✅ Good: Prioritize content
export default function Page() {
    return (
        <div>
            {/* Above the fold - load first */}
            <HeroSection />
            <Navigation />

            {/* Below the fold - stream later */}
            <Suspense fallback={<ContentSkeleton />}>
                <MainContent />
            </Suspense>

            <Suspense fallback={<SidebarSkeleton />}>
                <Sidebar />
            </Suspense>
        </div>
    );
}

// ✅ Good: Parallel loading
async function Page() {
    const [headerData, contentData] = await Promise.all([
        fetchHeaderData(),
        fetchContentData()
    ]);

    return (
        <div>
            <Header data={headerData} /> {/* Fast */}
            <Suspense fallback={<Skeleton />}>
                <Content data={contentData} /> {/* Streams */}
            </Suspense>
        </div>
    );
}
```

### Bundle Splitting with Streaming

```javascript
// Automatically split bundles for streaming
import dynamic from 'next/dynamic';

// Heavy component loads separately
const HeavyChart = dynamic(() => import('../components/HeavyChart'), {
    loading: () => <ChartSkeleton />
});

// Usage with streaming
<Suspense fallback={<ChartSkeleton />}>
    <HeavyChart />
</Suspense>
```

## Debugging Streaming

### Monitoring Stream Performance

```javascript
// components/StreamMonitor.js
'use client';

import { useEffect } from 'react';

export function StreamMonitor() {
    useEffect(() => {
        // Monitor streaming performance
        const observer = new PerformanceObserver((list) => {
            for (const entry of list.getEntries()) {
                if (entry.name.includes('streaming')) {
                    console.log('Streaming performance:', entry);
                }
            }
        });

        observer.observe({ entryTypes: ['measure'] });

        return () => observer.disconnect();
    }, []);

    return null;
}

// Add to layout
export default function Layout({ children }) {
    return (
        <html>
            <body>
                <StreamMonitor />
                {children}
            </body>
        </html>
    );
}
```

### Common Streaming Issues

```javascript
// ❌ Problem: Blocking renders
export default async function SlowPage() {
    const slowData = await fetchSlowData(); // Blocks everything
    const fastData = await fetchFastData(); // Waits for slow data

    return (
        <div>
            <FastComponent data={fastData} />
            <SlowComponent data={slowData} />
        </div>
    );
}

// ✅ Solution: Parallel fetching with streaming
export default async function OptimizedPage() {
    const [slowData, fastData] = await Promise.all([
        fetchSlowData(),
        fetchFastData()
    ]);

    return (
        <div>
            <FastComponent data={fastData} />
            <Suspense fallback={<SlowSkeleton />}>
                <SlowComponent data={slowData} />
            </Suspense>
        </div>
    );
}
```

## Testing Streaming Components

### Testing Suspense Boundaries

```javascript
// __tests__/StreamingComponent.test.js
import { render, screen, waitFor } from '@testing-library/react';
import { Suspense } from 'react';
import StreamingComponent from '../StreamingComponent';

// Mock the async operation
jest.mock('../api', () => ({
    fetchData: () => new Promise(resolve => setTimeout(() => resolve('data'), 100))
}));

test('shows loading then content', async () => {
    render(
        <Suspense fallback={<div>Loading...</div>}>
            <StreamingComponent />
        </Suspense>
    );

    // Initially shows loading
    expect(screen.getByText('Loading...')).toBeInTheDocument();

    // Then shows content
    await waitFor(() => {
        expect(screen.getByText('Content loaded')).toBeInTheDocument();
    });
});
```

### Testing Streamed API Responses

```javascript
// __tests__/api/stream-route.test.js
import { GET } from '../../../app/api/stream/route';

test('streams data correctly', async () => {
    const response = await GET();
    const reader = response.body.getReader();
    const decoder = new TextDecoder();

    let chunks = [];
    while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const chunk = decoder.decode(value);
        chunks.push(chunk);
    }

    expect(chunks.length).toBeGreaterThan(0);
    expect(chunks[0]).toContain('data:');
});
```

## Real-World Examples

### 1. **E-commerce Product Page**

```javascript
// app/products/[id]/page.js
import { Suspense } from 'react';

export default async function ProductPage({ params }) {
    const product = await getProduct(params.id); // Fast

    return (
        <div>
            <ProductInfo product={product} />

            {/* Stream related products */}
            <Suspense fallback={<RelatedSkeleton />}>
                <RelatedProducts productId={params.id} />
            </Suspense>

            {/* Stream reviews */}
            <Suspense fallback={<ReviewsSkeleton />}>
                <ProductReviews productId={params.id} />
            </Suspense>
        </div>
    );
}
```

### 2. **Social Media Feed**

```javascript
// app/feed/page.js
import { Suspense } from 'react';

export default async function FeedPage() {
    const initialPosts = await getInitialPosts(); // Fast

    return (
        <div>
            <PostComposer />

            {/* Initial posts load immediately */}
            <PostsList posts={initialPosts} />

            {/* Stream more posts */}
            <Suspense fallback={<LoadMoreSkeleton />}>
                <LoadMorePosts after={initialPosts[initialPosts.length - 1].id} />
            </Suspense>
        </div>
    );
}
```

### 3. **Analytics Dashboard**

```javascript
// app/analytics/page.js
import { Suspense } from 'react';

export default async function AnalyticsPage() {
    return (
        <div>
            <h1>Analytics</h1>

            {/* Critical metrics first */}
            <Suspense fallback={<MetricSkeleton />}>
                <KeyMetrics />
            </Suspense>

            {/* Charts stream in */}
            <div className="charts-grid">
                <Suspense fallback={<ChartSkeleton />}>
                    <RevenueChart />
                </Suspense>

                <Suspense fallback={<ChartSkeleton />}>
                    <UserGrowthChart />
                </Suspense>

                <Suspense fallback={<ChartSkeleton />}>
                    <ConversionChart />
                </Suspense>
            </div>

            {/* Detailed reports last */}
            <Suspense fallback={<ReportSkeleton />}>
                <DetailedReports />
            </Suspense>
        </div>
    );
}
```

## Best Practices

### 1. **Prioritize Content Loading**

```javascript
// Load above-the-fold content first
export default function Page() {
    return (
        <div>
            {/* Critical content - immediate */}
            <Header />
            <Hero />

            {/* Secondary content - streamed */}
            <Suspense fallback={<ContentSkeleton />}>
                <MainContent />
            </Suspense>

            {/* Least important - last */}
            <Suspense fallback={<FooterSkeleton />}>
                <Footer />
            </Suspense>
        </div>
    );
}
```

### 2. **Use Appropriate Loading States**

```javascript
// Match loading UI to actual content structure
<Suspense fallback={
    <div className="article-skeleton">
        <div className="title-skeleton"></div>
        <div className="content-skeleton">
            <div className="line"></div>
            <div className="line short"></div>
            <div className="line"></div>
        </div>
    </div>
}>
    <ArticleContent />
</Suspense>
```

### 3. **Handle Errors Gracefully**

```javascript
// Error boundaries with streaming
<ErrorBoundary fallback={<ErrorUI />}>
    <Suspense fallback={<LoadingUI />}>
        <StreamingComponent />
    </Suspense>
</ErrorBoundary>
```

### 4. **Monitor Performance**

```javascript
// Track streaming metrics
'use client';

export function StreamingTracker() {
    useEffect(() => {
        const observer = new PerformanceObserver((list) => {
            list.getEntries().forEach((entry) => {
                // Log streaming performance
                console.log('Stream loaded in:', entry.duration);
            });
        });

        observer.observe({ entryTypes: ['measure'] });
        return () => observer.disconnect();
    }, []);

    return null;
}
```

### Interview Tip:
*"Streaming in Next.js App Router uses React Suspense to progressively send UI from server to client as it becomes ready, improving perceived performance. Server Components render immediately when data is available, while Suspense boundaries show fallbacks for slower components, enabling parallel loading and better user experience."*