# Zustand vs Redux

## Question
When to use Zustand instead of Redux?

## Answer
Zustand for small/medium apps with less boilerplate. Redux for large apps with complex state management needs.

## Zustand Example
```javascript
import { create } from 'zustand';

const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 }))
}));

function Counter() {
  const { count, increment, decrement } = useStore();
  return (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

## Redux Example
```javascript
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: (state) => { state.value += 1; },
    decrement: (state) => { state.value -= 1; }
  }
});

function Counter() {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();
  return (
    <div>
      <button onClick={() => dispatch(decrement())}>-</button>
      <span>{count}</span>
      <button onClick={() => dispatch(increment())}>+</button>
    </div>
  );
}
```

## When to Use Zustand

✅ **Small to medium apps**  
✅ **Quick prototyping**  
✅ **Less boilerplate code**  
✅ **Simple state management**  
✅ **Smaller bundle size**  

## When to Use Redux

✅ **Large applications**  
✅ **Complex async logic**  
✅ **Many developers**  
✅ **Strict patterns needed**  
✅ **Advanced debugging tools**  

## Interview Q&A

**Q: When would you choose Zustand over Redux?**

A: For small to medium applications where I want less boilerplate and faster development. Zustand has minimal setup.

**Q: What's the main difference between Zustand and Redux?**

A: Zustand uses hooks directly for state management, Redux uses actions/reducers/selectors pattern with more structure.

**Q: Can Zustand scale to large applications?**

A: Yes for medium apps, but Redux is better for very large complex applications with many developers and strict patterns.

### 2. **Key Differences**

| Aspect | Redux | Zustand |
|--------|-------|---------|
| **Boilerplate** | High (RTK reduces it significantly) | Minimal |
| **Bundle Size** | ~2-3kb (RTK) | ~1kb |
| **Learning Curve** | Steep (concepts, patterns) | Gentle |
| **TypeScript Support** | Excellent | Good |
| **Middleware** | Powerful ecosystem | Limited but extensible |
| **DevTools** | Excellent (Redux DevTools) | Basic (can integrate) |
| **Testing** | Well-established patterns | Simple but less conventional |
| **Scalability** | Excellent for large apps | Good for medium apps |
| **Performance** | Excellent with memoization | Excellent with selective subscriptions |

## When to Choose Zustand

### 1. **Small to Medium Applications**

**Choose Zustand when:**
- Team size is small (1-5 developers)
- Application complexity is moderate
- You want to move fast without heavy setup
- The app doesn't need complex async logic patterns

**Example - Simple app state:**
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AppState {
    user: User | null;
    theme: 'light' | 'dark';
    sidebarOpen: boolean;
    login: (user: User) => void;
    logout: () => void;
    toggleTheme: () => void;
    toggleSidebar: () => void;
}

const useAppStore = create<AppState>()(
    persist(
        (set) => ({
            user: null,
            theme: 'light',
            sidebarOpen: true,
            login: (user) => set({ user }),
            logout: () => set({ user: null }),
            toggleTheme: () => set((state) => ({
                theme: state.theme === 'light' ? 'dark' : 'light'
            })),
            toggleSidebar: () => set((state) => ({
                sidebarOpen: !state.sidebarOpen
            })),
        }),
        { name: 'app-storage' }
    )
);

// Usage across components
function Header() {
    const { user, logout, toggleSidebar } = useAppStore();
    // ...
}

function ThemeToggle() {
    const { theme, toggleTheme } = useAppStore();
    // ...
}
```

### 2. **Rapid Prototyping**

**Zustand for quick MVPs:**
```typescript
// Define store in minutes
const useTodoStore = create((set, get) => ({
    todos: [],
    addTodo: (text) => set((state) => ({
        todos: [...state.todos, { id: Date.now(), text, completed: false }]
    })),
    toggleTodo: (id) => set((state) => ({
        todos: state.todos.map(todo =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo
        )
    })),
    clearCompleted: () => set((state) => ({
        todos: state.todos.filter(todo => !todo.completed)
    })),
    // Computed values
    get completedCount() {
        return get().todos.filter(todo => todo.completed).length;
    },
    get totalCount() {
        return get().todos.length;
    },
}));
```

### 3. **Component-Level State That Needs Sharing**

**When multiple components need the same local state:**
```typescript
// Instead of prop drilling or context
const useModalStore = create((set) => ({
    modals: {},
    openModal: (id, data) => set((state) => ({
        modals: { ...state.modals, [id]: { open: true, data } }
    })),
    closeModal: (id) => set((state) => ({
        modals: { ...state.modals, [id]: { open: false } }
    })),
    getModalState: (id) => (state) => state.modals[id],
}));

// Any component can control any modal
function SomeComponent() {
    const openModal = useModalStore((state) => state.openModal);

    return (
        <button onClick={() => openModal('confirm', { message: 'Are you sure?' })}>
            Delete Item
        </button>
    );
}
```

### 4. **Simple Async Operations**

**Zustand handles async naturally:**
```typescript
const useUserStore = create((set, get) => ({
    user: null,
    loading: false,
    error: null,

    fetchUser: async (id) => {
        set({ loading: true, error: null });
        try {
            const response = await fetch(`/api/users/${id}`);
            const user = await response.json();
            set({ user, loading: false });
        } catch (error) {
            set({ error: error.message, loading: false });
        }
    },

    updateUser: async (updates) => {
        const currentUser = get().user;
        if (!currentUser) return;

        // Optimistic update
        set({ user: { ...currentUser, ...updates } });

        try {
            await fetch(`/api/users/${currentUser.id}`, {
                method: 'PATCH',
                body: JSON.stringify(updates),
            });
        } catch (error) {
            // Revert on error
            set({ user: currentUser, error: error.message });
        }
    },
}));
```

### 5. **When You Want Flexibility**

**Zustand allows patterns that Redux discourages:**
```typescript
// Direct state mutations (with Immer-like API)
const useFlexibleStore = create((set) => ({
    data: { nested: { value: 0 } },

    // Direct nested updates
    updateNested: (newValue) => set((state) => {
        state.data.nested.value = newValue;
    }),

    // Async actions can be methods
    async complexOperation: async () => {
        const currentState = get();
        // Complex logic with access to current state
        // Can call other methods, make multiple API calls, etc.
    },

    // Getters/computed properties
    get computedValue() {
        return get().data.nested.value * 2;
    },
}));
```

## When to Choose Redux

### 1. **Large Applications with Many Developers**

**Choose Redux when:**
- Team size > 5 developers
- Complex application with many features
- Need strict patterns and conventions
- Heavy use of middleware (sagas, thunks)
- Complex async workflows

**Redux benefits at scale:**
```typescript
// Predictable patterns across large codebase
const featureSlice = createSlice({
    name: 'feature',
    initialState,
    reducers: { /* well-defined actions */ },
    extraReducers: (builder) => {
        // Standardized async handling
        builder.addCase(asyncAction.pending, /* ... */);
    },
});

// Type-safe across the entire app
type RootState = ReturnType<typeof store.getState>;
type AppDispatch = typeof store.dispatch;
```

### 2. **Complex Async Logic**

**Redux excels at complex async patterns:**
```typescript
// Complex async workflows with sagas
function* watchUserActions() {
    yield takeLatest('USER_LOGIN', handleLogin);
    yield takeEvery('USER_UPDATE_PROFILE', handleProfileUpdate);
}

function* handleLogin(action) {
    try {
        // Complex login flow
        const { user, token } = yield call(api.login, action.payload);
        yield put(loginSuccess({ user, token }));

        // Parallel actions
        yield all([
            call(fetchUserPreferences, user.id),
            call(fetchUserNotifications, user.id),
            call(initializeWebSocket, token),
        ]);
    } catch (error) {
        yield put(loginFailure(error));
    }
}
```

### 3. **Heavy Middleware Usage**

**Redux middleware ecosystem:**
```typescript
// Powerful middleware for logging, crash reporting, etc.
const store = configureStore({
    reducer: rootReducer,
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware().concat(
            logger,
            crashReporter,
            apiMiddleware,
            websocketMiddleware
        ),
});
```

### 4. **Time Travel Debugging**

**Redux DevTools are unmatched:**
```typescript
// Full state history, action replay, etc.
const store = configureStore({
    reducer: rootReducer,
    devTools: {
        trace: true,  // Stack traces for actions
        traceLimit: 25,
    },
});
```

## Zustand Advanced Patterns

### 1. **Store Composition**

**Combine multiple stores:**
```typescript
// Separate stores for different concerns
const useAuthStore = create((set) => ({
    user: null,
    login: (user) => set({ user }),
    logout: () => set({ user: null }),
}));

const useCartStore = create((set) => ({
    items: [],
    addItem: (item) => set((state) => ({
        items: [...state.items, item]
    })),
}));

// Or compose them
const useCombinedStore = create((set, get) => ({
    ...useAuthStore.getState(),
    ...useCartStore.getState(),
    // Add interactions between stores
    addToCart: (item) => {
        const user = get().user;
        if (!user) return; // Auth check

        set((state) => ({
            items: [...state.items, { ...item, userId: user.id }]
        }));
    },
}));
```

### 2. **Middleware and Plugins**

**Zustand middleware:**
```typescript
import { persist, devtools, immer } from 'zustand/middleware';

const useStore = create<StoreState>()(
    devtools(
        persist(
            immer((set) => ({
                // Store definition
            })),
            {
                name: 'store',
                // Persistence config
            }
        )
    )
);
```

### 3. **Selectors and Performance**

**Selective subscriptions:**
```typescript
// Only re-render when specific state changes
function UserProfile() {
    const userName = useStore((state) => state.user?.name);
    const userEmail = useStore((state) => state.user?.email);

    // Component only re-renders when name or email changes
    return <div>{userName} - {userEmail}</div>;
}

// Or use selectors
const selectUserDisplay = (state) => `${state.user?.name} - ${state.user?.email}`;

function UserProfile() {
    const userDisplay = useStore(selectUserDisplay);
    return <div>{userDisplay}</div>;
}
```

### 4. **Async State Management**

**Built-in async patterns:**
```typescript
const useAsyncStore = create((set, get) => ({
    data: null,
    loading: false,
    error: null,

    // Simple async action
    fetchData: async (id) => {
        set({ loading: true, error: null });
        try {
            const data = await api.fetchData(id);
            set({ data, loading: false });
        } catch (error) {
            set({ error: error.message, loading: false });
        }
    },

    // Optimistic updates
    updateData: async (id, updates) => {
        const previousData = get().data;

        // Optimistic update
        set({ data: { ...previousData, ...updates } });

        try {
            await api.updateData(id, updates);
        } catch (error) {
            // Revert on error
            set({ data: previousData, error: error.message });
        }
    },
}));
```

## Migration Strategies

### 1. **From Redux to Zustand**

**Gradual migration:**
```typescript
// Start by replacing leaf components
// Before: Redux-connected component
const ConnectedComponent = connect(
    (state) => ({ data: state.feature.data }),
    { action: featureActions.action }
)(Component);

// After: Zustand hook
function Component() {
    const { data, action } = useFeatureStore();
    // Same component logic
}
```

**Feature-by-feature migration:**
```typescript
// 1. Create Zustand store for one feature
const useFeatureStore = create((set) => ({
    // Migrate state and actions from Redux slice
}));

// 2. Replace Redux usage in components for that feature
// 3. Remove Redux slice once fully migrated
// 4. Repeat for other features
```

### 2. **Hybrid Approach**

**Use both in the same app:**
```typescript
// Keep Redux for complex features, use Zustand for simple ones
const store = configureStore({
    reducer: {
        complexFeature: complexSlice.reducer, // Redux
        // ... other Redux slices
    },
});

// Zustand for simpler features
const useSimpleStore = create(/* ... */);
```

## Performance Comparison

### 1. **Re-rendering Optimization**

**Redux:**
```typescript
// Requires memoization for performance
const selectData = createSelector(
    (state) => state.feature.data,
    (data) => data
);

function Component() {
    const data = useSelector(selectData); // Memoized
    // ...
}
```

**Zustand:**
```typescript
// Automatic optimization with selectors
function Component() {
    const data = useStore((state) => state.data); // Only re-renders when data changes
    // ...
}
```

### 2. **Bundle Size Impact**

| Library | Bundle Size (gzipped) |
|---------|----------------------|
| Redux + RTK | ~3-4kb |
| Zustand | ~1kb |
| Zustand + middleware | ~1.5-2kb |

## Testing Approaches

### 1. **Testing Zustand Stores**

```typescript
// Unit testing stores
describe('useCounterStore', () => {
    it('should increment counter', () => {
        const { result } = renderHook(() => useCounterStore());

        act(() => {
            result.current.increment();
        });

        expect(result.current.value).toBe(1);
    });

    it('should handle async operations', async () => {
        // Mock API
        const mockApi = jest.fn().mockResolvedValue({ data: 'test' });

        const { result } = renderHook(() => useAsyncStore());

        await act(async () => {
            await result.current.fetchData();
        });

        expect(result.current.data).toEqual({ data: 'test' });
    });
});
```

### 2. **Integration Testing**

```typescript
// Test store interactions
describe('store integration', () => {
    it('should handle complex state changes', () => {
        const { result } = renderHook(() => useAppStore());

        act(() => {
            result.current.login({ id: 1, name: 'John' });
            result.current.toggleTheme();
        });

        expect(result.current.user).toEqual({ id: 1, name: 'John' });
        expect(result.current.theme).toBe('dark');
    });
});
```

## Common Interview Questions

### Q: When would you choose Zustand over Redux?

**A:** I'd choose Zustand for small to medium applications where I want minimal boilerplate and quick setup. It's great for teams that want to move fast without the learning curve of Redux patterns. Redux is better for large applications with many developers, complex async logic, or when you need the extensive middleware ecosystem and time-travel debugging.

### Q: What's the main advantage of Zustand?

**A:** Zustand's main advantage is its simplicity and low boilerplate. You define your state and actions in one place with a hook-based API, and it handles reactivity automatically. There's no need for action types, action creators, or reducers - just define your state and methods.

### Q: Can Zustand scale to large applications?

**A:** Zustand can scale to medium-large applications, but Redux with RTK is generally better for very large applications with many developers because of its strict patterns, extensive middleware ecosystem, and better team coordination features. Zustand works well up to 10-15 stores in a typical application.

### Q: How does Zustand handle async operations?

**A:** Zustand handles async operations naturally - you just define async methods in your store. For loading states, you can set loading flags, and Zustand's reactivity will automatically update components. No special patterns like thunks or sagas are needed.

## Summary

**Choose Zustand When:**
- **Small to medium applications** (1-5 developers)
- **Rapid prototyping** and MVPs
- **Simple to moderate complexity**
- **Want minimal boilerplate** and quick setup
- **Component-level state** that needs sharing
- **Flexibility** over strict patterns

**Choose Redux When:**
- **Large applications** (5+ developers)
- **Complex async workflows** requiring middleware
- **Need extensive DevTools** and debugging
- **Strict patterns** and conventions are important
- **Heavy middleware usage** (sagas, complex thunks)
- **Time-travel debugging** is crucial

**Zustand Strengths:**
- **Minimal API** - just create() and useStore()
- **TypeScript-friendly** with excellent inference
- **Automatic reactivity** with selective subscriptions
- **Small bundle size** (~1kb)
- **Flexible patterns** - no strict conventions

**Redux Strengths:**
- **Battle-tested** in large applications
- **Rich ecosystem** of middleware and tools
- **Excellent DevTools** with time-travel debugging
- **Strict patterns** prevent team conflicts
- **Scales to hundreds** of developers

**Interview Tip:** "I'd choose Zustand for small to medium applications where I want minimal boilerplate and quick development velocity. It's perfect for teams that want to move fast without Redux's learning curve. Redux is better for large applications needing strict patterns, complex async logic, and extensive debugging tools."