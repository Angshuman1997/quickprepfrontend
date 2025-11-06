# Purpose of page.js

## Question
Purpose of page.js

## Answer

The `page.js` file in Next.js App Router defines the content and behavior for a specific route segment. It serves as the entry point for rendering UI components, handling data fetching, and managing route-specific logic. Each `page.js` file corresponds to a unique URL path in your application.

## Basic Page Structure

### Creating a Simple Page

```javascript
// app/page.js - Home page (/)
export default function HomePage() {
    return (
        <div>
            <h1>Welcome to My App</h1>
            <p>This is the home page.</p>
        </div>
    );
}

// app/about/page.js - About page (/about)
export default function AboutPage() {
    return (
        <div>
            <h1>About Us</h1>
            <p>Learn more about our company.</p>
        </div>
    );
}

// app/dashboard/page.js - Dashboard page (/dashboard)
export default function DashboardPage() {
    return (
        <div>
            <h1>Dashboard</h1>
            <DashboardContent />
        </div>
    );
}
```

**File Location to URL Mapping:**
- `app/page.js` → `/`
- `app/about/page.js` → `/about`
- `app/dashboard/page.js` → `/dashboard`
- `app/blog/post/page.js` → `/blog/post`

## Page Component Types

### Server Components (Default)

```javascript
// app/users/page.js
import { getUsers } from '../../lib/api';

export default async function UsersPage() {
    // Server-side data fetching
    const users = await getUsers();

    return (
        <div>
            <h1>Users</h1>
            <ul>
                {users.map(user => (
                    <li key={user.id}>{user.name}</li>
                ))}
            </ul>
        </div>
    );
}
```

**Server Component Benefits:**
- Direct database access
- No client-side JavaScript bundle increase
- Automatic SEO optimization
- Faster initial page loads

### Client Components

```javascript
// app/interactive/page.js
'use client';

import { useState } from 'react';

export default function InteractivePage() {
    const [count, setCount] = useState(0);

    return (
        <div>
            <h1>Interactive Page</h1>
            <button onClick={() => setCount(count + 1)}>
                Count: {count}
            </button>
        </div>
    );
}
```

**Client Component Benefits:**
- Interactive features
- Browser APIs access
- State management
- Event handlers

## Data Fetching in Pages

### Server-Side Data Fetching

```javascript
// app/products/page.js
import { getProducts, getCategories } from '../../lib/api';

export default async function ProductsPage() {
    // Parallel data fetching
    const [products, categories] = await Promise.all([
        getProducts(),
        getCategories()
    ]);

    return (
        <div>
            <h1>Products</h1>
            <CategoryFilter categories={categories} />
            <ProductGrid products={products} />
        </div>
    );
}
```

### Streaming and Suspense

```javascript
// app/dashboard/page.js
import { Suspense } from 'react';

function AnalyticsChart() {
    return (
        <Suspense fallback={<ChartSkeleton />}>
            <AnalyticsChartContent />
        </Suspense>
    );
}

function RecentActivity() {
    return (
        <Suspense fallback={<ActivitySkeleton />}>
            <RecentActivityContent />
        </Suspense>
    );
}

export default function DashboardPage() {
    return (
        <div>
            <h1>Dashboard</h1>
            <StatsOverview /> {/* Fast */}
            <AnalyticsChart /> {/* Slow */}
            <RecentActivity /> {/* Slow */}
        </div>
    );
}
```

## Dynamic Routes

### Dynamic Segments

```javascript
// app/blog/[slug]/page.js
import { getPost } from '../../../lib/api';

export default async function BlogPostPage({ params }) {
    const post = await getPost(params.slug);

    if (!post) {
        return <div>Post not found</div>;
    }

    return (
        <article>
            <h1>{post.title}</h1>
            <p>{post.content}</p>
        </article>
    );
}

// app/users/[id]/page.js
import { getUser } from '../../../lib/api';

export default async function UserPage({ params }) {
    const user = await getUser(params.id);

    return (
        <div>
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}
```

### Catch-All Routes

```javascript
// app/docs/[...slug]/page.js
import { getDocContent } from '../../../lib/api';

export default async function DocsPage({ params }) {
    const { slug } = params;
    const path = slug.join('/'); // ['getting-started', 'installation'] → 'getting-started/installation'

    const content = await getDocContent(path);

    return (
        <div>
            <DocNavigation />
            <DocContent content={content} />
        </div>
    );
}
```

### Optional Catch-All Routes

```javascript
// app/shop/[[...categories]]/page.js
import { getProducts } from '../../../lib/api';

export default async function ShopPage({ params }) {
    const categories = params.categories || [];
    const products = await getProducts({ categories });

    return (
        <div>
            <ShopFilters />
            <ProductGrid products={products} />
        </div>
    );
}
```

## Page Metadata

### Static Metadata

```javascript
// app/about/page.js
import type { Metadata } from 'next';

export const metadata: Metadata = {
    title: 'About Us',
    description: 'Learn about our company and mission',
    keywords: ['about', 'company', 'mission'],
};

export default function AboutPage() {
    return (
        <div>
            <h1>About Us</h1>
            <p>Our company story...</p>
        </div>
    );
}
```

### Dynamic Metadata

```javascript
// app/blog/[slug]/page.js
import type { Metadata } from 'next';
import { getPost } from '../../../lib/api';

export async function generateMetadata({ params }): Promise<Metadata> {
    const post = await getPost(params.slug);

    return {
        title: post.title,
        description: post.excerpt,
        openGraph: {
            title: post.title,
            description: post.excerpt,
            images: [post.image],
        },
    };
}

export default async function BlogPostPage({ params }) {
    const post = await getPost(params.slug);

    return (
        <article>
            <h1>{post.title}</h1>
            <div dangerouslySetInnerHTML={{ __html: post.content }} />
        </article>
    );
}
```

## Page Generation Strategies

### Static Generation (Default)

```javascript
// app/blog/page.js
import { getPosts } from '../../lib/api';

export default async function BlogPage() {
    const posts = await getPosts();

    return (
        <div>
            <h1>Blog</h1>
            {posts.map(post => (
                <article key={post.id}>
                    <h2>{post.title}</h2>
                    <p>{post.excerpt}</p>
                </article>
            ))}
        </div>
    );
}
```

### Dynamic Rendering

```javascript
// app/dashboard/page.js
import { getDashboardData } from '../../lib/api';

export default async function DashboardPage() {
    const data = await getDashboardData();

    return (
        <div>
            <h1>Dashboard</h1>
            <Stats data={data.stats} />
        </div>
    );
}

// Force dynamic rendering
export const dynamic = 'force-dynamic';
```

## Error Handling in Pages

### Error Boundaries

```javascript
// app/products/page.js
import { getProducts } from '../../lib/api';

export default async function ProductsPage() {
    try {
        const products = await getProducts();

        return (
            <div>
                <h1>Products</h1>
                <ProductGrid products={products} />
            </div>
        );
    } catch (error) {
        // This will be caught by error.js
        throw new Error('Failed to load products');
    }
}
```

### Client-Side Error Handling

```javascript
// app/contact/page.js
'use client';

import { useState } from 'react';

export default function ContactPage() {
    const [error, setError] = useState(null);

    const handleSubmit = async (formData) => {
        try {
            await submitContactForm(formData);
            // Success handling
        } catch (error) {
            setError(error.message);
        }
    };

    return (
        <div>
            <h1>Contact Us</h1>
            {error && <div className="error">{error}</div>}
            <ContactForm onSubmit={handleSubmit} />
        </div>
    );
}
```

## Page Loading States

### Loading UI

```javascript
// app/products/loading.js
export default function ProductsLoading() {
    return (
        <div>
            <h1>Products</h1>
            <div className="loading-grid">
                {Array.from({ length: 6 }).map((_, i) => (
                    <div key={i} className="product-skeleton"></div>
                ))}
            </div>
        </div>
    );
}

// app/products/page.js
export default async function ProductsPage() {
    const products = await getProducts(); // Shows loading.js during fetch

    return (
        <div>
            <h1>Products</h1>
            <ProductGrid products={products} />
        </div>
    );
}
```

## Advanced Page Patterns

### 1. **Hybrid Pages (Server + Client)**

```javascript
// app/dashboard/page.js
import { getDashboardData } from '../../lib/api';
import ClientDashboard from './ClientDashboard';

export default async function DashboardPage() {
    // Server-side data fetching
    const initialData = await getDashboardData();

    return (
        <div>
            <h1>Dashboard</h1>
            {/* Pass data to client component */}
            <ClientDashboard initialData={initialData} />
        </div>
    );
}

// app/dashboard/ClientDashboard.js
'use client';

import { useState } from 'react';

export default function ClientDashboard({ initialData }) {
    const [data, setData] = useState(initialData);

    const refreshData = async () => {
        const newData = await fetch('/api/dashboard').then(r => r.json());
        setData(newData);
    };

    return (
        <div>
            <button onClick={refreshData}>Refresh</button>
            <Stats data={data.stats} />
        </div>
    );
}
```

### 2. **Conditional Rendering**

```javascript
// app/admin/page.js
import { getCurrentUser } from '../../lib/auth';

export default async function AdminPage() {
    const user = await getCurrentUser();

    if (!user || user.role !== 'admin') {
        return (
            <div>
                <h1>Access Denied</h1>
                <p>You don't have permission to access this page.</p>
            </div>
        );
    }

    return (
        <div>
            <h1>Admin Dashboard</h1>
            <AdminControls />
        </div>
    );
}
```

### 3. **Page with Multiple Components**

```javascript
// app/blog/page.js
import { getPosts, getCategories } from '../../lib/api';
import BlogHeader from './BlogHeader';
import CategoryFilter from './CategoryFilter';
import PostList from './PostList';
import BlogSidebar from './BlogSidebar';

export default async function BlogPage() {
    const [posts, categories] = await Promise.all([
        getPosts(),
        getCategories()
    ]);

    return (
        <div className="blog-page">
            <BlogHeader />
            <div className="blog-content">
                <main>
                    <CategoryFilter categories={categories} />
                    <PostList posts={posts} />
                </main>
                <BlogSidebar categories={categories} />
            </div>
        </div>
    );
}
```

## Page Performance Optimization

### Code Splitting

```javascript
// Automatic code splitting by route
// Each page.js creates its own bundle

// app/products/page.js - Separate bundle
export default function ProductsPage() {
    return <ProductComponents />; // Only loaded on /products
}

// app/blog/page.js - Separate bundle
export default function BlogPage() {
    return <BlogComponents />; // Only loaded on /blog
}
```

### Image Optimization

```javascript
// app/gallery/page.js
import Image from 'next/image';

export default function GalleryPage() {
    return (
        <div>
            <h1>Gallery</h1>
            <div className="gallery-grid">
                {images.map(image => (
                    <Image
                        key={image.id}
                        src={image.url}
                        alt={image.alt}
                        width={300}
                        height={200}
                        priority={image.priority}
                    />
                ))}
            </div>
        </div>
    );
}
```

## Testing Pages

### Unit Testing

```javascript
// __tests__/page.test.js
import { render, screen } from '@testing-library/react';
import HomePage from '../app/page';

test('renders home page content', () => {
    render(<HomePage />);
    expect(screen.getByText('Welcome to My App')).toBeInTheDocument();
});
```

### Integration Testing

```javascript
// Test with data fetching
test('renders products page with data', async () => {
    // Mock API
    global.fetch = jest.fn(() =>
        Promise.resolve({
            json: () => Promise.resolve(mockProducts)
        })
    );

    render(<ProductsPage />);

    await waitFor(() => {
        expect(screen.getByText('Product 1')).toBeInTheDocument();
    });
});
```

## Common Page Patterns

### 1. **List Pages**

```javascript
// app/products/page.js
import { getProducts } from '../../lib/api';

export default async function ProductsPage({ searchParams }) {
    const { page = 1, category } = searchParams;
    const products = await getProducts({ page, category });

    return (
        <div>
            <h1>Products</h1>
            <ProductFilters />
            <ProductGrid products={products.data} />
            <Pagination
                currentPage={page}
                totalPages={products.totalPages}
            />
        </div>
    );
}
```

### 2. **Detail Pages**

```javascript
// app/products/[id]/page.js
import { getProduct } from '../../../lib/api';
import { notFound } from 'next/navigation';

export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);

    if (!product) {
        notFound();
    }

    return (
        <div>
            <ProductHeader product={product} />
            <ProductImages product={product} />
            <ProductDetails product={product} />
            <RelatedProducts product={product} />
        </div>
    );
}
```

### 3. **Form Pages**

```javascript
// app/contact/page.js
import ContactForm from './ContactForm';

export default function ContactPage() {
    return (
        <div>
            <h1>Contact Us</h1>
            <ContactForm />
        </div>
    );
}

// app/contact/ContactForm.js
'use client';

export default function ContactForm() {
    const handleSubmit = async (formData) => {
        'use server';
        await submitContactForm(formData);
        redirect('/thank-you');
    };

    return (
        <form action={handleSubmit}>
            <input name="name" required />
            <input name="email" type="email" required />
            <textarea name="message" required />
            <button type="submit">Send</button>
        </form>
    );
}
```

## Best Practices

### 1. **Keep Pages Focused**

```javascript
// ✅ Good: Single responsibility
export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);

    return (
        <div>
            <ProductHeader product={product} />
            <ProductDetails product={product} />
        </div>
    );
}

// ❌ Bad: Too many responsibilities
export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);
    const reviews = await getProductReviews(params.id);
    const related = await getRelatedProducts(params.id);
    const user = await getCurrentUser();

    // Complex logic mixed with rendering
    return (
        <div>
            {/* Lots of conditional rendering */}
            {/* Complex state management */}
            {/* Multiple API calls */}
        </div>
    );
}
```

### 2. **Use Appropriate Component Types**

```javascript
// ✅ Good: Server component for data fetching
export default async function BlogPage() {
    const posts = await getPosts(); // Server-side

    return (
        <div>
            <PostList posts={posts} />
        </div>
    );
}

// ✅ Good: Client component for interactivity
function PostList({ posts }) {
    return (
        <ul>
            {posts.map(post => (
                <PostItem key={post.id} post={post} />
            ))}
        </ul>
    );
}
```

### 3. **Handle Loading and Error States**

```javascript
// ✅ Good: Proper loading and error handling
export default async function DataPage() {
    try {
        const data = await getData();

        return (
            <div>
                <DataContent data={data} />
            </div>
        );
    } catch (error) {
        throw new Error('Failed to load data');
    }
}

// loading.js handles loading state
// error.js handles error state
```

### Interview Tip:
*"`page.js` in Next.js App Router defines the actual content for each route, serving as the entry point for UI rendering and data fetching. Each page corresponds to a URL path and can be either a Server Component (for data fetching and SEO) or Client Component (for interactivity), with automatic code splitting for optimal performance."*