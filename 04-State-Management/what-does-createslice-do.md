# What does createSlice() do?

## Question
What does createSlice() do?

## Answer

`createSlice()` is Redux Toolkit's primary API for creating Redux slices - self-contained pieces of Redux state logic that include actions, reducers, and selectors. It eliminates the boilerplate of writing action types, action creators, and reducers separately, while automatically integrating Immer for immutable updates.

## What createSlice Does

### 1. **Combines Multiple Redux Concepts**

**Before createSlice (Vanilla Redux):**
```javascript
// Action types (manual)
const INCREMENT = 'counter/increment';
const DECREMENT = 'counter/decrement';
const INCREMENT_BY_AMOUNT = 'counter/incrementByAmount';

// Action creators (manual)
function increment() {
    return { type: INCREMENT };
}

function decrement() {
    return { type: DECREMENT };
}

function incrementByAmount(amount) {
    return { type: INCREMENT_BY_AMOUNT, payload: amount };
}

// Reducer (manual)
function counterReducer(state = 0, action) {
    switch (action.type) {
        case INCREMENT:
            return state + 1;
        case DECREMENT:
            return state - 1;
        case INCREMENT_BY_AMOUNT:
            return state + action.payload;
        default:
            return state;
    }
}
```

**With createSlice:**
```typescript
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
    name: 'counter',
    initialState: 0,
    reducers: {
        increment: (state) => state + 1,
        decrement: (state) => state - 1,
        incrementByAmount: (state, action) => state + action.payload,
    },
});

// Auto-generated action creators
export const { increment, decrement, incrementByAmount } = counterSlice.actions;

// Auto-generated reducer
export default counterSlice.reducer;
```

### 2. **Automatic Action Type Generation**

**How createSlice generates action types:**
```typescript
const counterSlice = createSlice({
    name: 'counter',  // Slice name prefix
    initialState: 0,
    reducers: {
        increment: (state) => state + 1,           // → 'counter/increment'
        decrement: (state) => state - 1,           // → 'counter/decrement'
        incrementByAmount: (state, action) =>      // → 'counter/incrementByAmount'
            state + action.payload,
    },
});

// The generated action types are available as:
// counterSlice.actions.increment.type → 'counter/increment'
// counterSlice.actions.decrement.type → 'counter/decrement'
// counterSlice.actions.incrementByAmount.type → 'counter/incrementByAmount'
```

**Action type format:** `{sliceName}/{reducerName}`

### 3. **Automatic Action Creator Generation**

**What createSlice generates:**
```typescript
const counterSlice = createSlice({
    name: 'counter',
    initialState: 0,
    reducers: {
        increment: (state) => state + 1,
        decrement: (state) => state - 1,
        incrementByAmount: (state, action) => state + action.payload,
    },
});

// Generated action creators:
counterSlice.actions.increment()
// → { type: 'counter/increment' }

counterSlice.actions.decrement()
// → { type: 'counter/decrement' }

counterSlice.actions.incrementByAmount(5)
// → { type: 'counter/incrementByAmount', payload: 5 }
```

**Action creator behavior:**
- **No parameters**: Creates action with just `type`
- **One parameter**: Becomes the `payload` property
- **Multiple parameters**: Use an object and destructure in the reducer

### 4. **Automatic Reducer Generation**

**How createSlice builds the reducer:**
```typescript
const counterSlice = createSlice({
    name: 'counter',
    initialState: 0,
    reducers: {
        increment: (state) => state + 1,
        decrement: (state) => state - 1,
        incrementByAmount: (state, action) => state + action.payload,
    },
});

// Generated reducer function:
function counterReducer(state = 0, action) {
    switch (action.type) {
        case 'counter/increment':
            // Immer allows "mutating" syntax
            return immer.produce(state, draft => {
                return draft + 1;
            });
        case 'counter/decrement':
            return immer.produce(state, draft => {
                return draft - 1;
            });
        case 'counter/incrementByAmount':
            return immer.produce(state, draft => {
                return draft + action.payload;
            });
        default:
            return state;
    }
}
```

## Key Features of createSlice

### 1. **Immer Integration**

**"Mutating" immutable updates:**
```typescript
const todosSlice = createSlice({
    name: 'todos',
    initialState: [],
    reducers: {
        addTodo: (state, action) => {
            // Looks like mutation, but Immer makes it immutable
            state.push({
                id: Date.now(),
                text: action.payload,
                completed: false,
            });
        },
        toggleTodo: (state, action) => {
            // Direct "mutation" is safe
            const todo = state.find(todo => todo.id === action.payload);
            if (todo) {
                todo.completed = !todo.completed;
            }
        },
        updateTodo: (state, action) => {
            // Complex nested updates look natural
            const { id, updates } = action.payload;
            const todo = state.find(todo => todo.id === id);
            if (todo) {
                Object.assign(todo, updates);
            }
        },
    },
});
```

**What Immer does behind the scenes:**
```javascript
// Your "mutating" code:
state.push(newItem);

// Immer converts it to:
return {
    ...state,
    [state.length]: newItem,
    length: state.length + 1,
};
```

### 2. **Extra Reducers for Async Actions**

**Handling external actions:**
```typescript
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
        data: [],
        loading: false,
        error: null,
    },
    reducers: {
        // Synchronous reducers
        clearError: (state) => {
            state.error = null;
        },
    },
    extraReducers: (builder) => {
        // Handle async thunk actions
        builder
            .addCase(fetchUsers.pending, (state) => {
                state.loading = true;
                state.error = null;
            })
            .addCase(fetchUsers.fulfilled, (state, action) => {
                state.loading = false;
                state.data = action.payload;
            })
            .addCase(fetchUsers.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload;
            });
    },
});
```

### 3. **Advanced Reducer Patterns**

**Multiple action creators for one reducer:**
```typescript
const counterSlice = createSlice({
    name: 'counter',
    initialState: { value: 0, step: 1 },
    reducers: {
        increment: (state) => {
            state.value += state.step;
        },
        decrement: (state) => {
            state.value -= state.step;
        },
        setStep: (state, action) => {
            state.step = action.payload;
        },
        // Multiple ways to increment
        incrementBy: {
            reducer: (state, action) => {
                state.value += action.payload;
            },
            prepare: (amount: number) => ({
                payload: amount,
                meta: { timestamp: Date.now() },
            }),
        },
    },
});

// Usage
dispatch(counterSlice.actions.incrementBy(5));
// → { type: 'counter/incrementBy', payload: 5, meta: { timestamp: 1234567890 } }
```

**Conditional reducers:**
```typescript
const authSlice = createSlice({
    name: 'auth',
    initialState: { user: null, token: null },
    reducers: {
        login: (state, action) => {
            const { user, token } = action.payload;
            state.user = user;
            state.token = token;
        },
        logout: (state) => {
            // Only clear state if user exists
            if (state.user) {
                state.user = null;
                state.token = null;
            }
        },
        updateProfile: (state, action) => {
            // Only update if user is logged in
            if (state.user) {
                state.user = { ...state.user, ...action.payload };
            }
        },
    },
});
```

## createSlice API Deep Dive

### 1. **Parameters**

```typescript
createSlice({
    name: string,           // Required: Slice name (used as action type prefix)
    initialState: any,      // Required: Initial state value
    reducers: object,       // Optional: Synchronous reducer functions
    extraReducers: function, // Optional: Handle external actions (async thunks)
});
```

### 2. **Return Value**

```typescript
const slice = createSlice(options);

// Returns:
{
    name: string,              // The slice name
    reducer: function,         // The generated reducer function
    actions: object,           // Generated action creators
    caseReducers: object,      // Individual reducer functions (for testing)
    getInitialState: function, // Returns a fresh initial state
}
```

### 3. **Action Creator Types**

**TypeScript support:**
```typescript
const counterSlice = createSlice({
    name: 'counter',
    initialState: 0,
    reducers: {
        increment: (state) => state + 1,
        incrementByAmount: (state, action: PayloadAction<number>) => 
            state + action.payload,
    },
});

// Action types are automatically inferred
export type CounterActions =
    | ReturnType<typeof counterSlice.actions.increment>
    | ReturnType<typeof counterSlice.actions.incrementByAmount>;

// Action creators have proper types
const increment = counterSlice.actions.increment;
// typeof increment === () => { type: 'counter/increment' }

const incrementByAmount = counterSlice.actions.incrementByAmount;
// typeof incrementByAmount === (payload: number) => { type: 'counter/incrementByAmount', payload: number }
```

## Advanced createSlice Patterns

### 1. **Dynamic Slice Creation**

**Factory pattern for dynamic slices:**
```typescript
function createCounterSlice(name: string, initialValue = 0) {
    return createSlice({
        name,
        initialState: { value: initialValue },
        reducers: {
            increment: (state) => {
                state.value += 1;
            },
            decrement: (state) => {
                state.value -= 1;
            },
            reset: (state) => {
                state.value = initialValue;
            },
        },
    });
}

// Create multiple counter instances
const counterA = createCounterSlice('counterA', 10);
const counterB = createCounterSlice('counterB', 20);

// Different action types: 'counterA/increment', 'counterB/increment'
```

### 2. **Slice Composition**

**Combining slices:**
```typescript
// Base slice
const baseCounterSlice = createSlice({
    name: 'counter',
    initialState: 0,
    reducers: {
        increment: (state) => state + 1,
        decrement: (state) => state - 1,
    },
});

// Extended slice
const advancedCounterSlice = createSlice({
    name: 'advancedCounter',
    initialState: { ...baseCounterSlice.getInitialState(), step: 1 },
    reducers: {
        ...baseCounterSlice.caseReducers, // Include base reducers
        setStep: (state, action) => {
            state.step = action.payload;
        },
        increment: (state) => {
            state.value += state.step; // Override with step logic
        },
    },
});
```

### 3. **Slice with Entity Adapter**

**Normalized state with entities:**
```typescript
import { createSlice, createEntityAdapter } from '@reduxjs/toolkit';

const usersAdapter = createEntityAdapter();

const usersSlice = createSlice({
    name: 'users',
    initialState: usersAdapter.getInitialState({
        loading: false,
        error: null,
    }),
    reducers: {
        // Adapter provides common CRUD actions
        addUser: usersAdapter.addOne,
        removeUser: usersAdapter.removeOne,
        updateUser: usersAdapter.updateOne,
        setUsers: usersAdapter.setAll,
        clearUsers: usersAdapter.removeAll,
        // Custom actions
        setLoading: (state, action) => {
            state.loading = action.payload;
        },
        setError: (state, action) => {
            state.error = action.payload;
        },
    },
});

// Generated actions include entity actions
export const {
    addUser,
    removeUser,
    updateUser,
    setUsers,
    clearUsers,
    setLoading,
    setError
} = usersSlice.actions;

// Generated selectors
export const {
    selectAll: selectAllUsers,
    selectById: selectUserById,
    selectIds: selectUserIds,
} = usersAdapter.getSelectors((state: RootState) => state.users);
```

## Common Patterns and Best Practices

### 1. **Consistent Slice Structure**

```typescript
// ✅ Good: Consistent structure across slices
const usersSlice = createSlice({
    name: 'users',
    initialState: {
        // Data first
        users: [],
        // Status second
        loading: false,
        error: null,
        // Metadata last
        lastFetched: null,
    },
    reducers: {
        // Actions in logical order
        setLoading: (state, action) => {
            state.loading = action.payload;
        },
        setUsers: (state, action) => {
            state.users = action.payload;
            state.lastFetched = Date.now();
        },
        setError: (state, action) => {
            state.error = action.payload;
            state.loading = false;
        },
    },
});
```

### 2. **Action Naming Conventions**

```typescript
// ✅ Good: Clear, consistent action names
const todosSlice = createSlice({
    name: 'todos',
    initialState: [],
    reducers: {
        // CRUD operations
        addTodo: (state, action) => { /* ... */ },
        updateTodo: (state, action) => { /* ... */ },
        removeTodo: (state, action) => { /* ... */ },
        toggleTodo: (state, action) => { /* ... */ },

        // Status operations
        setTodoLoading: (state, action) => { /* ... */ },
        setTodoError: (state, action) => { /* ... */ },

        // Bulk operations
        clearCompletedTodos: (state) => { /* ... */ },
        markAllTodosCompleted: (state) => { /* ... */ },
    },
});
```

### 3. **Testing createSlice**

```typescript
// Testing the generated reducer
describe('counter slice', () => {
    const initialState = 0;

    it('should handle increment', () => {
        const actual = counterSlice.reducer(initialState, counterSlice.actions.increment());
        expect(actual).toEqual(1);
    });

    it('should handle decrement', () => {
        const actual = counterSlice.reducer(1, counterSlice.actions.decrement());
        expect(actual).toEqual(0);
    });

    it('should handle incrementByAmount', () => {
        const actual = counterSlice.reducer(5, counterSlice.actions.incrementByAmount(3));
        expect(actual).toEqual(8);
    });

    it('should return initial state for unknown action', () => {
        const actual = counterSlice.reducer(initialState, { type: 'unknown' });
        expect(actual).toEqual(initialState);
    });
});

// Testing action creators
describe('counter actions', () => {
    it('should create increment action', () => {
        const expectedAction = { type: 'counter/increment' };
        expect(counterSlice.actions.increment()).toEqual(expectedAction);
    });

    it('should create incrementByAmount action', () => {
        const expectedAction = {
            type: 'counter/incrementByAmount',
            payload: 5
        };
        expect(counterSlice.actions.incrementByAmount(5)).toEqual(expectedAction);
    });
});
```

## Performance Considerations

### 1. **Action Creator Memoization**

**createSlice action creators are automatically memoized:**
```typescript
const counterSlice = createSlice({ /* ... */ });

// Same reference every time
const increment1 = counterSlice.actions.increment;
const increment2 = counterSlice.actions.increment;
expect(increment1).toBe(increment2); // true
```

### 2. **Reducer Performance**

**Immer optimizes immutable updates:**
```typescript
// Immer only creates new objects for changed paths
const state = {
    todos: [
        { id: 1, text: 'Learn Redux', completed: false },
        { id: 2, text: 'Learn RTK', completed: false },
    ],
    filter: 'all',
};

// Only the todos array and the specific todo object are recreated
state.todos[0].completed = true; // With Immer
```

## Migration from Vanilla Redux

### 1. **Converting Switch Reducers**

**Before:**
```javascript
function todosReducer(state = [], action) {
    switch (action.type) {
        case 'ADD_TODO':
            return [...state, action.payload];
        case 'TOGGLE_TODO':
            return state.map(todo =>
                todo.id === action.payload.id
                    ? { ...todo, completed: !todo.completed }
                    : todo
            );
        default:
            return state;
    }
}
```

**After:**
```typescript
const todosSlice = createSlice({
    name: 'todos',
    initialState: [],
    reducers: {
        addTodo: (state, action) => {
            state.push(action.payload);
        },
        toggleTodo: (state, action) => {
            const todo = state.find(todo => todo.id === action.payload.id);
            if (todo) {
                todo.completed = !todo.completed;
            }
        },
    },
});
```

### 2. **Handling Complex State**

**Before:**
```javascript
function appReducer(state = initialState, action) {
    switch (action.type) {
        case 'UPDATE_USER_PROFILE':
            return {
                ...state,
                user: {
                    ...state.user,
                    profile: {
                        ...state.user.profile,
                        ...action.payload,
                    },
                },
            };
        // More complex nested updates...
    }
}
```

**After:**
```typescript
const userSlice = createSlice({
    name: 'user',
    initialState: { profile: {} },
    reducers: {
        updateProfile: (state, action) => {
            // Simple nested updates
            Object.assign(state.profile, action.payload);
        },
    },
});
```

## Common Interview Questions

### Q: What does createSlice do in Redux Toolkit?

**A:** `createSlice` is the primary API for creating Redux slices. It automatically generates action types, action creators, and reducers from a simple configuration object. It integrates Immer for immutable updates and eliminates the boilerplate of writing action types and creators manually.

### Q: How does createSlice handle immutability?

**A:** `createSlice` uses Immer internally, which allows you to write "mutating" code that looks like `state.todos.push(todo)`, but Immer converts it to proper immutable updates automatically. This makes reducer logic more readable and less error-prone.

### Q: What's the difference between reducers and extraReducers in createSlice?

**A:** `reducers` defines synchronous actions that are automatically generated with action creators. `extraReducers` handles external actions, typically from `createAsyncThunk` or other slices. `extraReducers` uses a builder pattern to add cases for pending, fulfilled, and rejected states of async actions.

### Q: Can createSlice be used for complex async logic?

**A:** `createSlice` is designed for synchronous reducers. For async logic, you use `createAsyncThunk` to create async actions, then handle their pending/fulfilled/rejected states in `extraReducers`. The slice itself remains synchronous.

## Summary

**What createSlice Does:**
1. **Generates action types** with `{sliceName}/{reducerName}` format
2. **Creates action creators** automatically for each reducer
3. **Builds reducer function** that handles all defined actions
4. **Integrates Immer** for "mutating" immutable updates
5. **Supports extraReducers** for external async actions

**Key Benefits:**
- **80% reduction** in Redux boilerplate code
- **Type-safe** action creators and state
- **Immer integration** for readable reducer logic
- **Automatic action type generation**
- **Consistent patterns** across the application

**Generated Outputs:**
- `slice.actions` - Action creator functions
- `slice.reducer` - Reducer function
- `slice.caseReducers` - Individual reducer functions
- Action types accessible via `action.type`

**Interview Tip:** "`createSlice` is Redux Toolkit's main API that eliminates Redux boilerplate by automatically generating action types, action creators, and reducers from a simple configuration. It uses Immer for immutable updates, allowing you to write `state.push(item)` instead of complex spread operations, reducing Redux code by about 80%."