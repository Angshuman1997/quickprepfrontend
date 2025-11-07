# What is Tree Shaking?

## Simple Definition
Tree shaking removes unused code from your bundle, making it smaller and faster to load.

## How It Works

### Before Tree Shaking
```javascript
// utils.js - You export 4 functions
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export const multiply = (a, b) => a * b;
export const divide = (a, b) => a / b;

// app.js - You only use 2 functions
import { add, multiply } from './utils.js';
console.log(add(2, 3));
```

### After Tree Shaking
```javascript
// Bundle only contains the 2 used functions
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;
console.log(add(2, 3));
// subtract and divide are removed!
```

## Why It Matters

### Bundle Size Impact
```javascript
// Without tree shaking: 500KB
// With tree shaking: 200KB
// Result: 60% smaller bundle, faster loading
```

## Vite Tree Shaking

### Automatic in Production
```javascript
// vite.config.js - No special config needed!
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()]
  // Tree shaking enabled automatically in build
})
```

### Build Command
```bash
npm run build  # Creates optimized, tree-shaken bundle
```

## ES6 Modules Required

### ✅ Tree Shakable
```javascript
// Named exports
export const helper1 = () => {};
export const helper2 = () => {};

// Usage - only imports what you use
import { helper1 } from './helpers';
```

### ❌ Not Tree Shakable
```javascript
// CommonJS style
const helpers = { helper1: () => {}, helper2: () => {} };
module.exports = helpers;

// Usage - imports entire object
const { helper1 } = require('./helpers');
```

## Side Effects

### Pure Functions (Tree Shakable)
```javascript
export const calculateTotal = (items) => {
  return items.reduce((sum, item) => sum + item.price, 0);
};
```

### Side Effects (Not Tree Shakable)
```javascript
// Modifies global state
export const initApp = () => {
  window.myApp = {};
};

// CSS imports
import './global-styles.css';
```

## Interview Questions & Answers

**Q: What is tree shaking?**
**A:** "Tree shaking is a bundler optimization that removes unused code from your final bundle. It analyzes which exports are actually used and eliminates the rest."

**Q: How does tree shaking work?**
**A:** "Modern bundlers like Vite analyze ES6 import/export statements statically. They can determine which functions are never called and remove them from the bundle."

**Q: Why is tree shaking important?**
**A:** "It reduces bundle size significantly, leading to faster load times and better performance. For large apps, this can mean 30-60% smaller bundles."

**Q: Does tree shaking work with CommonJS?**
**A:** "No, tree shaking requires ES6 modules with static imports/exports. CommonJS require/module.exports can't be analyzed statically."

**Q: When does tree shaking happen?**
**A:** "Only in production builds. Development mode doesn't tree shake to keep rebuilds fast."

**Q: What about CSS?**
**A:** "CSS is considered a side effect, so unused CSS might not be tree shaken unless you configure it specifically."

## Summary
- **Tree Shaking**: Removes unused code from bundles
- **ES6 Modules**: Required for static analysis
- **Production Only**: Development keeps all code for speed
- **Result**: Smaller bundles, faster apps
- **Vite**: Automatic tree shaking in builds