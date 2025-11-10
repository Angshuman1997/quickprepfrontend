# Difference between state, props, and context

## Question
Difference between state, props, and context.

## Answer

State, props, and context are three fundamental concepts in React for managing data and component communication. Each serves different purposes and has different use cases.

## State

State represents data that can change over time within a component. It's managed internally by the component and can be updated using state setters.

### Characteristics:
- **Mutable**: Can be changed within the component
- **Local**: Only accessible within the component that owns it
- **Triggers re-renders**: When state changes, component re-renders
- **Private**: Not accessible from parent or child components directly

### When to use:
- User input (form fields, checkboxes)
- UI state (dropdown open/closed, modal visibility)
- Data that changes over time (counters, timers)
- Component-specific data

### Example:

```javascript
import { useState } from 'react';

function Counter() {
    const [count, setCount] = useState(0);
    const [isVisible, setIsVisible] = useState(true);

    return (
        <div>
            {isVisible && <p>Count: {count}</p>}
            <button onClick={() => setCount(count + 1)}>Increment</button>
            <button onClick={() => setIsVisible(!isVisible)}>
                {isVisible ? 'Hide' : 'Show'}
            </button>
        </div>
    );
}
```

## Props

Props (properties) are read-only data passed from parent components to child components. They allow parent components to configure child components.

### Characteristics:
- **Immutable**: Cannot be modified by the receiving component
- **One-way data flow**: Data flows from parent to child
- **External data**: Comes from parent component
- **Required/Optional**: Can be marked as required or have defaults

### When to use:
- Configuration data (styling, behavior settings)
- Data that needs to be displayed
- Event handlers passed from parent
- Component customization

### Example:

```javascript
function UserCard({ name, age, onEdit, isAdmin = false }) {
    return (
        <div className="user-card">
            <h3>{name}</h3>
            <p>Age: {age}</p>
            {isAdmin && <span>Admin</span>}
            <button onClick={() => onEdit(name)}>Edit</button>
        </div>
    );
}

function UserList() {
    const [users, setUsers] = useState([
        { name: 'John', age: 25 },
        { name: 'Jane', age: 30 }
    ]);

    const handleEdit = (userName) => {
        console.log('Editing user:', userName);
    };

    return (
        <div>
            {users.map(user => (
                <UserCard
                    key={user.name}
                    name={user.name}
                    age={user.age}
                    onEdit={handleEdit}
                    isAdmin={user.name === 'John'}
                />
            ))}
        </div>
    );
}
```

## Context

Context provides a way to share data across the component tree without having to pass props down manually at every level (prop drilling).

### Characteristics:
- **Global state**: Accessible by any component in the tree
- **Provider/Consumer pattern**: Provider supplies data, consumers use it
- **Cross-component communication**: Shares data across multiple components
- **Subscription-based**: Components subscribe to context changes

### When to use:
- Theme data (dark/light mode, colors)
- User authentication state
- Language/locale settings
- Global application state
- Deep component trees where prop drilling is problematic

### Example:

```javascript
import { createContext, useContext, useState } from 'react';

// Create context
const ThemeContext = createContext();

// Provider component
function ThemeProvider({ children }) {
    const [theme, setTheme] = useState('light');

    const toggleTheme = () => {
        setTheme(prev => prev === 'light' ? 'dark' : 'light');
    };

    const value = {
        theme,
        toggleTheme
    };

    return (
        <ThemeContext.Provider value={value}>
            {children}
        </ThemeContext.Provider>
    );
}

// Consumer hook
function useTheme() {
    const context = useContext(ThemeContext);
    if (!context) {
        throw new Error('useTheme must be used within a ThemeProvider');
    }
    return context;
}

// Usage
function App() {
    return (
        <ThemeProvider>
            <Header />
            <Content />
        </ThemeProvider>
    );
}

function Header() {
    const { theme, toggleTheme } = useTheme();

    return (
        <header className={theme}>
            <h1>My App</h1>
            <button onClick={toggleTheme}>
                Switch to {theme === 'light' ? 'Dark' : 'Light'} Mode
            </button>
        </header>
    );
}

function Content() {
    const { theme } = useTheme();

    return (
        <main className={theme}>
            <p>This content uses the {theme} theme.</p>
            <NestedComponent />
        </main>
    );
}

function NestedComponent() {
    const { theme } = useTheme();

    return (
        <div className={theme}>
            <p>Even deeply nested components can access theme!</p>
        </div>
    );
}
```

## Comparison Table

| Aspect | State | Props | Context |
|--------|-------|-------|---------|
| **Data Flow** | Internal to component | Parent → Child | Provider → Consumers |
| **Mutability** | Mutable | Immutable | Mutable (via Provider) |
| **Scope** | Component only | Direct parent-child | Component tree |
| **Updates** | Component methods | Parent re-render | Provider update |
| **Performance** | Local re-renders | Parent re-renders child | Selective re-renders |
| **Use Case** | UI state, form data | Configuration, data display | Global state, themes |

## Real React Examples

### 1. **E-commerce Product Page**

```javascript
// State: Product quantity, selected variant
// Props: Product data, add to cart handler
// Context: User authentication, shopping cart

const AuthContext = createContext();
const CartContext = createContext();

function ProductPage({ product }) {
    const [quantity, setQuantity] = useState(1);
    const [selectedVariant, setSelectedVariant] = useState(product.variants[0]);
    const { user } = useContext(AuthContext);
    const { addToCart } = useContext(CartContext);

    const handleAddToCart = () => {
        if (!user) {
            alert('Please login to add items to cart');
            return;
        }

        addToCart({
            productId: product.id,
            variant: selectedVariant,
            quantity
        });
    };

    return (
        <div>
            <h1>{product.name}</h1> {/* Props */}
            <p>{product.description}</p> {/* Props */}

            <select
                value={selectedVariant.id}
                onChange={(e) => setSelectedVariant(
                    product.variants.find(v => v.id === e.target.value)
                )}
            >
                {product.variants.map(variant => (
                    <option key={variant.id} value={variant.id}>
                        {variant.name} - ${variant.price}
                    </option>
                ))}
            </select> {/* State */}

            <input
                type="number"
                value={quantity}
                onChange={(e) => setQuantity(Number(e.target.value))}
                min="1"
            /> {/* State */}

            <button onClick={handleAddToCart}>
                Add to Cart
            </button> {/* Context */}
        </div>
    );
}
```

### 2. **Dashboard with Multiple Components**

```javascript
// State: Individual component states (filters, sorting)
// Props: Component configuration, data
// Context: Global app state (sidebar open, theme, user)

const AppContext = createContext();

function Dashboard() {
    const [sidebarOpen, setSidebarOpen] = useState(true);
    const [theme, setTheme] = useState('light');
    const [user, setUser] = useState({ name: 'John Doe', role: 'admin' });

    const contextValue = {
        sidebarOpen,
        setSidebarOpen,
        theme,
        setTheme,
        user,
        setUser
    };

    return (
        <AppContext.Provider value={contextValue}>
            <div className={`dashboard ${theme}`}>
                <Sidebar />
                <MainContent />
            </div>
        </AppContext.Provider>
    );
}

function Sidebar() {
    const { sidebarOpen, setSidebarOpen } = useContext(AppContext);

    return (
        <aside className={sidebarOpen ? 'open' : 'closed'}>
            <button onClick={() => setSidebarOpen(!sidebarOpen)}>
                {sidebarOpen ? 'Close' : 'Open'} Sidebar
            </button>
            <Navigation />
        </aside>
    );
}

function MainContent() {
    return (
        <main>
            <Header />
            <StatsGrid />
            <DataTable data={[]} /> {/* Props */}
        </main>
    );
}

function Header() {
    const { user, theme, setTheme } = useContext(AppContext);

    return (
        <header>
            <h1>Welcome, {user.name}</h1> {/* Context */}
            <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
                Toggle Theme
            </button> {/* Context */}
        </header>
    );
}

function StatsGrid() {
    const [timeRange, setTimeRange] = useState('7d'); // State

    return (
        <div className="stats-grid">
            <select value={timeRange} onChange={(e) => setTimeRange(e.target.value)}>
                <option value="1d">Last 24 hours</option>
                <option value="7d">Last 7 days</option>
                <option value="30d">Last 30 days</option>
            </select> {/* State */}

            <StatCard title="Revenue" value="$12,345" timeRange={timeRange} /> {/* Props */}
            <StatCard title="Users" value="1,234" timeRange={timeRange} /> {/* Props */}
        </div>
    );
}

function StatCard({ title, value, timeRange }) { // Props
    const [isLoading, setIsLoading] = useState(false); // State

    useEffect(() => {
        setIsLoading(true);
        // Simulate API call
        setTimeout(() => setIsLoading(false), 1000);
    }, [timeRange]);

    return (
        <div className="stat-card">
            <h3>{title}</h3>
            {isLoading ? <div>Loading...</div> : <div className="value">{value}</div>}
        </div>
    );
}

function DataTable({ data }) { // Props
    const [sortBy, setSortBy] = useState('name'); // State
    const [sortOrder, setSortOrder] = useState('asc'); // State
    const [currentPage, setCurrentPage] = useState(1); // State

    const handleSort = (column) => {
        if (sortBy === column) {
            setSortOrder(sortOrder === 'asc' ? 'desc' : 'asc');
        } else {
            setSortBy(column);
            setSortOrder('asc');
        }
    };

    return (
        <table>
            <thead>
                <tr>
                    <th onClick={() => handleSort('name')}>
                        Name {sortBy === 'name' && (sortOrder === 'asc' ? '↑' : '↓')}
                    </th>
                    <th onClick={() => handleSort('email')}>
                        Email {sortBy === 'email' && (sortOrder === 'asc' ? '↑' : '↓')}
                    </th>
                </tr>
            </thead>
            <tbody>
                {data.map(item => (
                    <tr key={item.id}>
                        <td>{item.name}</td>
                        <td>{item.email}</td>
                    </tr>
                ))}
            </tbody>
        </table>
    );
}
```

### 3. **Form with Validation**

```javascript
// State: Form field values, validation errors, touched state
// Props: Initial values, submit handler, validation rules
// Context: Form theme, global validation settings

const FormContext = createContext();

function Form({ initialValues, onSubmit, validationRules }) {
    const [values, setValues] = useState(initialValues);
    const [errors, setErrors] = useState({});
    const [touched, setTouched] = useState({});
    const [isSubmitting, setIsSubmitting] = useState(false);

    const contextValue = {
        values,
        setValues,
        errors,
        setErrors,
        touched,
        setTouched,
        validationRules
    };

    const handleSubmit = async (e) => {
        e.preventDefault();
        setIsSubmitting(true);

        try {
            await onSubmit(values);
        } catch (error) {
            console.error('Submit failed:', error);
        } finally {
            setIsSubmitting(false);
        }
    };

    return (
        <FormContext.Provider value={contextValue}>
            <form onSubmit={handleSubmit}>
                {/* Form fields will use context */}
                <FormField name="email" type="email" />
                <FormField name="password" type="password" />
                <button type="submit" disabled={isSubmitting}>
                    {isSubmitting ? 'Submitting...' : 'Submit'}
                </button>
            </form>
        </FormContext.Provider>
    );
}

function FormField({ name, type }) {
    const { values, setValues, errors, setErrors, touched, setTouched, validationRules } = useContext(FormContext);

    const handleChange = (e) => {
        const { value } = e.target;
        setValues(prev => ({ ...prev, [name]: value }));

        // Clear error when user starts typing
        if (errors[name]) {
            setErrors(prev => ({ ...prev, [name]: '' }));
        }
    };

    const handleBlur = () => {
        setTouched(prev => ({ ...prev, [name]: true }));

        // Validate field
        const rule = validationRules[name];
        if (rule && rule.required && !values[name]) {
            setErrors(prev => ({ ...prev, [name]: `${name} is required` }));
        }
    };

    return (
        <div>
            <input
                type={type}
                value={values[name] || ''}
                onChange={handleChange}
                onBlur={handleBlur}
                placeholder={name}
            />
            {touched[name] && errors[name] && (
                <span className="error">{errors[name]}</span>
            )}
        </div>
    );
}

// Usage
function LoginForm() {
    const handleSubmit = async (values) => {
        console.log('Login:', values);
    };

    return (
        <Form
            initialValues={{ email: '', password: '' }}
            onSubmit={handleSubmit}
            validationRules={{
                email: { required: true },
                password: { required: true }
            }}
        />
    );
}
```

## Best Practices

### State:
- Keep state local when possible
- Lift state up only when needed by multiple components
- Use functional updates for state based on previous state
- Consider performance impact of frequent state updates

### Props:
- Use prop destructuring for cleaner code
- Provide default values for optional props
- Use PropTypes or TypeScript for type checking
- Avoid deeply nested prop structures

### Context:
- Don't overuse context - prefer props for simple cases
- Split contexts for different concerns (theme, auth, etc.)
- Use context selectors to prevent unnecessary re-renders
- Provide default values to prevent runtime errors

### Performance Considerations:
- State changes cause re-renders - minimize unnecessary updates
- Props changes trigger child re-renders - use memo when appropriate
- Context changes affect all consumers - use multiple contexts for different data

### Interview Tip:
*"State is for component-internal data that changes, props are for parent-to-child data flow, and context is for global data that needs to be accessed by multiple components without prop drilling. Choose based on scope and data flow needs."*