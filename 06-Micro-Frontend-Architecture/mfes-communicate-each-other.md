# How do MFEs communicate with each other?

## Question
How do MFEs communicate with each other?

# How do MFEs communicate with each other?

## Question
How do MFEs communicate with each other?

## Answer

Micro frontends (MFEs) need to communicate with each other despite being deployed as separate applications. Here are the main communication patterns:

## 1. Custom Events (Recommended)

### Publisher MFE
```javascript
// src/utils/eventBus.js
class EventBus {
  constructor() {
    this.events = {};
  }

  subscribe(event, callback) {
    if (!this.events[event]) {
      this.events[event] = [];
    }
    this.events[event].push(callback);
  }

  publish(event, data) {
    if (this.events[event]) {
      this.events[event].forEach(callback => callback(data));
    }
  }

  unsubscribe(event, callback) {
    if (this.events[event]) {
      this.events[event] = this.events[event].filter(cb => cb !== callback);
    }
  }
}

export const eventBus = new EventBus();
```

```javascript
// src/components/ProductList.js
import React, { useEffect } from 'react';
import { eventBus } from '../utils/eventBus';

const ProductList = () => {
  const handleProductSelect = (product) => {
    // Publish event to other MFEs
    eventBus.publish('PRODUCT_SELECTED', product);
  };

  return (
    <div>
      {products.map(product => (
        <div key={product.id} onClick={() => handleProductSelect(product)}>
          {product.name}
        </div>
      ))}
    </div>
  );
};
```

### Subscriber MFE
```javascript
// src/components/ProductDetail.js
import React, { useEffect, useState } from 'react';
import { eventBus } from '../utils/eventBus';

const ProductDetail = () => {
  const [selectedProduct, setSelectedProduct] = useState(null);

  useEffect(() => {
    // Subscribe to product selection events
    const handleProductSelected = (product) => {
      setSelectedProduct(product);
    };

    eventBus.subscribe('PRODUCT_SELECTED', handleProductSelected);

    // Cleanup on unmount
    return () => {
      eventBus.unsubscribe('PRODUCT_SELECTED', handleProductSelected);
    };
  }, []);

  return (
    <div>
      {selectedProduct ? (
        <div>
          <h2>{selectedProduct.name}</h2>
          <p>{selectedProduct.description}</p>
        </div>
      ) : (
        <p>Select a product to view details</p>
      )}
    </div>
  );
};
```

## 2. Shared State Management (Redux/Zustand)

### Shared Store Configuration
```javascript
// shared-store/src/store.js
import { configureStore, createSlice } from '@reduxjs/toolkit';

const productSlice = createSlice({
  name: 'product',
  initialState: {
    selectedProduct: null,
    products: []
  },
  reducers: {
    selectProduct: (state, action) => {
      state.selectedProduct = action.payload;
    },
    setProducts: (state, action) => {
      state.products = action.payload;
    }
  }
});

export const { selectProduct, setProducts } = productSlice.actions;

export const store = configureStore({
  reducer: {
    product: productSlice.reducer
  }
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Using Shared Store in MFEs
```javascript
// Product List MFE
import React from 'react';
import { useDispatch } from 'react-redux';
import { selectProduct } from 'shared-store';

const ProductList = () => {
  const dispatch = useDispatch();

  const handleSelect = (product) => {
    dispatch(selectProduct(product));
  };

  return (
    <div>
      {products.map(product => (
        <button key={product.id} onClick={() => handleSelect(product)}>
          {product.name}
        </button>
      ))}
    </div>
  );
};
```

```javascript
// Product Detail MFE
import React from 'react';
import { useSelector } from 'react-redux';

const ProductDetail = () => {
  const selectedProduct = useSelector(state => state.product.selectedProduct);

  return (
    <div>
      {selectedProduct ? (
        <div>
          <h2>{selectedProduct.name}</h2>
          <p>{selectedProduct.description}</p>
        </div>
      ) : (
        <p>No product selected</p>
      )}
    </div>
  );
};
```

## 3. URL/State Synchronization

### Using React Router State
```javascript
// src/components/ProductList.js
import { useNavigate } from 'react-router-dom';

const ProductList = ({ products }) => {
  const navigate = useNavigate();

  const handleProductClick = (product) => {
    // Navigate with state
    navigate(`/product/${product.id}`, { 
      state: { product, source: 'product-list' } 
    });
  };

  return (
    <div>
      {products.map(product => (
        <div key={product.id} onClick={() => handleProductClick(product)}>
          {product.name}
        </div>
      ))}
    </div>
  );
};
```

```javascript
// src/components/ProductDetail.js
import { useLocation } from 'react-router-dom';

const ProductDetail = () => {
  const location = useLocation();
  const { product, source } = location.state || {};

  return (
    <div>
      {product ? (
        <div>
          <h3>From: {source}</h3>
          <h2>{product.name}</h2>
          <p>{product.description}</p>
        </div>
      ) : (
        <p>Product not found</p>
      )}
    </div>
  );
};
```

## 4. API-Based Communication

### Shared API Service
```javascript
// shared-api/src/api.js
class ApiService {
  constructor() {
    this.baseURL = process.env.REACT_APP_API_URL;
    this.listeners = {};
  }

  // Event system for cross-MFE communication
  on(event, callback) {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event].push(callback);
  }

  emit(event, data) {
    if (this.listeners[event]) {
      this.listeners[event].forEach(callback => callback(data));
    }
  }

  async getProducts() {
    const response = await fetch(`${this.baseURL}/products`);
    const products = await response.json();
    
    // Notify other MFEs
    this.emit('PRODUCTS_LOADED', products);
    
    return products;
  }

  async createProduct(product) {
    const response = await fetch(`${this.baseURL}/products`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(product)
    });
    
    const newProduct = await response.json();
    
    // Notify other MFEs about new product
    this.emit('PRODUCT_CREATED', newProduct);
    
    return newProduct;
  }
}

export const apiService = new ApiService();
```

## 5. Window.postMessage (Cross-Origin)

### Sender MFE
```javascript
// src/utils/messaging.js
export const sendMessage = (targetOrigin, message) => {
  window.parent.postMessage({
    type: 'MFE_MESSAGE',
    payload: message,
    source: 'product-mfe'
  }, targetOrigin);
};

// Usage in component
const handleAction = () => {
  sendMessage('http://localhost:3000', {
    action: 'PRODUCT_UPDATED',
    productId: selectedProduct.id
  });
};
```

### Receiver MFE
```javascript
// src/utils/messaging.js
export const setupMessageListener = (callback) => {
  window.addEventListener('message', (event) => {
    // Validate origin for security
    if (event.origin !== 'http://localhost:3001') return;
    
    if (event.data.type === 'MFE_MESSAGE') {
      callback(event.data.payload);
    }
  });
};

// Usage in component
useEffect(() => {
  setupMessageListener((message) => {
    if (message.action === 'PRODUCT_UPDATED') {
      // Refresh product data
      fetchUpdatedProduct(message.productId);
    }
  });
}, []);
```

## Communication Patterns Comparison

| Method | Pros | Cons | Use Case |
|--------|------|------|----------|
| **Custom Events** | Simple, decoupled, real-time | No persistence, same-origin only | Intra-app communication |
| **Shared State** | Centralized, predictable, debuggable | Complex setup, single source of truth | Complex state management |
| **URL/State** | Browser-native, bookmarkable | Limited data size, navigation required | Page-level communication |
| **API** | Scalable, async, cross-platform | Network dependent, latency | Data synchronization |
| **postMessage** | Cross-origin, secure | Complex setup, message validation needed | Cross-domain MFEs |

## Best Practices

1. **Choose based on coupling needs**: Loose coupling = Events, Tight coupling = Shared state
2. **Security**: Always validate message origins and sanitize data
3. **Error handling**: Implement fallback mechanisms for failed communications
4. **Performance**: Debounce frequent events, use efficient serialization
5. **Testing**: Mock communication layers for isolated MFE testing

## Interview Tips
- **Custom events**: Most common and recommended approach
- **Shared state**: Use when MFEs need to share complex application state
- **URL params**: Good for bookmarkable cross-MFE navigation
- **postMessage**: Required for cross-origin deployments
- **Security**: Always validate origins and sanitize messages
- **Performance**: Consider message frequency and payload size