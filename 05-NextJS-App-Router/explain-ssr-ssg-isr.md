# Explain SSR vs SSG vs ISR

## Question
Explain SSR vs SSG vs ISR.

## Answer

Next.js provides different rendering strategies to optimize performance, SEO, and user experience. Understanding SSR (Server-Side Rendering), SSG (Static Site Generation), and ISR (Incremental Static Regeneration) is crucial for choosing the right approach for different types of content.

## Server-Side Rendering (SSR)

### What is SSR?

Server-Side Rendering generates the HTML for a page on each request. The server executes React components, fetches data, and sends fully rendered HTML to the browser.

### How SSR Works

```javascript
// Pages Router - SSR
export default function UserProfile({ user }) {
    return (
        <div>
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}

export async function getServerSideProps(context) {
    const { id } = context.params;
    const user = await fetchUser(id);

    return {
        props: { user }
    };
}

// App Router - SSR (default for dynamic routes)
export default async function UserProfile({ params }) {
    const user = await fetchUser(params.id);

    return (
        <div>
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}
```

### SSR Characteristics

**Pros:**
- **Fresh Data**: Always serves up-to-date content
- **SEO Friendly**: Search engines see fully rendered HTML
- **Dynamic Content**: Perfect for personalized content
- **No Client-Side JavaScript Required**: Works without JavaScript

**Cons:**
- **Slower TTFB**: Server must render on each request
- **Higher Server Load**: Each request requires server processing
- **Database Load**: Frequent database queries
- **Scalability**: More expensive to scale

### When to Use SSR

```javascript
// Real-time data that changes frequently
function StockPrice({ symbol }) {
    // Data changes every few seconds
    return <div>Current price: ${price}</div>;
}

// User-specific content
function Dashboard({ user }) {
    // Personalized for each user
    return <div>Welcome {user.name}</div>;
}

// Search results
function SearchResults({ query, results }) {
    // Results depend on user input
    return <div>Found {results.length} results for "{query}"</div>;
}
```

## Static Site Generation (SSG)

### What is SSG?

Static Site Generation pre-generates HTML pages at build time. Pages are created once and served as static files, making them extremely fast to deliver.

### How SSG Works

```javascript
// Pages Router - SSG
export default function BlogPost({ post }) {
    return (
        <article>
            <h1>{post.title}</h1>
            <div>{post.content}</div>
        </article>
    );
}

export async function getStaticProps({ params }) {
    const post = await fetchPost(params.slug);

    return {
        props: { post }
    };
}

export async function getStaticPaths() {
    const posts = await fetchAllPosts();

    return {
        paths: posts.map(post => ({
            params: { slug: post.slug }
        })),
        fallback: false
    };
}

// App Router - SSG
export async function generateStaticParams() {
    const posts = await fetchAllPosts();

    return posts.map(post => ({
        slug: post.slug
    }));
}

export default async function BlogPost({ params }) {
    const post = await fetchPost(params.slug);

    return (
        <article>
            <h1>{post.title}</h1>
            <div>{post.content}</div>
        </article>
    );
}
```

### SSG Characteristics

**Pros:**
- **Fastest TTFB**: Pre-generated static files
- **Excellent Performance**: Served from CDN
- **Low Server Load**: No server processing per request
- **Highly Scalable**: Can handle massive traffic
- **SEO Friendly**: Fully rendered HTML

**Cons:**
- **Build Time**: Long builds for large sites
- **Stale Content**: Content only updates on rebuild
- **Storage**: Many static files for large sites
- **Dynamic Content**: Not suitable for personalized content

### When to Use SSG

```javascript
// Marketing pages
function LandingPage() {
    // Content rarely changes
    return <div>Our amazing product...</div>;
}

// Blog posts
function BlogPost({ post }) {
    // Content is relatively static
    return <article>{post.content}</article>;
}

// Documentation
function DocsPage({ content }) {
    // Versioned content, updates with releases
    return <div>{content}</div>;
}

// Product catalogs (if not too large)
function ProductList({ products }) {
    // Products update periodically
    return <div>{products.map(product => <ProductCard key={product.id} product={product} />)}</div>;
}
```

## Incremental Static Regeneration (ISR)

### What is ISR?

Incremental Static Regeneration combines the benefits of SSG and SSR. Pages are pre-generated at build time, but can be regenerated on-demand or at intervals without a full rebuild.

### How ISR Works

```javascript
// Pages Router - ISR
export async function getStaticProps() {
    const posts = await fetchPosts();

    return {
        props: { posts },
        revalidate: 60 // Regenerate every 60 seconds
    };
}

// App Router - ISR
export const revalidate = 60; // Regenerate every 60 seconds

export default async function BlogPage() {
    const posts = await fetchPosts();
    return <PostList posts={posts} />;
}

// On-demand revalidation
export async function POST() {
    revalidatePath('/blog');
    return NextResponse.json({ success: true });
}
```

### ISR Characteristics

**Pros:**
- **Fast TTFB**: Static files served initially
- **Fresh Content**: Automatic background regeneration
- **Scalable**: Combines static performance with dynamic updates
- **SEO Friendly**: Always serves fully rendered HTML
- **Flexible**: Time-based or on-demand revalidation

**Cons:**
- **Complexity**: More complex to implement and debug
- **Stale Content Window**: Brief period with old content during regeneration
- **Resource Usage**: Background regeneration consumes resources
- **Cache Invalidation**: Need to manage when to revalidate

### When to Use ISR

```javascript
// E-commerce product pages
function ProductPage({ product }) {
    // Products update frequently but not constantly
    return <ProductDetails product={product} />;
}

// News articles
function NewsArticle({ article }) {
    // Breaking news needs to be fresh
    return <ArticleContent article={article} />;
}

// User-generated content
function ForumPost({ post }) {
    // Comments and reactions change over time
    return <PostContent post={post} />;
}

// Analytics dashboards
function AnalyticsPage({ data }) {
    // Data updates every few minutes
    return <Charts data={data} />;
}
```

## Advanced ISR Patterns

### Time-Based Revalidation

```javascript
// Revalidate every 5 minutes
export const revalidate = 300;

export default async function Page() {
    const data = await fetchData();
    return <div>{data}</div>;
}
```

### On-Demand Revalidation

```javascript
// app/api/revalidate/route.js
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(request) {
    const { path, tag } = await request.json();

    if (path) {
        revalidatePath(path);
    }

    if (tag) {
        revalidateTag(tag);
    }

    return NextResponse.json({ success: true });
}

// Usage in Server Action
'use server';

export async function updatePost(id, data) {
    await savePost(id, data);
    revalidatePath('/blog');
    revalidatePath(`/blog/${id}`);
}
```

### Tag-Based Revalidation

```javascript
// app/blog/page.js
import { unstable_cache } from 'next/cache';

const getPosts = unstable_cache(
    async () => {
        return await fetchPosts();
    },
    ['posts'],
    { revalidate: 3600, tags: ['posts'] }
);

export default async function BlogPage() {
    const posts = await getPosts();
    return <PostList posts={posts} />;
}

// Revalidate by tag
revalidateTag('posts');
```

## Performance Comparison

| Metric | SSR | SSG | ISR |
|--------|-----|-----|-----|
| **TTFB** | Slow | Fastest | Fast |
| **Server Load** | High | Low | Medium |
| **Scalability** | Low | High | High |
| **SEO** | Excellent | Excellent | Excellent |
| **Freshness** | Always fresh | Stale until rebuild | Configurable |
| **Build Time** | N/A | Slow for large sites | Fast |
| **Storage** | N/A | High for large sites | High |

## Choosing the Right Strategy

### Decision Tree

```javascript
function chooseRenderingStrategy(contentType, updateFrequency, traffic) {
    if (contentType === 'static' && updateFrequency === 'never') {
        return 'SSG';
    }

    if (contentType === 'dynamic' && updateFrequency === 'realtime') {
        return 'SSR';
    }

    if (traffic === 'high' && updateFrequency === 'periodic') {
        return 'ISR';
    }

    // Default recommendations
    if (updateFrequency === 'rare') return 'SSG';
    if (updateFrequency === 'frequent') return 'ISR';
    if (contentType === 'personalized') return 'SSR';

    return 'ISR'; // Safe default
}
```

### Content Type Guidelines

```javascript
// Static Content - SSG
const MarketingPages = () => 'SSG - rarely changes';
const Documentation = () => 'SSG - versioned releases';
const CompanyAbout = () => 'SSG - stable content';

// Dynamic Content - SSR
const UserProfiles = () => 'SSR - personalized';
const ShoppingCart = () => 'SSR - user-specific';
const SearchResults = () => 'SSR - query-dependent';

// Hybrid Content - ISR
const BlogPosts = () => 'ISR - updated periodically';
const ProductCatalog = () => 'ISR - inventory changes';
const NewsArticles = () => 'ISR - breaking news';
```

## Implementation Examples

### E-commerce Product Page

```javascript
// ISR for product pages - fast loading, fresh inventory
export const revalidate = 300; // 5 minutes

export async function generateStaticParams() {
    const products = await getAllProductIds();
    return products.map(id => ({ id: id.toString() }));
}

export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);
    const inventory = await getInventory(params.id);

    return (
        <div>
            <ProductInfo product={product} />
            <InventoryStatus inventory={inventory} />
        </div>
    );
}
```

### Blog with Comments

```javascript
// ISR for blog posts, SSR for comments
export const revalidate = 3600; // 1 hour for posts

export default async function BlogPost({ params }) {
    const post = await getPost(params.slug);

    return (
        <div>
            <PostContent post={post} />
            <Comments postId={post.id} /> {/* Client component for real-time comments */}
        </div>
    );
}
```

### Dashboard with Real-time Data

```javascript
// SSR for personalized dashboard
export default async function Dashboard({ params }) {
    const user = await getCurrentUser();
    const stats = await getUserStats(user.id);

    return (
        <div>
            <UserStats stats={stats} />
            <RealTimeUpdates userId={user.id} /> {/* WebSocket client component */}
        </div>
    );
}
```

## Caching Strategies

### HTTP Caching Headers

```javascript
// app/api/data/route.js
export async function GET() {
    const data = await fetchData();

    return NextResponse.json(data, {
        headers: {
            'Cache-Control': 'public, s-maxage=300, stale-while-revalidate=86400'
        }
    });
}
```

### CDN Configuration

```javascript
// next.config.js
module.exports = {
    async headers() {
        return [
            {
                source: '/api/:path*',
                headers: [
                    {
                        key: 'Cache-Control',
                        value: 'public, s-maxage=300, stale-while-revalidate=86400'
                    }
                ]
            }
        ];
    }
};
```

## Monitoring and Debugging

### Performance Monitoring

```javascript
// Measure rendering performance
export default async function Page() {
    const startTime = Date.now();

    const data = await fetchData();
    const renderTime = Date.now() - startTime;

    console.log(`Page rendered in ${renderTime}ms`);

    return <div>Data: {data}</div>;
}
```

### Cache Hit Monitoring

```javascript
// Track cache performance
const getCachedData = unstable_cache(
    async (key) => {
        console.log(`Cache miss for ${key}`);
        return await fetchData(key);
    },
    ['data'],
    {
        revalidate: 300,
        tags: ['data']
    }
);
```

## Common Pitfalls

### 1. **Over-using SSR**

```javascript
// ❌ Bad: SSR for static content
export async function getServerSideProps() {
    // This page never changes!
    return { props: { content: 'Static content' } };
}

// ✅ Good: SSG for static content
export async function getStaticProps() {
    return { props: { content: 'Static content' } };
}
```

### 2. **Ignoring Revalidation**

```javascript
// ❌ Bad: No revalidation strategy
export const revalidate = false; // Never updates!

// ✅ Good: Appropriate revalidation
export const revalidate = 3600; // Updates every hour
```

### 3. **Large Static Builds**

```javascript
// ❌ Bad: Generating thousands of pages
export async function generateStaticParams() {
    const allProducts = await getAllProducts(); // 100,000 products
    return allProducts.map(product => ({ id: product.id }));
}

// ✅ Good: Generate popular pages only
export async function generateStaticParams() {
    const popularProducts = await getPopularProducts(); // Top 1000
    return popularProducts.map(product => ({ id: product.id }));
}
```

## Migration Strategies

### From SSR to ISR

```javascript
// Before: SSR
export async function getServerSideProps() {
    const data = await fetchData();
    return { props: { data } };
}

// After: ISR
export const revalidate = 60;

export default async function Page() {
    const data = await fetchData();
    return <div>{data}</div>;
}
```

### From SSG to ISR

```javascript
// Before: SSG with manual redeploy
export async function getStaticProps() {
    const data = await fetchData();
    return { props: { data } };
}

// After: ISR with automatic updates
export const revalidate = 3600;

export default async function Page() {
    const data = await fetchData();
    return <div>{data}</div>;
}
```

### Interview Tip:
*"SSR renders pages on each request for fresh data but slower performance, SSG pre-generates pages at build time for fastest delivery but stale content, ISR combines both by regenerating pages in the background at specified intervals or on-demand for optimal performance and freshness."*