# How Next.js handles errors using error.jsx

## Question
How Next.js handles errors using error.jsx

## Answer

In the Next.js App Router, every route folder can have an `error.jsx` file. It works like an Error Boundary for that route only, catching rendering errors and providing fallback UI.

## Basic error.jsx Implementation

```javascript
// app/dashboard/error.jsx
"use client";

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>Something went wrong in Dashboard.</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

**Key Points:**
- Must be a **Client Component** (`"use client"`)
- Receives `error` and `reset` props automatically
- Only catches **rendering errors**, not event handler errors

## How It Works

### Route-Specific Error Handling

```
app/
├── error.jsx                    // Global error boundary
├── dashboard/
│   ├── error.jsx               // Dashboard-specific errors  
│   ├── page.jsx                // If this throws error, dashboard/error.jsx shows
│   └── users/
│       ├── error.jsx           // User-specific errors
│       └── page.jsx            // If this throws, users/error.jsx shows
```

### Error Catching Example

```javascript
// app/dashboard/page.jsx
export default async function Dashboard() {
  // This error will be caught by app/dashboard/error.jsx
  const res = await fetch("https://api.example.com/data");
  
  if (!res.ok) {
    throw new Error("Failed to fetch dashboard data");
  }
  
  const data = await res.json();
  return <div>{data.title}</div>;
}

// app/dashboard/error.jsx
"use client";

export default function DashboardError({ error, reset }) {
  return (
    <div>
      <h2>Dashboard Error</h2>
      <p>Could not load dashboard: {error.message}</p>
      <button onClick={reset}>Retry</button>
    </div>
  );
}
```

## Error Hierarchy

### Nested Error Boundaries

```
app/
├── error.jsx                    // Catches all unhandled errors
├── dashboard/
│   ├── error.jsx               // Overrides global for /dashboard/*
│   ├── page.jsx
│   └── analytics/
│       ├── error.jsx           // Overrides parent for /dashboard/analytics
│       └── page.jsx
```

**Error Resolution:**
1. **Closest error.jsx** catches the error first
2. If no local error.jsx, **inherits from parent**
3. Falls back to **global error.jsx**
4. Finally uses **Next.js default error page**

## Advanced Error Handling

### Custom Error Component

```javascript
// app/dashboard/error.jsx
"use client";

import { useEffect } from 'react';

export default function DashboardError({ error, reset }) {
  useEffect(() => {
    // Log error to monitoring service
    console.error('Dashboard error:', error);
  }, [error]);

  const isNetworkError = error.message.includes('fetch');
  const isAuthError = error.message.includes('unauthorized');

  return (
    <div className="error-container">
      <h2>Dashboard Unavailable</h2>
      
      {isNetworkError && (
        <p>Network issue. Check your connection.</p>
      )}
      
      {isAuthError && (
        <p>Session expired. Please log in again.</p>
      )}
      
      {!isNetworkError && !isAuthError && (
        <p>Unexpected error: {error.message}</p>
      )}
      
      <div className="error-actions">
        <button onClick={reset}>Try Again</button>
        <button onClick={() => window.location.reload()}>
          Reload Page
        </button>
      </div>
    </div>
  );
}
```

### Error Boundary with Recovery

```javascript
// app/reports/error.jsx
"use client";

import { useState } from 'react';

export default function ReportsError({ error, reset }) {
  const [retryCount, setRetryCount] = useState(0);
  const maxRetries = 3;

  const handleRetry = () => {
    if (retryCount < maxRetries) {
      setRetryCount(count => count + 1);
      reset();
    }
  };

  if (retryCount >= maxRetries) {
    return (
      <div>
        <h2>Reports Permanently Failed</h2>
        <p>Too many retry attempts. Please refresh the page.</p>
        <button onClick={() => window.location.reload()}>
          Refresh Page
        </button>
      </div>
    );
  }

  return (
    <div>
      <h2>Reports Error</h2>
      <p>Attempt {retryCount + 1} failed</p>
      <button onClick={handleRetry}>
        Retry ({maxRetries - retryCount} attempts left)
      </button>
    </div>
  );
}
```

## What Errors Are NOT Caught

### Event Handler Errors

```javascript
// app/dashboard/page.jsx
function Dashboard() {
  const handleClick = () => {
    // This error is NOT caught by error.jsx
    throw new Error("Button failed");
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

**Solution: Manual try/catch**

```javascript
function Dashboard() {
  const [error, setError] = useState(null);

  const handleClick = () => {
    try {
      throw new Error("Button failed");
    } catch (err) {
      setError(err.message);
    }
  };

  return (
    <div>
      {error && <div className="error">{error}</div>}
      <button onClick={handleClick}>Click me</button>
    </div>
  );
}
```

### Async Errors in Event Handlers

```javascript
function Dashboard() {
  const [error, setError] = useState(null);

  const handleAsyncAction = async () => {
    try {
      const response = await fetch('/api/action');
      if (!response.ok) throw new Error('Action failed');
      // Success handling
    } catch (err) {
      setError(err.message);
    }
  };

  return (
    <div>
      {error && <div className="error">{error}</div>}
      <button onClick={handleAsyncAction}>Async Action</button>
    </div>
  );
}
```

## Integration with loading.jsx

### Complete Route Error Handling

```
app/dashboard/
├── loading.jsx                 // Shows while page loads
├── error.jsx                   // Shows if loading/rendering fails
├── page.jsx                    // Main page component
└── not-found.jsx              // Shows for 404 errors
```

```javascript
// app/dashboard/loading.jsx
export default function DashboardLoading() {
  return <div>Loading dashboard...</div>;
}

// app/dashboard/error.jsx
"use client";

export default function DashboardError({ error, reset }) {
  return (
    <div>
      <h2>Dashboard Error</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  );
}

// app/dashboard/page.jsx
export default async function Dashboard() {
  // While this loads, loading.jsx is shown
  const data = await fetchDashboardData();
  
  // If this throws error, error.jsx is shown
  if (!data) {
    throw new Error("No dashboard data available");
  }
  
  return <DashboardContent data={data} />;
}
```

## Global vs Route-Specific Errors

### Global Error Boundary

```javascript
// app/error.jsx
"use client";

export default function GlobalError({ error, reset }) {
  return (
    <html>
      <body>
        <div>
          <h1>Application Error</h1>
          <p>Something went wrong with the entire app.</p>
          <button onClick={reset}>Try Again</button>
        </div>
      </body>
    </html>
  );
}
```

### Route-Specific Error

```javascript
// app/profile/error.jsx
"use client";

export default function ProfileError({ error, reset }) {
  return (
    <div>
      <h2>Profile Error</h2>
      <p>Could not load your profile.</p>
      <button onClick={reset}>Retry</button>
      <a href="/dashboard">Go to Dashboard</a>
    </div>
  );
}
```

## Best Practices

### 1. User-Friendly Messages

```javascript
// ❌ Bad: Technical error messages
export default function Error({ error }) {
  return <div>Error: {error.stack}</div>;
}

// ✅ Good: User-friendly messages
export default function Error({ error, reset }) {
  const getUserMessage = (error) => {
    if (error.message.includes('fetch')) return 'Connection problem. Please try again.';
    if (error.message.includes('404')) return 'Page not found.';
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

### 2. Error Logging

```javascript
"use client";

import { useEffect } from 'react';

export default function Error({ error, reset }) {
  useEffect(() => {
    // Log to error monitoring service
    if (process.env.NODE_ENV === 'production') {
      logErrorToService(error);
    }
    console.error('Route error:', error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={reset}>Try Again</button>
    </div>
  );
}
```

### 3. Progressive Fallbacks

```javascript
"use client";

export default function ProductsError({ error, reset }) {
  // Show cached data if available
  const cachedProducts = getCachedProducts();

  return (
    <div>
      <h2>Products temporarily unavailable</h2>
      
      {cachedProducts.length > 0 ? (
        <div>
          <p>Showing cached products:</p>
          <ProductList products={cachedProducts} />
        </div>
      ) : (
        <p>No products available right now.</p>
      )}
      
      <button onClick={reset}>Try Again</button>
    </div>
  );
}
```

## Interview Tips

- **error.jsx only catches rendering errors**: Event handlers need manual try/catch
- **Must be client component**: Always include `"use client"` directive
- **Closest error.jsx wins**: Nested folders override parent error boundaries
- **Reset function**: Allows users to retry without full page reload
- **Works with loading.jsx**: Create complete loading and error states
- **User experience focus**: Show helpful messages, not technical stack traces

## Quick Reference

| File Location | Catches Errors For |
|--------------|-------------------|
| `app/error.jsx` | Entire application |
| `app/dashboard/error.jsx` | `/dashboard/*` routes |
| `app/blog/[slug]/error.jsx` | Individual blog posts |

**Remember**: error.jsx only handles **rendering errors**. For event handlers and async operations, use manual try/catch with state management.