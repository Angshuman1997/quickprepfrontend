# Explain Generics with a simple custom hook example

## Question
Explain Generics with a simple custom hook example.

## Answer

Generics in TypeScript allow you to create reusable components and functions that work with multiple types while maintaining type safety. They're particularly powerful in React hooks for creating flexible, type-safe custom hooks.

## What are Generics?

Generics are like "type variables" - placeholders for types that are specified when the code is used, not when it's written.

```typescript
// Generic function syntax
function identity<T>(arg: T): T {
    return arg;
}

// Usage
const result1 = identity<string>("hello");  // T = string
const result2 = identity<number>(42);       // T = number
const result3 = identity({ name: "John" }); // T inferred as { name: string }
```

## Basic Generic Custom Hook Example

Let's create a simple `useLocalStorage` hook that demonstrates generics:

```typescript
import { useState, useEffect } from 'react';

// Generic hook for localStorage
function useLocalStorage<T>(
    key: string,
    initialValue: T
): [T, (value: T | ((prevValue: T) => T)) => void] {
    // State to store our value
    const [storedValue, setStoredValue] = useState<T>(() => {
        try {
            // Get from local storage by key
            const item = window.localStorage.getItem(key);
            // Parse stored json or if none return initialValue
            return item ? JSON.parse(item) : initialValue;
        } catch (error) {
            // If error, return initial value
            console.error(`Error reading localStorage key "${key}":`, error);
            return initialValue;
        }
    });

    // Return a wrapped version of useState's setter function that persists the new value to localStorage
    const setValue = (value: T | ((prevValue: T) => T)) => {
        try {
            // Allow value to be a function so we have the same API as useState
            const valueToStore = value instanceof Function ? value(storedValue) : value;
            // Save state
            setStoredValue(valueToStore);
            // Save to local storage
            window.localStorage.setItem(key, JSON.stringify(valueToStore));
        } catch (error) {
            console.error(`Error setting localStorage key "${key}":`, error);
        }
    };

    return [storedValue, setValue];
}

export default useLocalStorage;
```

## Usage Examples

```typescript
// Example 1: String value
function App() {
    const [name, setName] = useLocalStorage<string>('name', 'John Doe');

    return (
        <div>
            <input
                value={name}
                onChange={(e) => setName(e.target.value)}
            />
            <p>Hello, {name}!</p>
        </div>
    );
}

// Example 2: Object value
interface User {
    id: number;
    name: string;
    email: string;
}

function UserProfile() {
    const [user, setUser] = useLocalStorage<User>('user', {
        id: 1,
        name: 'John Doe',
        email: 'john@example.com'
    });

    const updateName = (newName: string) => {
        setUser(prevUser => ({
            ...prevUser,
            name: newName
        }));
    };

    return (
        <div>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
            <button onClick={() => updateName('Jane Doe')}>
                Change Name
            </button>
        </div>
    );
}

// Example 3: Array value
function TodoList() {
    const [todos, setTodos] = useLocalStorage<string[]>('todos', []);

    const addTodo = (todo: string) => {
        setTodos(prevTodos => [...prevTodos, todo]);
    };

    return (
        <div>
            <input
                placeholder="Add todo"
                onKeyPress={(e) => {
                    if (e.key === 'Enter') {
                        addTodo(e.currentTarget.value);
                        e.currentTarget.value = '';
                    }
                }}
            />
            <ul>
                {todos.map((todo, index) => (
                    <li key={index}>{todo}</li>
                ))}
            </ul>
        </div>
    );
}
```

## Advanced Generic Hook: useAPI

Let's create a more complex generic hook for API calls:

```typescript
import { useState, useEffect, useCallback } from 'react';

// Generic types for API responses
interface APIResponse<T> {
    data: T | null;
    loading: boolean;
    error: string | null;
}

// Generic hook for API calls
function useAPI<T>(
    url: string,
    options?: {
        method?: 'GET' | 'POST' | 'PUT' | 'DELETE';
        body?: any;
        headers?: Record<string, string>;
        autoFetch?: boolean;
    }
): APIResponse<T> & {
    refetch: () => Promise<void>;
    setData: React.Dispatch<React.SetStateAction<T | null>>;
} {
    const [data, setData] = useState<T | null>(null);
    const [loading, setLoading] = useState<boolean>(false);
    const [error, setError] = useState<string | null>(null);

    const fetchData = useCallback(async () => {
        setLoading(true);
        setError(null);

        try {
            const response = await fetch(url, {
                method: options?.method || 'GET',
                headers: {
                    'Content-Type': 'application/json',
                    ...options?.headers,
                },
                body: options?.body ? JSON.stringify(options.body) : undefined,
            });

            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }

            const result: T = await response.json();
            setData(result);
        } catch (err) {
            setError(err instanceof Error ? err.message : 'An error occurred');
        } finally {
            setLoading(false);
        }
    }, [url, options]);

    useEffect(() => {
        if (options?.autoFetch !== false) {
            fetchData();
        }
    }, [fetchData, options?.autoFetch]);

    return {
        data,
        loading,
        error,
        refetch: fetchData,
        setData,
    };
}

export default useAPI;
```

## Usage of Advanced Hook

```typescript
// Define types for your API responses
interface User {
    id: number;
    name: string;
    email: string;
}

interface Post {
    id: number;
    title: string;
    body: string;
    userId: number;
}

// Component using the generic API hook
function UserDashboard() {
    // Fetch user data
    const {
        data: user,
        loading: userLoading,
        error: userError,
        refetch: refetchUser
    } = useAPI<User>('/api/user/1');

    // Fetch user's posts
    const {
        data: posts,
        loading: postsLoading,
        error: postsError,
        refetch: refetchPosts
    } = useAPI<Post[]>('/api/posts?userId=1');

    if (userLoading || postsLoading) return <div>Loading...</div>;
    if (userError || postsError) return <div>Error: {userError || postsError}</div>;
    if (!user || !posts) return <div>No data</div>;

    return (
        <div>
            <h1>Welcome, {user.name}!</h1>
            <p>Email: {user.email}</p>

            <h2>Your Posts:</h2>
            {posts.map(post => (
                <article key={post.id}>
                    <h3>{post.title}</h3>
                    <p>{post.body}</p>
                </article>
            ))}

            <button onClick={refetchUser}>Refresh User</button>
            <button onClick={refetchPosts}>Refresh Posts</button>
        </div>
    );
}

// POST request example
function CreatePost() {
    const {
        data: newPost,
        loading,
        error,
        refetch: createPost
    } = useAPI<Post>('/api/posts', {
        method: 'POST',
        body: {
            title: 'New Post',
            body: 'Post content',
            userId: 1
        },
        autoFetch: false // Don't fetch on mount
    });

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        await createPost();
    };

    return (
        <form onSubmit={handleSubmit}>
            <button type="submit" disabled={loading}>
                {loading ? 'Creating...' : 'Create Post'}
            </button>
            {error && <p>Error: {error}</p>}
            {newPost && <p>Post created: {newPost.title}</p>}
        </form>
    );
}
```

## Generic Constraints

You can constrain generics to ensure they have certain properties:

```typescript
// Constraint: T must have an 'id' property
function useEntityManager<T extends { id: number }>(
    entityName: string
) {
    const [entities, setEntities] = useState<T[]>([]);

    const addEntity = (entity: Omit<T, 'id'>) => {
        const newEntity = {
            ...entity,
            id: Date.now() // Generate ID
        } as T;

        setEntities(prev => [...prev, newEntity]);
    };

    const updateEntity = (id: number, updates: Partial<T>) => {
        setEntities(prev =>
            prev.map(entity =>
                entity.id === id ? { ...entity, ...updates } : entity
            )
        );
    };

    const deleteEntity = (id: number) => {
        setEntities(prev => prev.filter(entity => entity.id !== id));
    };

    return {
        entities,
        addEntity,
        updateEntity,
        deleteEntity
    };
}

// Usage
interface Product {
    id: number;
    name: string;
    price: number;
    category: string;
}

function ProductManager() {
    const { entities: products, addEntity, updateEntity, deleteEntity } =
        useEntityManager<Product>('product');

    return (
        <div>
            {/* Product management UI */}
        </div>
    );
}
```

## Multiple Generic Parameters

```typescript
// Hook with multiple generic parameters
function useFormField<T, K extends keyof T>(
    object: T,
    fieldName: K
): [T[K], (value: T[K]) => void] {
    const [value, setValue] = useState<T[K]>(object[fieldName]);

    const updateValue = (newValue: T[K]) => {
        setValue(newValue);
    };

    return [value, updateValue];
}

// Usage
interface User {
    name: string;
    age: number;
    email: string;
}

function UserForm() {
    const user = { name: 'John', age: 30, email: 'john@example.com' };

    const [name, setName] = useFormField(user, 'name');
    const [age, setAge] = useFormField(user, 'age');
    const [email, setEmail] = useFormField(user, 'email');

    return (
        <form>
            <input value={name} onChange={(e) => setName(e.target.value)} />
            <input
                type="number"
                value={age}
                onChange={(e) => setAge(Number(e.target.value))}
            />
            <input
                type="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
            />
        </form>
    );
}
```

## Generic Hook with Context

```typescript
import React, { createContext, useContext, ReactNode } from 'react';

// Generic context type
interface DataContextType<T> {
    data: T[];
    addItem: (item: T) => void;
    removeItem: (id: string | number) => void;
    updateItem: (id: string | number, updates: Partial<T>) => void;
}

// Generic context hook
function createDataContext<T extends { id: string | number }>() {
    const DataContext = createContext<DataContextType<T> | null>(null);

    function DataProvider({
        children,
        initialData = []
    }: {
        children: ReactNode;
        initialData?: T[]
    }) {
        const [data, setData] = useState<T[]>(initialData);

        const addItem = (item: T) => {
            setData(prev => [...prev, item]);
        };

        const removeItem = (id: string | number) => {
            setData(prev => prev.filter(item => item.id !== id));
        };

        const updateItem = (id: string | number, updates: Partial<T>) => {
            setData(prev =>
                prev.map(item =>
                    item.id === id ? { ...item, ...updates } : item
                )
            );
        };

        const value = {
            data,
            addItem,
            removeItem,
            updateItem
        };

        return (
            <DataContext.Provider value={value}>
                {children}
            </DataContext.Provider>
        );
    }

    function useDataContext() {
        const context = useContext(DataContext);
        if (!context) {
            throw new Error('useDataContext must be used within DataProvider');
        }
        return context;
    }

    return { DataProvider, useDataContext };
}

// Usage
interface Todo {
    id: number;
    text: string;
    completed: boolean;
}

const { DataProvider: TodoProvider, useDataContext: useTodos } =
    createDataContext<Todo>();

function TodoApp() {
    return (
        <TodoProvider initialData={[
            { id: 1, text: 'Learn TypeScript', completed: false },
            { id: 2, text: 'Build React app', completed: true }
        ]}>
            <TodoList />
        </TodoProvider>
    );
}

function TodoList() {
    const { data: todos, addItem, removeItem, updateItem } = useTodos();

    return (
        <div>
            {todos.map(todo => (
                <div key={todo.id}>
                    <span style={{
                        textDecoration: todo.completed ? 'line-through' : 'none'
                    }}>
                        {todo.text}
                    </span>
                    <button onClick={() => updateItem(todo.id, { completed: !todo.completed })}>
                        Toggle
                    </button>
                    <button onClick={() => removeItem(todo.id)}>
                        Delete
                    </button>
                </div>
            ))}
        </div>
    );
}
```

## Key Benefits of Generics in Hooks

### 1. **Type Safety**
- Compile-time type checking
- IntelliSense support
- Prevents runtime errors

### 2. **Reusability**
- Same hook works with different data types
- Reduces code duplication
- Consistent API across different use cases

### 3. **Maintainability**
- Changes to types are automatically propagated
- Easier refactoring
- Better developer experience

### 4. **Flexibility**
- Works with primitives, objects, arrays, etc.
- Supports complex type operations
- Can be constrained for specific requirements

## Common Patterns

### 1. **Generic Data Fetching Hook**
```typescript
function useFetch<T>(url: string): {
    data: T | null;
    loading: boolean;
    error: Error | null;
} {
    // Implementation
}
```

### 2. **Generic Form Hook**
```typescript
function useForm<T extends Record<string, any>>(
    initialValues: T
): {
    values: T;
    setValues: (values: T) => void;
    handleChange: (field: keyof T) => (value: any) => void;
} {
    // Implementation
}
```

### 3. **Generic Storage Hook**
```typescript
function useStorage<T>(
    key: string,
    initialValue: T,
    storage: Storage = localStorage
): [T, (value: T) => void] {
    // Implementation
}
```

## Interview Tips

**Q: Why use generics in custom hooks?**
- Type safety across different data types
- Reusable logic without sacrificing type information
- Better IntelliSense and developer experience
- Prevents type-related bugs at compile time

**Q: How do you constrain generics?**
- Use `extends` keyword: `function useHook<T extends SomeType>`
- Common constraints: objects with IDs, arrays, specific interfaces

**Q: What's the difference between `<T>` and `<T,>`?**
- `<T,>` is used when you have only one generic parameter but want to avoid JSX confusion
- In TSX files, `<T>` might be interpreted as JSX, so `<T,>` disambiguates

**Q: Can you have default generic types?**
- Yes: `function useHook<T = string>(value: T)`
- Useful for providing sensible defaults

## Summary

Generics enable you to write flexible, reusable hooks that maintain type safety. They allow your hooks to work with any data type while providing compile-time guarantees about type correctness. The examples above demonstrate how generics can be used to create powerful, type-safe custom hooks for local storage, API calls, forms, and data management.

**Key Takeaway:** Generics in TypeScript hooks provide the perfect balance between flexibility and type safety, making your React applications more robust and maintainable.