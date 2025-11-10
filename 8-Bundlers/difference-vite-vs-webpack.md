# Vite vs Webpack

## Simple Comparison

### Vite (What I Use)
- **Fast dev server**: Starts in 1-3 seconds
- **HMR**: Updates in 50-200ms
- **Simple config**: Just a few lines
- **Modern browsers**: ES2015+ only

### Webpack (Traditional)
- **Slower dev server**: Takes 10-30 seconds
- **HMR**: Updates in 500ms-2 seconds
- **Complex config**: Lots of loaders and plugins
- **All browsers**: Including IE11

## Why Vite is Faster

### Development Mode
```javascript
// Vite: Serve files directly, transform on-demand
// No bundling needed - browser handles ES modules

// Webpack: Bundle everything upfront
// Creates one big file, then serves it
```

### Production Mode
```javascript
// Both create optimized bundles
// Vite uses Rollup, Webpack uses its own optimizer
```

## Simple Vite Config
```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000
  }
})
```

## Simple Webpack Config
```javascript
// webpack.config.js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        use: 'babel-loader'
      }
    ]
  }
};
```

## Interview Questions & Answers

**Q: Why did you choose Vite over Webpack?**
**A:** "Vite provides much faster development experience with instant HMR and quick cold starts. Since I work with modern browsers, Vite's ESM-first approach works perfectly for my projects."

**Q: What's the main difference between Vite and Webpack?**
**A:** "Webpack bundles everything upfront in development, while Vite serves files directly and transforms them on-demand using native ES modules."

**Q: Is Vite production-ready?**
**A:** "Yes! Vite uses Rollup for production builds, creating optimized, tree-shaken bundles just like Webpack."

**Q: When would you still use Webpack?**
**A:** "For legacy browser support (IE11) or very complex build pipelines with many custom loaders that don't have Vite equivalents."

**Q: How does Vite achieve such fast HMR?**
**A:** "Vite only transforms the changed file and its dependencies, not the entire bundle. The browser's native ES module system handles the rest."

## Summary
- **Vite**: Fast dev, simple config, modern browsers
- **Webpack**: Slower dev, complex config, all browsers
- **Both**: Excellent production builds
- **My Choice**: Vite for speed and simplicity