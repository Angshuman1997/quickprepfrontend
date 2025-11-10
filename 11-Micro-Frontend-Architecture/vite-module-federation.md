# Vite Module Federation

## Simple Explanation
Module Federation allows different JavaScript apps to share code at runtime. One app can use components from another app without rebuilding everything.

## How It Works

### Basic Idea
```javascript
// App A exposes a component
const AppA = {
  exposes: {
    './Button': './src/Button'  // Share this component
  }
};

// App B consumes the component
const AppB = {
  remotes: {
    appA: 'appA@http://localhost:3001/assets/remoteEntry.js'
  }
};

// App B can now use App A's Button
import Button from 'appA/Button';
```

## Key Concepts

### Host vs Remote
- **Host**: Main app that uses other apps' code
- **Remote**: App that shares its code with others

### RemoteEntry.js
- **What it is**: A manifest file that lists available modules
- **Purpose**: Tells other apps what code is available to use

## Simple Configuration

### Remote App (Shares Code)
```javascript
// vite.config.js
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    federation({
      name: 'productApp',
      filename: 'remoteEntry.js',  // Creates this file
      exposes: {
        './ProductCard': './src/ProductCard',
        './ProductList': './src/ProductList'
      },
      shared: ['react', 'react-dom']  // Share these libraries
    })
  ]
})
```

### Host App (Uses Shared Code)
```javascript
// vite.config.js
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    federation({
      name: 'mainApp',
      remotes: {
        products: 'productApp@http://localhost:3001/assets/remoteEntry.js'
      },
      shared: ['react', 'react-dom']
    })
  ]
})
```

## Usage Example

```javascript
// In the host app
import React, { Suspense } from 'react';

const ProductCard = React.lazy(() => import('products/ProductCard'));
const ProductList = React.lazy(() => import('products/ProductList'));

function App() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <ProductList />
      </Suspense>
    </div>
  );
}
```

## Benefits

### 1. **Independent Deployment**
```javascript
// Teams can deploy separately
// Product team deploys: Only affects product features
// Cart team deploys: Only affects cart features
// No need to redeploy entire app
```

### 2. **Shared Dependencies**
```javascript
// React is loaded only once, even with multiple apps
// Saves bandwidth and memory
shared: {
  react: { singleton: true },  // One React instance only
  lodash: { singleton: true }
}
```

### 3. **Different Technologies**
```javascript
// Different teams can use different frameworks
const Teams = {
  Search: 'React',
  Cart: 'Vue.js',
  Checkout: 'Angular'
};
```

## Interview Questions & Answers

**Q: What is Vite Module Federation?**
**A:** "It's a Vite plugin that allows JavaScript applications to share code at runtime. Apps can expose modules that other apps can consume without rebuilding."

**Q: What's the difference between host and remote?**
**A:** "Host applications consume code from remote applications. Remote applications expose code that can be consumed by hosts."

**Q: How does it work?**
**A:** "Remote apps create a remoteEntry.js file that lists available modules. Host apps load this file and can import the exposed modules dynamically."

**Q: What's the benefit of shared dependencies?**
**A:** "Libraries like React are loaded only once instead of being duplicated in every app, reducing bundle size and improving performance."

**Q: When would you use Module Federation?**
**A:** "When building micro frontends where different teams need to develop and deploy features independently while sharing some common code."

## Summary
- **Module Federation** = Apps sharing code at runtime
- **Host** = Consumer app, **Remote** = Provider app
- **Benefits**: Independent deployment, shared dependencies, tech flexibility
- **Key file**: remoteEntry.js (manifest of available modules)