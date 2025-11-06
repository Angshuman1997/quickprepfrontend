# What is a host and what is a remote?

## Question
What is a host and what is a remote?

# What is a host and what is a remote?

## Question
What is a host and what is a remote?

## Answer

In Webpack Module Federation, **host** and **remote** are two key roles that applications can play in a micro frontend architecture:

### Host Application
- **Definition**: The host is the main application that consumes and orchestrates remote modules
- **Responsibilities**: 
  - Loads and initializes remote modules
  - Provides the main application shell/layout
  - Manages routing and navigation
  - Handles shared dependencies
- **Configuration**: Uses `remotes` in webpack config to specify which remote applications to consume

### Remote Application
- **Definition**: Remote applications expose modules that can be consumed by host applications
- **Responsibilities**:
  - Exposes specific modules/components for consumption
  - Manages its own build process and dependencies
  - Can be developed and deployed independently
- **Configuration**: Uses `exposes` in webpack config to specify which modules to share

## Practical Example

### Remote Application (Product Service)
```javascript
// webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  // ... other config
  plugins: [
    new ModuleFederationPlugin({
      name: 'product_service',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/components/ProductList',
        './ProductDetail': './src/components/ProductDetail',
        './productApi': './src/api/productApi'
      },
      shared: ['react', 'react-dom', '@reduxjs/toolkit']
    })
  ]
};
```

### Host Application (Main App)
```javascript
// webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin');

module.exports = {
  // ... other config
  plugins: [
    new ModuleFederationPlugin({
      name: 'main_app',
      remotes: {
        productService: 'product_service@http://localhost:3001/remoteEntry.js',
        userService: 'user_service@http://localhost:3002/remoteEntry.js'
      },
      shared: ['react', 'react-dom', '@reduxjs/toolkit']
    })
  ]
};
```

### Host App Usage
```javascript
// src/App.js
import React, { Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

// Lazy load remote components
const ProductList = React.lazy(() => import('productService/ProductList'));
const ProductDetail = React.lazy(() => import('productService/ProductDetail'));
const UserProfile = React.lazy(() => import('userService/UserProfile'));

function App() {
  return (
    <BrowserRouter>
      <div className="app">
        <header>
          <nav>
            <Link to="/products">Products</Link>
            <Link to="/profile">Profile</Link>
          </nav>
        </header>
        
        <main>
          <Suspense fallback={<div>Loading...</div>}>
            <Routes>
              <Route path="/products" element={<ProductList />} />
              <Route path="/products/:id" element={<ProductDetail />} />
              <Route path="/profile" element={<UserProfile />} />
            </Routes>
          </Suspense>
        </main>
      </div>
    </BrowserRouter>
  );
}
```

## Key Differences

| Aspect | Host | Remote |
|--------|------|--------|
| **Purpose** | Consumes modules | Provides modules |
| **Webpack Config** | `remotes: {}` | `exposes: {}` |
| **Entry Point** | `remoteEntry.js` (consumes) | `remoteEntry.js` (provides) |
| **Dependencies** | Can share dependencies | Can share dependencies |
| **Deployment** | Can be deployed independently | Can be deployed independently |
| **Scope** | Application shell | Feature modules |

## Advanced Patterns

### Host as Remote (Hybrid)
```javascript
// webpack.config.js - Host that also exposes modules
new ModuleFederationPlugin({
  name: 'main_app',
  filename: 'remoteEntry.js',
  exposes: {
    './Header': './src/components/Header',
    './Footer': './src/components/Footer'
  },
  remotes: {
    productService: 'product_service@http://localhost:3001/remoteEntry.js'
  },
  shared: ['react', 'react-dom']
})
```

### Multiple Hosts, One Remote
```javascript
// Different host apps consuming same remote
// Host App 1 (Admin Panel)
remotes: {
  productService: 'product_service@http://localhost:3001/remoteEntry.js'
}

// Host App 2 (Customer Portal)  
remotes: {
  productService: 'product_service@http://localhost:3001/remoteEntry.js'
}
```

## Interview Tips
- **Host**: Think of it as the "orchestrator" or "container" app
- **Remote**: Think of it as the "provider" or "supplier" of functionality
- **Real-world analogy**: Host is like a shopping mall, remotes are like individual stores
- **Configuration**: `remotes` for consuming, `exposes` for providing
- **Communication**: Hosts load remotes dynamically at runtime

## Common Interview Questions
- "Can a remote also be a host?" (Yes, hybrid approach)
- "What's the difference between `exposes` and `remotes`?" (exposes shares out, remotes consumes in)
- "How does the host know where to find remotes?" (URL to remoteEntry.js file)