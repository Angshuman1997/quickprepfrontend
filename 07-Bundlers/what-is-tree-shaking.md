# What is tree-shaking?

## Question
What is tree-shaking?

# What is tree-shaking?

## Question
What is tree-shaking?

## Answer

Tree-shaking is a dead code elimination technique used by modern JavaScript bundlers to remove unused code from the final bundle, resulting in smaller bundle sizes and better performance.

## How Tree-Shaking Works

### Before Tree-Shaking
```javascript
// utils.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export const multiply = (a, b) => a * b;
export const divide = (a, b) => a / b;

// app.js
import { add, multiply } from './utils.js';

console.log(add(2, 3)); // Used
// subtract and divide are not used
```

### After Tree-Shaking
```javascript
// Final bundle only contains used functions
const add = (a, b) => a + b;
const multiply = (a, b) => a * b;

console.log(add(2, 3));
```

## Tree-Shaking in Different Bundlers

### Webpack Configuration
```javascript
// webpack.config.js
module.exports = {
  mode: 'production', // Enables tree-shaking
  optimization: {
    usedExports: true, // Mark used exports
    minimize: true, // Remove unused code
  }
};
```

### Rollup Configuration
```javascript
// rollup.config.js
export default {
  input: 'src/index.js',
  output: {
    file: 'dist/bundle.js',
    format: 'es'
  },
  treeshake: true // Explicitly enable
};
```

### Vite Configuration
```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      treeshake: true
    }
  }
};
```

## ES6 Modules vs CommonJS

### Tree-Shakable (ES6)
```javascript
// ✅ Tree-shakable
export const helper1 = () => {};
export const helper2 = () => {};

// Usage
import { helper1 } from './helpers'; // Only helper1 is included
```

### Not Tree-Shakable (CommonJS)
```javascript
// ❌ Not tree-shakable
const helpers = {
  helper1: () => {},
  helper2: () => {}
};

module.exports = helpers;

// Usage
const { helper1 } = require('./helpers'); // Entire helpers object included
```

## Side Effects and Package.json

### Marking Side Effects
```json
// package.json
{
  "name": "my-library",
  "sideEffects": [
    "*.css",
    "*.scss",
    "./src/polyfills.js"
  ]
}
```

```json
// Or mark as side-effect free
{
  "name": "my-library",
  "sideEffects": false
}
```

### Side Effect Examples
```javascript
// ✅ No side effects - tree-shakable
export const pureFunction = (x) => x * 2;

// ❌ Has side effects - not tree-shakable
export const initializeApp = () => {
  // Modifies global state
  window.app = {};
};

// ❌ CSS import - side effect
import './styles.css';
```

## Advanced Tree-Shaking Techniques

### Dynamic Imports
```javascript
// Tree-shaking friendly
const loadModule = async () => {
  if (condition) {
    const { moduleA } = await import('./moduleA'); // Only loaded when needed
    return moduleA();
  } else {
    const { moduleB } = await import('./moduleB'); // Only loaded when needed
    return moduleB();
  }
};
```

### Barrel Exports
```javascript
// utils/index.js
export { add } from './math';
export { format } from './string';
export { fetchData } from './api';

// app.js
import { add, format } from './utils'; // Only add and format are included
// fetchData is tree-shaken away
```

### Conditional Exports
```javascript
// utils.js
export const isDevelopment = process.env.NODE_ENV === 'development';
export const isProduction = process.env.NODE_ENV === 'production';

export const devOnlyFunction = () => {
  if (isDevelopment) {
    console.log('Development mode');
  }
};

// In production build, devOnlyFunction can be tree-shaken if not used
```

## Common Tree-Shaking Issues

### 1. Re-exports
```javascript
// ❌ Problematic re-export
export * from './utils'; // Can't determine what's used

// ✅ Better approach
export { helper1, helper2 } from './utils'; // Explicit exports
```

### 2. Class Properties
```javascript
// ❌ Not tree-shakable
class Utils {
  static helper1 = () => {};
  static helper2 = () => {};
}

// ✅ Tree-shakable
export const helper1 = () => {};
export const helper2 = () => {};
```

### 3. Prototype Methods
```javascript
// ❌ Not tree-shakable
function MyClass() {}
MyClass.prototype.method1 = () => {};
MyClass.prototype.method2 = () => {};

// ✅ Tree-shakable
export class MyClass {
  method1() {}
  method2() {}
}
```

## Measuring Tree-Shaking Effectiveness

### Bundle Analyzer
```javascript
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer');

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false
    })
  ]
};
```

### Build Output Analysis
```bash
# Check bundle size
npm run build
ls -lh dist/

# Analyze with source-map-explorer
npx source-map-explorer dist/bundle.js dist/bundle.js.map
```

## Best Practices

1. **Use ES6 modules**: `import`/`export` instead of `require`/`module.exports`
2. **Avoid side effects**: Mark pure functions and avoid global modifications
3. **Use dynamic imports**: For conditional loading
4. **Configure sideEffects**: In package.json for libraries
5. **Avoid wildcard exports**: Use named exports
6. **Test in production mode**: Tree-shaking only works in production builds

## Interview Tips
- **Dead code elimination**: Removes unused exports from bundle
- **ES6 modules**: Static analysis enables tree-shaking
- **Side effects**: Code that modifies global state can't be tree-shaken
- **Production mode**: Only enabled in production builds
- **Bundle size**: Directly impacts app performance
- **Dynamic imports**: Enable code splitting and lazy loading