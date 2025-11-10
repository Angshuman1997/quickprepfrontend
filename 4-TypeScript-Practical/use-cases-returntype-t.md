# Use cases for ReturnType<T>

## Question
Use cases for ReturnType<T>

## Answer

`ReturnType<T>` is a TypeScript utility type that extracts the return type of a function type `T`. It's incredibly useful for creating derived types based on function return values, ensuring type consistency, and avoiding manual type duplication.

## Basic Usage

```typescript
// Function definitions
function getUser(id: number): User {
    return { id, name: 'John', email: 'john@example.com' };
}

async function fetchPosts(): Promise<Post[]> {
    const response = await fetch('/api/posts');
    return response.json();
}

// Extract return types
type UserType = ReturnType<typeof getUser>;        // User
type PostsType = ReturnType<typeof fetchPosts>;    // Promise<Post[]>

// For async functions, you might want the resolved type
type ResolvedPostsType = Awaited<ReturnType<typeof fetchPosts>>; // Post[]
```

## React Hook Return Types

### 1. **Custom Hook Return Types**

```typescript
import { useState, useEffect } from 'react';

interface User {
    id: number;
    name: string;
    email: string;
}

// Custom hook
function useUser(userId: number) {
    const [user, setUser] = useState<User | null>(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState<string | null>(null);

    useEffect(() => {
        setLoading(true);
        setError(null);

        fetch(`/api/users/${userId}`)
            .then(response => response.json())
            .then(setUser)
            .catch(error => setError(error.message))
            .finally(() => setLoading(false));
    }, [userId]);

    const updateUser = (updates: Partial<User>) => {
        if (!user) return;

        const updatedUser = { ...user, ...updates };
        setUser(updatedUser);

        // Optimistic update to API
        fetch(`/api/users/${userId}`, {
            method: 'PATCH',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(updates)
        }).catch(() => {
            // Revert on error
            setUser(user);
            setError('Failed to update user');
        });
    };

    return {
        user,
        loading,
        error,
        updateUser
    };
}

// Extract the hook's return type
type UseUserReturn = ReturnType<typeof useUser>;

// Usage with proper typing
function UserProfile({ userId }: { userId: number }) {
    const { user, loading, error, updateUser }: UseUserReturn = useUser(userId);

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;
    if (!user) return <div>User not found</div>;

    return (
        <div>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
            <button onClick={() => updateUser({ name: 'Updated Name' })}>
                Update Name
            </button>
        </div>
    );
}
```

### 2. **API Function Return Types**

```typescript
// API functions
const api = {
    getUsers: (): Promise<User[]> => fetch('/api/users').then(r => r.json()),
    getUser: (id: number): Promise<User> => fetch(`/api/users/${id}`).then(r => r.json()),
    createUser: (user: Omit<User, 'id'>): Promise<User> => fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(user)
    }).then(r => r.json()),
    updateUser: (id: number, user: Partial<User>): Promise<User> => fetch(`/api/users/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(user)
    }).then(r => r.json()),
    deleteUser: (id: number): Promise<void> => fetch(`/api/users/${id}`, {
        method: 'DELETE'
    }).then(() => undefined)
};

// Extract API return types
type APIFunctions = typeof api;
type GetUsersReturn = ReturnType<typeof api.getUsers>;      // Promise<User[]>
type GetUserReturn = ReturnType<typeof api.getUser>;        // Promise<User>
type CreateUserReturn = ReturnType<typeof api.createUser>;  // Promise<User>
type UpdateUserReturn = ReturnType<typeof api.updateUser>;  // Promise<User>
type DeleteUserReturn = ReturnType<typeof api.deleteUser>;  // Promise<void>

// For resolved types (without Promise wrapper)
type UsersData = Awaited<GetUsersReturn>;    // User[]
type UserData = Awaited<GetUserReturn>;      // User

// Generic API response type
type APIResponse<T extends keyof APIFunctions> = Awaited<ReturnType<APIFunctions[T]>>;

// Usage
async function loadUserData(userId: number): Promise<UserData> {
    return api.getUser(userId);
}

async function loadUsersData(): Promise<UsersData> {
    return api.getUsers();
}

// Type-safe API wrapper
function createAPIWrapper<T extends keyof APIFunctions>(
    apiFunction: T
): APIFunctions[T] {
    return (...args: Parameters<APIFunctions[T]>) => {
        // Add logging, error handling, etc.
        console.log(`Calling ${apiFunction}`, args);
        return api[apiFunction](...args);
    };
}
```

## Redux/Action Creator Patterns

### 1. **Redux Action Types**

```typescript
// Action creators
const userActions = {
    setUsers: (users: User[]) => ({ type: 'SET_USERS', payload: users } as const),
    addUser: (user: User) => ({ type: 'ADD_USER', payload: user } as const),
    updateUser: (user: User) => ({ type: 'UPDATE_USER', payload: user } as const),
    deleteUser: (id: number) => ({ type: 'DELETE_USER', payload: id } as const),
    setLoading: (loading: boolean) => ({ type: 'SET_LOADING', payload: loading } as const),
    setError: (error: string | null) => ({ type: 'SET_ERROR', payload: error } as const)
};

// Extract action types
type UserAction = ReturnType<typeof userActions.setUsers> |
                  ReturnType<typeof userActions.addUser> |
                  ReturnType<typeof userActions.updateUser> |
                  ReturnType<typeof userActions.deleteUser> |
                  ReturnType<typeof userActions.setLoading> |
                  ReturnType<typeof userActions.setError>;

// Or more concisely with a union
type UserActions = ReturnType<(typeof userActions)[keyof typeof userActions]>;

// State type
interface UserState {
    users: User[];
    loading: boolean;
    error: string | null;
}

// Reducer
function userReducer(state: UserState, action: UserAction): UserState {
    switch (action.type) {
        case 'SET_USERS':
            return { ...state, users: action.payload, loading: false, error: null };
        case 'ADD_USER':
            return { ...state, users: [...state.users, action.payload] };
        case 'UPDATE_USER':
            return {
                ...state,
                users: state.users.map(user =>
                    user.id === action.payload.id ? action.payload : user
                )
            };
        case 'DELETE_USER':
            return {
                ...state,
                users: state.users.filter(user => user.id !== action.payload)
            };
        case 'SET_LOADING':
            return { ...state, loading: action.payload };
        case 'SET_ERROR':
            return { ...state, error: action.payload, loading: false };
        default:
            return state;
    }
}
```

### 2. **Thunk Action Types**

```typescript
import { ThunkAction } from 'redux-thunk';

// Async action creators (thunks)
const asyncUserActions = {
    fetchUsers: () => async (dispatch: Dispatch) => {
        dispatch(userActions.setLoading(true));
        try {
            const users = await api.getUsers();
            dispatch(userActions.setUsers(users));
        } catch (error) {
            dispatch(userActions.setError(error.message));
        }
    },

    createUser: (userData: Omit<User, 'id'>) => async (dispatch: Dispatch) => {
        try {
            const newUser = await api.createUser(userData);
            dispatch(userActions.addUser(newUser));
        } catch (error) {
            dispatch(userActions.setError('Failed to create user'));
        }
    }
};

// Extract thunk return types
type FetchUsersThunk = ReturnType<typeof asyncUserActions.fetchUsers>;
type CreateUserThunk = ReturnType<typeof asyncUserActions.createUser>;

// For Redux Thunk, the return type is complex, but we can extract the function signature
type AppThunk<ReturnType = void> = ThunkAction<
    ReturnType,
    UserState,
    unknown,
    UserAction
>;

// Type-safe dispatch
type AppDispatch = Dispatch<UserAction> & {
    <T extends ReturnType<(typeof asyncUserActions)[keyof typeof asyncUserActions]>>(
        thunkAction: T
    ): ReturnType<T extends (...args: any[]) => infer R ? R : any>;
};
```

## Factory Functions and Builders

### 1. **Factory Function Return Types**

```typescript
// Factory function
function createLogger(prefix: string) {
    return {
        info: (message: string) => console.log(`[${prefix}] INFO: ${message}`),
        warn: (message: string) => console.warn(`[${prefix}] WARN: ${message}`),
        error: (message: string) => console.error(`[${prefix}] ERROR: ${message}`),
        debug: (message: string) => console.debug(`[${prefix}] DEBUG: ${message}`)
    };
}

// Extract logger type
type Logger = ReturnType<typeof createLogger>;

// Usage
function createService(name: string) {
    const logger = createLogger(name);

    return {
        name,
        logger,
        start: () => logger.info(`Starting ${name} service`),
        stop: () => logger.info(`Stopping ${name} service`)
    };
}

type Service = ReturnType<typeof createService>;

// Service registry
const services: Record<string, Service> = {};

function registerService(name: string): Service {
    const service = createService(name);
    services[name] = service;
    return service;
}

// Usage
const userService = registerService('UserService');
const authService = registerService('AuthService');

// Both have the same type
userService.logger.info('User service initialized');
authService.logger.info('Auth service initialized');
```

### 2. **Builder Pattern**

```typescript
// Builder pattern
function createQueryBuilder(table: string) {
    let query = `SELECT * FROM ${table}`;
    let whereClause = '';
    let orderByClause = '';
    let limitClause = '';

    return {
        where: (condition: string) => {
            whereClause = `WHERE ${condition}`;
            return builder;
        },
        orderBy: (column: string, direction: 'ASC' | 'DESC' = 'ASC') => {
            orderByClause = `ORDER BY ${column} ${direction}`;
            return builder;
        },
        limit: (count: number) => {
            limitClause = `LIMIT ${count}`;
            return builder;
        },
        build: () => {
            const parts = [query, whereClause, orderByClause, limitClause].filter(Boolean);
            return parts.join(' ');
        }
    };

    // This needs to be defined after the return object for self-reference
    const builder = {
        where: (condition: string) => { whereClause = `WHERE ${condition}`; return builder; },
        orderBy: (column: string, direction: 'ASC' | 'DESC' = 'ASC') => {
            orderByClause = `ORDER BY ${column} ${direction}`;
            return builder;
        },
        limit: (count: number) => { limitClause = `LIMIT ${count}`; return builder; },
        build: () => {
            const parts = [query, whereClause, orderByClause, limitClause].filter(Boolean);
            return parts.join(' ');
        }
    };

    return builder;
}

// Extract builder type
type QueryBuilder = ReturnType<typeof createQueryBuilder>;

// Usage
function createUserRepository() {
    const builder = createQueryBuilder('users');

    return {
        findAll: (): QueryBuilder => builder,
        findById: (id: number): string => builder.where(`id = ${id}`).limit(1).build(),
        findByEmail: (email: string): string => builder.where(`email = '${email}'`).build(),
        findActive: (): string => builder.where('active = true').orderBy('created_at', 'DESC').build()
    };
}

type UserRepository = ReturnType<typeof createUserRepository>;

// Usage
const repo = createUserRepository();
const query1 = repo.findById(123); // "SELECT * FROM users WHERE id = 123 LIMIT 1"
const query2 = repo.findActive();  // "SELECT * FROM users WHERE active = true ORDER BY created_at DESC"
```

## React Component Props from Functions

### 1. **Component Props from Event Handlers**

```typescript
// Event handler functions
const formHandlers = {
    handleSubmit: (e: React.FormEvent) => {
        e.preventDefault();
        console.log('Form submitted');
    },
    handleChange: (field: string) => (e: React.ChangeEvent<HTMLInputElement>) => {
        console.log(`${field} changed to:`, e.target.value);
    },
    handleBlur: (field: string) => () => {
        console.log(`${field} blurred`);
    }
};

// Extract handler types for component props
type FormHandlers = typeof formHandlers;
type HandleSubmit = FormHandlers['handleSubmit'];
type HandleChange = ReturnType<FormHandlers['handleChange']>;
type HandleBlur = ReturnType<FormHandlers['handleBlur']>;

// Component props
interface FormProps {
    onSubmit: HandleSubmit;
    onChange: HandleChange;
    onBlur: HandleBlur;
}

// Usage
const Form: React.FC<FormProps> = ({ onSubmit, onChange, onBlur }) => {
    return (
        <form onSubmit={onSubmit}>
            <input onChange={onChange('name')} onBlur={onBlur('name')} />
            <input onChange={onChange('email')} onBlur={onBlur('email')} />
            <button type="submit">Submit</button>
        </form>
    );
};

// Create handlers and pass to component
const handlers = formHandlers;
<Form onSubmit={handlers.handleSubmit} onChange={handlers.handleChange} onBlur={handlers.handleBlur} />
```

### 2. **Render Props Pattern**

```typescript
// Render prop function
function createListRenderer<T>() {
    return function ListRenderer({
        items,
        renderItem
    }: {
        items: T[];
        renderItem: (item: T, index: number) => React.ReactNode;
    }) {
        return (
            <ul>
                {items.map((item, index) => (
                    <li key={index}>{renderItem(item, index)}</li>
                ))}
            </ul>
        );
    };
}

// Extract render prop type
type ListRendererComponent<T> = ReturnType<typeof createListRenderer<T>>;
type RenderItemFunction<T> = Parameters<ListRendererComponent<T>>['renderItem'];

// Usage
const StringList = createListRenderer<string>();
const NumberList = createListRenderer<number>();

// Both renderItem props have the correct signature
<StringList
    items={['apple', 'banana', 'cherry']}
    renderItem={(item, index) => <span>{index + 1}. {item}</span>}
/>

<NumberList
    items={[1, 2, 3, 4, 5]}
    renderItem={(item, index) => <span>Item {index}: {item * 2}</span>}
/>
```

## Advanced Patterns

### 1. **Function Composition**

```typescript
// Function composition utilities
function compose<A, B, C>(
    f: (x: B) => C,
    g: (x: A) => B
): (x: A) => C {
    return (x) => f(g(x));
}

function pipe<A, B, C, D>(
    f: (x: A) => B,
    g: (x: B) => C,
    h: (x: C) => D
): (x: A) => D {
    return (x) => h(g(f(x)));
}

// Extract composed function types
type ComposedFunction<A, B, C> = ReturnType<typeof compose<A, B, C>>;
type PipedFunction<A, B, C, D> = ReturnType<typeof pipe<A, B, C, D>>;

// Usage
const add = (x: number) => x + 1;
const multiply = (x: number) => x * 2;
const toString = (x: number) => x.toString();

const addThenMultiply = compose(multiply, add); // (x: number) => number
const complexPipe = pipe(add, multiply, toString); // (x: number) => string

type AddThenMultiply = ReturnType<typeof addThenMultiply>; // number
type ComplexPipe = ReturnType<typeof complexPipe>; // string
```

### 2. **Method Chaining with ReturnType**

```typescript
// Method chaining
class Calculator {
    private value: number;

    constructor(initialValue: number = 0) {
        this.value = initialValue;
    }

    add(x: number): this {
        this.value += x;
        return this;
    }

    subtract(x: number): this {
        this.value -= x;
        return this;
    }

    multiply(x: number): this {
        this.value *= x;
        return this;
    }

    divide(x: number): this {
        this.value /= x;
        return this;
    }

    result(): number {
        return this.value;
    }
}

// Extract method return types
type CalculatorMethod = ReturnType<Calculator['add']>; // Calculator (this)
type ResultMethod = ReturnType<Calculator['result']>;  // number

// Create a type-safe calculator wrapper
function createCalculator(initialValue?: number): Calculator {
    return new Calculator(initialValue);
}

type CalculatorInstance = ReturnType<typeof createCalculator>;

// Usage
const calc = createCalculator(10)
    .add(5)
    .multiply(2)
    .subtract(3)
    .divide(2);

const result = calc.result(); // 6
```

## Best Practices

### 1. **Use ReturnType for Hook Types**

```typescript
// ✅ Good: Extract hook return types
function useCounter(initialValue = 0) {
    const [count, setCount] = useState(initialValue);
    const increment = () => setCount(c => c + 1);
    const decrement = () => setCount(c => c - 1);
    const reset = () => setCount(initialValue);

    return { count, increment, decrement, reset };
}

type UseCounterReturn = ReturnType<typeof useCounter>;

// ❌ Avoid: Manual type duplication
// type UseCounterReturn = {
//     count: number;
//     increment: () => void;
//     decrement: () => void;
//     reset: () => void;
// };
```

### 2. **Combine with Awaited for Async Functions**

```typescript
// ✅ Good: Get resolved types from async functions
async function fetchUser(id: number): Promise<User> {
    // Implementation
}

type FetchUserReturn = ReturnType<typeof fetchUser>;        // Promise<User>
type UserData = Awaited<ReturnType<typeof fetchUser>>;     // User

// ❌ Avoid: Manual Promise unwrapping
// type UserData = User; // Error-prone if function signature changes
```

### 3. **Use ReturnType for API Types**

```typescript
// ✅ Good: Type-safe API responses
const api = {
    getUsers: (): Promise<User[]> => fetch('/api/users').then(r => r.json()),
    getUser: (id: number): Promise<User> => fetch(`/api/users/${id}`).then(r => r.json()),
};

type APIResponse<T extends keyof typeof api> = Awaited<ReturnType<typeof api[T]>>;

// ❌ Avoid: Hardcoded response types
// type GetUsersResponse = User[];
// type GetUserResponse = User; // Becomes outdated if API changes
```

### 4. **Document Complex Return Types**

```typescript
// ✅ Good: Document complex return types
type ComplexHookReturn = ReturnType<typeof useComplexHook>;
// ComplexHookReturn: { data: T[]; loading: boolean; error: Error | null; refetch: () => void }

// ✅ Good: Break down complex types
type UseComplexHookReturn = ReturnType<typeof useComplexHook>;
type HookData<T> = UseComplexHookReturn extends { data: T } ? T : never;
type HookError = UseComplexHookReturn extends { error: infer E } ? E : never;
```

## Common Interview Questions

### Q: When should you use ReturnType<T> instead of defining types manually?

**A:** Use `ReturnType<T>` when the return type depends on a function that might change. It ensures type consistency and reduces duplication. Manual types become outdated if the function signature changes.

### Q: How do you get the return type of an async function without the Promise wrapper?

**A:** Use `Awaited<ReturnType<T>>` for async functions. `ReturnType<T>` gives you `Promise<T>`, and `Awaited` extracts the `T` from the Promise.

### Q: Can ReturnType work with function overloads?

**A:** For function overloads, `ReturnType` will infer the return type of the last overload. If you need specific overload return types, you should define them explicitly or use conditional types.

### Q: What's the difference between ReturnType<T> and the return type annotation?

**A:** `ReturnType<T>` extracts the return type from an existing function type, while a return type annotation defines what a function should return. `ReturnType` is useful for deriving types from existing functions.

## Summary

`ReturnType<T>` is essential for:

1. **Hook Return Types**: Extracting types from custom React hooks
2. **API Function Types**: Creating type-safe API response types
3. **Action Creator Types**: Redux action and thunk types
4. **Factory Functions**: Types for objects created by factories
5. **Function Composition**: Types for composed and piped functions
6. **Method Chaining**: Types for fluent interfaces

**Key Benefits:**
- Automatic type inference from function implementations
- Maintains consistency when functions change
- Reduces manual type duplication
- Enables type-safe generic programming
- Works with complex function signatures

**Interview Tip:** "`ReturnType<T>` extracts the return type of a function T. It's perfect for creating derived types from existing functions, ensuring type consistency, and avoiding manual duplication in APIs, hooks, and action creators."