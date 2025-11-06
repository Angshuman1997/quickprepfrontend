# How do you share dependencies (e.g., React) across MFEs?

## Question
How do you share dependencies (e.g., React) across MFEs?

# How do you share dependencies (e.g., React) across MFEs?

## Question
How do you share dependencies (e.g., React) across MFEs?

## Answer

Sharing dependencies across micro frontends is crucial for performance optimization and bundle size reduction. Module Federation provides the `shared` configuration to handle this.

## Basic Shared Dependencies

### Host Application
```javascript
// webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host_app',
      remotes: {
        products: 'products@http://localhost:3001/remoteEntry.js',
        cart: 'cart@http://localhost:3002/remoteEntry.js'
      },
      shared: {
        react: {
          singleton: true,
          eager: true
        },
        'react-dom': {
          singleton: true,
          eager: true
        },
        '@reduxjs/toolkit': {
          singleton: true
        }
      }
    })
  ]
};
```

### Remote Application (Products)
```javascript
// webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'products',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/components/ProductList'
      },
      shared: {
        react: {
          singleton: true,
          eager: true
        },
        'react-dom': {
          singleton: true,
          eager: true
        },
        '@reduxjs/toolkit': {
          singleton: true
        }
      }
    })
  ]
};
```

## Advanced Shared Configuration

### Version-Specific Sharing
```javascript
shared: {
  react: {
    singleton: true,
    eager: true,
    requiredVersion: '^18.0.0',
    version: '18.2.0'
  },
  'react-dom': {
    singleton: true,
    eager: true,
    requiredVersion: '^18.0.0'
  },
  lodash: {
    eager: false, // Load on demand
    singleton: false // Allow multiple versions
  }
}
```

### Conditional Sharing
```javascript
// webpack.config.js
const isProduction = process.env.NODE_ENV === 'production';

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host_app',
      shared: {
        react: {
          singleton: true,
          eager: true
        },
        ...(isProduction && {
          // Only share in production
          'react-query': {
            singleton: true
          }
        })
      }
    })
  ]
};
```

## Shared Scope Management

### Custom Shared Scope
```javascript
// webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host_app',
      shared: {
        react: {
          singleton: true,
          shareScope: 'default'
        }
      },
      shareScope: 'custom-scope'
    })
  ]
};
```

### Multiple Share Scopes
```javascript
// Different MFEs can use different scopes
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

// MFE 1 - Business Logic
new ModuleFederationPlugin({
  name: 'business_mfe',
  shared: {
    'business-logic': {
      shareScope: 'business'
    }
  }
});

// MFE 2 - UI Components
new ModuleFederationPlugin({
  name: 'ui_mfe',
  shared: {
    'ui-library': {
      shareScope: 'ui'
    }
  }
});
```

## Runtime Shared Dependencies

### Dynamic Import with Shared Modules
```javascript
// src/components/LazyComponent.js
import React, { Suspense } from 'react';

// Dynamic import that respects shared dependencies
const LazyProductList = React.lazy(() =>
  import('products/ProductList')
);

const LazyComponent = () => {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <LazyProductList />
    </Suspense>
  );
};
```

### Shared Utility Functions
```javascript
// shared-utils/src/index.js
export const formatCurrency = (amount, currency = 'USD') => {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency
  }).format(amount);
};

export const debounce = (func, wait) => {
  let timeout;
  return function executedFunction(...args) {
    const later = () => {
      clearTimeout(timeout);
      func(...args);
    };
    clearTimeout(timeout);
    timeout = setTimeout(later, wait);
  };
};
```

```javascript
// webpack.config.js - Shared Utils
new ModuleFederationPlugin({
  name: 'shared_utils',
  filename: 'remoteEntry.js',
  exposes: {
    './utils': './src/index.js'
  },
  shared: ['lodash'] // Dependencies of shared utils
});
```

## Bundle Analysis and Optimization

### Analyzing Shared Dependencies
```javascript
// webpack.config.js
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
      reportFilename: 'bundle-report.html'
    }),
    new ModuleFederationPlugin({
      // ... shared config
    })
  ]
};
```

### Selective Sharing Strategy
```javascript
// Only share large libraries
const sharedDependencies = {
  react: { singleton: true, eager: true },
  'react-dom': { singleton: true, eager: true },
  '@reduxjs/toolkit': { singleton: true },
  'react-router-dom': { singleton: true },
  // Don't share small utilities
  // lodash: { singleton: true } // Too small to share
};
```

## Error Handling and Fallbacks

### Version Conflict Resolution
```javascript
// webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host_app',
      shared: {
        react: {
          singleton: true,
          eager: true,
          requiredVersion: '^18.0.0',
          version: '18.2.0'
        }
      }
    })
  ]
};
```

### Fallback for Missing Shared Dependencies
```javascript
// src/bootstrap.js
import React from 'react';
import ReactDOM from 'react-dom';

// Check if shared dependencies are available
const initializeApp = async () => {
  try {
    // Try to load remote with shared dependencies
    const { default: App } = await import('./App');
    ReactDOM.render(<App />, document.getElementById('root'));
  } catch (error) {
    console.warn('Shared dependencies not available, loading standalone version');
    // Fallback to standalone version
    const { default: StandaloneApp } = await import('./StandaloneApp');
    ReactDOM.render(<StandaloneApp />, document.getElementById('root'));
  }
};

initializeApp();
```

## Performance Considerations

### Eager vs Lazy Loading
```javascript
shared: {
  // Load immediately with host
  react: {
    eager: true, // Loads with host bundle
    singleton: true
  },
  // Load when needed
  'react-query': {
    eager: false, // Loads when first MFE needs it
    singleton: true
  }
}
```

### Bundle Size Monitoring
```javascript
// scripts/analyze-shared.js
const fs = require('fs');
const path = require('path');

const analyzeSharedDependencies = () => {
  const stats = JSON.parse(fs.readFileSync('./dist/stats.json', 'utf8'));
  
  const sharedModules = stats.modules.filter(module => 
    module.issuer.includes('shared')
  );
  
  console.log('Shared Dependencies Analysis:');
  sharedModules.forEach(module => {
    console.log(`${module.name}: ${module.size} bytes`);
  });
};

analyzeSharedDependencies();
```

## Best Practices

1. **Singleton Pattern**: Use `singleton: true` for libraries that shouldn't have multiple instances
2. **Eager Loading**: Use `eager: true` for critical dependencies loaded by the host
3. **Version Pinning**: Specify `requiredVersion` to prevent version conflicts
4. **Bundle Analysis**: Regularly analyze bundle sizes to optimize sharing strategy
5. **Error Boundaries**: Implement fallbacks for shared dependency loading failures

## Interview Tips
- **Shared config**: Most important part of Module Federation setup
- **Singleton**: Prevents multiple React instances (causes errors)
- **Eager**: Critical dependencies load with host, others load on demand
- **Version conflicts**: Can break MFEs, use requiredVersion
- **Performance**: Sharing reduces bundle size but increases initial load
- **Analysis**: Use bundle analyzer to monitor shared dependency sizes