# Turbopack in Next.js

## Simple Definition
Turbopack is a super-fast bundler built by Vercel specifically for Next.js development. It's written in Rust and much faster than traditional bundlers.

## Why Turbopack?

### Performance Comparison
```javascript
// Traditional bundlers (Webpack):
// Cold start: 15-30 seconds
// HMR updates: 500-2000ms
// Memory usage: High

// Turbopack:
// Cold start: 1-3 seconds (10x faster)
// HMR updates: 50ms (700x faster)
// Memory usage: 75% less
```

## How to Use in Next.js

### Enable Turbopack
```bash
# Start dev server with Turbopack
npx next dev --turbo

# Or add to package.json scripts
{
  "scripts": {
    "dev": "next dev --turbo"
  }
}
```

### Basic Configuration
```javascript
// next.config.js
module.exports = {
  experimental: {
    turbo: {
      // Custom rules and aliases
      rules: {
        '*.svg': {
          loaders: ['@svgr/webpack'],
          as: '*.js'
        }
      }
    }
  }
}
```

## Key Features

### Fast Refresh
```javascript
// Instant component updates
// State preservation during edits
// CSS hot reloading
// Error boundaries
```

### Framework Optimized
```javascript
// Built specifically for Next.js:
// - App Router support
// - Server Components
// - API Routes
// - Middleware
// - Image optimization
```

## When to Use Turbopack

### ✅ Good for Next.js projects
```javascript
// Large Next.js applications
// Teams focused on development speed
// Projects with frequent hot reloads
```

### ❌ Not for other frameworks
```javascript
// Turbopack is Next.js only
// For React/Vue projects, use Vite instead
// For Angular, use Angular CLI
```

## Current Status

### Beta Features
```javascript
// Still in development (2024)
// Production builds use Webpack
// Development only for now
// Limited plugin ecosystem
```

## Interview Questions & Answers

**Q: What is Turbopack?**
**A:** "Turbopack is a high-performance bundler built by Vercel specifically for Next.js. It's written in Rust and provides extremely fast development builds and hot reloads."

**Q: How does Turbopack compare to Webpack?**
**A:** "Turbopack is 10x faster for cold starts and 700x faster for HMR updates. It uses less memory and is optimized specifically for Next.js development."

**Q: Is Turbopack production-ready?**
**A:** "Currently, Turbopack is used for development only. Production builds in Next.js still use Webpack, but future versions may support production builds."

**Q: When should you use Turbopack?**
**A:** "Use Turbopack for Next.js development when you need extremely fast build times and hot reloads. It's perfect for large Next.js applications."

**Q: What's the trade-off with Turbopack?**
**A:** "It's currently in beta, has a smaller plugin ecosystem, and only works with Next.js. For other frameworks, Vite is a better choice."

## Summary
- **Turbopack**: Rust-based bundler for Next.js
- **Performance**: 10x faster dev builds
- **Usage**: `next dev --turbo`
- **Status**: Beta, dev-only currently
- **Alternative**: Vite for non-Next.js projects