# Purpose of template.js

## Question
Purpose of template.js

## Answer

The `template.js` file in Next.js App Router provides a specialized layout component that re-renders on every navigation within its segment, unlike `layout.js` which persists. Templates are useful for scenarios requiring fresh component instances, state resets, or dynamic behavior that should change with each route transition.

## Basic Template Structure

### Creating a Template

```javascript
// app/dashboard/template.js
export default function DashboardTemplate({ children }) {
    return (
        <div className="dashboard-template">
            <div className="template-header">
                <h2>Dashboard Section</h2>
                <div className="template-timestamp">
                    Rendered at: {new Date().toLocaleTimeString()}
                </div>
            </div>
            <div className="template-content">
                {children}
            </div>
        </div>
    );
}
```

**Template vs Layout Behavior:**
- **Layout**: Persists across navigation, maintains state
- **Template**: Re-renders on each navigation, resets state

## Template Re-rendering

### State Reset on Navigation

```javascript
// app/dashboard/template.js
'use client';

import { useState, useEffect } from 'react';

export default function DashboardTemplate({ children }) {
    const [renderCount, setRenderCount] = useState(0);
    const [timestamp, setTimestamp] = useState(Date.now());

    useEffect(() => {
        setRenderCount(count => count + 1);
        setTimestamp(Date.now());
    }, []);

    return (
        <div>
            <div className="template-info">
                <p>Template rendered {renderCount} times</p>
                <p>Last render: {new Date(timestamp).toLocaleTimeString()}</p>
            </div>
            {children}
        </div>
    );
}

// app/dashboard/analytics/page.js
export default function AnalyticsPage() {
    return <div>Analytics Content</div>;
}

// app/dashboard/users/page.js
export default function UsersPage() {
    return <div>Users Content</div>;
}
```

**Navigation Behavior:**
- Navigate from `/dashboard/analytics` to `/dashboard/users`
- Template re-renders with fresh state
- `renderCount` resets to 1
- `timestamp` updates to current time

## When to Use Templates

### 1. **Per-Route Analytics or Logging**

```javascript
// app/analytics/template.js
'use client';

import { useEffect } from 'react';
import { usePathname } from 'next/navigation';

export default function AnalyticsTemplate({ children }) {
    const pathname = usePathname();

    useEffect(() => {
        // Track page view on every navigation
        trackPageView(pathname);

        // Log route change
        console.log(`Navigated to: ${pathname}`);
    }, [pathname]);

    return (
        <div>
            <RouteTracker currentPath={pathname} />
            {children}
        </div>
    );
}
```

### 2. **Modal or Overlay Management**

```javascript
// app/dashboard/template.js
'use client';

import { useState } from 'react';

export default function DashboardTemplate({ children }) {
    const [modalOpen, setModalOpen] = useState(false);

    return (
        <div>
            <button onClick={() => setModalOpen(true)}>
                Open Modal
            </button>

            {children}

            {modalOpen && (
                <Modal onClose={() => setModalOpen(false)}>
                    <div>This modal resets on navigation</div>
                </Modal>
            )}
        </div>
    );
}
```

### 3. **Fresh Component Instances**

```javascript
// app/forms/template.js
'use client';

import { useState } from 'react';

export default function FormsTemplate({ children }) {
    const [formData, setFormData] = useState({});
    const [errors, setErrors] = useState({});

    // Form state resets on navigation between different forms
    const resetForm = () => {
        setFormData({});
        setErrors({});
    };

    return (
        <div>
            <FormContext.Provider value={{ formData, setFormData, errors, setErrors, resetForm }}>
                {children}
            </FormContext.Provider>
        </div>
    );
}
```

### 4. **Loading States per Route**

```javascript
// app/shop/template.js
'use client';

import { useTransition } from 'react';

export default function ShopTemplate({ children }) {
    const [isPending, startTransition] = useTransition();

    return (
        <div>
            {isPending && <div className="route-loading">Loading...</div>}
            {children}
        </div>
    );
}
```

## Template Hierarchy

### Nested Templates

```javascript
// app/layout.js - Root layout (persistent)
export default function RootLayout({ children }) {
    return (
        <html>
            <body>
                <GlobalHeader />
                {children}
                <GlobalFooter />
            </body>
        </html>
    );
}

// app/dashboard/layout.js - Dashboard layout (persistent)
export default function DashboardLayout({ children }) {
    return (
        <div className="dashboard">
            <DashboardSidebar />
            {children}
        </div>
    );
}

// app/dashboard/template.js - Dashboard template (re-renders)
export default function DashboardTemplate({ children }) {
    return (
        <div className="dashboard-template">
            <DashboardBreadcrumb />
            {children}
        </div>
    );
}

// app/dashboard/analytics/template.js - Specific template (re-renders)
export default function AnalyticsTemplate({ children }) {
    return (
        <div className="analytics-template">
            <AnalyticsControls />
            {children}
        </div>
    );
}
```

**Rendering Order:**
1. `RootLayout` (persistent)
2. `DashboardLayout` (persistent)
3. `DashboardTemplate` (re-renders on dashboard navigation)
4. `AnalyticsTemplate` (re-renders on analytics navigation)
5. Page content

## Template vs Layout Comparison

### Layout (Persistent)

```javascript
// app/dashboard/layout.js
'use client';

import { useState } from 'react';

export default function DashboardLayout({ children }) {
    const [sidebarCollapsed, setSidebarCollapsed] = useState(false);

    return (
        <div>
            <button onClick={() => setSidebarCollapsed(!sidebarCollapsed)}>
                Toggle Sidebar
            </button>
            <Sidebar collapsed={sidebarCollapsed} />
            {children}
        </div>
    );
}
```

**Layout Behavior:**
- State persists across navigation
- Component doesn't unmount/remount
- Good for: Navigation, persistent UI, shared state

### Template (Re-rendering)

```javascript
// app/dashboard/template.js
'use client';

import { useState } from 'react';

export default function DashboardTemplate({ children }) {
    const [activeTab, setActiveTab] = useState('overview');

    return (
        <div>
            <TabNavigation activeTab={activeTab} onChange={setActiveTab} />
            {children}
        </div>
    );
}
```

**Template Behavior:**
- State resets on navigation
- Component re-mounts for each route
- Good for: Per-route state, modals, fresh instances

## Advanced Template Patterns

### 1. **Route-Based Conditional Rendering**

```javascript
// app/admin/template.js
'use client';

import { usePathname } from 'next/navigation';

export default function AdminTemplate({ children }) {
    const pathname = usePathname();
    const isUsersPage = pathname.includes('/users');
    const isSettingsPage = pathname.includes('/settings');

    return (
        <div className="admin-template">
            <AdminHeader />

            {isUsersPage && <UserManagementToolbar />}
            {isSettingsPage && <SettingsToolbar />}

            <main>
                {children}
            </main>

            <AdminFooter />
        </div>
    );
}
```

### 2. **Template with Data Fetching**

```javascript
// app/blog/template.js
import { getCategories } from '../../lib/api';

export default async function BlogTemplate({ children }) {
    const categories = await getCategories();

    return (
        <div className="blog-template">
            <BlogHeader categories={categories} />
            <div className="blog-layout">
                <BlogSidebar categories={categories} />
                <main>
                    {children}
                </main>
            </div>
        </div>
    );
}
```

### 3. **Template with Context Reset**

```javascript
// app/forms/template.js
'use client';

import { FormProvider } from '../contexts/FormContext';

export default function FormsTemplate({ children }) {
    // Fresh FormProvider instance for each form route
    return (
        <FormProvider>
            <div className="form-template">
                <FormHeader />
                {children}
                <FormFooter />
            </div>
        </FormProvider>
    );
}
```

### 4. **Animation Templates**

```javascript
// app/gallery/template.js
'use client';

import { useEffect, useState } from 'react';

export default function GalleryTemplate({ children }) {
    const [isVisible, setIsVisible] = useState(false);

    useEffect(() => {
        // Trigger entrance animation on navigation
        setIsVisible(false);
        const timer = setTimeout(() => setIsVisible(true), 50);
        return () => clearTimeout(timer);
    }, []);

    return (
        <div className={`gallery-template ${isVisible ? 'visible' : ''}`}>
            {children}
        </div>
    );
}
```

## Template Performance Considerations

### Re-rendering Impact

```javascript
// Templates re-render on every navigation
// Consider performance implications

// ✅ Good: Lightweight template
export default function Template({ children }) {
    return (
        <div>
            <SimpleHeader />
            {children}
        </div>
    );
}

// ❌ Bad: Heavy template with expensive operations
export default function Template({ children }) {
    const expensiveData = performExpensiveCalculation();

    return (
        <div>
            <HeavyComponent data={expensiveData} />
            {children}
        </div>
    );
}
```

### Memory Management

```javascript
// Templates create new component instances
// Clean up resources properly

// ✅ Good: Proper cleanup
'use client';

import { useEffect } from 'react';

export default function Template({ children }) {
    useEffect(() => {
        const subscription = subscribeToData();

        return () => {
            subscription.unsubscribe(); // Cleanup on navigation
        };
    }, []);

    return <div>{children}</div>;
}
```

## Template with Loading States

### Integration with loading.js

```javascript
// app/shop/template.js
'use client';

import { useTransition } from 'react';

export default function ShopTemplate({ children }) {
    const [isPending, startTransition] = useTransition();

    return (
        <div className="shop-template">
            <ShopHeader />

            {isPending ? (
                <ShopLoadingSkeleton />
            ) : (
                <div className="shop-content">
                    {children}
                </div>
            )}

            <ShopFooter />
        </div>
    );
}

// app/shop/loading.js - Still works with template
export default function ShopLoading() {
    return <ShopLoadingSkeleton />;
}
```

## Testing Templates

### Testing Re-rendering Behavior

```javascript
// __tests__/template.test.js
import { render, screen } from '@testing-library/react';
import { useRouter } from 'next/navigation';
import DashboardTemplate from '../app/dashboard/template';

// Mock router
jest.mock('next/navigation', () => ({
    useRouter: () => ({
        push: jest.fn(),
        pathname: '/dashboard'
    })
}));

test('template re-renders on route change', () => {
    const { rerender } = render(
        <DashboardTemplate>
            <div>Content 1</div>
        </DashboardTemplate>
    );

    expect(screen.getByText('Content 1')).toBeInTheDocument();

    // Simulate route change by re-rendering with different content
    rerender(
        <DashboardTemplate>
            <div>Content 2</div>
        </DashboardTemplate>
    );

    expect(screen.getByText('Content 2')).toBeInTheDocument();
});
```

### Testing State Reset

```javascript
test('template state resets on navigation', () => {
    let renderCount = 0;

    function TestTemplate({ children }) {
        renderCount++;
        return (
            <div>
                <p>Render count: {renderCount}</p>
                {children}
            </div>
        );
    }

    const { rerender } = render(<TestTemplate>Content</TestTemplate>);
    expect(screen.getByText('Render count: 1')).toBeInTheDocument();

    // Simulate navigation (template re-mount)
    rerender(<TestTemplate>Content</TestTemplate>);
    expect(screen.getByText('Render count: 2')).toBeInTheDocument();
});
```

## Common Template Use Cases

### 1. **Breadcrumb Navigation**

```javascript
// app/docs/template.js
'use client';

import { usePathname } from 'next/navigation';
import Link from 'next/link';

export default function DocsTemplate({ children }) {
    const pathname = usePathname();
    const breadcrumbs = generateBreadcrumbs(pathname);

    return (
        <div className="docs-template">
            <nav className="breadcrumb">
                {breadcrumbs.map((crumb, index) => (
                    <span key={crumb.path}>
                        {index > 0 && ' / '}
                        <Link href={crumb.path}>{crumb.label}</Link>
                    </span>
                ))}
            </nav>

            <div className="docs-content">
                {children}
            </div>
        </div>
    );
}
```

### 2. **Page Transition Effects**

```javascript
// app/portfolio/template.js
'use client';

import { useEffect, useState } from 'react';

export default function PortfolioTemplate({ children }) {
    const [transitionStage, setTransitionStage] = useState('entering');

    useEffect(() => {
        setTransitionStage('entering');
        const timer = setTimeout(() => setTransitionStage('entered'), 300);
        return () => clearTimeout(timer);
    }, []);

    return (
        <div className={`portfolio-template ${transitionStage}`}>
            <PortfolioHeader />
            <div className="portfolio-content">
                {children}
            </div>
        </div>
    );
}
```

### 3. **Authentication Guards**

```javascript
// app/protected/template.js
'use client';

import { useAuth } from '../hooks/useAuth';
import { useRouter } from 'next/navigation';

export default function ProtectedTemplate({ children }) {
    const { user, loading } = useAuth();
    const router = useRouter();

    useEffect(() => {
        if (!loading && !user) {
            router.push('/login');
        }
    }, [user, loading, router]);

    if (loading) {
        return <div>Loading...</div>;
    }

    if (!user) {
        return null; // Will redirect
    }

    return (
        <div className="protected-template">
            <UserMenu user={user} />
            {children}
        </div>
    );
}
```

### 4. **Theme Switching per Section**

```javascript
// app/themes/template.js
'use client';

import { usePathname } from 'next/navigation';

export default function ThemeTemplate({ children }) {
    const pathname = usePathname();

    // Different theme for different sections
    const theme = pathname.includes('/dark') ? 'dark' :
                 pathname.includes('/light') ? 'light' : 'default';

    return (
        <div className={`theme-template theme-${theme}`}>
            <ThemeSwitcher currentTheme={theme} />
            {children}
        </div>
    );
}
```

## Best Practices

### 1. **Use Templates Sparingly**

```javascript
// ✅ Good: Use templates only when re-rendering is needed
// Use layouts for persistent UI
// Use templates for per-route behavior

// ❌ Bad: Using template when layout would suffice
export default function UnnecessaryTemplate({ children }) {
    return (
        <div>
            <StaticHeader /> {/* This could be in a layout */}
            {children}
        </div>
    );
}
```

### 2. **Keep Templates Lightweight**

```javascript
// ✅ Good: Minimal template logic
export default function Template({ children }) {
    return (
        <div>
            <Breadcrumb />
            {children}
        </div>
    );
}

// ❌ Bad: Heavy template with complex logic
export default function Template({ children }) {
    // Complex state management
    // Heavy computations
    // Multiple API calls
    return <div>{/* Complex template */}</div>;
}
```

### 3. **Clear Separation of Concerns**

```javascript
// ✅ Good: Template focuses on layout, page handles content
// app/dashboard/template.js
export default function DashboardTemplate({ children }) {
    return (
        <div>
            <DashboardControls />
            {children}
        </div>
    );
}

// app/dashboard/analytics/page.js
export default function AnalyticsPage() {
    return <AnalyticsContent />;
}
```

### Interview Tip:
*"`template.js` in Next.js App Router creates re-rendering layout components that reset state and re-mount on every navigation within their segment, unlike persistent layouts. Use templates for per-route analytics, modal management, fresh component instances, or any scenario requiring state reset on navigation."*