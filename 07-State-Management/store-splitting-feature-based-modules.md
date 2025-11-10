# How to split Redux store by features?

## Question
How to split Redux store by features?

## Answer
Split store by features instead of one big reducer. Each feature has its own slice, actions, and state.

## Bad: Monolithic Store
```javascript
// One huge reducer with everything mixed
const rootReducer = combineReducers({
  users: usersReducer,
  products: productsReducer,
  cart: cartReducer,
  auth: authReducer
  // All in one place - hard to maintain!
});
```

## Good: Feature-Based Modules
```javascript
// features/users/usersSlice.js
const usersSlice = createSlice({
  name: 'users',
  initialState: { data: [], loading: false },
  reducers: {
    setUsers: (state, action) => { state.data = action.payload; },
    setLoading: (state, action) => { state.loading = action.payload; }
  }
});

// features/products/productsSlice.js
const productsSlice = createSlice({
  name: 'products',
  initialState: { items: [], loading: false },
  reducers: {
    setProducts: (state, action) => { state.items = action.payload; }
  }
});

// store.js
const store = configureStore({
  reducer: {
    users: usersSlice.reducer,
    products: productsSlice.reducer,
  }
});
```

## Benefits

✅ **Easier to maintain** - each feature in own file  
✅ **Better code organization** - related logic together  
✅ **Team collaboration** - different developers work on different features  
✅ **Clear separation** - no conflicts between features  
✅ **Easier testing** - test features independently  

## Interview Q&A

**Q: Why split Redux store by features?**

A: Makes code more maintainable. Each feature has its own slice, reducer, and actions in separate files.

**Q: How do you combine feature slices?**

A: Pass them as an object to configureStore's reducer option: { users: usersSlice.reducer, products: productsSlice.reducer }

**Q: What's the benefit for teams?**

A: Different developers can work on different features without merge conflicts or stepping on each other's code.

**Problems with monolithic stores:**
- **Hard to maintain**: One huge file with hundreds of lines
- **Tight coupling**: Features depend on each other's state structure
- **Difficult testing**: Large reducer functions are hard to test
- **Merge conflicts**: Multiple developers working on the same file
- **Poor code organization**: Related logic scattered throughout

### 2. **Benefits of Store Splitting**

- **Separation of concerns**: Each feature manages its own state
- **Easier testing**: Smaller, focused reducers
- **Better maintainability**: Changes isolated to specific features
- **Parallel development**: Teams can work on different features
- **Code reusability**: Feature modules can be reused
- **Clear boundaries**: Well-defined interfaces between features

## Feature-Based Module Structure

### 1. **Basic Feature Module**

```
src/
├── store/
│   ├── index.ts                 # Store configuration
│   ├── features/
│   │   ├── users/
│   │   │   ├── usersSlice.ts    # User state logic
│   │   │   ├── usersAPI.ts      # User API calls
│   │   │   ├── usersSelectors.ts # User selectors
│   │   │   └── index.ts         # Feature exports
│   │   ├── products/
│   │   │   ├── productsSlice.ts
│   │   │   ├── productsAPI.ts
│   │   │   └── index.ts
│   │   └── cart/
│   │       ├── cartSlice.ts
│   │       └── index.ts
│   └── ui/
│       ├── uiSlice.ts
│       └── index.ts
```

**Example feature module:**
```typescript
// features/users/usersSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchUsers = createAsyncThunk(
    'users/fetchUsers',
    async () => {
        const response = await fetch('/api/users');
        return response.json();
    }
);

const usersSlice = createSlice({
    name: 'users',
    initialState: {
        users: [],
        loading: false,
        error: null,
        currentUser: null,
    },
    reducers: {
        setCurrentUser: (state, action) => {
            state.currentUser = action.payload;
        },
        clearError: (state) => {
            state.error = null;
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(fetchUsers.pending, (state) => {
                state.loading = true;
                state.error = null;
            })
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.loading = false;
                state.users = action.payload;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload;
            });
    },
});

export const { setCurrentUser, clearError } = usersSlice.actions;
export default usersSlice.reducer;
```

```typescript
// features/users/index.ts
export { default as usersReducer } from './usersSlice';
export * from './usersSlice';
export * from './usersSelectors';
export * from './usersAPI';
```

### 2. **Store Configuration**

```typescript
// store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import { usersReducer } from '../features/users';
import { productsReducer } from '../features/products';
import { cartReducer } from '../features/cart';
import { uiReducer } from '../features/ui';

export const store = configureStore({
    reducer: {
        users: usersReducer,
        products: productsReducer,
        cart: cartReducer,
        ui: uiReducer,
    },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

## Advanced Splitting Patterns

### 1. **Domain-Driven Design (DDD) Approach**

**Organize by business domains:**
```
features/
├── auth/           # Authentication domain
│   ├── authSlice.ts
│   ├── loginAPI.ts
│   └── index.ts
├── catalog/        # Product catalog domain
│   ├── productsSlice.ts
│   ├── categoriesSlice.ts
│   └── index.ts
├── ordering/       # Order management domain
│   ├── cartSlice.ts
│   ├── checkoutSlice.ts
│   └── index.ts
├── inventory/      # Inventory management
│   ├── stockSlice.ts
│   └── index.ts
```

**Domain boundaries:**
```typescript
// features/catalog/productsSlice.ts
import { createSlice } from '@reduxjs/toolkit';

const productsSlice = createSlice({
    name: 'products',
    initialState: {
        items: [],
        categories: [],
        filters: {},
        loading: false,
    },
    reducers: {
        // Only catalog-related actions
        setProducts: (state, action) => {
            state.items = action.payload;
        },
        setFilters: (state, action) => {
            state.filters = action.payload;
        },
    },
});
```

### 2. **Cross-Feature Communication**

**Method 1: Selectors from other features**
```typescript
// features/cart/cartSlice.ts
import { createSlice } from '@reduxjs/toolkit';
import { selectProductById } from '../catalog/productsSelectors';

const cartSlice = createSlice({
    name: 'cart',
    initialState: {
        items: [],
        total: 0,
    },
    reducers: {
        addToCart: (state, action) => {
            const { productId, quantity } = action.payload;
            // Cart logic doesn't need product details
            // Use selectors in components for enriched data
        },
    },
});

// In components, combine data from multiple features
function CartItem({ item }) {
    const product = useSelector(state => selectProductById(state, item.productId));
    const cartItem = useSelector(state => selectCartItemById(state, item.id));

    return (
        <div>
            {product?.name} - Quantity: {cartItem?.quantity}
        </div>
    );
}
```

**Method 2: Thunks for cross-feature coordination**
```typescript
// features/ordering/orderThunks.ts
import { createAsyncThunk } from '@reduxjs/toolkit';

export const placeOrder = createAsyncThunk(
    'orders/placeOrder',
    async (_, { dispatch, getState, rejectWithValue }) => {
        const state = getState();

        // Get data from multiple features
        const cartItems = state.cart.items;
        const user = state.auth.currentUser;
        const products = state.catalog.products;

        // Validate order across features
        if (!user) {
            return rejectWithValue('User not authenticated');
        }

        if (cartItems.length === 0) {
            return rejectWithValue('Cart is empty');
        }

        // Create order
        const orderData = {
            userId: user.id,
            items: cartItems.map(item => ({
                productId: item.productId,
                quantity: item.quantity,
                price: products.find(p => p.id === item.productId)?.price,
            })),
        };

        const response = await fetch('/api/orders', {
            method: 'POST',
            body: JSON.stringify(orderData),
        });

        const order = await response.json();

        // Update multiple features
        dispatch(clearCart());
        dispatch(addOrder(order));

        return order;
    }
);
```

### 3. **Shared/Common State**

**Approach 1: Separate common feature**
```
features/
├── common/
│   ├── loadingSlice.ts    # Global loading states
│   ├── notificationsSlice.ts
│   ├── uiSlice.ts         # Global UI state
│   └── index.ts
```

```typescript
// features/common/loadingSlice.ts
import { createSlice } from '@reduxjs/toolkit';

const loadingSlice = createSlice({
    name: 'loading',
    initialState: {
        globalLoading: false,
        loadings: {}, // Feature-specific loading
    },
    reducers: {
        setGlobalLoading: (state, action) => {
            state.globalLoading = action.payload;
        },
        setFeatureLoading: (state, action) => {
            const { feature, loading } = action.payload;
            state.loadings[feature] = loading;
        },
    },
});

export const { setGlobalLoading, setFeatureLoading } = loadingSlice.actions;
```

**Approach 2: Middleware for cross-cutting concerns**
```typescript
// store/middleware/loadingMiddleware.ts
export const loadingMiddleware = (store) => (next) => (action) => {
    // Automatically track loading states
    if (action.type?.endsWith('/pending')) {
        const feature = action.type.split('/')[0];
        store.dispatch(setFeatureLoading({ feature, loading: true }));
    }

    if (action.type?.endsWith('/fulfilled') || action.type?.endsWith('/rejected')) {
        const feature = action.type.split('/')[0];
        store.dispatch(setFeatureLoading({ feature, loading: false }));
    }

    return next(action);
};
```

## Code Splitting and Dynamic Imports

### 1. **Dynamic Store Injection**

```typescript
// store/dynamicStore.ts
import { combineReducers } from '@reduxjs/toolkit';

const staticReducers = {
    common: commonReducer,
};

let asyncReducers = {};

export function injectReducer(key: string, reducer: any) {
    if (!asyncReducers[key]) {
        asyncReducers[key] = reducer;
        store.replaceReducer(
            combineReducers({
                ...staticReducers,
                ...asyncReducers,
            })
        );
    }
}

// Lazy load features
const AdminPage = lazy(() => import('../features/admin'));

function AdminRoute() {
    useEffect(() => {
        // Inject admin reducer when component mounts
        import('../features/admin/adminSlice').then(({ adminReducer }) => {
            injectReducer('admin', adminReducer);
        });
    }, []);

    return <AdminPage />;
}
```

### 2. **Feature Flags and Conditional Loading**

```typescript
// store/featureFlags.ts
export const FEATURE_FLAGS = {
    ADVANCED_SEARCH: process.env.REACT_APP_ADVANCED_SEARCH === 'true',
    ANALYTICS: process.env.REACT_APP_ANALYTICS === 'true',
};

// Dynamic reducer loading based on features
const getReducers = () => {
    const reducers = {
        common: commonReducer,
        auth: authReducer,
    };

    if (FEATURE_FLAGS.ADVANCED_SEARCH) {
        // Lazy load search reducer
        const searchModule = require('../features/search');
        reducers.search = searchModule.searchReducer;
    }

    if (FEATURE_FLAGS.ANALYTICS) {
        const analyticsModule = require('../features/analytics');
        reducers.analytics = analyticsModule.analyticsReducer;
    }

    return reducers;
};
```

## Testing Feature Modules

### 1. **Isolated Feature Testing**

```typescript
// features/users/usersSlice.test.ts
import usersReducer, { fetchUsers, setCurrentUser } from './usersSlice';

describe('users slice', () => {
    const initialState = {
        users: [],
        loading: false,
        error: null,
        currentUser: null,
    };

    it('should handle setCurrentUser', () => {
        const user = { id: 1, name: 'John' };
        const action = setCurrentUser(user);
        const result = usersReducer(initialState, action);

        expect(result.currentUser).toEqual(user);
    });

    it('should handle fetchUsers.pending', () => {
        const action = fetchUsers.pending();
        const result = usersReducer(initialState, action);

        expect(result.loading).toBe(true);
        expect(result.error).toBe(null);
    });
});
```

### 2. **Integration Testing**

```typescript
// features/cart/cartSlice.integration.test.ts
import { configureStore } from '@reduxjs/toolkit';
import cartReducer, { addToCart } from './cartSlice';
import { usersReducer } from '../users';

describe('cart integration', () => {
    let store: any;

    beforeEach(() => {
        store = configureStore({
            reducer: {
                cart: cartReducer,
                users: usersReducer,
            },
        });
    });

    it('should add item to cart for authenticated user', () => {
        // Set up user state
        store.dispatch(setCurrentUser({ id: 1, name: 'John' }));

        // Add item to cart
        store.dispatch(addToCart({ productId: 1, quantity: 2 }));

        const state = store.getState();
        expect(state.cart.items).toHaveLength(1);
        expect(state.cart.items[0].productId).toBe(1);
    });
});
```

## Best Practices

### 1. **Feature Module Guidelines**

```typescript
// ✅ Good: Clear feature boundaries
features/
├── users/
│   ├── usersSlice.ts      # State logic
│   ├── usersAPI.ts        # API calls
│   ├── usersSelectors.ts  # Computed values
│   ├── usersTypes.ts      # TypeScript types
│   └── index.ts           # Public API

// ❌ Avoid: Mixed concerns
features/
├── userManagement.ts      # Everything in one file
```

### 2. **State Shape Consistency**

```typescript
// ✅ Consistent loading/error patterns across features
interface FeatureState {
    data: any;
    loading: boolean;
    error: string | null;
}

// ✅ Consistent action naming
// fetchUsers, createUser, updateUser, deleteUser
// setUsers, clearUsersError
```

### 3. **Selector Organization**

```typescript
// features/users/usersSelectors.ts
import { createSelector } from '@reduxjs/toolkit';

export const selectUsersState = (state: RootState) => state.users;

export const selectAllUsers = createSelector(
    selectUsersState,
    (users) => users.users
);

export const selectUserById = createSelector(
    [selectAllUsers, (state, userId) => userId],
    (users, userId) => users.find(user => user.id === userId)
);

export const selectUsersLoading = createSelector(
    selectUsersState,
    (users) => users.loading
);

export const selectUsersError = createSelector(
    selectUsersState,
    (users) => users.error
);
```

### 4. **Cross-Feature Dependencies**

```typescript
// ✅ Good: Use selectors for cross-feature data access
function CartTotal() {
    const cartItems = useSelector(selectCartItems);
    const products = useSelector(selectAllProducts);

    const total = useMemo(() => {
        return cartItems.reduce((sum, item) => {
            const product = products.find(p => p.id === item.productId);
            return sum + (product?.price || 0) * item.quantity;
        }, 0);
    }, [cartItems, products]);

    return <div>Total: ${total.toFixed(2)}</div>;
}

// ❌ Avoid: Direct state access across features
// Don't do: state.products.items in cart selectors
```

## Common Patterns and Anti-Patterns

### 1. **Entity Management**

```typescript
// ✅ Good: Use normalized state for entities
import { createEntityAdapter } from '@reduxjs/toolkit';

const usersAdapter = createEntityAdapter();

const usersSlice = createSlice({
    name: 'users',
    initialState: usersAdapter.getInitialState({
        loading: false,
        error: null,
    }),
    reducers: {
        usersReceived: usersAdapter.setAll,
        userUpdated: usersAdapter.updateOne,
    },
});

export const {
    selectAll: selectAllUsers,
    selectById: selectUserById,
} = usersAdapter.getSelectors((state: RootState) => state.users);
```

### 2. **Avoid God Features**

```typescript
// ❌ Bad: One feature handles everything
const appSlice = createSlice({
    name: 'app',
    initialState: {
        users: [],
        products: [],
        cart: [],
        orders: [],
        notifications: [],
        ui: {},
    },
    // 200+ lines of mixed logic
});

// ✅ Good: Separate concerns
// usersSlice, productsSlice, cartSlice, ordersSlice, etc.
```

### 3. **Feature Communication**

```typescript
// ✅ Good: Event-driven communication
// Feature A dispatches actions that Feature B listens to
dispatch(userLoggedIn(user));
// cartSlice can react to USER_LOGGED_IN action

// ✅ Good: Selector composition
const selectCartTotal = createSelector(
    selectCartItems,
    selectProductPrices,
    (items, prices) => /* calculate total */
);
```

## Migration Strategies

### 1. **Incremental Migration**

```typescript
// Step 1: Extract one feature at a time
// Before: monolithic reducer
function appReducer(state, action) { /* 500 lines */ }

// After: Extract users feature
const usersSlice = createSlice({
    name: 'users',
    initialState: { /* user state */ },
    reducers: { /* user actions */ },
});

// Step 2: Update store
const store = configureStore({
    reducer: {
        users: usersSlice.reducer,
        // Keep monolithic reducer temporarily
        app: appReducer,
    },
});

// Step 3: Migrate components gradually
// Old: useSelector(state => state.app.users)
// New: useSelector(state => state.users.users)
```

### 2. **Parallel Migration**

```typescript
// Run both old and new systems in parallel
const store = configureStore({
    reducer: {
        // New modular approach
        users: usersSlice.reducer,
        products: productsSlice.reducer,

        // Legacy monolithic state (gradually remove)
        legacy: legacyReducer,
    },
});

// Migration utilities
export const selectUsers = createSelector(
    // Try new state first, fall back to legacy
    (state) => state.users?.users || state.legacy?.users || [],
    (users) => users
);
```

## Performance Considerations

### 1. **Selector Memoization**

```typescript
// ✅ Good: Memoized selectors prevent unnecessary re-renders
export const selectFilteredUsers = createSelector(
    [selectAllUsers, selectUserFilters],
    (users, filters) => {
        return users.filter(user => {
            // Expensive filtering logic
            return matchesFilters(user, filters);
        });
    }
);
```

### 2. **Lazy Loading**

```typescript
// ✅ Good: Load features on demand
const AdminPanel = lazy(() =>
    import('../features/admin').then(module => ({
        default: module.AdminPanel,
    }))
);

// Only load admin reducer when needed
function AdminRoute() {
    useEffect(() => {
        import('../features/admin/adminSlice').then(({ adminReducer }) => {
            injectReducer('admin', adminReducer);
        });
    }, []);

    return <AdminPanel />;
}
```

## Common Interview Questions

### Q: Why should you split Redux store by features?

**A:** Feature-based splitting improves maintainability by separating concerns, makes testing easier with smaller focused modules, enables parallel development, and prevents merge conflicts. Each feature manages its own state and logic independently.

### Q: How do features communicate with each other?

**A:** Features communicate through selectors (reading other features' state), shared actions that multiple features can listen to, or thunks that coordinate multi-feature operations. Avoid direct state mutations across feature boundaries.

### Q: What's the difference between domain-driven and feature-based splitting?

**A:** Domain-driven design organizes by business domains (auth, catalog, ordering), while feature-based organizes by user-facing features (login, product-list, checkout). Domain-driven is more business-focused, feature-based is more UI-focused.

### Q: How do you handle shared state between features?

**A:** For truly shared state, create a separate "common" or "shared" feature. For derived state, use selectors that combine data from multiple features. Avoid duplicating state across features.

## Summary

**Store Splitting Benefits:**
- **Maintainability**: Smaller, focused modules
- **Testability**: Isolated feature testing
- **Scalability**: Parallel development support
- **Reusability**: Feature modules can be shared
- **Performance**: Better memoization and lazy loading

**Key Patterns:**
1. **Feature Modules**: Group related state, actions, selectors
2. **Cross-Feature Communication**: Use selectors and shared actions
3. **Consistent Structure**: Standard file organization per feature
4. **Lazy Loading**: Dynamic reducer injection for code splitting
5. **Entity Normalization**: Use adapters for related data

**Best Practices:**
- Clear feature boundaries
- Consistent state shapes
- Memoized selectors
- Isolated testing
- Gradual migration path

**Interview Tip:** "Store splitting organizes Redux state by features or domains, making the codebase more maintainable and testable. Each feature has its own slice with state, actions, and selectors, communicating through selectors and shared actions rather than direct state access."