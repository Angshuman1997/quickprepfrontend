# What is a Higher-Order Component (HOC)? Why are hooks preferred now?

## Question
What is a Higher-Order Component (HOC)? Why are hooks preferred now?

## Answer

A Higher-Order Component (HOC) is a pattern in React where a function takes a component and returns a new component with additional props or behavior. Hooks are now preferred over HOCs because they provide a simpler, more composable way to reuse logic without the complexity of component wrapping.

## What is a Higher-Order Component (HOC)?

An HOC is a function that:
- Takes a component as input
- Returns an enhanced component
- Adds functionality without modifying the original component
- Follows the pattern: `const EnhancedComponent = hoc(OriginalComponent)`

### Basic HOC Structure:

```javascript
// HOC function
function withLogging(WrappedComponent) {
    return function EnhancedComponent(props) {
        console.log('Component rendered with props:', props);
        return <WrappedComponent {...props} />;
    };
}

// Usage
const LoggedButton = withLogging(Button);
const LoggedInput = withLogging(Input);
```

### Common HOC Patterns:

#### 1. **Props Enhancement HOC**

```javascript
function withUserData(WrappedComponent) {
    return function UserDataComponent(props) {
        const [user, setUser] = useState(null);
        const [loading, setLoading] = useState(true);

        useEffect(() => {
            fetchUserData().then(userData => {
                setUser(userData);
                setLoading(false);
            });
        }, []);

        if (loading) return <div>Loading...</div>;

        return <WrappedComponent {...props} user={user} />;
    };
}

// Usage
function UserProfile({ user }) {
    return (
        <div>
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}

const UserProfileWithData = withUserData(UserProfile);
```

#### 2. **Conditional Rendering HOC**

```javascript
function withAuthentication(WrappedComponent) {
    return function AuthenticatedComponent(props) {
        const { user, loading } = useAuth();

        if (loading) return <div>Loading...</div>;
        if (!user) return <LoginPrompt />;

        return <WrappedComponent {...props} user={user} />;
    };
}

// Usage
function AdminPanel({ user }) {
    return <div>Welcome to admin panel, {user.name}!</div>;
}

const ProtectedAdminPanel = withAuthentication(AdminPanel);
```

#### 3. **Styling HOC**

```javascript
function withTheme(WrappedComponent) {
    return function ThemedComponent(props) {
        const theme = useTheme();

        return (
            <div style={{ backgroundColor: theme.background }}>
                <WrappedComponent {...props} theme={theme} />
            </div>
        );
    };
}
```

## Problems with HOCs

### 1. **Wrapper Hell**
HOCs create multiple layers of component wrapping:

```javascript
// Multiple HOCs lead to deeply nested components
const EnhancedComponent = withTheme(
    withAuthentication(
        withUserData(
            withLogging(MyComponent)
        )
    )
);

// Results in: withTheme(withAuthentication(withUserData(withLogging(MyComponent))))
```

### 2. **Props Collision**
Different HOCs might inject props with the same name:

```javascript
function withUserData(Component) {
    return function(props) {
        return <Component {...props} user={userData} />;
    };
}

function withCurrentUser(Component) {
    return function(props) {
        return <Component {...props} user={currentUser} />;
    };
}

// Problem: both inject 'user' prop
const Component = withCurrentUser(withUserData(MyComponent));
```

### 3. **Ref Forwarding Issues**
Refs don't automatically forward through HOCs:

```javascript
const EnhancedInput = withValidation(Input);

// This won't work as expected
const inputRef = useRef();
<EnhancedInput ref={inputRef} />; // ref won't reach the actual Input
```

### 4. **Static Method Loss**
HOCs don't automatically copy static methods:

```javascript
class MyComponent extends React.Component {
    static getData() { /* ... */ }
}

// HOC doesn't preserve static methods
const EnhancedComponent = withData(MyComponent);
EnhancedComponent.getData(); // undefined
```

### 5. **Complex Debugging**
Component tree becomes hard to debug with multiple HOC layers.

## Why Hooks are Preferred

Hooks solve all the HOC problems by providing direct access to React features without component wrapping.

### 1. **No Wrapper Components**

```javascript
// Instead of HOC
function useUserData() {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        fetchUserData().then(userData => {
            setUser(userData);
            setLoading(false);
        });
    }, []);

    return { user, loading };
}

// Usage - no wrapping needed
function UserProfile() {
    const { user, loading } = useUserData();

    if (loading) return <div>Loading...</div>;

    return (
        <div>
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}
```

### 2. **No Props Collision**

```javascript
// Multiple hooks can coexist without conflicts
function MyComponent() {
    const { user: userData } = useUserData();
    const { user: currentUser } = useCurrentUser();
    const theme = useTheme();

    // No naming conflicts
    return <div>...</div>;
}
```

### 3. **Better Ref Handling**

```javascript
function useForwardedRef(ref) {
    const innerRef = useRef();

    useEffect(() => {
        if (ref) {
            if (typeof ref === 'function') {
                ref(innerRef.current);
            } else {
                ref.current = innerRef.current;
            }
        }
    });

    return innerRef;
}

function MyInput(props, ref) {
    const innerRef = useForwardedRef(ref);

    return <input {...props} ref={innerRef} />;
}

const ForwardedInput = forwardRef(MyInput);
```

### 4. **Static Methods Preserved**

```javascript
// Hooks don't affect component structure
class MyComponent extends React.Component {
    static getData() { /* ... */ }

    render() {
        const { user } = useUserData(); // Hook usage doesn't break statics
        return <div>...</div>;
    }
}

MyComponent.getData(); // Still works
```

### 5. **Easier Testing**

```javascript
// Testing hooks is straightforward
import { renderHook } from '@testing-library/react-hooks';

test('useUserData fetches user data', async () => {
    const { result, waitForNextUpdate } = renderHook(() => useUserData());

    expect(result.current.loading).toBe(true);

    await waitForNextUpdate();

    expect(result.current.loading).toBe(false);
    expect(result.current.user).toBeDefined();
});
```

## Real React Examples

### 1. **Converting HOC to Hook**

```javascript
// Old HOC approach
function withWindowSize(WrappedComponent) {
    return class WindowSizeComponent extends React.Component {
        state = { width: window.innerWidth, height: window.innerHeight };

        componentDidMount() {
            window.addEventListener('resize', this.handleResize);
        }

        componentWillUnmount() {
            window.removeEventListener('resize', this.handleResize);
        }

        handleResize = () => {
            this.setState({
                width: window.innerWidth,
                height: window.innerHeight
            });
        };

        render() {
            return (
                <WrappedComponent
                    {...this.props}
                    windowWidth={this.state.width}
                    windowHeight={this.state.height}
                />
            );
        }
    };
}

// New Hook approach
function useWindowSize() {
    const [size, setSize] = useState({
        width: window.innerWidth,
        height: window.innerHeight
    });

    useEffect(() => {
        const handleResize = () => {
            setSize({
                width: window.innerWidth,
                height: window.innerHeight
            });
        };

        window.addEventListener('resize', handleResize);
        return () => window.removeEventListener('resize', handleResize);
    }, []);

    return size;
}

// Usage
function ResponsiveComponent() {
    const { width, height } = useWindowSize();

    return (
        <div>
            <p>Window size: {width} x {height}</p>
            {width < 768 ? <MobileView /> : <DesktopView />}
        </div>
    );
}
```

### 2. **Complex HOC Composition vs Hook Composition**

```javascript
// HOC approach - complex composition
const withAuth = withAuthentication;
const withTheme = withThemeProvider;
const withData = withDataFetching;
const withLogging = withLogger;

const ComplexComponent = withAuth(
    withTheme(
        withData(
            withLogging(MyComponent)
        )
    )
);

// Hook approach - simple composition
function MyComponent() {
    const { user } = useAuth();
    const theme = useTheme();
    const { data, loading } = useData();
    useLogger('MyComponent rendered');

    // All logic in one place, easy to understand
    if (!user) return <Login />;
    if (loading) return <Spinner />;

    return (
        <div style={{ backgroundColor: theme.background }}>
            <DataDisplay data={data} />
        </div>
    );
}
```

### 3. **HOC with Multiple Parameters vs Configurable Hook**

```javascript
// HOC with configuration - cumbersome
function withApi(endpoint, options) {
    return function(WrappedComponent) {
        return function ApiComponent(props) {
            const [data, setData] = useState(null);
            // ... API logic
            return <WrappedComponent {...props} data={data} />;
        };
    };
}

// Usage requires creating multiple HOCs
const withUserApi = withApi('/users', { method: 'GET' });
const withPostsApi = withApi('/posts', { method: 'GET' });

const UserComponent = withUserApi(MyComponent);
const PostsComponent = withPostsApi(MyComponent);

// Hook approach - flexible configuration
function useApi(endpoint, options = {}) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        fetch(endpoint, options)
            .then(response => response.json())
            .then(setData)
            .catch(setError)
            .finally(() => setLoading(false));
    }, [endpoint, JSON.stringify(options)]);

    return { data, loading, error };
}

// Usage - same hook, different configurations
function UserComponent() {
    const { data, loading } = useApi('/users');
    // ...
}

function PostsComponent() {
    const { data, loading } = useApi('/posts');
    // ...
}
```

### 4. **HOC Ref Issues vs Hook Solution**

```javascript
// HOC approach - ref forwarding issues
function withValidation(WrappedComponent) {
    return class ValidationComponent extends React.Component {
        state = { isValid: true };

        validate = (value) => {
            // validation logic
        };

        render() {
            return (
                <WrappedComponent
                    {...this.props}
                    isValid={this.state.isValid}
                    onValidate={this.validate}
                />
            );
        }
    };
}

// Usage - ref doesn't work
const ValidatedInput = withValidation('input');
const inputRef = useRef();
<ValidatedInput ref={inputRef} />; // ref is lost

// Hook approach - proper ref handling
function useValidation(initialValue = '') {
    const [value, setValue] = useState(initialValue);
    const [isValid, setIsValid] = useState(true);
    const [error, setError] = useState('');

    const validate = useCallback((newValue) => {
        // validation logic
        const valid = newValue.length > 3;
        setIsValid(valid);
        setError(valid ? '' : 'Must be at least 3 characters');
        return valid;
    }, []);

    const onChange = useCallback((e) => {
        const newValue = e.target.value;
        setValue(newValue);
        validate(newValue);
    }, [validate]);

    return {
        value,
        isValid,
        error,
        onChange,
        validate
    };
}

function ValidatedInput(props, ref) {
    const { value, isValid, error, onChange } = useValidation();

    return (
        <div>
            <input
                {...props}
                ref={ref}
                value={value}
                onChange={onChange}
                className={isValid ? '' : 'invalid'}
            />
            {error && <span className="error">{error}</span>}
        </div>
    );
}

const ForwardedValidatedInput = forwardRef(ValidatedInput);

// Usage - ref works correctly
const inputRef = useRef();
<ForwardedValidatedInput ref={inputRef} />;
```

## Migration Strategy

### Converting HOCs to Hooks:

1. **Identify HOC logic**: Extract the logic that the HOC provides
2. **Create custom hook**: Convert the HOC logic into a custom hook
3. **Update components**: Replace HOC usage with hook usage
4. **Remove HOC**: Delete the HOC once all usages are converted

### Example Migration:

```javascript
// Before: HOC
const withCounter = (WrappedComponent) => {
    return class extends React.Component {
        state = { count: 0 };

        increment = () => {
            this.setState({ count: this.state.count + 1 });
        };

        render() {
            return (
                <WrappedComponent
                    {...this.props}
                    count={this.state.count}
                    increment={this.increment}
                />
            );
        }
    };
};

// After: Hook
const useCounter = (initialValue = 0) => {
    const [count, setCount] = useState(initialValue);

    const increment = useCallback(() => {
        setCount(c => c + 1);
    }, []);

    const decrement = useCallback(() => {
        setCount(c => c - 1);
    }, []);

    const reset = useCallback(() => {
        setCount(initialValue);
    }, [initialValue]);

    return { count, increment, decrement, reset };
};

// Usage migration
// Before: const CounterButton = withCounter(Button);
// After: function CounterButton() { const { count, increment } = useCounter(); ... }
```

## When to Still Use HOCs

While hooks are preferred, HOCs are still useful for:

1. **Library code**: When creating reusable component libraries
2. **Legacy code**: When maintaining existing HOC-based codebases
3. **Render props**: When you need to share code between different component types
4. **Complex prop manipulation**: When you need to transform props extensively

## Best Practices

### For Hooks:
- **Custom hooks**: Extract reusable logic into custom hooks
- **Single responsibility**: Each hook should do one thing well
- **Naming convention**: Prefix custom hooks with `use`
- **Dependencies**: Include all dependencies in useEffect/useCallback

### For HOCs:
- **Convention**: Use `with` prefix for HOC names
- **Composition**: Prefer composition over deep nesting
- **Display name**: Set `displayName` for better debugging
- **Forward refs**: Always forward refs properly

### Interview Tip:
*"HOCs are functions that wrap components to add functionality, but they create wrapper hell and prop collision issues. Hooks solve these problems by allowing direct access to React features without component wrapping, making code more composable and easier to test."*