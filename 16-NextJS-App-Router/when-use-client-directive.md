# When must we write "use client"?

## Question
When must we write "use client"?

## Answer

The `'use client'` directive in Next.js App Router marks a component as a Client Component, meaning it runs on the client-side (browser) instead of the server. You must use `'use client'` when your component needs browser-specific features that aren't available during server rendering.

## When You MUST Use 'use client'

### 1. **Using Browser APIs**

```javascript
// ❌ Error: window is undefined on server
export default function Component() {
    const [theme, setTheme] = useState(window.localStorage.getItem('theme'));
    return <div>Theme: {theme}</div>;
}

// ✅ Correct: Use 'use client'
'use client';

export default function Component() {
    const [theme, setTheme] = useState('light');

    useEffect(() => {
        const saved = localStorage.getItem('theme');
        if (saved) setTheme(saved);
    }, []);

    return <div>Theme: {theme}</div>;
}
```

**Browser APIs that require 'use client':**
- `window`, `document`, `navigator`
- `localStorage`, `sessionStorage`
- `location`, `history`
- `fetch` (in client-only scenarios)
- Geolocation, WebRTC, WebSockets

### 2. **Using React State and Lifecycle**

```javascript
// ❌ Error: useState not available on server
export default function Counter() {
    const [count, setCount] = useState(0);
    return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// ✅ Correct: Use 'use client'
'use client';

export default function Counter() {
    const [count, setCount] = useState(0);
    return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**React hooks that require 'use client':**
- `useState`, `useReducer`
- `useEffect`, `useLayoutEffect`
- `useCallback`, `useMemo`
- `useRef`, `useContext`
- `useTransition`, `useDeferredValue`

### 3. **Event Handlers**

```javascript
// ❌ Error: onClick not available on server
export default function Button() {
    return <button onClick={() => alert('Clicked!')}>Click me</button>;
}

// ✅ Correct: Use 'use client'
'use client';

export default function Button() {
    return <button onClick={() => alert('Clicked!')}>Click me</button>;
}
```

**Common event handlers:**
- `onClick`, `onSubmit`, `onChange`
- `onMouseEnter`, `onMouseLeave`
- `onFocus`, `onBlur`
- `onKeyDown`, `onKeyUp`

### 4. **Third-Party Libraries with Browser Dependencies**

```javascript
// ❌ Error: Chart.js requires browser environment
import Chart from 'chart.js';

export default function ChartComponent() {
    useEffect(() => {
        new Chart(canvasRef.current, config);
    }, []);

    return <canvas ref={useRef()} />;
}

// ✅ Correct: Use 'use client'
'use client';

import Chart from 'chart.js';

export default function ChartComponent() {
    const canvasRef = useRef();

    useEffect(() => {
        new Chart(canvasRef.current, config);
    }, []);

    return <canvas ref={canvasRef} />;
}
```

**Libraries requiring 'use client':**
- Chart libraries (Chart.js, D3.js)
- Animation libraries (Framer Motion client-side)
- Date pickers, modals, carousels
- Video/audio players
- Maps (Google Maps, Mapbox)

### 5. **Real-time Features**

```javascript
// ❌ Error: WebSocket connection on server
export default function Chat() {
    const [messages, setMessages] = useState([]);
    const ws = new WebSocket('ws://localhost:8080');

    useEffect(() => {
        ws.onmessage = (event) => {
            setMessages(prev => [...prev, JSON.parse(event.data)]);
        };
    }, []);

    return <div>{/* render messages */}</div>;
}

// ✅ Correct: Use 'use client'
'use client';

export default function Chat() {
    const [messages, setMessages] = useState([]);
    const [ws, setWs] = useState(null);

    useEffect(() => {
        const socket = new WebSocket('ws://localhost:8080');
        setWs(socket);

        socket.onmessage = (event) => {
            setMessages(prev => [...prev, JSON.parse(event.data)]);
        };

        return () => socket.close();
    }, []);

    return <div>{/* render messages */}</div>;
}
```

**Real-time features requiring 'use client':**
- WebSocket connections
- Server-Sent Events (SSE)
- WebRTC (video/audio)
- Live data updates
- Push notifications

### 6. **Custom Hooks with Client Dependencies**

```javascript
// ❌ Error: Custom hook uses browser APIs
function useLocalStorage(key, initialValue) {
    const [value, setValue] = useState(initialValue);

    useEffect(() => {
        const stored = localStorage.getItem(key);
        if (stored) setValue(JSON.parse(stored));
    }, [key]);

    const setStoredValue = (newValue) => {
        setValue(newValue);
        localStorage.setItem(key, JSON.stringify(newValue));
    };

    return [value, setStoredValue];
}

export default function Component() {
    const [name, setName] = useLocalStorage('name', '');
    return <input value={name} onChange={(e) => setName(e.target.value)} />;
}

// ✅ Correct: Use 'use client' in component using the hook
'use client';

export default function Component() {
    const [name, setName] = useLocalStorage('name', '');
    return <input value={name} onChange={(e) => setName(e.target.value)} />;
}
```

## When You DON'T Need 'use client'

### 1. **Server Components (Default)**

```javascript
// ✅ No 'use client' needed - Server Component
export default async function BlogPost({ params }) {
    const post = await getPost(params.id);

    return (
        <article>
            <h1>{post.title}</h1>
            <p>{post.content}</p>
        </article>
    );
}
```

**Server Component features:**
- Data fetching
- Direct database access
- File system operations
- Static content rendering

### 2. **Server Actions**

```javascript
// ✅ No 'use client' needed - Server Action
export async function createPost(formData) {
    'use server';

    const title = formData.get('title');
    const content = formData.get('content');

    await prisma.post.create({
        data: { title, content }
    });

    revalidatePath('/posts');
}
```

### 3. **Static Components**

```javascript
// ✅ No 'use client' needed - Static content
export function Header() {
    return (
        <header>
            <nav>
                <Link href="/">Home</Link>
                <Link href="/about">About</Link>
            </nav>
        </header>
    );
}

export function Footer() {
    return (
        <footer>
            <p>© 2024 My App</p>
        </footer>
    );
}
```

## Component Composition Patterns

### Server Component with Client Child

```javascript
// ✅ Server Component (no 'use client')
export default async function ProductPage({ params }) {
    const product = await getProduct(params.id);

    return (
        <div>
            <ProductInfo product={product} />
            <AddToCart productId={product.id} /> {/* Client Component */}
        </div>
    );
}

// ✅ Client Component (needs 'use client')
'use client';

function AddToCart({ productId }) {
    const [quantity, setQuantity] = useState(1);

    const addToCart = () => {
        // Add to cart logic
    };

    return (
        <div>
            <input
                type="number"
                value={quantity}
                onChange={(e) => setQuantity(e.target.value)}
            />
            <button onClick={addToCart}>Add to Cart</button>
        </div>
    );
}
```

### Client Component with Server Child

```javascript
// ✅ Client Component (needs 'use client')
'use client';

export default function Dashboard() {
    const [activeTab, setActiveTab] = useState('overview');

    return (
        <div>
            <TabNavigation
                activeTab={activeTab}
                onChange={setActiveTab}
            />
            {activeTab === 'overview' && <OverviewTab />}
            {activeTab === 'analytics' && <AnalyticsTab />}
        </div>
    );
}

// ✅ Server Component (no 'use client')
async function AnalyticsTab() {
    const data = await getAnalyticsData();

    return <AnalyticsChart data={data} />;
}
```

## Common Mistakes and Solutions

### 1. **Unnecessary 'use client'**

```javascript
// ❌ Unnecessary: Static component doesn't need client
'use client';

export function StaticCard({ title, content }) {
    return (
        <div>
            <h3>{title}</h3>
            <p>{content}</p>
        </div>
    );
}

// ✅ Correct: Remove 'use client'
export function StaticCard({ title, content }) {
    return (
        <div>
            <h3>{title}</h3>
            <p>{content}</p>
        </div>
    );
}
```

### 2. **Missing 'use client' for Interactivity**

```javascript
// ❌ Missing 'use client': Component won't work
export default function SearchBar() {
    const [query, setQuery] = useState('');

    return (
        <input
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            placeholder="Search..."
        />
    );
}

// ✅ Correct: Add 'use client'
'use client';

export default function SearchBar() {
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

### 3. **'use client' Too High in Tree**

```javascript
// ❌ Bad: Entire page as client component
'use client';

export default function Page() {
    return (
        <div>
            <h1>Page Title</h1> {/* Static content in bundle */}
            <p>Static content</p> {/* Static content in bundle */}
            <InteractiveWidget /> {/* Only this needs client */}
        </div>
    );
}

// ✅ Good: Only interactive parts are client
export default function Page() {
    return (
        <div>
            <h1>Page Title</h1>
            <p>Static content</p>
            <InteractiveWidget /> {/* Only this has 'use client' */}
        </div>
    );
}
```

## Performance Implications

### Bundle Size Impact

```javascript
// Client components increase bundle size
'use client';

export function HeavyInteractiveComponent() {
    // This entire component and its dependencies
    // are sent to the browser
    return <div>Heavy component</div>;
}

// Server components don't affect client bundle
export function LightStaticComponent() {
    // Only HTML is sent to browser
    return <div>Light component</div>;
}
```

### Hydration Considerations

```javascript
// Client components require hydration
'use client';

export function InteractiveForm() {
    const [email, setEmail] = useState('');

    return (
        <form>
            <input
                value={email}
                onChange={(e) => setEmail(e.target.value)}
            />
        </form>
    );
    // This component needs to hydrate on client
}
```

## Migration from Pages Router

### Converting Components

```javascript
// Pages Router - All components are client by default
function MyComponent() {
    const [state, setState] = useState(0);
    return <div>Component</div>;
}

// App Router - Add 'use client' for interactive components
'use client';

function MyComponent() {
    const [state, setState] = useState(0);
    return <div>Component</div>;
}

// App Router - No 'use client' for static components
function StaticComponent() {
    return <div>Static content</div>;
}
```

## Debugging 'use client' Issues

### Common Error Messages

```javascript
// Error: "window is not defined"
export default function Component() {
    console.log(window.location.href); // ❌ window undefined on server
}

// Error: "useState is not defined"
export default function Component() {
    const [state, setState] = useState(0); // ❌ useState undefined on server
}

// Error: "event handlers cannot be used on server"
export default function Component() {
    return <button onClick={() => {}}>Click</button>; // ❌ onClick not allowed on server
}
```

### Debugging Steps

```javascript
// 1. Check if component uses browser APIs
function checkBrowserUsage(component) {
    // Look for: window, document, localStorage, etc.
}

// 2. Check if component uses React hooks
function checkHooksUsage(component) {
    // Look for: useState, useEffect, useCallback, etc.
}

// 3. Check if component has event handlers
function checkEventHandlers(component) {
    // Look for: onClick, onChange, onSubmit, etc.
}

// 4. Check if component uses third-party libraries
function checkLibraries(component) {
    // Look for: Chart.js, date pickers, maps, etc.
}
```

## Best Practices

### 1. **Minimize Client Components**

```javascript
// ✅ Good: Small client component boundary
export default function Page() {
    return (
        <div>
            <StaticHeader />
            <StaticContent />
            <InteractiveForm /> {/* Only this needs 'use client' */}
            <StaticFooter />
        </div>
    );
}

// ❌ Bad: Large client component boundary
'use client';

export default function Page() {
    return (
        <div>
            <StaticHeader /> {/* Unnecessarily in client bundle */}
            <StaticContent /> {/* Unnecessarily in client bundle */}
            <InteractiveForm />
            <StaticFooter /> {/* Unnecessarily in client bundle */}
        </div>
    );
}
```

### 2. **Use Server Components by Default**

```javascript
// ✅ Default to server components
export default async function BlogPage() {
    const posts = await getPosts();

    return (
        <div>
            {posts.map(post => (
                <BlogPost key={post.id} post={post} />
            ))}
        </div>
    );
}

// Add 'use client' only when necessary
'use client';

function BlogPost({ post }) {
    const [expanded, setExpanded] = useState(false);

    return (
        <article>
            <h2>{post.title}</h2>
            {expanded && <p>{post.content}</p>}
            <button onClick={() => setExpanded(!expanded)}>
                {expanded ? 'Collapse' : 'Expand'}
            </button>
        </article>
    );
}
```

### 3. **Colocate Client and Server Logic**

```javascript
// ✅ Good: Keep related logic together
// components/
  // ProductCard.js (server - static content)
  // AddToCart.js (client - interactivity)
  // ProductPage.js (server - data fetching)

// ❌ Bad: Split related logic
// components/
  // ProductCard.js (client - unnecessary)
  // ProductPage.js (server - data fetching)
  // AddToCart.js (client - interactivity)
```

### 4. **Test Both Server and Client**

```javascript
// Test server rendering
test('renders on server', () => {
    render(<ServerComponent />);
    // Test static content
});

// Test client hydration
test('hydrates on client', () => {
    render(<ClientComponent />);
    // Test interactivity
});
```

### Interview Tip:
*"'use client' is required in Next.js App Router when a component needs browser-specific features like React hooks (useState, useEffect), browser APIs (window, localStorage), event handlers (onClick), or third-party libraries with DOM dependencies. Server Components are the default and should be used for static content and data fetching."*