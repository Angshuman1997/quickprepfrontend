# What is code splitting and when does it occur?

## Question
What is code splitting and when does it occur?

# What is code splitting and when does it occur?

## Question
What is code splitting and when does it occur?

## Answer

Code splitting is a technique that allows JavaScript bundlers to break a large bundle into smaller chunks that can be loaded on demand, improving application performance and user experience.

## What is Code Splitting?

### Basic Concept
```javascript
// Instead of one large bundle:
// bundle.js (2MB)

// Split into smaller chunks:
// app.js (500KB) - Main application
// dashboard.js (800KB) - Dashboard feature
// admin.js (700KB) - Admin panel
// vendor.js (300KB) - Third-party libraries
```

### Benefits
- **Faster initial load**: Only load essential code first
- **Better caching**: Smaller chunks cache more efficiently
- **Reduced bandwidth**: Users download only needed code
- **Improved performance**: Parallel loading of chunks

## When Code Splitting Occurs

### 1. Entry Point Splitting
```javascript
// webpack.config.js
module.exports = {
  entry: {
    main: './src/index.js',
    admin: './src/admin.js',
    vendor: ['react', 'react-dom']
  }
};

// Result: main.js, admin.js, vendor.js
```

### 2. Dynamic Imports (Most Common)
```javascript
// Route-based splitting
const loadDashboard = () => import('./components/Dashboard');
const loadAdmin = () => import('./components/AdminPanel');

// Event-based splitting
const loadHeavyComponent = () => import('./components/HeavyChart');

// Conditional splitting
if (userHasPermission) {
  import('./admin/AdvancedSettings').then(module => {
    // Use module
  });
}
```

### 3. Vendor/Library Splitting
```javascript
// Automatically split third-party libraries
// webpack.config.js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        chunks: 'all'
      }
    }
  }
}
```

## Code Splitting Techniques

### Route-Based Splitting (SPA)
```javascript
// React Router with code splitting
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { Suspense } from 'react';

const Home = lazy(() => import('./pages/Home'));
const About = lazy(() => import('./pages/About'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/about" element={<About />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### Component-Based Splitting
```javascript
// Heavy components loaded on demand
import { useState, Suspense } from 'react';
import { ErrorBoundary } from './ErrorBoundary';

const HeavyChart = lazy(() => import('./components/HeavyChart'));
const DataTable = lazy(() => import('./components/DataTable'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  const [showTable, setShowTable] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      <button onClick={() => setShowTable(true)}>Show Table</button>
      
      <ErrorBoundary>
        <Suspense fallback={<div>Loading chart...</div>}>
          {showChart && <HeavyChart />}
        </Suspense>
        
        <Suspense fallback={<div>Loading table...</div>}>
          {showTable && <DataTable />}
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

### Library Splitting
```javascript
// Split large libraries
const loadLodash = () => import('lodash');
const loadMoment = () => import('moment');

// Usage
async function processData(data) {
  const { default: _ } = await loadLodash();
  const { default: moment } = await loadMoment();
  
  // Process data
}
```

## Advanced Code Splitting

### Named Chunks
```javascript
// webpack chunk naming
import(
  /* webpackChunkName: "dashboard" */
  './components/Dashboard'
);

// Result: dashboard.chunk.js
```

### Magic Comments
```javascript
// Webpack magic comments
import(
  /* webpackChunkName: "admin-panel" */
  /* webpackMode: "lazy" */
  /* webpackPrefetch: true */
  './admin/AdminPanel'
);
```

### Preloading and Prefetching
```javascript
// Prefetch on hover
const handleMouseEnter = () => {
  import('./components/Tooltip');
};

// Preload critical resources
if ('requestIdleCallback' in window) {
  requestIdleCallback(() => {
    import('./non-critical/FeedbackForm');
  });
}
```

## Bundler-Specific Implementation

### Webpack Code Splitting
```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: 3,
      maxAsyncRequests: 5,
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        },
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all'
        }
      }
    }
  }
};
```

### Vite Code Splitting
```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          utils: ['lodash', 'moment']
        }
      }
    }
  }
};
```

### Rollup Code Splitting
```javascript
// rollup.config.js
export default {
  input: 'src/index.js',
  output: {
    dir: 'dist',
    format: 'es',
    manualChunks: {
      vendor: ['react']
    }
  }
};
```

## Performance Optimization

### Bundle Analysis
```javascript
// Analyze bundle sizes
import { BundleAnalyzerPlugin } from 'webpack-bundle-analyzer';

plugins: [
  new BundleAnalyzerPlugin()
];

// Identify large chunks and split them
```

### Critical Path Optimization
```javascript
// Load critical CSS/JS immediately
// Split non-critical code

// Above the fold content loads first
// Heavy components load after interaction
```

### Caching Strategies
```javascript
// Content hashing for cache busting
output: {
  filename: '[name].[contenthash].js',
  chunkFilename: '[name].[contenthash].chunk.js'
}

// Long-term caching for vendor chunks
// Short-term caching for app chunks
```

## Common Patterns

### Page-Based Splitting
```javascript
// Next.js automatic code splitting
// Each page is a separate chunk
export default function HomePage() {
  return <div>Home</div>;
}

// Manual splitting in Next.js
import dynamic from 'next/dynamic';

const HeavyComponent = dynamic(() => import('../components/Heavy'), {
  loading: () => <div>Loading...</div>
});
```

### Feature-Based Splitting
```javascript
// Feature flags
const ExperimentalFeature = lazy(() =>
  import('./features/ExperimentalFeature')
);

// User permissions
const AdminPanel = lazy(() =>
  user.isAdmin ? import('./admin/AdminPanel') : Promise.resolve(() => null)
);
```

## Monitoring and Metrics

### Performance Metrics
```javascript
// Track chunk loading performance
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.name.includes('.js')) {
      console.log(`Chunk ${entry.name} loaded in ${entry.duration}ms`);
    }
  }
});

observer.observe({ entryTypes: ['resource'] });
```

### Error Handling
```javascript
// Handle chunk loading failures
const loadComponent = async () => {
  try {
    const module = await import('./HeavyComponent');
    return module.default;
  } catch (error) {
    console.error('Failed to load component:', error);
    // Fallback to lighter version
    const fallback = await import('./LightComponent');
    return fallback.default;
  }
};
```

## Best Practices

1. **Route-based splitting**: Split at route level for SPAs
2. **Component-based splitting**: Split heavy components
3. **Vendor splitting**: Separate third-party libraries
4. **Lazy loading**: Load code on demand
5. **Prefetching**: Preload likely-needed chunks
6. **Error boundaries**: Handle loading failures gracefully
7. **Bundle analysis**: Monitor chunk sizes regularly

## Interview Tips
- **Dynamic imports**: `import()` creates split points
- **Route splitting**: Each route loads its own chunk
- **Vendor chunks**: Third-party libraries in separate files
- **Lazy loading**: Components load when needed
- **Performance**: Reduces initial bundle size
- **Caching**: Smaller chunks cache better
- **Suspense**: React component for loading states