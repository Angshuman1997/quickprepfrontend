# Purpose of layout.js

## Question
Purpose of layout.js

## Answer

The `layout.js` file in Next.js App Router provides a way to define shared UI and state that persists across multiple route segments. Layouts wrap child route components, enabling shared navigation, sidebars, headers, footers, and other common UI elements without re-rendering on navigation.

## Basic Layout Structure

### Creating a Root Layout

```javascript
// app/layout.js
export default function RootLayout({ children }) {
    return (
        <html lang="en">
            <body>
                <header>
                    <nav>
                        <Link href="/">Home</Link>
                        <Link href="/about">About</Link>
                        <Link href="/contact">Contact</Link>
                    </nav>
                </header>

                <main>
                    {children}
                </main>

                <footer>
                    <p>© 2024 My App</p>
                </footer>
            </body>
        </html>
    );
}
```

**Layout Hierarchy:**
- `app/layout.js` - Root layout for entire application
- `app/dashboard/layout.js` - Layout for dashboard routes
- `app/blog/layout.js` - Layout for blog routes
- `app/(auth)/layout.js` - Layout for auth routes (route groups)

## How Layouts Work

### Nested Layouts

```javascript
// app/layout.js - Root layout
export default function RootLayout({ children }) {
    return (
        <html>
            <body>
                <GlobalHeader />
                <div className="app-container">
                    {children}
                </div>
                <GlobalFooter />
            </body>
        </html>
    );
}

// app/dashboard/layout.js - Dashboard layout
export default function DashboardLayout({ children }) {
    return (
        <div className="dashboard-layout">
            <DashboardSidebar />
            <main className="dashboard-main">
                <DashboardHeader />
                {children}
            </main>
        </div>
    );
}

// app/dashboard/analytics/page.js - Page component
export default function AnalyticsPage() {
    return (
        <div>
            <h1>Analytics</h1>
            <AnalyticsChart />
        </div>
    );
}
```

**Rendered Structure:**
```html
<html>
<body>
    <GlobalHeader />
    <div class="app-container">
        <div class="dashboard-layout">
            <DashboardSidebar />
            <main class="dashboard-main">
                <DashboardHeader />
                <div>
                    <h1>Analytics</h1>
                    <AnalyticsChart />
                </div>
            </main>
        </div>
    </div>
    <GlobalFooter />
</body>
</html>
```

## Layout Persistence

### State Preservation

```javascript
// app/dashboard/layout.js
'use client';

import { useState } from 'react';

export default function DashboardLayout({ children }) {
    const [sidebarCollapsed, setSidebarCollapsed] = useState(false);
    const [theme, setTheme] = useState('light');

    return (
        <div className={`dashboard ${theme}`}>
            <Sidebar
                collapsed={sidebarCollapsed}
                onToggle={() => setSidebarCollapsed(!sidebarCollapsed)}
            />
            <main>
                <Header
                    theme={theme}
                    onThemeChange={setTheme}
                />
                {children}
            </main>
        </div>
    );
}
```

**State Persistence Benefits:**
- Sidebar collapse state maintained during navigation
- Theme preferences preserved
- Form state in layout components
- Authentication state

### Context Providers

```javascript
// contexts/AuthContext.js
'use client';

import { createContext, useContext, useState } from 'react';

const AuthContext = createContext();

export function AuthProvider({ children }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    return (
        <AuthContext.Provider value={{ user, setUser, loading }}>
            {children}
        </AuthContext.Provider>
    );
}

export function useAuth() {
    return useContext(AuthContext);
}

// app/layout.js
import { AuthProvider } from '../contexts/AuthContext';

export default function RootLayout({ children }) {
    return (
        <html>
            <body>
                <AuthProvider>
                    <GlobalHeader />
                    {children}
                    <GlobalFooter />
                </AuthProvider>
            </body>
        </html>
    );
}
```

## Layout Composition Patterns

### 1. **Shared Navigation**

```javascript
// components/Navigation.js
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';

export function Navigation() {
    const pathname = usePathname();

    return (
        <nav>
            <Link href="/" className={pathname === '/' ? 'active' : ''}>
                Home
            </Link>
            <Link href="/products" className={pathname === '/products' ? 'active' : ''}>
                Products
            </Link>
            <Link href="/about" className={pathname === '/about' ? 'active' : ''}>
                About
            </Link>
        </nav>
    );
}

// app/layout.js
import { Navigation } from '../components/Navigation';

export default function RootLayout({ children }) {
    return (
        <html>
            <body>
                <header>
                    <Navigation />
                </header>
                <main>{children}</main>
            </body>
        </html>
    );
}
```

### 2. **Conditional Layouts**

```javascript
// app/(auth)/layout.js - Auth routes
export default function AuthLayout({ children }) {
    return (
        <div className="auth-layout">
            <div className="auth-container">
                <div className="auth-branding">
                    <h1>My App</h1>
                    <p>Welcome back!</p>
                </div>
                <div className="auth-form">
                    {children}
                </div>
            </div>
        </div>
    );
}

// app/(dashboard)/layout.js - Dashboard routes
export default function DashboardLayout({ children }) {
    return (
        <div className="dashboard-layout">
            <Sidebar />
            <main>{children}</main>
        </div>
    );
}
```

### 3. **Dynamic Layouts**

```javascript
// app/blog/layout.js
import { getCategories } from '../lib/api';

export default async function BlogLayout({ children }) {
    const categories = await getCategories();

    return (
        <div className="blog-layout">
            <aside className="blog-sidebar">
                <h3>Categories</h3>
                <ul>
                    {categories.map(category => (
                        <li key={category.id}>
                            <Link href={`/blog/category/${category.slug}`}>
                                {category.name}
                            </Link>
                        </li>
                    ))}
                </ul>
            </aside>
            <main className="blog-content">
                {children}
            </main>
        </div>
    );
}
```

## Layout vs Template

### Layout (Persistent)

```javascript
// app/dashboard/layout.js
'use client';

import { useState } from 'react';

export default function DashboardLayout({ children }) {
    const [count, setCount] = useState(0);

    return (
        <div>
            <button onClick={() => setCount(count + 1)}>
                Count: {count}
            </button>
            {children}
        </div>
    );
}
```

**Layout Behavior:**
- State persists across navigation
- Component doesn't re-render on route changes
- Children content changes but layout stays

### Template (Re-render)

```javascript
// app/dashboard/template.js
'use client';

import { useState } from 'react';

export default function DashboardTemplate({ children }) {
    const [count, setCount] = useState(0);

    return (
        <div>
            <button onClick={() => setCount(count + 1)}>
                Count: {count}
            </button>
            {children}
        </div>
    );
}
```

**Template Behavior:**
- Re-renders on every navigation
- State resets on route changes
- Useful for modals, loading states

## Advanced Layout Patterns

### 1. **Route Groups for Organization**

```javascript
// app/(marketing)/layout.js
export default function MarketingLayout({ children }) {
    return (
        <div className="marketing-layout">
            <MarketingHeader />
            {children}
            <MarketingFooter />
        </div>
    );
}

// app/(shop)/layout.js
export default function ShopLayout({ children }) {
    return (
        <div className="shop-layout">
            <ShopHeader />
            <CartProvider>
                {children}
            </CartProvider>
            <ShopFooter />
        </div>
    );
}

// File structure:
// app/
//   (marketing)/
//     layout.js
//     page.js
//     about/page.js
//   (shop)/
//     layout.js
//     products/page.js
//     cart/page.js
```

### 2. **Parallel Routes**

```javascript
// app/dashboard/layout.js
export default function DashboardLayout({ children, modal }) {
    return (
        <div>
            {children}
            {modal}
        </div>
    );
}

// app/dashboard/@modal/page.js
export default function ModalPage() {
    return (
        <dialog open>
            <p>This is a modal</p>
        </dialog>
    );
}
```

### 3. **Intercepting Routes**

```javascript
// app/dashboard/layout.js
export default function DashboardLayout({ children }) {
    return (
        <div>
            <h1>Dashboard</h1>
            {children}
        </div>
    );
}

// app/dashboard/(..)photo/[id]/page.js - Intercepting route
export default function PhotoModal({ params }) {
    return (
        <dialog open>
            <Photo id={params.id} />
        </dialog>
    );
}
```

## Layout Data Fetching

### Server Component Layouts

```javascript
// app/blog/layout.js
import { getUser } from '../lib/auth';
import { getBlogSettings } from '../lib/api';

export default async function BlogLayout({ children }) {
    const user = await getUser();
    const settings = await getBlogSettings();

    return (
        <div className="blog-layout">
            <BlogHeader user={user} settings={settings} />
            <div className="blog-content">
                {children}
            </div>
            <BlogFooter settings={settings} />
        </div>
    );
}
```

### Client Component Layouts

```javascript
// app/dashboard/layout.js
'use client';

import { useEffect, useState } from 'react';

export default function DashboardLayout({ children }) {
    const [user, setUser] = useState(null);

    useEffect(() => {
        fetchUser().then(setUser);
    }, []);

    if (!user) {
        return <div>Loading...</div>;
    }

    return (
        <div>
            <DashboardHeader user={user} />
            {children}
        </div>
    );
}
```

## Layout Metadata

### SEO and Metadata

```javascript
// app/layout.js
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
    title: 'My App',
    description: 'A modern web application',
};

export default function RootLayout({ children }) {
    return (
        <html lang="en">
            <body className={inter.className}>
                {children}
            </body>
        </html>
    );
}

// app/blog/layout.js
export const metadata = {
    title: {
        default: 'Blog',
        template: '%s | My Blog'
    },
    description: 'Read our latest blog posts',
};
```

## Performance Considerations

### Layout Optimization

```javascript
// ✅ Good: Minimal layout re-renders
export default function Layout({ children }) {
    return (
        <div>
            <StaticHeader />
            {children}
        </div>
    );
}

// ❌ Bad: Heavy computations in layout
export default function Layout({ children }) {
    const expensiveData = useMemo(() => {
        // Heavy computation on every render
        return processLargeDataset();
    }, []);

    return (
        <div>
            <Header data={expensiveData} />
            {children}
        </div>
    );
}
```

### Bundle Splitting

```javascript
// Layouts enable automatic code splitting
// Each route segment loads its layout independently

// app/dashboard/layout.js - Only loaded for dashboard routes
export default function DashboardLayout({ children }) {
    return (
        <div>
            <HeavyDashboardLibrary />
            {children}
        </div>
    );
}

// app/blog/layout.js - Only loaded for blog routes
export default function BlogLayout({ children }) {
    return (
        <div>
            <HeavyBlogComponents />
            {children}
        </div>
    );
}
```

## Testing Layouts

### Unit Testing

```javascript
// __tests__/layout.test.js
import { render, screen } from '@testing-library/react';
import RootLayout from '../app/layout';

test('renders layout with children', () => {
    render(
        <RootLayout>
            <div>Test Content</div>
        </RootLayout>
    );

    expect(screen.getByText('Test Content')).toBeInTheDocument();
    expect(screen.getByRole('navigation')).toBeInTheDocument();
});
```

### Integration Testing

```javascript
// Test layout persistence
test('layout state persists across navigation', async () => {
    render(<App />);

    // Open sidebar
    fireEvent.click(screen.getByText('Toggle Sidebar'));
    expect(screen.getByTestId('sidebar')).toHaveClass('open');

    // Navigate to different page
    fireEvent.click(screen.getByText('Go to Analytics'));

    // Sidebar should still be open
    expect(screen.getByTestId('sidebar')).toHaveClass('open');
});
```

## Common Layout Patterns

### 1. **Authentication Layout**

```javascript
// app/(auth)/layout.js
export default function AuthLayout({ children }) {
    return (
        <div className="auth-layout">
            <div className="auth-card">
                <div className="auth-header">
                    <Logo />
                    <h1>Welcome</h1>
                </div>
                {children}
            </div>
        </div>
    );
}
```

### 2. **Dashboard Layout**

```javascript
// app/dashboard/layout.js
'use client';

import { useState } from 'react';

export default function DashboardLayout({ children }) {
    const [sidebarOpen, setSidebarOpen] = useState(true);

    return (
        <div className="dashboard">
            <Sidebar open={sidebarOpen} onToggle={setSidebarOpen} />
            <main className={sidebarOpen ? 'sidebar-open' : 'sidebar-closed'}>
                <TopBar onMenuClick={() => setSidebarOpen(!sidebarOpen)} />
                <div className="content">
                    {children}
                </div>
            </main>
        </div>
    );
}
```

### 3. **Blog Layout**

```javascript
// app/blog/layout.js
import { getCategories } from '../lib/api';

export default async function BlogLayout({ children }) {
    const categories = await getCategories();

    return (
        <div className="blog-container">
            <BlogHeader />
            <div className="blog-grid">
                <BlogSidebar categories={categories} />
                <main className="blog-content">
                    {children}
                </main>
            </div>
            <BlogFooter />
        </div>
    );
}
```

## Best Practices

### 1. **Keep Layouts Simple**

```javascript
// ✅ Good: Focused layout
export default function Layout({ children }) {
    return (
        <div>
            <Header />
            <main>{children}</main>
            <Footer />
        </div>
    );
}

// ❌ Bad: Layout doing too much
export default function Layout({ children }) {
    // Complex business logic in layout
    const [user, setUser] = useState(null);
    const [cart, setCart] = useState([]);

    useEffect(() => {
        loadUser();
        loadCart();
    }, []);

    return (
        <AuthProvider>
            <CartProvider>
                <ThemeProvider>
                    <div>
                        <ComplexHeader user={user} cart={cart} />
                        {children}
                    </div>
                </ThemeProvider>
            </CartProvider>
        </AuthProvider>
    );
}
```

### 2. **Use Route Groups for Organization**

```javascript
// app/
//   (auth)/
//     layout.js
//     login/page.js
//     register/page.js
//   (dashboard)/
//     layout.js
//     analytics/page.js
//     users/page.js
//   (marketing)/
//     layout.js
//     home/page.js
//     about/page.js
```

### 3. **Consistent Layout Structure**

```javascript
// Consistent patterns across layouts
export default function Layout({ children }) {
    return (
        <div className="layout">
            <header className="layout-header">
                {/* Header content */}
            </header>
            <main className="layout-main">
                {children}
            </main>
            <footer className="layout-footer">
                {/* Footer content */}
            </footer>
        </div>
    );
}
```

### Interview Tip:
*"`layout.js` in Next.js App Router creates persistent UI containers that wrap route segments, maintaining state and shared components across navigation. Unlike pages that re-render, layouts persist during route changes, enabling shared navigation, sidebars, and context providers without losing component state."*