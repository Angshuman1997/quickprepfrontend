# Host vs Remote in Micro Frontends

## Simple Explanation

### Host Application
- **What it is**: The main app that brings everything together
- **What it does**: Loads and displays remote apps, handles navigation
- **Example**: Like the shopping mall that contains all the stores

### Remote Application
- **What it is**: Individual feature apps that provide specific functionality
- **What it does**: Contains the actual features (products, cart, user accounts)
- **Example**: Like individual stores within the mall

## Real-World Example

```javascript
// Shopping Mall (HOST) - Main container
const ShoppingMall = () => (
  <div>
    <Header />           {/* Mall's header */}
    <ProductStore />     {/* Remote: Product listings */}
    <CartStore />        {/* Remote: Shopping cart */}
    <UserStore />        {/* Remote: User accounts */}
  </div>
);

// Product Store (REMOTE) - Independent feature
const ProductStore = () => (
  <div>
    <h2>Products</h2>
    <ProductList />
    <ProductSearch />
  </div>
);
```

## Key Differences

| Feature | Host | Remote |
|---------|------|--------|
| **Role** | Container/Main app | Feature provider |
| **Purpose** | Orchestrates everything | Provides specific functionality |
| **Deployment** | Can deploy independently | Can deploy independently |
| **Example** | Main website shell | Product catalog, User profile |

## Simple Configuration

### Host (Main App)
```javascript
// vite.config.js
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    federation({
      name: 'main_app',
      remotes: {
        products: 'products@http://localhost:3001/assets/remoteEntry.js',
        cart: 'cart@http://localhost:3002/assets/remoteEntry.js'
      }
    })
  ]
})
```

### Remote (Feature App)
```javascript
// vite.config.js
import { defineConfig } from 'vite'
import federation from '@originjs/vite-plugin-federation'

export default defineConfig({
  plugins: [
    federation({
      name: 'products',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductList': './src/ProductList',
        './ProductDetail': './src/ProductDetail'
      }
    })
  ]
})
```

## Interview Questions & Answers

**Q: What's the difference between host and remote?**
**A:** "Host is the main application that loads and orchestrates remote applications. Remote applications provide specific features that the host consumes."

**Q: Can one app be both host and remote?**
**A:** "Yes, this is called a hybrid approach. An app can both consume other remotes and expose its own modules."

**Q: How does the host know where to find remotes?**
**A:** "Through URLs pointing to each remote's remoteEntry.js file, specified in the Vite configuration."

**Q: What's the benefit of this separation?**
**A:** "Teams can develop and deploy their features independently without affecting the main application."

## Summary
- **Host** = Main container app that brings everything together
- **Remote** = Individual feature apps that provide specific functionality
- **Benefit**: Independent development and deployment
- **Communication**: Host loads remotes via URLs to their entry files