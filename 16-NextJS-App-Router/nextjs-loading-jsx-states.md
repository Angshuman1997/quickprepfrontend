# How Next.js handles loading states using loading.jsx

## Question
How Next.js handles loading states using loading.jsx

## Answer

`loading.jsx` is automatically shown whenever Next.js is waiting for data to finish loading. It provides instant loading UI while async operations complete, improving user experience by showing immediate feedback.

## Basic loading.jsx Implementation

```javascript
// app/dashboard/loading.jsx
export default function Loading() {
  return <div>Loading dashboard...</div>;
}
```

**Then in your page:**

```javascript
// app/dashboard/page.jsx
export default async function Dashboard() {
  // While this fetch happens, loading.jsx is displayed
  const res = await fetch("https://api.example.com/data");
  const data = await res.json();
  
  return <div>{data.title}</div>;
}
```

**How it works:**
- **Automatic**: No manual state management needed
- **Instant**: Shows immediately when navigation starts
- **Route-specific**: Each route can have its own loading UI

## Loading Hierarchy

### Nested Loading States

```
app/
├── loading.jsx                  // Global loading for all routes
├── dashboard/
│   ├── loading.jsx             // Overrides global for /dashboard/*
│   ├── page.jsx
│   └── users/
│       ├── loading.jsx         // Overrides parent for /dashboard/users
│       └── page.jsx
```

**Loading Resolution:**
1. **Closest loading.jsx** is used first
2. If no local loading.jsx, **inherits from parent**
3. Falls back to **global loading.jsx**
4. Finally uses **Next.js default loading**

## Real-World Examples

### Dashboard Loading

```javascript
// app/dashboard/loading.jsx
export default function DashboardLoading() {
  return (
    <div className="dashboard-loading">
      <div className="skeleton-header"></div>
      <div className="skeleton-cards">
        <div className="skeleton-card"></div>
        <div className="skeleton-card"></div>
        <div className="skeleton-card"></div>
      </div>
      <div className="skeleton-chart"></div>
    </div>
  );
}

// app/dashboard/page.jsx
export default async function Dashboard() {
  // Multiple data fetches - loading.jsx shows during all of this
  const [stats, users, revenue] = await Promise.all([
    fetch('/api/stats').then(r => r.json()),
    fetch('/api/users').then(r => r.json()),
    fetch('/api/revenue').then(r => r.json())
  ]);

  return (
    <div className="dashboard">
      <StatsCards data={stats} />
      <UsersList users={users} />
      <RevenueChart data={revenue} />
    </div>
  );
}
```

### Product List Loading

```javascript
// app/products/loading.jsx
export default function ProductsLoading() {
  return (
    <div className="products-loading">
      <h1>Products</h1>
      <div className="product-grid">
        {[...Array(8)].map((_, i) => (
          <div key={i} className="product-skeleton">
            <div className="skeleton-image"></div>
            <div className="skeleton-title"></div>
            <div className="skeleton-price"></div>
          </div>
        ))}
      </div>
    </div>
  );
}

// app/products/page.jsx
export default async function Products() {
  const products = await fetch('/api/products').then(r => r.json());
  
  return (
    <div className="products">
      <h1>Products</h1>
      <div className="product-grid">
        {products.map(product => (
          <ProductCard key={product.id} product={product} />
        ))}
      </div>
    </div>
  );
}
```

## Advanced Loading Patterns

### Skeleton Loading

```javascript
// components/SkeletonLoader.jsx
export function SkeletonCard() {
  return (
    <div className="skeleton-card">
      <div className="skeleton-avatar"></div>
      <div className="skeleton-content">
        <div className="skeleton-line long"></div>
        <div className="skeleton-line medium"></div>
        <div className="skeleton-line short"></div>
      </div>
    </div>
  );
}

// app/users/loading.jsx
import { SkeletonCard } from '@/components/SkeletonLoader';

export default function UsersLoading() {
  return (
    <div className="users-loading">
      <h1>Users</h1>
      <div className="users-grid">
        {[...Array(6)].map((_, i) => (
          <SkeletonCard key={i} />
        ))}
      </div>
    </div>
  );
}
```

### Progressive Loading

```javascript
// app/profile/loading.jsx
export default function ProfileLoading() {
  return (
    <div className="profile-loading">
      {/* Show header immediately */}
      <div className="profile-header">
        <div className="skeleton-avatar large"></div>
        <div className="skeleton-info">
          <div className="skeleton-line long"></div>
          <div className="skeleton-line medium"></div>
        </div>
      </div>
      
      {/* Loading placeholders for content */}
      <div className="profile-content">
        <div className="skeleton-section">
          <div className="skeleton-title"></div>
          <div className="skeleton-paragraph"></div>
        </div>
        
        <div className="skeleton-section">
          <div className="skeleton-title"></div>
          <div className="skeleton-list">
            {[...Array(5)].map((_, i) => (
              <div key={i} className="skeleton-list-item"></div>
            ))}
          </div>
        </div>
      </div>
    </div>
  );
}
```

## Integration with Suspense

### Manual Suspense Boundaries

```javascript
// app/dashboard/page.jsx
import { Suspense } from 'react';

function FastComponent() {
  // Renders immediately
  return <div>Fast content here</div>;
}

async function SlowComponent() {
  // Takes time to load
  await new Promise(resolve => setTimeout(resolve, 2000));
  const data = await fetch('/api/slow-data').then(r => r.json());
  return <div>Slow content: {data.result}</div>;
}

export default function Dashboard() {
  return (
    <div>
      <FastComponent />
      
      <Suspense fallback={<div>Loading slow content...</div>}>
        <SlowComponent />
      </Suspense>
    </div>
  );
}
```

### Nested Suspense with loading.jsx

```javascript
// app/reports/loading.jsx - Shows while entire page loads
export default function ReportsLoading() {
  return <div>Loading reports page...</div>;
}

// app/reports/page.jsx
import { Suspense } from 'react';

export default async function Reports() {
  // This data loads quickly
  const basicInfo = await fetch('/api/reports/basic').then(r => r.json());
  
  return (
    <div>
      <h1>Reports</h1>
      <BasicInfo data={basicInfo} />
      
      {/* Heavy chart loads separately */}
      <Suspense fallback={<div>Loading charts...</div>}>
        <HeavyChart />
      </Suspense>
      
      {/* User data loads separately */}
      <Suspense fallback={<div>Loading user data...</div>}>
        <UserAnalytics />
      </Suspense>
    </div>
  );
}
```

## Loading States for Different Data Types

### API Data Loading

```javascript
// app/blog/loading.jsx
export default function BlogLoading() {
  return (
    <div className="blog-loading">
      <div className="skeleton-post">
        <div className="skeleton-title"></div>
        <div className="skeleton-meta"></div>
        <div className="skeleton-content">
          {[...Array(4)].map((_, i) => (
            <div key={i} className="skeleton-paragraph"></div>
          ))}
        </div>
      </div>
    </div>
  );
}

// app/blog/page.jsx
export default async function Blog() {
  const posts = await fetch('/api/posts').then(r => r.json());
  
  return (
    <div className="blog">
      {posts.map(post => (
        <BlogPost key={post.id} post={post} />
      ))}
    </div>
  );
}
```

### Database Loading

```javascript
// app/orders/loading.jsx
export default function OrdersLoading() {
  return (
    <div className="orders-loading">
      <h1>Your Orders</h1>
      <div className="orders-list">
        {[...Array(5)].map((_, i) => (
          <div key={i} className="order-skeleton">
            <div className="skeleton-order-number"></div>
            <div className="skeleton-order-date"></div>
            <div className="skeleton-order-items"></div>
            <div className="skeleton-order-total"></div>
          </div>
        ))}
      </div>
    </div>
  );
}

// app/orders/page.jsx
export default async function Orders() {
  const orders = await getOrdersFromDB(userId);
  
  return (
    <div className="orders">
      <h1>Your Orders</h1>
      <OrdersList orders={orders} />
    </div>
  );
}
```

## Loading with Error Handling

### Complete Route States

```
app/dashboard/
├── loading.jsx                 // While loading
├── error.jsx                   // If error occurs
├── not-found.jsx              // If 404
└── page.jsx                    // Success state
```

```javascript
// app/dashboard/loading.jsx
export default function DashboardLoading() {
  return (
    <div className="loading-container">
      <div className="spinner"></div>
      <p>Loading your dashboard...</p>
    </div>
  );
}

// app/dashboard/error.jsx
"use client";

export default function DashboardError({ error, reset }) {
  return (
    <div className="error-container">
      <h2>Dashboard Error</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  );
}

// app/dashboard/page.jsx
export default async function Dashboard() {
  // loading.jsx shows during this
  const data = await fetchDashboardData();
  
  // error.jsx shows if this throws
  if (!data) {
    throw new Error("Dashboard data unavailable");
  }
  
  // Success: show actual dashboard
  return <DashboardContent data={data} />;
}
```

## Performance Optimization

### Streaming with loading.jsx

```javascript
// app/products/page.jsx
import { Suspense } from 'react';

// Fast loading header
async function ProductHeader() {
  const categories = await fetch('/api/categories').then(r => r.json());
  return <ProductNav categories={categories} />;
}

// Slower product list
async function ProductList() {
  const products = await fetch('/api/products').then(r => r.json());
  return <ProductGrid products={products} />;
}

export default function Products() {
  return (
    <div>
      {/* Shows immediately */}
      <Suspense fallback={<div>Loading navigation...</div>}>
        <ProductHeader />
      </Suspense>
      
      {/* Shows loading.jsx then streams in */}
      <Suspense fallback={<div>Loading products...</div>}>
        <ProductList />
      </Suspense>
    </div>
  );
}
```

## CSS for Loading States

```css
/* Skeleton loading styles */
.skeleton-line {
  height: 1rem;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
  border-radius: 4px;
  margin: 0.5rem 0;
}

.skeleton-line.long { width: 100%; }
.skeleton-line.medium { width: 75%; }
.skeleton-line.short { width: 50%; }

.skeleton-card {
  padding: 1rem;
  border: 1px solid #eee;
  border-radius: 8px;
}

.skeleton-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
  background-size: 200% 100%;
  animation: loading 1.5s infinite;
}

@keyframes loading {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}
```

## Best Practices

### 1. Match Content Structure

```javascript
// ✅ Good: Loading matches actual content structure
export default function BlogLoading() {
  return (
    <article>
      <header>
        <div className="skeleton-title"></div>
        <div className="skeleton-meta"></div>
      </header>
      <div className="skeleton-content">
        {[...Array(3)].map((_, i) => (
          <div key={i} className="skeleton-paragraph"></div>
        ))}
      </div>
    </article>
  );
}

// ❌ Bad: Generic loading that doesn't match content
export default function BlogLoading() {
  return <div>Loading...</div>;
}
```

### 2. Progressive Loading

```javascript
// Show structure immediately, load content progressively
export default function ProfileLoading() {
  return (
    <div className="profile">
      {/* Shell structure visible immediately */}
      <nav className="profile-nav">
        <div className="skeleton-nav-item"></div>
        <div className="skeleton-nav-item"></div>
      </nav>
      
      {/* Content areas with placeholders */}
      <main className="profile-content">
        <div className="skeleton-section"></div>
        <div className="skeleton-section"></div>
      </main>
    </div>
  );
}
```

### 3. Accessible Loading

```javascript
export default function AccessibleLoading() {
  return (
    <div 
      role="status" 
      aria-label="Loading content"
      className="loading-container"
    >
      <div className="spinner" aria-hidden="true"></div>
      <span className="sr-only">Loading...</span>
    </div>
  );
}
```

## Interview Tips

- **Automatic behavior**: loading.jsx shows automatically during async operations
- **No state management**: No useState or loading flags needed
- **Route-specific**: Each route can have custom loading UI
- **Inheritance**: Child routes inherit parent loading if they don't have their own
- **Works with Suspense**: Can combine with manual Suspense boundaries for granular control
- **Structure matching**: Loading UI should match the final content structure
- **Instant feedback**: Shows immediately on navigation, improving perceived performance

## Quick Reference

| File Location | Shows Loading For |
|--------------|-------------------|
| `app/loading.jsx` | All routes |
| `app/dashboard/loading.jsx` | `/dashboard/*` routes |
| `app/blog/[slug]/loading.jsx` | Individual blog posts |

**Remember**: loading.jsx shows during **server-side data fetching**. For client-side loading, use useState with loading flags.