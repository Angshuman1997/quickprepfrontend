# Sharing Dependencies Across Micro Frontends

## Why Share Dependencies?

### Problem Without Sharing
```javascript
// Each MFE includes React separately
// Total bundle size: 500KB + 500KB + 500KB = 1.5MB

MFE1: React (500KB) + Components (100KB) = 600KB
MFE2: React (500KB) + Components (150KB) = 650KB
MFE3: React (500KB) + Components (80KB) = 580KB
```

### Solution With Sharing
```javascript
// React loaded once, shared across MFEs
// Total bundle size: 500KB + 100KB + 150KB + 80KB = 830KB

Host: React (500KB)
MFE1: Components (100KB)
MFE2: Components (150KB)
MFE3: Components (80KB)
```

## How to Share Dependencies

### Basic Configuration
```javascript
// vite.config.js - Host App
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    federation({
      name: 'host',
      remotes: {
        products: 'products@http://localhost:3001/assets/remoteEntry.js'
      },
      shared: {
        react: {
          singleton: true,  // Only one React instance
          eager: true       // Load immediately
        },
        'react-dom': {
          singleton: true,
          eager: true
        }
      }
    })
  ]
})

// vite.config.js - Remote App
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    federation({
      name: 'products',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/ProductList'
      },
      shared: {
        react: {
          singleton: true,
          eager: true
        },
        'react-dom': {
          singleton: true,
          eager: true
        }
      }
    })
  ]
})
```

## Key Options

### Singleton
```javascript
shared: {
  react: {
    singleton: true  // ✅ Only one instance (recommended)
  },
  lodash: {
    singleton: false // ❌ Allow multiple versions
  }
}
```

### Eager Loading
```javascript
shared: {
  react: {
    eager: true   // ✅ Load with host app immediately
  },
  'react-query': {
    eager: false  // ❌ Load when first MFE needs it
  }
}
```

### Version Control
```javascript
shared: {
  react: {
    singleton: true,
    requiredVersion: '^18.0.0',  // Accept compatible versions
    version: '18.2.0'           // Preferred version
  }
}
```

## Interview Questions & Answers

**Q: Why share dependencies in micro frontends?**
**A:** "Sharing dependencies prevents duplication of libraries like React across multiple micro frontends, reducing total bundle size and improving performance."

**Q: What does 'singleton: true' mean?**
**A:** "It ensures only one instance of the library exists across all micro frontends. Multiple React instances can cause errors and waste memory."

**Q: What's the difference between eager and lazy loading?**
**A:** "Eager loading loads the dependency immediately with the host app. Lazy loading loads it only when the first micro frontend that needs it is loaded."

**Q: How do you handle version conflicts?**
**A:** "Use requiredVersion to specify acceptable version ranges, and ensure all micro frontends use compatible versions of shared dependencies."

**Q: What happens if you don't share dependencies?**
**A:** "Each micro frontend will bundle its own copy of libraries like React, leading to larger bundle sizes and potential version conflicts."

## Summary
- **Singleton**: One instance of libraries (React, etc.)
- **Eager**: Load critical deps immediately
- **Version control**: Prevent conflicts with version ranges
- **Benefit**: Smaller bundles, better performance
- **Risk**: Version conflicts if not managed properly