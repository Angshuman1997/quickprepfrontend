# What are Server Components vs Client Components?

## Question
What are Server Components vs Client Components?

## Answer

Server Components and Client Components are two fundamental component types in Next.js App Router. Server Components run on the server and are the default, while Client Components run in the browser. Understanding when and how to use each is crucial for building performant React applications.

## Server Components (Default)

### What are Server Components?

```javascript
// Server Component (default - no 'use client')
export default async function BlogPost({ params }) {
    // Runs on server
    const post = await fetchPost(params.id);

    return (
        <article>
            <h1>{post.title}</h1>
            <p>{post.content}</p>
        </article>
    );
}
```

**Key Characteristics:**
- Run on the server during build time or request time
- No JavaScript bundle sent to browser
- Direct access to server resources (database, file system, APIs)
- Automatic code splitting and optimization
- Cannot use browser APIs or state

## Client Components

### What are Client Components?

```javascript
// Client Component (requires 'use client')
'use client';

import { useState } from 'react';

export function InteractiveButton() {
    const [count, setCount] = useState(0);

    return (
        <button onClick={() => setCount(count + 1)}>
            Clicked {count} times
        </button>
    );
}
```

**Key Characteristics:**
- Run in the browser
- Include JavaScript in client bundle
- Can use browser APIs, state, and event handlers
- Hydrated after server rendering

## When to Use Server Components

### ✅ **Use Server Components for:**

#### 1. **Data Fetching**
```javascript
// app/posts/page.js
export default async function PostsPage() {
    // Direct database access
    const posts = await prisma.post.findMany({
        include: { author: true }
    });

    return (
        <div>
            {posts.map(post => (
                <PostCard key={post.id} post={post} />
            ))}
        </div>
    );
}
```

#### 2. **Static Content**
```javascript
// app/about/page.js
export default function AboutPage() {
    return (
        <div>
            <h1>About Our Company</h1>
            <p>We are a leading technology company...</p>
            <TeamSection />
            <ContactInfo />
        </div>
    );
}
```

#### 3. **SEO-Critical Content**
```javascript
// app/blog/[slug]/page.js
export default async function BlogPost({ params }) {
    const post = await getPost(params.slug);

    return (
        <article>
            <h1>{post.title}</h1>
            <meta name="description" content={post.excerpt} />
            <div dangerouslySetInnerHTML={{ __html: post.content }} />
        </article>
    );
}
```

#### 4. **Heavy Computation**
```javascript
// app/analytics/page.js
export default async function AnalyticsPage() {
    // Process large datasets on server
    const analytics = await processAnalyticsData();

    return <AnalyticsDashboard data={analytics} />;
}
```

#### 5. **Accessing Server Resources**
```javascript
// app/admin/page.js
export default async function AdminPage() {
    // Access environment variables, internal APIs
    const config = await getServerConfig();
    const stats = await getInternalStats();

    return <AdminDashboard config={config} stats={stats} />;
}
```

## When to Use Client Components

### ✅ **Use Client Components for:**

#### 1. **Interactivity**
```javascript
// components/SearchBar.js
'use client';

import { useState } from 'react';

export function SearchBar() {
    const [query, setQuery] = useState('');

    return (
        <input
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            placeholder="Search..."
        />
    );
}
```

#### 2. **Browser API Usage**
```javascript
// components/ThemeToggle.js
'use client';

import { useState, useEffect } from 'react';

export function ThemeToggle() {
    const [theme, setTheme] = useState('light');

    useEffect(() => {
        const saved = localStorage.getItem('theme');
        if (saved) setTheme(saved);
    }, []);

    const toggleTheme = () => {
        const newTheme = theme === 'light' ? 'dark' : 'light';
        setTheme(newTheme);
        localStorage.setItem('theme', newTheme);
        document.documentElement.className = newTheme;
    };

    return (
        <button onClick={toggleTheme}>
            Switch to {theme === 'light' ? 'dark' : 'light'} mode
        </button>
    );
}
```

#### 3. **Event Handlers**
```javascript
// components/Accordion.js
'use client';

import { useState } from 'react';

export function Accordion({ items }) {
    const [openIndex, setOpenIndex] = useState(null);

    return (
        <div>
            {items.map((item, index) => (
                <div key={index}>
                    <button onClick={() => setOpenIndex(openIndex === index ? null : index)}>
                        {item.title}
                    </button>
                    {openIndex === index && <div>{item.content}</div>}
                </div>
            ))}
        </div>
    );
}
```

#### 4. **Real-time Features**
```javascript
// components/ChatWidget.js
'use client';

import { useEffect, useState } from 'react';

export function ChatWidget() {
    const [messages, setMessages] = useState([]);
    const [socket, setSocket] = useState(null);

    useEffect(() => {
        const ws = new WebSocket('ws://localhost:8080');
        setSocket(ws);

        ws.onmessage = (event) => {
            setMessages(prev => [...prev, JSON.parse(event.data)]);
        };

        return () => ws.close();
    }, []);

    return (
        <div className="chat">
            {messages.map((msg, i) => <div key={i}>{msg.text}</div>)}
        </div>
    );
}
```

#### 5. **Third-party Libraries**
```javascript
// components/Charts.js
'use client';

import { useEffect, useRef } from 'react';
import Chart from 'chart.js';

export function BarChart({ data }) {
    const canvasRef = useRef(null);

    useEffect(() => {
        const chart = new Chart(canvasRef.current, {
            type: 'bar',
            data: data
        });

        return () => chart.destroy();
    }, [data]);

    return <canvas ref={canvasRef} />;
}
```

## Component Composition Patterns

### Server Component with Client Components

```javascript
// app/dashboard/page.js (Server Component)
import { DashboardHeader } from './DashboardHeader';
import { StatsCards } from './StatsCards';
import { RecentActivity } from './RecentActivity';

export default async function DashboardPage() {
    // Fetch data on server
    const stats = await getDashboardStats();
    const activity = await getRecentActivity();

    return (
        <div>
            <DashboardHeader /> {/* Can be Server or Client */}
            <StatsCards stats={stats} /> {/* Can be Server or Client */}
            <RecentActivity activity={activity} /> {/* Can be Server or Client */}
        </div>
    );
}

// components/DashboardHeader.js (Client Component)
'use client';

export function DashboardHeader() {
    const [notifications, setNotifications] = useState([]);

    return (
        <header>
            <h1>Dashboard</h1>
            <NotificationBell notifications={notifications} />
        </header>
    );
}
```

### Passing Data Between Components

```javascript
// Server Component fetching data
export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);
    const reviews = await getProductReviews(params.id);

    return (
        <div>
            <ProductDetails product={product} />
            <ProductReviews reviews={reviews} />
        </div>
    );
}

// Client Component for interactivity
'use client';

export function ProductReviews({ reviews: initialReviews }) {
    const [reviews, setReviews] = useState(initialReviews);
    const [newReview, setNewReview] = useState('');

    const addReview = async () => {
        const review = await submitReview(newReview);
        setReviews(prev => [review, ...prev]);
        setNewReview('');
    };

    return (
        <div>
            <textarea
                value={newReview}
                onChange={(e) => setNewReview(e.target.value)}
            />
            <button onClick={addReview}>Add Review</button>
            {reviews.map(review => <div key={review.id}>{review.text}</div>)}
        </div>
    );
}
```

## Performance Implications

### Bundle Size Impact

```javascript
// Server Components don't add to client bundle
export default async function StaticPage() {
    const data = await getStaticData();

    return (
        <div>
            {/* No JavaScript sent to browser */}
            <h1>{data.title}</h1>
            <p>{data.content}</p>
        </div>
    );
}

// Client Components add to bundle
'use client';

export function InteractiveComponent() {
    const [state, setState] = useState(0);

    return (
        <button onClick={() => setState(state + 1)}>
            Count: {state}
        </button>
    );
    // This code is sent to browser
}
```

### Hydration Considerations

```javascript
// Minimize client components for better performance
// Wrap only interactive parts in client components

// ✅ Good: Minimal client component
export default function Article({ article }) {
    return (
        <article>
            <h1>{article.title}</h1>
            <div>{article.content}</div>
            <LikeButton articleId={article.id} /> {/* Only this is client */}
        </article>
    );
}

// ❌ Bad: Entire component as client
'use client';

export default function Article({ article }) {
    return (
        <article>
            <h1>{article.title}</h1> {/* Static content in bundle */}
            <div>{article.content}</div> {/* Static content in bundle */}
            <LikeButton articleId={article.id} />
        </article>
    );
}
```

## Migration from Pages Router

### Converting Class Components

```javascript
// Pages Router (all client)
class MyComponent extends React.Component {
    render() {
        return <div>Hello</div>;
    }
}

// App Router - becomes Server Component by default
export default function MyComponent() {
    return <div>Hello</div>;
}

// Add 'use client' only if needed
'use client';

export default function MyComponent() {
    const [state, setState] = useState(0);
    return <div>Hello</div>;
}
```

### Converting getServerSideProps

```javascript
// Pages Router
export async function getServerSideProps() {
    const data = await fetchData();
    return { props: { data } };
}

export default function Page({ data }) {
    return <div>{data}</div>;
}

// App Router - data fetching in component
export default async function Page() {
    const data = await fetchData();
    return <div>{data}</div>;
}
```

## Common Patterns and Anti-patterns

### ✅ **Good Patterns**

#### 1. **Server Components for Data Fetching**
```javascript
// Fetch in server component, pass to client component
export default async function PostsPage() {
    const posts = await getPosts(); // Server

    return <PostsList posts={posts} />; // Client component handles UI
}

'use client';
function PostsList({ posts }) {
    const [filtered, setFiltered] = useState(posts);
    // Handle filtering logic
}
```

#### 2. **Colocation of Concerns**
```javascript
// Keep related logic together
export default async function UserProfile({ params }) {
    const user = await getUser(params.id);

    return (
        <div>
            <UserInfo user={user} />
            <UserPosts userId={user.id} />
        </div>
    );
}
```

#### 3. **Progressive Enhancement**
```javascript
// Server component provides base functionality
export default async function CommentSection({ postId }) {
    const comments = await getComments(postId);

    return (
        <div>
            <CommentsList comments={comments} />
            <AddCommentForm postId={postId} /> {/* Can work without JS */}
        </div>
    );
}
```

### ❌ **Anti-patterns**

#### 1. **Unnecessary Client Components**
```javascript
// ❌ Bad: Client component for static content
'use client';

export function StaticHeader() {
    return <header><h1>My App</h1></header>;
}

// ✅ Good: Server component for static content
export function StaticHeader() {
    return <header><h1>My App</h1></header>;
}
```

#### 2. **Client Components for Data Fetching**
```javascript
// ❌ Bad: Fetching in client component
'use client';

export function PostsList() {
    const [posts, setPosts] = useState([]);

    useEffect(() => {
        fetch('/api/posts').then(r => r.json()).then(setPosts);
    }, []);

    return <div>{/* render posts */}</div>;
}

// ✅ Good: Fetch in server component
export default async function PostsPage() {
    const posts = await getPosts();
    return <PostsList posts={posts} />;
}
```

#### 3. **Large Client Component Trees**
```javascript
// ❌ Bad: Entire page as client component
'use client';

export default function LargePage() {
    return (
        <div>
            <Header /> {/* Static */}
            <Navigation /> {/* Static */}
            <Content /> {/* Static */}
            <InteractiveWidget /> {/* Interactive */}
            <Footer /> {/* Static */}
        </div>
    );
}

// ✅ Good: Mix server and client appropriately
export default function LargePage() {
    return (
        <div>
            <Header />
            <Navigation />
            <Content />
            <InteractiveWidget /> {/* Only this needs 'use client' */}
            <Footer />
        </div>
    );
}
```

## Debugging Component Types

### Identifying Component Types

```javascript
// Check if component is server or client
function MyComponent() {
    console.log(typeof window); // 'undefined' = server, 'object' = client

    return <div>Component</div>;
}

// Use React DevTools to see component boundaries
// Server components show as "Server Component"
// Client components show as regular React components
```

### Common Issues

#### 1. **Window Undefined Error**
```javascript
// ❌ Error: Server component trying to use browser API
export default function ServerComponent() {
    const theme = localStorage.getItem('theme'); // localStorage undefined on server
    return <div>Theme: {theme}</div>;
}

// ✅ Fix: Move to client component
'use client';

export default function ClientComponent() {
    const [theme, setTheme] = useState('light');

    useEffect(() => {
        const saved = localStorage.getItem('theme');
        if (saved) setTheme(saved);
    }, []);

    return <div>Theme: {theme}</div>;
}
```

#### 2. **State Not Persisting**
```javascript
// ❌ State lost on navigation (server component)
export default function CounterPage() {
    const [count, setCount] = useState(0); // State resets on navigation

    return (
        <button onClick={() => setCount(count + 1)}>
            Count: {count}
        </button>
    );
}

// ✅ Fix: Use client component
'use client';

export default function CounterPage() {
    const [count, setCount] = useState(0);

    return (
        <button onClick={() => setCount(count + 1)}>
            Count: {count}
        </button>
    );
}
```

## Testing Considerations

### Testing Server Components

```javascript
// __tests__/ServerComponent.test.js
import { render } from '@testing-library/react';
import ServerComponent from '../ServerComponent';

// Mock server-side functions
jest.mock('../lib/api', () => ({
    getData: () => Promise.resolve({ title: 'Test' })
}));

test('renders server component', async () => {
    const component = await ServerComponent();
    const { getByText } = render(component);

    expect(getByText('Test')).toBeInTheDocument();
});
```

### Testing Client Components

```javascript
// __tests__/ClientComponent.test.js
import { render, fireEvent } from '@testing-library/react';
import ClientComponent from '../ClientComponent';

test('handles user interaction', () => {
    const { getByText } = render(<ClientComponent />);

    fireEvent.click(getByText('Click me'));
    expect(getByText('Clicked')).toBeInTheDocument();
});
```

## Best Practices

### 1. **Default to Server Components**
```javascript
// Start with server components
export default function MyComponent() {
    // No 'use client' needed for static/interactive content
    return <div>Content</div>;
}

// Add 'use client' only when necessary
'use client';

export function InteractiveComponent() {
    const [state, setState] = useState(0);
    // Only add when browser APIs or state are needed
}
```

### 2. **Minimize Client Component Scope**
```javascript
// ✅ Good: Small client component
export function LikeButton({ postId }) {
    'use client';
    // Only the interactive part
}

// ❌ Bad: Large client component
export function EntirePost({ post }) {
    'use client';
    // Everything including static content
}
```

### 3. **Use Composition Effectively**
```javascript
// Server component handles data
export default async function Page() {
    const data = await fetchData();
    return <ClientWrapper data={data} />;
}

// Client component handles interaction
'use client';

function ClientWrapper({ data }) {
    const [state, setState] = useState(data);
    // Handle client-side logic
}
```

### 4. **Consider Performance Impact**
```javascript
// Profile bundle size impact
// Use code splitting for large client components
import dynamic from 'next/dynamic';

const HeavyClientComponent = dynamic(() => import('./HeavyComponent'), {
    loading: () => <div>Loading...</div>
});
```

### Interview Tip:
*"Server Components run on the server and are the default in Next.js App Router, enabling direct data fetching, better performance, and SEO without client JavaScript. Client Components run in the browser with 'use client' directive, enabling interactivity, state management, and browser APIs. Use Server Components for data fetching and static content, Client Components only for interactivity."*