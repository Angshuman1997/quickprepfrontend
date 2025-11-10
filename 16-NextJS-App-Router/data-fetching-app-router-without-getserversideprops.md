# How is data fetched in App Router without getServerSideProps?

## Question
How is data fetched in App Router without getServerSideProps?

## Answer

In Next.js App Router, data fetching is fundamentally different from Pages Router. Instead of `getServerSideProps`, you fetch data directly in Server Components using async/await. This approach is more intuitive and allows for better performance through advanced caching strategies.

## Core Concepts

### Server Components vs Client Components

**Server Components:**
- Run on server during build/request time
- Can fetch data directly
- No JavaScript bundle impact
- Cannot use browser APIs or React hooks

**Client Components:**
- Run in browser
- Use `useEffect` for data fetching
- Can use browser APIs and hooks
- Add to JavaScript bundle

## Method 1: Basic Server Component Data Fetching

```javascript
// app/users/page.js
async function getUsers() {
    const res = await fetch('https://api.example.com/users');
    if (!res.ok) {
        throw new Error('Failed to fetch users');
    }
    return res.json();
}

export default async function UsersPage() {
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

## Method 2: Parallel Data Fetching

```javascript
// app/dashboard/page.js
async function getUserStats() {
    const res = await fetch('https://api.example.com/user/stats');
    return res.json();
}

async function getRecentActivity() {
    const res = await fetch('https://api.example.com/user/activity');
    return res.json();
}

async function getNotifications() {
    const res = await fetch('https://api.example.com/user/notifications');
    return res.json();
}

export default async function DashboardPage() {
    // All requests run in parallel
    const [stats, activity, notifications] = await Promise.all([
        getUserStats(),
        getRecentActivity(),
        getNotifications()
    ]);

    return (
        <div>
            <StatsCard data={stats} />
            <ActivityFeed data={activity} />
            <NotificationList data={notifications} />
        </div>
    );
}
```

## Method 3: Streaming with Loading UI

```javascript
// app/products/page.js
import { Suspense } from 'react';

async function getProducts() {
    // Simulate slow API
    await new Promise(resolve => setTimeout(resolve, 2000));
    const res = await fetch('https://api.example.com/products');
    return res.json();
}

function ProductList() {
    return (
        <Suspense fallback={<div>Loading products...</div>}>
            <ProductListContent />
        </Suspense>
    );
}

async function ProductListContent() {
    const products = await getProducts();

    return (
        <div>
            {products.map(product => (
                <div key={product.id}>
                    <h3>{product.name}</h3>
                    <p>{product.description}</p>
                </div>
            ))}
        </div>
    );
}

export default function ProductsPage() {
    return (
        <div>
            <h1>Products</h1>
            <ProductList />
        </div>
    );
}
```

## Method 4: Error Handling

```javascript
// app/users/[id]/page.js
async function getUser(id) {
    const res = await fetch(`https://api.example.com/users/${id}`);

    if (!res.ok) {
        if (res.status === 404) {
            throw new Error('User not found');
        }
        throw new Error('Failed to fetch user');
    }

    return res.json();
}

export default async function UserPage({ params }) {
    let user;

    try {
        user = await getUser(params.id);
    } catch (error) {
        if (error.message === 'User not found') {
            return <div>User not found</div>;
        }
        return <div>Something went wrong</div>;
    }

    return (
        <div>
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}
```

## Method 5: Caching Strategies

### Request Memoization

```javascript
// lib/data.js
const cache = new Map();

async function fetchWithCache(url, options = {}) {
    const key = JSON.stringify({ url, ...options });

    if (cache.has(key)) {
        return cache.get(key);
    }

    const res = await fetch(url, options);
    const data = await res.json();

    cache.set(key, data);
    return data;
}

export async function getPosts() {
    return fetchWithCache('https://api.example.com/posts');
}

export async function getPost(id) {
    return fetchWithCache(`https://api.example.com/posts/${id}`);
}
```

### Next.js Built-in Caching

```javascript
// app/blog/page.js
import { unstable_cache } from 'next/cache';

const getPosts = unstable_cache(
    async () => {
        const res = await fetch('https://api.example.com/posts');
        return res.json();
    },
    ['posts'],
    { revalidate: 3600 } // Cache for 1 hour
);

export default async function BlogPage() {
    const posts = await getPosts();

    return (
        <div>
            <h1>Blog Posts</h1>
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

## Method 6: Dynamic Data with Search Params

```javascript
// app/search/page.js
async function searchProducts(query, category) {
    const params = new URLSearchParams();
    if (query) params.set('q', query);
    if (category) params.set('category', category);

    const res = await fetch(`https://api.example.com/search?${params}`);
    return res.json();
}

export default async function SearchPage({ searchParams }) {
    const { q: query, category } = searchParams;
    const products = await searchProducts(query, category);

    return (
        <div>
            <h1>Search Results</h1>
            {query && <p>Searching for: {query}</p>}
            {category && <p>Category: {category}</p>}

            <div>
                {products.map(product => (
                    <div key={product.id}>
                        <h3>{product.name}</h3>
                        <p>{product.price}</p>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

## Method 7: Server Actions for Mutations

```javascript
// app/actions.js
'use server';

import { revalidatePath } from 'next/navigation';

export async function createPost(formData) {
    const title = formData.get('title');
    const content = formData.get('content');

    const res = await fetch('https://api.example.com/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title, content })
    });

    if (!res.ok) {
        throw new Error('Failed to create post');
    }

    revalidatePath('/blog'); // Invalidate cached data
    redirect('/blog');
}

export async function updatePost(id, formData) {
    const title = formData.get('title');
    const content = formData.get('content');

    const res = await fetch(`https://api.example.com/posts/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title, content })
    });

    if (!res.ok) {
        throw new Error('Failed to update post');
    }

    revalidatePath(`/blog/${id}`);
    revalidatePath('/blog');
}
```

```javascript
// app/blog/new/page.js
import { createPost } from '../../actions';

export default function NewPostPage() {
    return (
        <div>
            <h1>Create New Post</h1>
            <form action={createPost}>
                <input name="title" placeholder="Title" required />
                <textarea name="content" placeholder="Content" required />
                <button type="submit">Create Post</button>
            </form>
        </div>
    );
}
```

## Method 8: Database Queries in Server Components

```javascript
// lib/db.js
import { sql } from '@vercel/postgres';

export async function getUsers() {
    const { rows } = await sql`SELECT * FROM users ORDER BY created_at DESC`;
    return rows;
}

export async function getUser(id) {
    const { rows } = await sql`SELECT * FROM users WHERE id = ${id}`;
    return rows[0];
}

export async function getPostsByUser(userId) {
    const { rows } = await sql`
        SELECT p.* FROM posts p
        JOIN users u ON p.user_id = u.id
        WHERE u.id = ${userId}
        ORDER BY p.created_at DESC
    `;
    return rows;
}
```

```javascript
// app/users/[id]/page.js
import { getUser, getPostsByUser } from '../../../lib/db';

export default async function UserPage({ params }) {
    const [user, posts] = await Promise.all([
        getUser(params.id),
        getPostsByUser(params.id)
    ]);

    if (!user) {
        return <div>User not found</div>;
    }

    return (
        <div>
            <h1>{user.name}</h1>
            <p>{user.email}</p>

            <h2>Posts</h2>
            <ul>
                {posts.map(post => (
                    <li key={post.id}>
                        <h3>{post.title}</h3>
                        <p>{post.content}</p>
                    </li>
                ))}
            </ul>
        </div>
    );
}
```

## Method 9: Third-party API Integration

```javascript
// lib/github.js
export async function getGitHubUser(username) {
    const res = await fetch(`https://api.github.com/users/${username}`, {
        headers: {
            'Authorization': `Bearer ${process.env.GITHUB_TOKEN}`,
            'Accept': 'application/vnd.github.v3+json'
        }
    });

    if (!res.ok) {
        throw new Error('Failed to fetch GitHub user');
    }

    return res.json();
}

export async function getGitHubRepos(username) {
    const res = await fetch(`https://api.github.com/users/${username}/repos`, {
        headers: {
            'Authorization': `Bearer ${process.env.GITHUB_TOKEN}`,
            'Accept': 'application/vnd.github.v3+json'
        }
    });

    return res.json();
}
```

```javascript
// app/github/[username]/page.js
import { getGitHubUser, getGitHubRepos } from '../../../lib/github';

export default async function GitHubUserPage({ params }) {
    const [user, repos] = await Promise.all([
        getGitHubUser(params.username),
        getGitHubRepos(params.username)
    ]);

    return (
        <div>
            <div>
                <img src={user.avatar_url} alt={user.login} width={100} />
                <h1>{user.name || user.login}</h1>
                <p>{user.bio}</p>
                <p>Followers: {user.followers}</p>
            </div>

            <h2>Repositories</h2>
            <div>
                {repos.slice(0, 6).map(repo => (
                    <div key={repo.id}>
                        <h3>{repo.name}</h3>
                        <p>{repo.description}</p>
                        <p>⭐ {repo.stargazers_count}</p>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

## Method 10: Incremental Static Regeneration (ISR)

```javascript
// app/blog/[slug]/page.js
import { notFound } from 'next/navigation';

async function getPost(slug) {
    const res = await fetch(`https://api.example.com/posts/${slug}`, {
        next: { revalidate: 3600 } // Revalidate every hour
    });

    if (!res.ok) {
        return null;
    }

    return res.json();
}

export async function generateStaticParams() {
    const posts = await fetch('https://api.example.com/posts').then(res => res.json());

    return posts.map(post => ({
        slug: post.slug
    }));
}

export default async function BlogPostPage({ params }) {
    const post = await getPost(params.slug);

    if (!post) {
        notFound();
    }

    return (
        <article>
            <h1>{post.title}</h1>
            <p>{post.publishedAt}</p>
            <div dangerouslySetInnerHTML={{ __html: post.content }} />
        </article>
    );
}
```

## Advanced Patterns

### 1. **Data Fetching with Loading States**

```javascript
// components/DataWrapper.js
'use client';

import { Suspense, use } from 'react';

function DataWrapper({ promise, children, fallback }) {
    return (
        <Suspense fallback={fallback}>
            <DataContent promise={promise}>
                {children}
            </DataContent>
        </Suspense>
    );
}

function DataContent({ promise, children }) {
    const data = use(promise);
    return children(data);
}

export default DataWrapper;
```

```javascript
// app/dashboard/page.js
import DataWrapper from '../../components/DataWrapper';

async function getDashboardData() {
    const [stats, activity] = await Promise.all([
        fetch('https://api.example.com/stats').then(res => res.json()),
        fetch('https://api.example.com/activity').then(res => res.json())
    ]);

    return { stats, activity };
}

export default function DashboardPage() {
    const dataPromise = getDashboardData();

    return (
        <div>
            <DataWrapper
                promise={dataPromise}
                fallback={<div>Loading dashboard...</div>}
            >
                {({ stats, activity }) => (
                    <div>
                        <StatsCard data={stats} />
                        <ActivityFeed data={activity} />
                    </div>
                )}
            </DataWrapper>
        </div>
    );
}
```

### 2. **Optimistic Updates**

```javascript
// app/todos/page.js
import { useOptimistic } from 'react';
import { toggleTodo } from '../actions';

function TodoList({ todos }) {
    const [optimisticTodos, addOptimisticTodo] = useOptimistic(
        todos,
        (state, updatedTodo) => {
            return state.map(todo =>
                todo.id === updatedTodo.id
                    ? { ...todo, completed: updatedTodo.completed }
                    : todo
            );
        }
    );

    async function handleToggle(todo) {
        addOptimisticTodo({ ...todo, completed: !todo.completed });
        await toggleTodo(todo.id);
    }

    return (
        <ul>
            {optimisticTodos.map(todo => (
                <li key={todo.id}>
                    <input
                        type="checkbox"
                        checked={todo.completed}
                        onChange={() => handleToggle(todo)}
                    />
                    {todo.text}
                </li>
            ))}
        </ul>
    );
}
```

### 3. **Error Boundaries for Data Fetching**

```javascript
// components/ErrorBoundary.js
'use client';

import { Component } from 'react';

class ErrorBoundary extends Component {
    constructor(props) {
        super(props);
        this.state = { hasError: false, error: null };
    }

    static getDerivedStateFromError(error) {
        return { hasError: true, error };
    }

    componentDidCatch(error, errorInfo) {
        console.error('Data fetching error:', error, errorInfo);
    }

    render() {
        if (this.state.hasError) {
            return (
                <div>
                    <h2>Something went wrong</h2>
                    <p>{this.state.error?.message}</p>
                    <button onClick={() => this.setState({ hasError: false })}>
                        Try again
                    </button>
                </div>
            );
        }

        return this.props.children;
    }
}

export default ErrorBoundary;
```

```javascript
// app/blog/page.js
import ErrorBoundary from '../../components/ErrorBoundary';

async function getPosts() {
    const res = await fetch('https://api.example.com/posts');
    if (!res.ok) {
        throw new Error('Failed to fetch posts');
    }
    return res.json();
}

export default async function BlogPage() {
    const posts = await getPosts();

    return (
        <ErrorBoundary>
            <div>
                <h1>Blog Posts</h1>
                {posts.map(post => (
                    <article key={post.id}>
                        <h2>{post.title}</h2>
                        <p>{post.excerpt}</p>
                    </article>
                ))}
            </div>
        </ErrorBoundary>
    );
}
```

## Performance Optimization

### 1. **Request Deduplication**

```javascript
// lib/fetch-cache.js
const requestCache = new Map();

export async function cachedFetch(url, options = {}) {
    const key = JSON.stringify({ url, options });

    if (requestCache.has(key)) {
        return requestCache.get(key);
    }

    const promise = fetch(url, options).then(async res => {
        const data = await res.json();
        requestCache.delete(key); // Clean up after request
        return data;
    });

    requestCache.set(key, promise);
    return promise;
}
```

### 2. **Data Prefetching**

```javascript
// app/blog/layout.js
import { prefetchData } from '../../lib/prefetch';

export default async function BlogLayout({ children }) {
    // Prefetch common data
    await prefetchData('/api/categories');
    await prefetchData('/api/tags');

    return (
        <div>
            <BlogSidebar />
            {children}
        </div>
    );
}
```

### 3. **Partial Hydration**

```javascript
// app/products/page.js
import dynamic from 'next/dynamic';

// Only hydrate the interactive parts
const ProductFilters = dynamic(() => import('../../components/ProductFilters'), {
    ssr: false
});

export default async function ProductsPage() {
    const products = await getProducts();

    return (
        <div>
            <ProductFilters />
            <ProductGrid products={products} />
        </div>
    );
}
```

## Best Practices

### 1. **Error Handling**

```javascript
// Always handle errors gracefully
async function safeFetch(url) {
    try {
        const res = await fetch(url);
        if (!res.ok) {
            throw new Error(`HTTP error! status: ${res.status}`);
        }
        return await res.json();
    } catch (error) {
        console.error(`Fetch failed for ${url}:`, error);
        throw error;
    }
}
```

### 2. **Loading States**

```javascript
// Use Suspense for granular loading
function ProductCard({ product }) {
    return (
        <Suspense fallback={<ProductSkeleton />}>
            <ProductCardContent product={product} />
        </Suspense>
    );
}
```

### 3. **Caching Strategy**

```javascript
// Choose appropriate caching based on data type
const getStaticData = unstable_cache(
    () => fetch('/api/static-data'),
    ['static'],
    { revalidate: false } // Never revalidate
);

const getDynamicData = unstable_cache(
    () => fetch('/api/dynamic-data'),
    ['dynamic'],
    { revalidate: 60 } // Revalidate every minute
);
```

### 4. **Type Safety**

```javascript
// Use TypeScript for better data fetching
interface User {
    id: number;
    name: string;
    email: string;
}

async function getUser(id: string): Promise<User> {
    const res = await fetch(`/api/users/${id}`);
    return res.json();
}
```

## Common Patterns Comparison

| Pattern | Pages Router | App Router |
|---------|-------------|------------|
| Basic fetching | `getServerSideProps` | `async` component |
| Static generation | `getStaticProps` | `export const dynamic = 'force-static'` |
| ISR | `revalidate` in `getStaticProps` | `export const revalidate = 60` |
| Client-side fetching | `useEffect` | `useEffect` (same) |
| API routes | `pages/api/` | `app/api/` |

## Real-World Example: E-commerce Product Page

```javascript
// app/products/[id]/page.js
import { notFound } from 'next/navigation';
import { Metadata } from 'next';

async function getProduct(id) {
    const res = await fetch(`https://api.example.com/products/${id}`, {
        next: { revalidate: 3600 } // ISR
    });

    if (!res.ok) {
        if (res.status === 404) {
            notFound();
        }
        throw new Error('Failed to fetch product');
    }

    return res.json();
}

async function getRelatedProducts(category) {
    const res = await fetch(
        `https://api.example.com/products?category=${category}&limit=4`
    );
    return res.json();
}

export async function generateMetadata({ params }) {
    const product = await getProduct(params.id);

    return {
        title: product.name,
        description: product.description,
        openGraph: {
            images: [product.image]
        }
    };
}

export default async function ProductPage({ params }) {
    const [product, relatedProducts] = await Promise.all([
        getProduct(params.id),
        getRelatedProducts(product.category)
    ]);

    return (
        <div>
            <div>
                <img src={product.image} alt={product.name} />
                <h1>{product.name}</h1>
                <p>{product.description}</p>
                <p>${product.price}</p>
                <AddToCartButton product={product} />
            </div>

            <div>
                <h2>Related Products</h2>
                <ProductGrid products={relatedProducts} />
            </div>
        </div>
    );
}
```

### Interview Tip:
*"In Next.js App Router, data fetching happens directly in Server Components using async/await, eliminating the need for getServerSideProps. Use parallel fetching with Promise.all, implement proper caching with unstable_cache, and handle errors gracefully with try/catch blocks."*