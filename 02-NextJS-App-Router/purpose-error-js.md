# Purpose of error.js

## Question
Purpose of error.js

## Answer

The `error.js` file in Next.js App Router serves as an error boundary for handling runtime errors in specific route segments. It provides a way to gracefully handle and display errors that occur during rendering, data fetching, or other operations within a route, preventing the entire application from crashing.

## Basic Error Boundary

### Creating an error.js File

```javascript
// app/dashboard/error.js
'use client'; // Error boundaries must be client components

export default function Error({ error, reset }) {
    return (
        <div>
            <h2>Something went wrong!</h2>
            <p>{error.message}</p>
            <button onClick={reset}>Try again</button>
        </div>
    );
}
```

**File Location:**
- `app/error.js` - Catches errors for the entire application
- `app/dashboard/error.js` - Catches errors within the dashboard route
- `app/blog/[slug]/error.js` - Catches errors for specific blog posts

## How Error Boundaries Work

### Error Isolation

```javascript
// app/dashboard/page.js
export default async function Dashboard() {
    const data = await fetchDashboardData();

    if (!data) {
        throw new Error('Failed to load dashboard data');
    }

    return <DashboardContent data={data} />;
}

// app/dashboard/error.js
'use client';

export default function DashboardError({ error, reset }) {
    return (
        <div className="error-container">
            <h2>Dashboard Error</h2>
            <p>We couldn't load your dashboard.</p>
            <p>Error: {error.message}</p>
            <button onClick={reset}>Retry</button>
        </div>
    );
}
```

**Error Containment:**
- Errors in `/dashboard/*` routes are caught by `app/dashboard/error.js`
- Errors in other routes remain unaffected
- Parent error boundaries don't catch child route errors

### Automatic Error Catching

```javascript
// Errors caught automatically:
export default async function ProblematicPage() {
    // 1. Data fetching errors
    const data = await fetchData(); // Network error

    // 2. Rendering errors
    return <div>{data.nonexistent.property}</div>; // TypeError

    // 3. Component errors
    return <FaultyComponent />; // Component throws error
}
```

## Error Boundary Lifecycle

### Error Props

```javascript
'use client';

export default function Error({
    error,      // The error that was thrown
    reset       // Function to reset the error boundary
}) {
    // Log error for monitoring
    console.error('Error caught by boundary:', error);

    return (
        <div>
            <h2>Error Occurred</h2>
            <p>{error.message}</p>
            <p>Stack: {error.stack}</p>
            <button onClick={reset}>
                Try Again
            </button>
        </div>
    );
}
```

### Reset Functionality

```javascript
'use client';

export default function Error({ error, reset }) {
    const [retryCount, setRetryCount] = useState(0);

    const handleReset = () => {
        setRetryCount(c => c + 1);
        reset(); // Attempts to re-render the failed component
    };

    return (
        <div>
            <p>Attempt {retryCount + 1} failed</p>
            <button onClick={handleReset}>Retry</button>
        </div>
    );
}
```

## Advanced Error Handling Patterns

### 1. **Error Logging and Monitoring**

```javascript
// lib/errorLogger.js
export function logError(error, context) {
    // Send to error monitoring service
    if (typeof window !== 'undefined' && window.errorLogger) {
        window.errorLogger.captureException(error, {
            tags: { context },
            extra: { url: window.location.href }
        });
    }

    // Log to console in development
    if (process.env.NODE_ENV === 'development') {
        console.error('Error logged:', error, context);
    }
}

// app/error.js
'use client';

import { logError } from '../lib/errorLogger';

export default function GlobalError({ error, reset }) {
    // Log the error
    logError(error, 'global-error-boundary');

    return (
        <div>
            <h1>Something went wrong</h1>
            <p>We've been notified and are working on it.</p>
            <button onClick={reset}>Try again</button>
        </div>
    );
}
```

### 2. **Error Recovery Strategies**

```javascript
// app/dashboard/error.js
'use client';

import { useEffect } from 'react';

export default function DashboardError({ error, reset }) {
    useEffect(() => {
        // Attempt automatic recovery for network errors
        if (error.message.includes('network') || error.message.includes('fetch')) {
            const timer = setTimeout(() => {
                reset();
            }, 5000); // Auto-retry after 5 seconds

            return () => clearTimeout(timer);
        }
    }, [error, reset]);

    const handleManualReset = () => {
        // Clear any cached data that might be causing issues
        localStorage.removeItem('dashboard-cache');
        reset();
    };

    return (
        <div>
            <h2>Dashboard temporarily unavailable</h2>
            <p>We'll automatically retry, or you can try manually:</p>
            <button onClick={handleManualReset}>Retry Now</button>
        </div>
    );
}
```

### 3. **Context-Aware Error Messages**

```javascript
// app/blog/[slug]/error.js
'use client';

export default function BlogPostError({ error, reset }) {
    // Different error messages based on error type
    const getErrorMessage = (error) => {
        if (error.message.includes('404') || error.message.includes('not found')) {
            return 'This blog post could not be found.';
        }

        if (error.message.includes('network')) {
            return 'Unable to load the blog post. Please check your connection.';
        }

        if (error.message.includes('unauthorized')) {
            return 'You don\'t have permission to view this post.';
        }

        return 'An error occurred while loading this blog post.';
    };

    return (
        <div>
            <h2>Blog Post Error</h2>
            <p>{getErrorMessage(error)}</p>
            <button onClick={reset}>Try again</button>
        </div>
    );
}
```

### 4. **Fallback UI Components**

```javascript
// components/ErrorFallback.js
'use client';

export function InlineError({ error, onRetry }) {
    return (
        <div className="inline-error">
            <span>⚠️ {error.message}</span>
            {onRetry && <button onClick={onRetry}>Retry</button>}
        </div>
    );
}

export function PageError({ error, onRetry }) {
    return (
        <div className="page-error">
            <h1>Oops!</h1>
            <p>{error.message}</p>
            <button onClick={onRetry}>Try Again</button>
        </div>
    );
}

export function SectionError({ error, onRetry, children }) {
    return (
        <div className="section-error">
            <p>Section failed to load: {error.message}</p>
            <button onClick={onRetry}>Reload Section</button>
            {children && <div className="fallback-content">{children}</div>}
        </div>
    );
}
```

## Error Boundary Hierarchy

### Nested Error Boundaries

```javascript
// app/layout.js - Root error boundary
export default function RootLayout({ children }) {
    return (
        <html>
            <body>
                <GlobalErrorBoundary>
                    {children}
                </GlobalErrorBoundary>
            </body>
        </html>
    );
}

// app/dashboard/layout.js - Dashboard-specific boundary
export default function DashboardLayout({ children }) {
    return (
        <DashboardErrorBoundary>
            <DashboardSidebar />
            <main>{children}</main>
        </DashboardErrorBoundary>
    );
}

// app/dashboard/analytics/error.js - Specific page boundary
'use client';

export default function AnalyticsError({ error, reset }) {
    return (
        <div>
            <h2>Analytics Error</h2>
            <p>Unable to load analytics data.</p>
            <button onClick={reset}>Retry</button>
        </div>
    );
}
```

**Error Boundary Priority:**
1. Closest `error.js` catches the error first
2. Bubbles up to parent error boundaries if not handled
3. Falls back to root error boundary
4. Finally falls back to Next.js default error page

## Integration with Data Fetching

### Handling Fetch Errors

```javascript
// app/users/page.js
async function getUsers() {
    const res = await fetch('/api/users');

    if (!res.ok) {
        throw new Error(`Failed to fetch users: ${res.status}`);
    }

    return res.json();
}

export default async function UsersPage() {
    const users = await getUsers();

    return (
        <ul>
            {users.map(user => (
                <li key={user.id}>{user.name}</li>
            ))}
        </ul>
    );
}

// app/users/error.js
'use client';

export default function UsersError({ error, reset }) {
    return (
        <div>
            <h2>Unable to load users</h2>
            <p>{error.message}</p>
            <button onClick={reset}>Try again</button>
        </div>
    );
}
```

### Error Boundaries with Loading States

```javascript
// app/products/page.js
import { Suspense } from 'react';

function ProductList() {
    return (
        <Suspense fallback={<div>Loading products...</div>}>
            <ProductListContent />
        </Suspense>
    );
}

async function ProductListContent() {
    const products = await getProducts();
    return <div>{/* render products */}</div>;
}

export default function ProductsPage() {
    return (
        <ErrorBoundary fallback={<div>Failed to load products</div>}>
            <ProductList />
        </ErrorBoundary>
    );
}
```

## Server vs Client Error Handling

### Server Component Errors

```javascript
// Server component errors are caught by error.js
export default async function ServerPage() {
    // These errors are caught by error.js
    const data = await fetchData(); // Network error
    const processed = processData(data); // Processing error

    return <div>{processed}</div>;
}
```

### Client Component Errors

```javascript
// Client component errors also caught by error.js
'use client';

export default function ClientPage() {
    const [data, setData] = useState(null);

    useEffect(() => {
        fetchData().catch(error => {
            // This error would be caught by error.js
            throw error;
        });
    }, []);

    return <div>{data}</div>;
}
```

## Best Practices

### 1. **User-Friendly Error Messages**

```javascript
// ❌ Bad: Technical error messages
export default function Error({ error }) {
    return <div>Error: {error.message}</div>;
}

// ✅ Good: User-friendly messages
export default function Error({ error }) {
    const getUserMessage = (error) => {
        if (error.message.includes('network')) {
            return 'Please check your internet connection and try again.';
        }
        if (error.message.includes('unauthorized')) {
            return 'You need to log in to access this content.';
        }
        return 'Something went wrong. Please try again.';
    };

    return (
        <div>
            <h2>Oops!</h2>
            <p>{getUserMessage(error)}</p>
            <button onClick={reset}>Try Again</button>
        </div>
    );
}
```

### 2. **Error Recovery Options**

```javascript
'use client';

export default function Error({ error, reset }) {
    const [isRetrying, setIsRetrying] = useState(false);

    const handleRetry = async () => {
        setIsRetrying(true);

        try {
            await reset();
        } catch (newError) {
            // Handle retry failure
            console.error('Retry failed:', newError);
        } finally {
            setIsRetrying(false);
        }
    };

    return (
        <div>
            <p>{error.message}</p>
            <button onClick={handleRetry} disabled={isRetrying}>
                {isRetrying ? 'Retrying...' : 'Try Again'}
            </button>
        </div>
    );
}
```

### 3. **Error Logging**

```javascript
'use client';

import { useEffect } from 'react';

export default function Error({ error, reset }) {
    useEffect(() => {
        // Log to error monitoring service
        logErrorToService(error, {
            url: window.location.href,
            userAgent: navigator.userAgent,
            timestamp: new Date().toISOString()
        });
    }, [error]);

    return (
        <div>
            <h2>Something went wrong</h2>
            <p>We've been notified and are working on fixing it.</p>
            <button onClick={reset}>Try Again</button>
        </div>
    );
}
```

### 4. **Graceful Degradation**

```javascript
// app/dashboard/error.js
'use client';

export default function DashboardError({ error, reset }) {
    // Provide fallback content
    const fallbackData = {
        stats: { users: 0, revenue: 0 },
        recentActivity: []
    };

    return (
        <div>
            <h2>Dashboard temporarily unavailable</h2>
            <p>Here's some cached information:</p>

            <DashboardContent data={fallbackData} isFallback />

            <button onClick={reset}>Reload Full Dashboard</button>
        </div>
    );
}
```

## Common Error Scenarios

### 1. **API Failures**

```javascript
// app/api/users/route.js
export async function GET() {
    try {
        const users = await getUsersFromDB();
        return NextResponse.json({ users });
    } catch (error) {
        console.error('API Error:', error);
        return NextResponse.json(
            { error: 'Failed to fetch users' },
            { status: 500 }
        );
    }
}

// Client-side API errors are caught by error.js
async function fetchUsers() {
    const res = await fetch('/api/users');
    if (!res.ok) throw new Error('API request failed');
    return res.json();
}
```

### 2. **Database Connection Issues**

```javascript
// lib/db.js
export async function connectToDB() {
    try {
        return await createConnection();
    } catch (error) {
        throw new Error('Database connection failed');
    }
}

// Errors bubble up to error.js
export default async function DataPage() {
    await connectToDB(); // Throws error if DB down
    const data = await fetchData();
    return <div>{data}</div>;
}
```

### 3. **Third-party Service Failures**

```javascript
// lib/payment.js
export async function processPayment(paymentData) {
    try {
        const result = await stripe.charges.create(paymentData);
        return result;
    } catch (error) {
        throw new Error('Payment processing failed');
    }
}

// Payment errors caught by error.js
export default async function CheckoutPage() {
    const handleSubmit = async (formData) => {
        'use server';
        await processPayment(formData);
        redirect('/success');
    };

    return <CheckoutForm action={handleSubmit} />;
}
```

## Testing Error Boundaries

### Unit Testing

```javascript
// __tests__/error.test.js
import { render, screen, fireEvent } from '@testing-library/react';
import Error from '../app/error';

const mockError = new Error('Test error');
const mockReset = jest.fn();

test('displays error message and reset button', () => {
    render(<Error error={mockError} reset={mockReset} />);

    expect(screen.getByText('Test error')).toBeInTheDocument();
    expect(screen.getByText('Try again')).toBeInTheDocument();
});

test('calls reset when button clicked', () => {
    render(<Error error={mockError} reset={mockReset} />);

    fireEvent.click(screen.getByText('Try again'));
    expect(mockReset).toHaveBeenCalled();
});
```

### Integration Testing

```javascript
// Test error boundary in context
test('error boundary catches component errors', () => {
    const BadComponent = () => {
        throw new Error('Component crashed');
    };

    render(
        <ErrorBoundary>
            <BadComponent />
        </ErrorBoundary>
    );

    expect(screen.getByText('Something went wrong!')).toBeInTheDocument();
});
```

## Performance Considerations

### Error Boundary Impact

```javascript
// Error boundaries add minimal overhead
// Only rendered when errors occur
// No impact on successful renders

// ✅ Good: Lightweight error UI
export default function Error({ error, reset }) {
    return (
        <div style={{ padding: '1rem', color: 'red' }}>
            <p>{error.message}</p>
            <button onClick={reset}>Retry</button>
        </div>
    );
}

// ❌ Bad: Heavy error UI with many dependencies
export default function Error({ error, reset }) {
    return (
        <div>
            <HeavyAnimationComponent />
            <LargeIconLibrary />
            <ComplexErrorReportingForm />
        </div>
    );
}
```

### Interview Tip:
*"`error.js` in Next.js App Router acts as an error boundary that catches runtime errors in route segments, providing graceful error handling and recovery options. It isolates errors to specific route areas, displays user-friendly error messages, and includes a reset function to attempt recovery without full page reloads."*