# Difference between Pages Router and App Router

## Question
Difference between Pages Router and App Router.

## Answer

Next.js has evolved significantly with the introduction of the App Router in version 13. The App Router is a newer, more advanced routing system that replaces the Pages Router. Understanding the differences is crucial for modern Next.js development.

## Core Architecture Differences

### File Structure

**Pages Router:**
```
pages/
├── _app.js
├── _document.js
├── index.js
├── about.js
├── api/
│   ├── users.js
│   └── posts.js
└── blog/
    ├── index.js
    └── [slug].js
```

**App Router:**
```
app/
├── layout.js
├── page.js
├── loading.js
├── error.js
├── not-found.js
├── global-error.js
├── api/
│   └── users/
│       └── route.js
└── blog/
    ├── page.js
    ├── layout.js
    ├── loading.js
    └── [slug]/
        └── page.js
```

### Component Model

**Pages Router:**
- File-based routing with page components
- `_app.js` wraps all pages
- `_document.js` for custom document structure
- Limited nested layouts

**App Router:**
- Route-based components with `page.js`
- `layout.js` for shared UI and nested layouts
- `loading.js` and `error.js` for route-specific states
- Server and Client Components

## Data Fetching

### Pages Router

```javascript
// pages/users.js
export default function UsersPage({ users }) {
    return (
        <div>
            {users.map(user => (
                <div key={user.id}>{user.name}</div>
            ))}
        </div>
    );
}

export async function getServerSideProps() {
    const res = await fetch('https://api.example.com/users');
    const users = await res.json();

    return {
        props: { users }
    };
}

// Static generation
export async function getStaticProps() {
    const res = await fetch('https://api.example.com/posts');
    const posts = await res.json();

    return {
        props: { posts },
        revalidate: 60
    };
}
```

### App Router

```javascript
// app/users/page.js
async function getUsers() {
    const res = await fetch('https://api.example.com/users');
    return res.json();
}

export default async function UsersPage() {
    const users = await getUsers();

    return (
        <div>
            {users.map(user => (
                <div key={user.id}>{user.name}</div>
            ))}
        </div>
    );
}

// Static generation with ISR
export const revalidate = 60;
```

## Server Components vs Client Components

### Pages Router
- All components run on client by default
- Server-side rendering through `getServerSideProps`
- Limited server-side logic

### App Router
- Server Components by default (run on server)
- Client Components with `'use client'` directive
- Rich server-side capabilities

```javascript
// Server Component (default)
export default function ServerComponent() {
    // Runs on server
    // Can fetch data directly
    // No JavaScript bundle impact
    return <div>Server rendered</div>;
}

// Client Component
'use client';

export default function ClientComponent() {
    // Runs on client
    // Can use hooks and browser APIs
    // Adds to JavaScript bundle
    return <div>Client rendered</div>;
}
```

## Layout System

### Pages Router

```javascript
// pages/_app.js
import Layout from '../components/Layout';

export default function App({ Component, pageProps }) {
    return (
        <Layout>
            <Component {...pageProps} />
        </Layout>
    );
}

// components/Layout.js
export default function Layout({ children }) {
    return (
        <div>
            <nav>Navigation</nav>
            <main>{children}</main>
            <footer>Footer</footer>
        </div>
    );
}
```

### App Router

```javascript
// app/layout.js
export default function RootLayout({ children }) {
    return (
        <html lang="en">
            <body>
                <nav>Navigation</nav>
                <main>{children}</main>
                <footer>Footer</footer>
            </body>
        </html>
    );
}

// app/blog/layout.js
export default function BlogLayout({ children }) {
    return (
        <div>
            <h1>My Blog</h1>
            <div>{children}</div>
        </div>
    );
}
```

## API Routes

### Pages Router

```javascript
// pages/api/users.js
export default function handler(req, res) {
    if (req.method === 'GET') {
        // Handle GET request
        res.status(200).json({ users: [] });
    } else if (req.method === 'POST') {
        // Handle POST request
        res.status(201).json({ user: req.body });
    }
}
```

### App Router

```javascript
// app/api/users/route.js
import { NextResponse } from 'next/server';

export async function GET() {
    const users = await getUsersFromDB();
    return NextResponse.json({ users });
}

export async function POST(request) {
    const body = await request.json();
    const user = await createUser(body);
    return NextResponse.json({ user }, { status: 201 });
}
```

## Loading and Error States

### Pages Router
- No built-in loading states
- Custom loading implementations
- Limited error boundaries

### App Router
- `loading.js` for route-specific loading UI
- `error.js` for route-specific error handling
- `not-found.js` for 404 pages

```javascript
// app/blog/loading.js
export default function Loading() {
    return <div>Loading blog posts...</div>;
}

// app/blog/error.js
'use client';

export default function Error({ error, reset }) {
    return (
        <div>
            <h2>Something went wrong!</h2>
            <p>{error.message}</p>
            <button onClick={reset}>Try again</button>
        </div>
    );
}

// app/blog/not-found.js
export default function NotFound() {
    return (
        <div>
            <h1>404 - Blog post not found</h1>
            <p>The blog post you're looking for doesn't exist.</p>
        </div>
    );
}
```

## Middleware

### Pages Router
- No built-in middleware
- Custom server setup required

### App Router
- `middleware.js` in root
- Runs on every request
- Can modify requests/responses

```javascript
// middleware.js
import { NextResponse } from 'next/server';

export function middleware(request) {
    const token = request.cookies.get('auth-token');

    if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
        return NextResponse.redirect(new URL('/login', request.url));
    }

    return NextResponse.next();
}
```

## Caching and Performance

### Pages Router
- Basic caching with `revalidate`
- Limited control over caching strategies

### App Router
- Advanced caching with `unstable_cache`
- Request memoization
- Full control over cache strategies

```javascript
// App Router caching
import { unstable_cache } from 'next/cache';

const getCachedData = unstable_cache(
    async (id) => {
        return await fetchData(id);
    },
    ['data'],
    { revalidate: 3600 }
);
```

## Server Actions

### Pages Router
- No built-in server actions
- API routes for mutations

### App Router
- Server Actions with `'use server'`
- Form handling without API routes
- Progressive enhancement

```javascript
// app/actions.js
'use server';

export async function createPost(formData) {
    const title = formData.get('title');
    const content = formData.get('content');

    await savePostToDB({ title, content });
    revalidatePath('/blog');
}

// app/blog/new/page.js
import { createPost } from '../../actions';

export default function NewPostPage() {
    return (
        <form action={createPost}>
            <input name="title" required />
            <textarea name="content" required />
            <button type="submit">Create Post</button>
        </form>
    );
}
```

## Bundle Splitting and Code Splitting

### Pages Router
- Automatic code splitting per page
- Limited control over chunks

### App Router
- More granular code splitting
- Server Components don't add to bundle
- Better tree shaking

## Migration Considerations

### Gradual Migration
```javascript
// You can use both routers in the same app
// pages/ - Pages Router
// app/ - App Router

// middleware.js can handle routing between them
export function middleware(request) {
    if (request.nextUrl.pathname.startsWith('/old-route')) {
        return NextResponse.rewrite(new URL('/pages/old-route', request.url));
    }
}
```

### Breaking Changes
- Component structure changes
- Data fetching patterns change
- API route structure changes
- Some third-party libraries may need updates

## Performance Comparison

| Feature | Pages Router | App Router |
|---------|-------------|------------|
| First Load | Good | Better (Server Components) |
| Navigation | Good | Better (nested layouts) |
| Bundle Size | Good | Better (Server Components) |
| Developer Experience | Good | Excellent |
| Advanced Features | Limited | Rich |

## When to Use Which

### Use Pages Router when:
- Simple applications
- Existing Pages Router apps (gradual migration)
- Limited team resources for migration
- Third-party dependencies not compatible with App Router

### Use App Router when:
- New applications
- Complex nested layouts
- Advanced caching needs
- Server-side heavy applications
- Better performance requirements

## Real-World Examples

### E-commerce with App Router

```javascript
// app/layout.js
export default function RootLayout({ children }) {
    return (
        <html>
            <body>
                <Header />
                {children}
                <Footer />
            </body>
        </html>
    );
}

// app/products/layout.js
export default function ProductsLayout({ children }) {
    return (
        <div>
            <ProductFilters />
            {children}
        </div>
    );
}

// app/products/page.js
async function getProducts() {
    return await fetch('/api/products').then(res => res.json());
}

export default async function ProductsPage() {
    const products = await getProducts();

    return <ProductGrid products={products} />;
}

// app/products/[id]/page.js
async function getProduct(id) {
    return await fetch(`/api/products/${id}`).then(res => res.json());
}

export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);

    return <ProductDetail product={product} />;
}
```

### Blog with Pages Router

```javascript
// pages/_app.js
function MyApp({ Component, pageProps }) {
    return (
        <div>
            <Header />
            <Component {...pageProps} />
            <Footer />
        </div>
    );
}

// pages/blog/[slug].js
export default function BlogPost({ post }) {
    return (
        <article>
            <h1>{post.title}</h1>
            <div>{post.content}</div>
        </article>
    );
}

export async function getStaticProps({ params }) {
    const post = await getPostBySlug(params.slug);
    return { props: { post } };
}

export async function getStaticPaths() {
    const posts = await getAllPosts();
    return {
        paths: posts.map(post => ({ params: { slug: post.slug } })),
        fallback: false
    };
}
```

## Best Practices

### App Router Best Practices
1. Use Server Components by default
2. Keep client components minimal
3. Use layouts for shared UI
4. Implement proper loading and error states
5. Use Server Actions for mutations
6. Leverage advanced caching

### Pages Router Best Practices
1. Use appropriate data fetching methods
2. Implement proper error handling
3. Use custom `_app.js` and `_document.js`
4. Optimize with `next.config.js`
5. Use API routes effectively

## Future of Next.js

- App Router is the future of Next.js
- Pages Router will be maintained but not receive new features
- New projects should use App Router
- Existing projects should plan migration

### Interview Tip:
*"App Router introduces Server Components, nested layouts, and advanced caching, while Pages Router uses traditional React patterns with getServerSideProps. App Router provides better performance through Server Components and more flexible routing with layouts, loading states, and error boundaries."*