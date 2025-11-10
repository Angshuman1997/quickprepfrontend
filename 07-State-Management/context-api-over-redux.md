# Context API vs Redux

## Question
When to use Context API vs Redux?

## Answer
Context API for simple state sharing in small apps. Redux for complex state management in large apps.

## Context API Example
```javascript
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function ThemedButton() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      {theme}
    </button>
  );
}
```

## Redux Example
```javascript
const themeSlice = createSlice({
  name: 'theme',
  initialState: 'light',
  reducers: {
    toggleTheme: (state) => state === 'light' ? 'dark' : 'light'
  }
});

function ThemedButton() {
  const theme = useSelector(state => state.theme);
  const dispatch = useDispatch();
  return (
    <button onClick={() => dispatch(toggleTheme())}>
      {theme}
    </button>
  );
}
```

## When to Use Context API

✅ **Small to medium apps**  
✅ **Simple state sharing**  
✅ **Theme/user preferences**  
✅ **No complex async logic**  
✅ **Built-in React (no extra libraries)**  

## When to Use Redux

✅ **Large applications**  
✅ **Complex state logic**  
✅ **Many developers working together**  
✅ **Time-travel debugging needed**  
✅ **Advanced middleware required**  
✅ **Predictable state updates**  

## Interview Q&A

**Q: When do you choose Context API over Redux?**

A: For small apps with simple state needs. Context API is built-in React and easier for basic state sharing.

**Q: What's the main difference between Context API and Redux?**

A: Context API is for sharing state between components. Redux is for managing complex application state with predictable updates.

**Q: Can Context API replace Redux?**

A: For simple cases yes, but Redux is better for complex apps with many features, multiple developers, and advanced debugging needs.
```

### 2. **Context API with useReducer**

```typescript
import React, { createContext, useContext, useReducer, ReactNode } from 'react';

// State type
interface CounterState {
    count: number;
}

// Action types
type CounterAction =
    | { type: 'INCREMENT' }
    | { type: 'DECREMENT' }
    | { type: 'RESET' };

// Reducer
const counterReducer = (state: CounterState, action: CounterAction): CounterState => {
    switch (action.type) {
        case 'INCREMENT':
            return { count: state.count + 1 };
        case 'DECREMENT':
            return { count: state.count - 1 };
        case 'RESET':
            return { count: 0 };
        default:
            return state;
    }
};

// Context
interface CounterContextType {
    state: CounterState;
    dispatch: React.Dispatch<CounterAction>;
}

const CounterContext = createContext<CounterContextType | undefined>(undefined);

// Provider
export const CounterProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const [state, dispatch] = useReducer(counterReducer, { count: 0 });

    return (
        <CounterContext.Provider value={{ state, dispatch }}>
            {children}
        </CounterContext.Provider>
    );
};

// Hook
export const useCounter = (): CounterContextType => {
    const context = useContext(CounterContext);
    if (!context) {
        throw new Error('useCounter must be used within CounterProvider');
    }
    return context;
};
```

## When to Choose Context API Over Redux

### 1. **Small to Medium Applications**

**Context API is ideal when:**
- Application has simple state management needs
- State is mostly local to a component tree
- No complex state interactions between distant components
- Team prefers built-in React solutions

```typescript
// ✅ Good for Context API: Theme management
const App = () => (
    <ThemeProvider>
        <AuthProvider>
            <Router>
                <Header />
                <MainContent />
                <Footer />
            </Router>
        </AuthProvider>
    </ThemeProvider>
);
```

### 2. **Static or Rarely Changing Data**

**Perfect for:**
- User authentication state
- Theme preferences
- Language/locale settings
- Feature flags
- App configuration

```typescript
// ✅ Good: User authentication context
interface AuthContextType {
    user: User | null;
    login: (credentials: LoginCredentials) => Promise<void>;
    logout: () => void;
    isLoading: boolean;
}

const AuthProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const [user, setUser] = useState<User | null>(null);
    const [isLoading, setIsLoading] = useState(true);

    useEffect(() => {
        // Check for existing session
        checkAuthStatus();
    }, []);

    const login = async (credentials: LoginCredentials) => {
        setIsLoading(true);
        try {
            const response = await api.login(credentials);
            setUser(response.user);
        } finally {
            setIsLoading(false);
        }
    };

    const logout = () => {
        setUser(null);
        // Clear tokens, etc.
    };

    return (
        <AuthContext.Provider value={{ user, login, logout, isLoading }}>
            {children}
        </AuthContext.Provider>
    );
};
```

### 3. **Component-Specific State**

**When state is:**
- Only used within a specific component subtree
- Doesn't need to be accessed globally
- Simple and predictable

```typescript
// ✅ Good: Form state management
const FormProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const [formData, setFormData] = useState<FormData>({});
    const [errors, setErrors] = useState<FormErrors>({});
    const [isSubmitting, setIsSubmitting] = useState(false);

    const updateField = (field: string, value: any) => {
        setFormData(prev => ({ ...prev, [field]: value }));
        // Clear error when field is updated
        if (errors[field]) {
            setErrors(prev => ({ ...prev, [field]: undefined }));
        }
    };

    const validateForm = (): boolean => {
        const newErrors: FormErrors = {};
        // Validation logic
        setErrors(newErrors);
        return Object.keys(newErrors).length === 0;
    };

    const submitForm = async () => {
        if (!validateForm()) return;

        setIsSubmitting(true);
        try {
            await api.submitForm(formData);
            // Handle success
        } finally {
            setIsSubmitting(false);
        }
    };

    return (
        <FormContext.Provider value={{
            formData,
            errors,
            isSubmitting,
            updateField,
            submitForm,
        }}>
            {children}
        </FormContext.Provider>
    );
};
```

### 4. **Prototyping and MVPs**

**Context API advantages for quick development:**
- No additional dependencies
- Built into React
- Faster setup
- Less boilerplate

```typescript
// Quick prototype setup
const AppStateProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const [state, setState] = useState({
        user: null,
        theme: 'light',
        notifications: [],
    });

    const updateState = (updates: Partial<typeof state>) => {
        setState(prev => ({ ...prev, ...updates }));
    };

    return (
        <AppContext.Provider value={{ state, updateState }}>
            {children}
        </AppContext.Provider>
    );
};
```

### 5. **When Redux Would Be Overkill**

**Choose Context API when:**
- No complex async logic (thunks, sagas)
- No need for advanced debugging (Redux DevTools)
- No time-travel debugging requirements
- No complex state selectors or memoization needs
- Team is small and prefers simpler solutions

## Performance Considerations

### 1. **Context API Re-renders**

**Context API triggers re-renders when:**
- Provider value changes (reference equality)
- Any component consuming the context re-renders

```typescript
// ❌ Bad: New object created on every render
const AppProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const [user, setUser] = useState(null);

    // This creates a new object every render, causing all consumers to re-render
    const value = {
        user,
        setUser,
        logout: () => setUser(null),
        login: (userData) => setUser(userData),
    };

    return (
        <AppContext.Provider value={value}>
            {children}
        </AppContext.Provider>
    );
};

// ✅ Good: Stable references with useCallback/useMemo
const AppProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const [user, setUser] = useState(null);

    const logout = useCallback(() => setUser(null), []);
    const login = useCallback((userData) => setUser(userData), []);

    const value = useMemo(() => ({
        user,
        setUser,
        logout,
        login,
    }), [user, setUser, logout, login]);

    return (
        <AppContext.Provider value={value}>
            {children}
        </AppContext.Provider>
    );
};
```

### 2. **Splitting Contexts**

**For better performance:**
- Split unrelated state into separate contexts
- Use multiple providers for different concerns

```typescript
// ✅ Good: Separate contexts for different concerns
const ThemeProvider = ({ children }) => (
    <ThemeContext.Provider value={themeValue}>
        {children}
    </ThemeContext.Provider>
);

const AuthProvider = ({ children }) => (
    <AuthContext.Provider value={authValue}>
        {children}
    </AuthContext.Provider>
);

// Usage
const App = () => (
    <ThemeProvider>
        <AuthProvider>
            <AppContent />
        </AuthProvider>
    </ThemeProvider>
);
```

## Migration from Context API to Redux

### When to Consider Migration:

1. **State becomes complex** - Multiple related pieces of state
2. **Performance issues** - Too many re-renders
3. **Debugging difficulties** - Hard to track state changes
4. **Team growth** - Need better developer tools
5. **Async operations** - Complex side effects

### Migration Strategy:

```typescript
// Before: Context API
const useTodos = () => {
    const { todos, addTodo, toggleTodo } = useContext(TodosContext);
    // ...
};

// After: Redux
const useTodos = () => {
    const todos = useSelector(state => state.todos.items);
    const dispatch = useDispatch();

    return {
        todos,
        addTodo: (text) => dispatch(addTodo(text)),
        toggleTodo: (id) => dispatch(toggleTodo(id)),
    };
};
```

## Real-World Examples

### 1. **E-commerce App State**

```typescript
// Context API for user session and cart
const UserProvider = ({ children }) => {
    const [user, setUser] = useState(null);
    const [cart, setCart] = useState([]);

    // Simple cart operations
    const addToCart = (item) => setCart(prev => [...prev, item]);
    const removeFromCart = (id) => setCart(prev => prev.filter(item => item.id !== id));

    return (
        <UserContext.Provider value={{ user, setUser, cart, addToCart, removeFromCart }}>
            {children}
        </UserContext.Provider>
    );
};

// Redux for complex product catalog, filters, search
// (products, filters, search state, pagination, etc.)
```

### 2. **Dashboard Application**

```typescript
// Context API for:
- User preferences (theme, language)
- Authentication state
- UI state (sidebar open/closed, active tab)

// Redux for:
- Dashboard data (charts, metrics, tables)
- Real-time updates
- Complex filtering and sorting
- Data caching and synchronization
```

### 3. **Content Management System**

```typescript
// Context API for:
- Current user and permissions
- UI state (modals, drawers)
- Form state for simple forms

// Redux for:
- Content entities (pages, posts, media)
- Complex form state with validation
- Bulk operations
- Undo/redo functionality
```

## Testing Context API

```typescript
import { render, screen } from '@testing-library/react';
import { ThemeProvider, useTheme } from './ThemeContext';

const TestComponent = () => {
    const { theme, toggleTheme } = useTheme();
    return (
        <div>
            <span>Current theme: {theme}</span>
            <button onClick={toggleTheme}>Toggle</button>
        </div>
    );
};

describe('ThemeContext', () => {
    it('provides theme context to children', () => {
        render(
            <ThemeProvider>
                <TestComponent />
            </ThemeProvider>
        );

        expect(screen.getByText('Current theme: light')).toBeInTheDocument();
    });

    it('allows theme toggling', () => {
        render(
            <ThemeProvider>
                <TestComponent />
            </ThemeProvider>
        );

        const button = screen.getByText('Toggle');
        fireEvent.click(button);

        expect(screen.getByText('Current theme: dark')).toBeInTheDocument();
    });
});
```

## Best Practices

### 1. **Context API Best Practices**

```typescript
// ✅ Good: TypeScript interfaces
interface ContextType {
    state: State;
    actions: Actions;
}

// ✅ Good: Custom hooks
const useMyContext = () => {
    const context = useContext(MyContext);
    if (!context) {
        throw new Error('useMyContext must be used within MyProvider');
    }
    return context;
};

// ✅ Good: Separate providers
const AppProviders = ({ children }) => (
    <ThemeProvider>
        <AuthProvider>
            <DataProvider>
                {children}
            </DataProvider>
        </AuthProvider>
    </ThemeProvider>
);
```

### 2. **When to Switch to Redux**

- App has 10+ components using the same state
- State logic becomes complex
- Need advanced debugging (time travel, action replay)
- Team needs better developer experience
- App requires complex async operations

## Common Interview Questions

### Q: When would you choose Context API over Redux?

**A:** I'd choose Context API for small to medium apps with simple state needs like theme, authentication, or form state. It's built into React, requires no extra dependencies, and is perfect when you don't need Redux's advanced features like middleware, dev tools, or complex async logic.

### Q: What's the main performance concern with Context API?

**A:** Context API can cause unnecessary re-renders if the provider value changes on every render. You need to use useMemo and useCallback to prevent this, or split contexts for different concerns.

### Q: Can Context API replace Redux entirely?

**A:** For simple apps, yes. But for complex applications with frequent state updates, async operations, or debugging needs, Redux provides better performance, developer tools, and maintainability.

### Q: How do you handle async operations with Context API?

**A:** You can use useEffect and async functions within your context provider, but for complex async logic, Redux with middleware like thunks or sagas is usually better.

## Summary

**Choose Context API when:**
- Small to medium application
- Simple, predictable state changes
- State is mostly static or rarely changes
- No complex async operations
- Prefer built-in React solutions
- Quick prototyping

**Choose Redux when:**
- Large, complex application
- Frequent state updates across many components
- Complex async operations
- Need advanced debugging tools
- Team requires predictable state management
- Time-travel debugging is valuable

**Migration Path:**
- Start with Context API for simplicity
- Switch to Redux when complexity grows
- Consider Redux Toolkit for easier Redux usage

**Interview Tip:** "I prefer Context API for simple state management like themes, authentication, or component-specific state. It's built into React and perfect for small apps. I'd switch to Redux when the app grows, needs complex async operations, or requires advanced debugging features."