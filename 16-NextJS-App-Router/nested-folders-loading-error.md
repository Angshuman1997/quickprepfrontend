# If the dashboard has nested folders

## Question
If the dashboard has nested folders

## Answer

The closest `error.jsx` or `loading.jsx` file is used. Nested folders can override parent loading and error states, or inherit from parent if they don't have their own.

## Folder Hierarchy Rules

### File Resolution Order
1. **Closest file wins**: Use the nearest loading/error file
2. **Inherit from parent**: If no local file, use parent's file
3. **Bubble up**: Continue up the tree until a file is found
4. **Default fallback**: Use Next.js default if none found

## Example Folder Structure

```
app/
├── loading.jsx                 // Global loading
├── error.jsx                   // Global error  
├── dashboard/
│   ├── loading.jsx            // Overrides global for all /dashboard/* routes
│   ├── error.jsx              // Overrides global for all /dashboard/* routes
│   ├── page.jsx               // Uses dashboard/loading.jsx and dashboard/error.jsx
│   ├── users/
│   │   ├── loading.jsx        // Overrides parent for /dashboard/users/* only
│   │   ├── page.jsx           // Uses users/loading.jsx and dashboard/error.jsx
│   │   └── [id]/
│   │       ├── page.jsx       // Uses users/loading.jsx and dashboard/error.jsx
│   │       └── edit/
│   │           ├── error.jsx  // Overrides for /dashboard/users/[id]/edit only
│   │           └── page.jsx   // Uses users/loading.jsx and edit/error.jsx
│   └── reports/
│       ├── error.jsx          // Overrides parent for /dashboard/reports/* only
│       └── page.jsx           // Uses dashboard/loading.jsx and reports/error.jsx
```

## Route Resolution Examples

### 1. `/dashboard` route
- **Loading**: `app/dashboard/loading.jsx` (local file)
- **Error**: `app/dashboard/error.jsx` (local file)

### 2. `/dashboard/users` route  
- **Loading**: `app/dashboard/users/loading.jsx` (overrides parent)
- **Error**: `app/dashboard/error.jsx` (inherits from parent)

### 3. `/dashboard/users/123` route
- **Loading**: `app/dashboard/users/loading.jsx` (inherits from parent folder)  
- **Error**: `app/dashboard/error.jsx` (bubbles up to grandparent)

### 4. `/dashboard/users/123/edit` route
- **Loading**: `app/dashboard/users/loading.jsx` (inherits from parent)
- **Error**: `app/dashboard/users/[id]/edit/error.jsx` (local file)

### 5. `/dashboard/reports` route
- **Loading**: `app/dashboard/loading.jsx` (inherits from parent)
- **Error**: `app/dashboard/reports/error.jsx` (overrides parent)

## Real Implementation Examples

### Parent Dashboard Loading

```javascript
// app/dashboard/loading.jsx
export default function DashboardLoading() {
  return (
    <div className="dashboard-layout">
      <div className="dashboard-sidebar">
        <div className="skeleton-nav">
          {[...Array(5)].map((_, i) => (
            <div key={i} className="skeleton-nav-item"></div>
          ))}
        </div>
      </div>
      <div className="dashboard-content">
        <div className="skeleton-header"></div>
        <div className="skeleton-content"></div>
      </div>
    </div>
  );
}
```

### Child Users Override

```javascript
// app/dashboard/users/loading.jsx (overrides parent)
export default function UsersLoading() {
  return (
    <div className="users-loading">
      <h1>Users</h1>
      <div className="users-table-skeleton">
        <div className="skeleton-table-header"></div>
        {[...Array(10)].map((_, i) => (
          <div key={i} className="skeleton-table-row"></div>
        ))}
      </div>
    </div>
  );
}
```

### Grandchild Inherits

```javascript
// app/dashboard/users/[id]/page.jsx
// Uses app/dashboard/users/loading.jsx (inherited)
// Uses app/dashboard/error.jsx (inherited)

export default async function UserDetail({ params }) {
  const user = await fetch(`/api/users/${params.id}`).then(r => r.json());
  
  return (
    <div>
      <h1>User: {user.name}</h1>
      <UserProfile user={user} />
    </div>
  );
}
```

## Selective Overriding

### Only Override Loading

```
app/dashboard/
├── loading.jsx                 // Shared loading for all children
├── error.jsx                   // Shared error for all children
├── analytics/
│   ├── loading.jsx            // Custom loading for analytics only
│   └── page.jsx               // Uses analytics/loading.jsx + dashboard/error.jsx
└── settings/
    └── page.jsx                // Uses dashboard/loading.jsx + dashboard/error.jsx
```

### Only Override Error

```
app/dashboard/
├── loading.jsx                 // Shared loading for all children
├── error.jsx                   // Shared error for all children  
├── billing/
│   ├── error.jsx              // Custom error for billing only
│   └── page.jsx               // Uses dashboard/loading.jsx + billing/error.jsx
└── profile/
    └── page.jsx                // Uses dashboard/loading.jsx + dashboard/error.jsx
```

## Practical Implementation

### Different Loading for Different Sections

```javascript
// app/dashboard/loading.jsx - Generic dashboard loading
export default function DashboardLoading() {
  return <div>Loading dashboard...</div>;
}

// app/dashboard/analytics/loading.jsx - Analytics-specific loading
export default function AnalyticsLoading() {
  return (
    <div className="analytics-loading">
      <h1>Analytics</h1>
      <div className="chart-skeletons">
        <div className="skeleton-chart large"></div>
        <div className="skeleton-chart-row">
          <div className="skeleton-chart small"></div>
          <div className="skeleton-chart small"></div>
        </div>
      </div>
    </div>
  );
}

// app/dashboard/settings/loading.jsx - Settings-specific loading  
export default function SettingsLoading() {
  return (
    <div className="settings-loading">
      <h1>Settings</h1>
      <div className="settings-skeleton">
        <div className="skeleton-form">
          {[...Array(6)].map((_, i) => (
            <div key={i} className="skeleton-form-field"></div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

### Different Error Handling for Different Sections

```javascript
// app/dashboard/error.jsx - Generic dashboard error
"use client";

export default function DashboardError({ error, reset }) {
  return (
    <div>
      <h2>Dashboard Error</h2>
      <p>Something went wrong with the dashboard.</p>
      <button onClick={reset}>Try Again</button>
    </div>
  );
}

// app/dashboard/payments/error.jsx - Payment-specific error handling
"use client";

export default function PaymentsError({ error, reset }) {
  const isPaymentError = error.message.includes('payment');
  
  return (
    <div>
      <h2>Payment System Error</h2>
      {isPaymentError ? (
        <div>
          <p>Our payment system is temporarily unavailable.</p>
          <p>Your data is safe. Please try again in a few minutes.</p>
        </div>
      ) : (
        <p>Unable to load payment information.</p>
      )}
      <button onClick={reset}>Retry</button>
      <a href="/dashboard">Back to Dashboard</a>
    </div>
  );
}
```

## Complex Nested Example

```
app/
├── error.jsx                   // Global fallback
├── loading.jsx                 // Global fallback  
├── dashboard/
│   ├── loading.jsx            // All dashboard routes (unless overridden)
│   ├── error.jsx              // All dashboard routes (unless overridden)
│   ├── page.jsx               // ✓ Uses dashboard loading/error
│   ├── users/
│   │   ├── loading.jsx        // ✓ Overrides for /users/* routes
│   │   ├── page.jsx           // ✓ Uses users/loading + dashboard/error
│   │   ├── [id]/
│   │   │   ├── page.jsx       // ✓ Uses users/loading + dashboard/error
│   │   │   └── edit/
│   │   │       ├── error.jsx  // ✓ Overrides for /edit route only
│   │   │       └── page.jsx   // ✓ Uses users/loading + edit/error
│   │   └── create/
│   │       └── page.jsx       // ✓ Uses users/loading + dashboard/error
│   └── reports/
│       ├── error.jsx          // ✓ Overrides for /reports/* routes
│       ├── page.jsx           // ✓ Uses dashboard/loading + reports/error
│       └── export/
│           └── page.jsx       // ✓ Uses dashboard/loading + reports/error
```

## Dynamic Route Considerations

### Route Parameters Inherit

```javascript
// app/dashboard/users/[id]/page.jsx
export default async function UserPage({ params }) {
  // Uses app/dashboard/users/loading.jsx (if exists)
  // Falls back to app/dashboard/loading.jsx
  // Falls back to app/loading.jsx
  
  const user = await fetch(`/api/users/${params.id}`).then(r => r.json());
  return <UserProfile user={user} />;
}
```

### Catch-All Routes

```javascript
// app/dashboard/docs/[...slug]/page.jsx
export default async function DocsPage({ params }) {
  // Uses closest loading/error in this order:
  // 1. app/dashboard/docs/[...slug]/loading.jsx
  // 2. app/dashboard/docs/loading.jsx  
  // 3. app/dashboard/loading.jsx
  // 4. app/loading.jsx
  
  const doc = await fetchDoc(params.slug.join('/'));
  return <DocContent doc={doc} />;
}
```

## Best Practices

### 1. **Strategic Placement**

```javascript
// ✅ Good: Place loading/error at logical boundaries
app/dashboard/
├── loading.jsx                 // Covers all dashboard features
├── error.jsx                   // Handles dashboard-wide errors
├── users/                      // User management section
│   ├── loading.jsx            // User-specific loading (table view)
│   └── [id]/
│       └── edit/
│           └── error.jsx      // Critical error handling for edits
```

### 2. **Consistent User Experience**

```javascript
// Keep similar loading patterns within sections
// app/dashboard/users/loading.jsx
export default function UsersLoading() {
  return <TableSkeleton columns={4} rows={10} title="Users" />;
}

// app/dashboard/orders/loading.jsx  
export default function OrdersLoading() {
  return <TableSkeleton columns={5} rows={8} title="Orders" />;
}
```

### 3. **Meaningful Error Boundaries**

```javascript
// Place error boundaries where failures have different impacts
// app/dashboard/billing/error.jsx - Critical financial operations
"use client";

export default function BillingError({ error, reset }) {
  return (
    <div className="critical-error">
      <h2>Billing System Unavailable</h2>
      <p>For account security, billing operations are temporarily disabled.</p>
      <button onClick={reset}>Try Again</button>
      <a href="/support">Contact Support</a>
    </div>
  );
}
```

## Interview Tips

- **Closest file wins**: Nested folders override parent files
- **Inheritance**: Missing files inherit from parent directories
- **Strategic placement**: Place loading/error files at logical feature boundaries
- **Override selectively**: Only override when you need different behavior
- **Consider user experience**: Match loading states to content structure
- **Error boundaries**: Place error handling where failures have different impacts

## Quick Reference

| Route | Loading File Used | Error File Used |
|-------|------------------|-----------------|
| `/dashboard` | `dashboard/loading.jsx` | `dashboard/error.jsx` |
| `/dashboard/users` | `users/loading.jsx` (override) | `dashboard/error.jsx` (inherit) |
| `/dashboard/users/123` | `users/loading.jsx` (inherit) | `dashboard/error.jsx` (inherit) |
| `/dashboard/reports` | `dashboard/loading.jsx` (inherit) | `reports/error.jsx` (override) |

**Remember**: Files cascade down the folder tree unless overridden at a specific level.