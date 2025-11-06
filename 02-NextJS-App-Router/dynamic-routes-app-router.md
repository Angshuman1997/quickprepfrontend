# How do dynamic routes work in App Router?

## Question
How do dynamic routes work in App Router?

## Answer

Dynamic routes in Next.js App Router allow you to create pages that match multiple URL patterns using dynamic segments. Unlike Pages Router which uses file-based dynamic routes, App Router uses folder-based routing with special naming conventions.

## Basic Dynamic Routes

### Single Dynamic Segment

```javascript
// app/blog/[slug]/page.js
export default function BlogPost({ params }) {
    return (
        <div>
            <h1>Blog Post: {params.slug}</h1>
            <p>Reading post with slug: {params.slug}</p>
        </div>
    );
}
```

**URL Patterns:**
- `/blog/hello-world` → `params.slug = "hello-world"`
- `/blog/nextjs-guide` → `params.slug = "nextjs-guide"`

### Multiple Dynamic Segments

```javascript
// app/shop/[category]/[product]/page.js
export default function ProductPage({ params }) {
    const { category, product } = params;

    return (
        <div>
            <h1>{product} in {category}</h1>
            <p>Category: {category}</p>
            <p>Product: {product}</p>
        </div>
    );
}
```

**URL Patterns:**
- `/shop/electronics/phone` → `params = { category: "electronics", product: "phone" }`
- `/shop/clothing/shirt` → `params = { category: "clothing", product: "shirt" }`

## Catch-All Routes

### Basic Catch-All

```javascript
// app/docs/[...slug]/page.js
export default function DocsPage({ params }) {
    const { slug } = params; // slug is an array

    return (
        <div>
            <h1>Docs</h1>
            <p>Path: {slug.join(' / ')}</p>
            <p>Full path: {slug.join('/')}</p>
        </div>
    );
}
```

**URL Patterns:**
- `/docs` → `params.slug = []`
- `/docs/getting-started` → `params.slug = ["getting-started"]`
- `/docs/api/reference` → `params.slug = ["api", "reference"]`
- `/docs/guides/advanced/usage` → `params.slug = ["guides", "advanced", "usage"]`

### Optional Catch-All

```javascript
// app/blog/[[...slug]]/page.js
export default function BlogPage({ params }) {
    const { slug } = params; // slug can be undefined

    if (!slug) {
        return <div>All blog posts</div>;
    }

    return (
        <div>
            <h1>Blog</h1>
            <p>Path: {slug.join(' / ')}</p>
        </div>
    );
}
```

**URL Patterns:**
- `/blog` → `params.slug = undefined`
- `/blog/javascript` → `params.slug = ["javascript"]`
- `/blog/react/hooks` → `params.slug = ["react", "hooks"]`

## Advanced Dynamic Routing Patterns

### Parallel Routes

```javascript
// app/dashboard/@sidebar/page.js
export default function Sidebar() {
    return (
        <div>
            <h2>Dashboard Sidebar</h2>
            <nav>
                <a href="/dashboard">Overview</a>
                <a href="/dashboard/analytics">Analytics</a>
            </nav>
        </div>
    );
}

// app/dashboard/@content/page.js
export default function Content() {
    return (
        <div>
            <h1>Dashboard Content</h1>
            <p>Main dashboard content</p>
        </div>
    );
}

// app/dashboard/layout.js
export default function DashboardLayout({ sidebar, content }) {
    return (
        <div style={{ display: 'flex' }}>
            <aside>{sidebar}</aside>
            <main>{content}</main>
        </div>
    );
}
```

### Intercepting Routes

```javascript
// app/photos/[id]/page.js
export default function PhotoPage({ params }) {
    return (
        <div>
            <h1>Photo {params.id}</h1>
            <img src={`/photos/${params.id}.jpg`} alt={`Photo ${params.id}`} />
        </div>
    );
}

// app/photos/(..)photo/[id]/page.js (intercepting route)
export default function InterceptedPhotoPage({ params }) {
    return (
        <div>
            <h1>Intercepted Photo {params.id}</h1>
            <img src={`/photos/${params.id}.jpg`} alt={`Photo ${params.id}`} />
            <button>Close Modal</button>
        </div>
    );
}
```

## Data Fetching with Dynamic Routes

### Static Generation

```javascript
// app/blog/[slug]/page.js
import { notFound } from 'next/navigation';

async function getPost(slug) {
    const res = await fetch(`https://api.example.com/posts/${slug}`);
    if (!res.ok) return null;
    return res.json();
}

export async function generateStaticParams() {
    const posts = await fetch('https://api.example.com/posts').then(res => res.json());

    return posts.map(post => ({
        slug: post.slug
    }));
}

export default async function BlogPost({ params }) {
    const post = await getPost(params.slug);

    if (!post) {
        notFound();
    }

    return (
        <article>
            <h1>{post.title}</h1>
            <div>{post.content}</div>
        </article>
    );
}
```

### Dynamic Rendering

```javascript
// app/products/[id]/page.js
async function getProduct(id) {
    const res = await fetch(`https://api.example.com/products/${id}`, {
        cache: 'no-store' // Always fetch fresh data
    });

    if (!res.ok) {
        throw new Error('Failed to fetch product');
    }

    return res.json();
}

export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);

    return (
        <div>
            <h1>{product.name}</h1>
            <p>{product.description}</p>
            <p>${product.price}</p>
        </div>
    );
}
```

## Route Groups and Organization

### Route Groups

```javascript
// app/(auth)/login/page.js
export default function LoginPage() {
    return <div>Login Form</div>;
}

// app/(auth)/register/page.js
export default function RegisterPage() {
    return <div>Register Form</div>;
}

// app/(dashboard)/analytics/page.js
export default function AnalyticsPage() {
    return <div>Analytics Dashboard</div>;
}

// app/(dashboard)/users/page.js
export default function UsersPage() {
    return <div>Users Management</div>;
}
```

**Benefits:**
- Organize routes without affecting URL structure
- Apply different layouts to route groups
- Keep related routes together

### Layouts with Route Groups

```javascript
// app/(auth)/layout.js
export default function AuthLayout({ children }) {
    return (
        <div>
            <h1>Authentication</h1>
            {children}
        </div>
    );
}

// app/(dashboard)/layout.js
export default function DashboardLayout({ children }) {
    return (
        <div>
            <nav>Dashboard Navigation</nav>
            {children}
        </div>
    );
}
```

## Advanced Patterns

### Dynamic Metadata

```javascript
// app/blog/[slug]/page.js
import { Metadata } from 'next';

export async function generateMetadata({ params }) {
    const post = await getPost(params.slug);

    return {
        title: post.title,
        description: post.description,
        openGraph: {
            title: post.title,
            description: post.description,
            images: [post.image]
        }
    };
}
```

### Dynamic Redirects

```javascript
// app/redirects/[slug]/page.js
import { redirect } from 'next/navigation';

const redirects = {
    'old-page': '/new-page',
    'blog-post-1': '/blog/modern-react-patterns',
    'docs-v1': '/docs/v2/getting-started'
};

export default function RedirectPage({ params }) {
    const destination = redirects[params.slug];

    if (destination) {
        redirect(destination);
    }

    return <div>Redirect not found</div>;
}
```

### API Routes with Dynamic Segments

```javascript
// app/api/posts/[id]/route.js
import { NextResponse } from 'next/server';

export async function GET(request, { params }) {
    const post = await getPost(params.id);

    if (!post) {
        return NextResponse.json(
            { error: 'Post not found' },
            { status: 404 }
        );
    }

    return NextResponse.json({ post });
}

export async function PUT(request, { params }) {
    const body = await request.json();
    const updatedPost = await updatePost(params.id, body);

    return NextResponse.json({ post: updatedPost });
}

export async function DELETE(request, { params }) {
    await deletePost(params.id);
    return NextResponse.json({ success: true });
}
```

## Error Handling

### 404 Handling

```javascript
// app/blog/[slug]/page.js
import { notFound } from 'next/navigation';

export default async function BlogPost({ params }) {
    const post = await getPost(params.slug);

    if (!post) {
        notFound(); // Shows app/not-found.js or default 404
    }

    return <PostContent post={post} />;
}
```

### Custom Not Found Pages

```javascript
// app/blog/not-found.js
export default function BlogNotFound() {
    return (
        <div>
            <h1>Blog Post Not Found</h1>
            <p>The blog post you're looking for doesn't exist.</p>
            <a href="/blog">Back to Blog</a>
        </div>
    );
}
```

## Performance Optimization

### Static Generation for Dynamic Routes

```javascript
// app/products/[id]/page.js
export async function generateStaticParams() {
    // Generate static pages for popular products
    const popularProducts = await getPopularProducts();

    return popularProducts.map(product => ({
        id: product.id.toString()
    }));
}

export const revalidate = 3600; // ISR: revalidate every hour

export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);
    // ... render product
}
```

### On-Demand Revalidation

```javascript
// app/api/revalidate/route.js
import { revalidatePath } from 'next/server';

export async function POST(request) {
    const { path } = await request.json();

    if (!path) {
        return NextResponse.json(
            { error: 'Path is required' },
            { status: 400 }
        );
    }

    revalidatePath(path);
    return NextResponse.json({ success: true });
}
```

## Real-World Examples

### E-commerce Product Pages

```javascript
// app/products/[category]/[slug]/page.js
async function getProduct(category, slug) {
    const res = await fetch(
        `https://api.example.com/products/${category}/${slug}`
    );
    return res.json();
}

export async function generateStaticParams() {
    const products = await fetch('https://api.example.com/products').then(res => res.json());

    return products.map(product => ({
        category: product.category,
        slug: product.slug
    }));
}

export default async function ProductPage({ params }) {
    const product = await getProduct(params.category, params.slug);

    return (
        <div>
            <Breadcrumb category={params.category} />
            <ProductDetails product={product} />
            <RelatedProducts category={params.category} />
        </div>
    );
}
```

### Multi-tenant Application

```javascript
// app/[tenant]/dashboard/page.js
async function getTenantData(tenant) {
    // Validate tenant exists
    const tenantData = await getTenantBySlug(tenant);
    if (!tenantData) {
        notFound();
    }

    return tenantData;
}

export default async function TenantDashboard({ params }) {
    const tenant = await getTenantData(params.tenant);

    return (
        <div>
            <h1>{tenant.name} Dashboard</h1>
            <TenantStats tenantId={tenant.id} />
            <TenantUsers tenantId={tenant.id} />
        </div>
    );
}
```

### Documentation Site

```javascript
// app/docs/[[...slug]]/page.js
async function getDocContent(slug) {
    if (!slug || slug.length === 0) {
        return getIndexContent();
    }

    const path = slug.join('/');
    return await getDocByPath(path);
}

export async function generateStaticParams() {
    const docs = await getAllDocs();

    const paths = docs.map(doc => ({
        slug: doc.path.split('/')
    }));

    // Add root docs page
    paths.push({ slug: [] });

    return paths;
}

export default async function DocsPage({ params }) {
    const content = await getDocContent(params.slug);

    return (
        <div>
            <DocsSidebar />
            <DocsContent content={content} />
        </div>
    );
}
```

## Best Practices

### 1. **Consistent Naming**

```javascript
// ✅ Good: Use descriptive names
app/blog/[slug]/page.js
app/users/[id]/page.js
app/products/[category]/[product]/page.js

// ❌ Bad: Generic names
app/[param]/page.js
app/[id]/page.js
```

### 2. **Type Safety**

```javascript
// Use TypeScript for better parameter handling
interface PageProps {
    params: {
        slug: string;
        id: string;
    };
}

export default function BlogPost({ params }: PageProps) {
    // params is now typed
}
```

### 3. **Error Boundaries**

```javascript
// app/blog/[slug]/error.js
'use client';

export default function BlogError({ error, reset }) {
    return (
        <div>
            <h2>Failed to load blog post</h2>
            <p>{error.message}</p>
            <button onClick={reset}>Try again</button>
        </div>
    );
}
```

### 4. **Loading States**

```javascript
// app/blog/[slug]/loading.js
export default function BlogLoading() {
    return (
        <div>
            <div>Loading blog post...</div>
            <div>Skeleton content here</div>
        </div>
    );
}
```

### 5. **SEO Optimization**

```javascript
// app/blog/[slug]/page.js
export async function generateMetadata({ params }) {
    const post = await getPost(params.slug);

    return {
        title: post.title,
        description: post.excerpt,
        keywords: post.tags,
        authors: [{ name: post.author }],
        openGraph: {
            title: post.title,
            description: post.excerpt,
            images: [post.featuredImage]
        }
    };
}
```

## Common Patterns and Anti-patterns

### ✅ Good Patterns

1. **Static generation for known routes**
2. **Dynamic rendering for user-specific content**
3. **Proper error handling with `notFound()`**
4. **Type-safe parameter handling**
5. **SEO-friendly metadata generation**

### ❌ Anti-patterns

1. **Overusing catch-all routes**
2. **Not handling 404 cases**
3. **Complex nested dynamic segments**
4. **Missing loading states**
5. **Inefficient data fetching**

## Migration from Pages Router

### Pages Router Dynamic Routes

```javascript
// pages/blog/[slug].js
export default function BlogPost({ post }) {
    return <div>{post.title}</div>;
}

export async function getStaticProps({ params }) {
    const post = await getPost(params.slug);
    return { props: { post } };
}
```

### App Router Equivalent

```javascript
// app/blog/[slug]/page.js
async function getPost(slug) {
    return await fetch(`/api/posts/${slug}`).then(res => res.json());
}

export default async function BlogPost({ params }) {
    const post = await getPost(params.slug);
    return <div>{post.title}</div>;
}
```

### Interview Tip:
*"Dynamic routes in App Router use folder-based naming with square brackets like [slug] for single segments and [...slug] for catch-all routes. They support static generation with generateStaticParams(), parallel routes with @folder syntax, and advanced patterns like intercepting routes."*