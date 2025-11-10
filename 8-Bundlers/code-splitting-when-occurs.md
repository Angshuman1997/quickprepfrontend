# Code Splitting in Vite

## Simple Definition
Code splitting breaks your app into smaller chunks that load on demand, making initial page loads faster.

## Why Code Splitting?

### Without Splitting
```javascript
// One massive bundle: 2MB
// User waits for everything to load before using app
```

### With Splitting
```javascript
// Multiple smaller chunks:
// app.js (300KB) - loads immediately
// dashboard.js (500KB) - loads when needed
// admin.js (400KB) - loads when needed
// vendor.js (200KB) - React, etc.
```

## How It Works in Vite

### Dynamic Imports
```javascript
// Automatic code splitting with dynamic imports
const loadDashboard = () => import('./components/Dashboard');
const loadAdmin = () => import('./components/AdminPanel');

// Usage
function App() {
  const [currentPage, setCurrentPage] = useState('home');

  return (
    <div>
      <button onClick={() => setCurrentPage('dashboard')}>
        Load Dashboard
      </button>

      {currentPage === 'dashboard' && (
        <Suspense fallback={<div>Loading...</div>}>
          <DashboardLoader />
        </Suspense>
      )}
    </div>
  );
}
```

### React Lazy Loading
```javascript
import { lazy, Suspense } from 'react';

// Lazy load components
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Admin = lazy(() => import('./pages/Admin'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/admin" element={<Admin />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

## Vite Configuration

### Manual Chunk Splitting
```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // Separate vendor libraries
          vendor: ['react', 'react-dom'],
          // Group utilities
          utils: ['lodash', 'moment']
        }
      }
    }
  }
})
```

### Automatic Splitting
```javascript
// Vite automatically splits:
// - Dynamic imports become separate chunks
// - Large dependencies get their own chunks
// - Common code shared between chunks
```

## When Code Splitting Happens

### 1. Dynamic Imports
```javascript
// Creates split point
const HeavyComponent = () => import('./HeavyComponent');

// Loads when user interacts
<button onClick={() => setShowHeavy(true)}>
  Load Heavy Feature
</button>
```

### 2. Route Changes
```javascript
// Each route loads its chunk
<Route path="/dashboard" element={<Dashboard />} />
// Loads dashboard.chunk.js when navigating
```

### 3. User Interactions
```javascript
// Load on button click
const handleClick = async () => {
  const { Modal } = await import('./Modal');
  // Modal code loads here
};
```

## Benefits

### Performance Impact
```javascript
// Initial load: 300KB instead of 2MB
// Dashboard load: 500KB only when needed
// Result: 85% faster initial page load
```

### Caching Benefits
```javascript
// Small chunks cache better
// Vendor libraries rarely change
// App updates don't invalidate vendor cache
```

## Interview Questions & Answers

**Q: What is code splitting?**
**A:** "Code splitting breaks a large bundle into smaller chunks that load on demand. Instead of loading everything upfront, users only download code when they need it."

**Q: How does code splitting work in Vite?**
**A:** "Vite automatically creates separate chunks for dynamic imports. Each `import()` statement becomes a split point, and Vite generates optimized chunks during build."

**Q: When should you use code splitting?**
**A:** "For large applications where initial bundle size matters. Split at route boundaries, for heavy components, or features that aren't immediately needed."

**Q: What's the difference between lazy loading and code splitting?**
**A:** "Code splitting is the bundler technique that creates separate chunks. Lazy loading is the pattern of loading those chunks on demand."

**Q: How do you implement code splitting in React?**
**A:** "Use `React.lazy()` with dynamic imports and wrap with `Suspense` for loading states. Vite automatically handles the chunk creation."

**Q: What about vendor libraries?**
**A:** "Vite automatically separates vendor chunks. You can also manually configure chunks in vite.config.js to group related libraries."

## Summary
- **Code Splitting**: Breaks app into smaller, loadable chunks
- **Dynamic Imports**: `import()` creates split points
- **Lazy Loading**: Load components on demand
- **Vite**: Automatic chunk optimization
- **Result**: Faster initial loads, better caching