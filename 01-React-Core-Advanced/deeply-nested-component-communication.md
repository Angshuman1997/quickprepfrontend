# How do you manage deeply nested component communication?

## Question
How do you manage deeply nested component communication?

## Answer

Managing communication between deeply nested components is a common challenge in React applications. As components get nested deeper, passing data through props (prop drilling) becomes cumbersome and error-prone. React provides several patterns and tools to handle this effectively.

## The Problem: Prop Drilling

When components are deeply nested, passing data through multiple levels becomes problematic:

```javascript
// ❌ Prop drilling - passing props through multiple levels
function App() {
    const [user, setUser] = useState({ name: 'John', role: 'admin' });

    return (
        <div>
            <Header user={user} />
        </div>
    );
}

function Header({ user }) {
    return (
        <div>
            <Navbar user={user} />
        </div>
    );
}

function Navbar({ user }) {
    return (
        <div>
            <UserMenu user={user} />
        </div>
    );
}

function UserMenu({ user }) {
    return (
        <div>
            <UserAvatar user={user} />
            <UserName user={user} />
        </div>
    );
}

function UserAvatar({ user }) {
    return <img src={user.avatar} alt={user.name} />;
}

function UserName({ user }) {
    return <span>{user.name}</span>;
}
```

## Solution 1: React Context API

Context provides a way to share data without explicitly passing props through every level:

```javascript
// Create context
const UserContext = createContext();

// Context provider
function UserProvider({ children }) {
    const [user, setUser] = useState({ name: 'John', role: 'admin' });

    const updateUser = (updates) => {
        setUser(prev => ({ ...prev, ...updates }));
    };

    const value = {
        user,
        setUser,
        updateUser
    };

    return (
        <UserContext.Provider value={value}>
            {children}
        </UserContext.Provider>
    );
}

// Custom hook for consuming context
function useUser() {
    const context = useContext(UserContext);
    if (!context) {
        throw new Error('useUser must be used within a UserProvider');
    }
    return context;
}

// Usage in deeply nested components
function UserAvatar() {
    const { user } = useUser();
    return <img src={user.avatar} alt={user.name} />;
}

function UserName() {
    const { user, updateUser } = useUser();

    const handleNameChange = () => {
        updateUser({ name: 'Jane' });
    };

    return (
        <div>
            <span>{user.name}</span>
            <button onClick={handleNameChange}>Change Name</button>
        </div>
    );
}

// App structure
function App() {
    return (
        <UserProvider>
            <Header />
        </UserProvider>
    );
}

function Header() {
    return (
        <div>
            <Navbar />
        </div>
    );
}

function Navbar() {
    return (
        <div>
            <UserMenu />
        </div>
    );
}

function UserMenu() {
    return (
        <div>
            <UserAvatar />
            <UserName />
        </div>
    );
}
```

## Solution 2: State Management Libraries

For complex applications, dedicated state management libraries provide better organization:

### Redux Toolkit

```javascript
// userSlice.js
import { createSlice } from '@reduxjs/toolkit';

const userSlice = createSlice({
    name: 'user',
    initialState: {
        name: 'John',
        role: 'admin',
        isLoggedIn: true
    },
    reducers: {
        updateUser: (state, action) => {
            return { ...state, ...action.payload };
        },
        logout: (state) => {
            state.isLoggedIn = false;
        }
    }
});

export const { updateUser, logout } = userSlice.actions;
export default userSlice.reducer;

// store.js
import { configureStore } from '@reduxjs/toolkit';
import userReducer from './userSlice';

export const store = configureStore({
    reducer: {
        user: userReducer
    }
});

// UserAvatar.jsx
import { useSelector } from 'react-redux';

function UserAvatar() {
    const user = useSelector(state => state.user);
    return <img src={user.avatar} alt={user.name} />;
}

// UserName.jsx
import { useSelector, useDispatch } from 'react-redux';
import { updateUser } from './userSlice';

function UserName() {
    const user = useSelector(state => state.user);
    const dispatch = useDispatch();

    const handleNameChange = () => {
        dispatch(updateUser({ name: 'Jane' }));
    };

    return (
        <div>
            <span>{user.name}</span>
            <button onClick={handleNameChange}>Change Name</button>
        </div>
    );
}
```

### Zustand (Lightweight Alternative)

```javascript
// useUserStore.js
import { create } from 'zustand';

const useUserStore = create((set, get) => ({
    user: {
        name: 'John',
        role: 'admin',
        isLoggedIn: true
    },
    updateUser: (updates) => set((state) => ({
        user: { ...state.user, ...updates }
    })),
    logout: () => set({ user: { ...get().user, isLoggedIn: false } })
}));

// Usage in components
function UserAvatar() {
    const user = useUserStore(state => state.user);
    return <img src={user.avatar} alt={user.name} />;
}

function UserName() {
    const { user, updateUser } = useUserStore();

    const handleNameChange = () => {
        updateUser({ name: 'Jane' });
    };

    return (
        <div>
            <span>{user.name}</span>
            <button onClick={handleNameChange}>Change Name</button>
        </div>
    );
}
```

## Solution 3: Component Composition with Render Props

Render props allow components to share logic while maintaining flexibility:

```javascript
function UserDataProvider({ children }) {
    const [user, setUser] = useState({ name: 'John', role: 'admin' });

    const updateUser = (updates) => {
        setUser(prev => ({ ...prev, ...updates }));
    };

    const value = {
        user,
        setUser,
        updateUser
    };

    return children(value);
}

// Usage
function App() {
    return (
        <UserDataProvider>
            {({ user, updateUser }) => (
                <div>
                    <Header user={user} updateUser={updateUser} />
                </div>
            )}
        </UserDataProvider>
    );
}

function Header({ user, updateUser }) {
    return (
        <div>
            <UserMenu user={user} updateUser={updateUser} />
        </div>
    );
}

function UserMenu({ user, updateUser }) {
    return (
        <div>
            <UserAvatar user={user} />
            <UserName user={user} updateUser={updateUser} />
        </div>
    );
}
```

## Solution 4: Custom Hooks for Logic Sharing

Create custom hooks that encapsulate related logic:

```javascript
// useUserData.js
function useUserData() {
    const [user, setUser] = useState({ name: 'John', role: 'admin' });
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const updateUser = async (updates) => {
        setLoading(true);
        try {
            // Simulate API call
            await new Promise(resolve => setTimeout(resolve, 1000));
            setUser(prev => ({ ...prev, ...updates }));
            setError(null);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    };

    const logout = () => {
        setUser({ name: '', role: '', isLoggedIn: false });
    };

    return {
        user,
        loading,
        error,
        updateUser,
        logout
    };
}

// Usage in any component
function UserProfile() {
    const { user, loading, error, updateUser } = useUserData();

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <div>
            <h2>{user.name}</h2>
            <p>Role: {user.role}</p>
            <button onClick={() => updateUser({ name: 'Jane' })}>
                Change Name
            </button>
        </div>
    );
}

function UserSettings() {
    const { user, logout } = useUserData();

    return (
        <div>
            <p>Current user: {user.name}</p>
            <button onClick={logout}>Logout</button>
        </div>
    );
}
```

## Solution 5: Event Emitters / Pub-Sub Pattern

For complex component interactions that don't follow parent-child relationships:

```javascript
// eventEmitter.js
class EventEmitter {
    constructor() {
        this.events = {};
    }

    on(event, callback) {
        if (!this.events[event]) {
            this.events[event] = [];
        }
        this.events[event].push(callback);
    }

    off(event, callback) {
        if (!this.events[event]) return;
        this.events[event] = this.events[event].filter(cb => cb !== callback);
    }

    emit(event, data) {
        if (!this.events[event]) return;
        this.events[event].forEach(callback => callback(data));
    }
}

const eventEmitter = new EventEmitter();

// Custom hook for event handling
function useEventEmitter() {
    const [events, setEvents] = useState({});

    useEffect(() => {
        const handleEvent = (data) => {
            setEvents(prev => ({ ...prev, ...data }));
        };

        eventEmitter.on('userUpdate', handleEvent);

        return () => {
            eventEmitter.off('userUpdate', handleEvent);
        };
    }, []);

    const emitUserUpdate = (userData) => {
        eventEmitter.emit('userUpdate', userData);
    };

    return { events, emitUserUpdate };
}

// Usage
function UserEditor() {
    const { emitUserUpdate } = useEventEmitter();

    const handleSave = () => {
        const updatedUser = { name: 'Jane', role: 'admin' };
        emitUserUpdate(updatedUser);
    };

    return (
        <div>
            <input placeholder="Name" />
            <button onClick={handleSave}>Save</button>
        </div>
    );
}

function UserDisplay() {
    const { events } = useEventEmitter();

    return (
        <div>
            <h2>User Info</h2>
            <p>Name: {events.name || 'John'}</p>
            <p>Role: {events.role || 'admin'}</p>
        </div>
    );
}
```

## Solution 6: React Query / SWR for Server State

For server state management across deeply nested components:

```javascript
// useUser.js
import { useQuery, useMutation, useQueryClient } from 'react-query';

function useUser(userId) {
    return useQuery(['user', userId], () =>
        fetch(`/api/users/${userId}`).then(res => res.json())
    );
}

function useUpdateUser() {
    const queryClient = useQueryClient();

    return useMutation(
        (updates) => fetch('/api/users/update', {
            method: 'POST',
            body: JSON.stringify(updates)
        }),
        {
            onSuccess: () => {
                queryClient.invalidateQueries('user');
            }
        }
    );
}

// Usage in deeply nested components
function UserProfile({ userId }) {
    const { data: user, isLoading } = useUser(userId);
    const updateUserMutation = useUpdateUser();

    if (isLoading) return <div>Loading...</div>;

    const handleUpdate = () => {
        updateUserMutation.mutate({ name: 'Jane' });
    };

    return (
        <div>
            <h2>{user.name}</h2>
            <button onClick={handleUpdate}>Update Name</button>
        </div>
    );
}

function UserAvatar({ userId }) {
    const { data: user } = useUser(userId);
    return <img src={user?.avatar} alt={user?.name} />;
}
```

## Advanced Patterns

### 1. **Compound Components with Context**

```javascript
const MenuContext = createContext();

function Menu({ children, onSelect }) {
    const [selectedItem, setSelectedItem] = useState(null);

    const selectItem = (item) => {
        setSelectedItem(item);
        onSelect?.(item);
    };

    return (
        <MenuContext.Provider value={{ selectedItem, selectItem }}>
            <ul className="menu">
                {children}
            </ul>
        </MenuContext.Provider>
    );
}

function MenuItem({ children, value }) {
    const { selectedItem, selectItem } = useContext(MenuContext);

    return (
        <li
            className={selectedItem === value ? 'selected' : ''}
            onClick={() => selectItem(value)}
        >
            {children}
        </li>
    );
}

// Usage
function App() {
    return (
        <Menu onSelect={(item) => console.log('Selected:', item)}>
            <MenuItem value="home">Home</MenuItem>
            <MenuItem value="about">About</MenuItem>
            <MenuItem value="contact">Contact</MenuItem>
        </Menu>
    );
}
```

### 2. **Reducer Pattern for Complex State**

```javascript
const initialState = {
    user: { name: 'John', role: 'admin' },
    settings: { theme: 'light', notifications: true },
    ui: { sidebarOpen: false, modalOpen: false }
};

function appReducer(state, action) {
    switch (action.type) {
        case 'UPDATE_USER':
            return { ...state, user: { ...state.user, ...action.payload } };
        case 'TOGGLE_SIDEBAR':
            return { ...state, ui: { ...state.ui, sidebarOpen: !state.ui.sidebarOpen } };
        case 'UPDATE_SETTINGS':
            return { ...state, settings: { ...state.settings, ...action.payload } };
        default:
            return state;
    }
}

const AppContext = createContext();

function AppProvider({ children }) {
    const [state, dispatch] = useReducer(appReducer, initialState);

    return (
        <AppContext.Provider value={{ state, dispatch }}>
            {children}
        </AppContext.Provider>
    );
}

// Custom hooks for specific domains
function useUser() {
    const { state, dispatch } = useContext(AppContext);
    return {
        user: state.user,
        updateUser: (updates) => dispatch({ type: 'UPDATE_USER', payload: updates })
    };
}

function useUI() {
    const { state, dispatch } = useContext(AppContext);
    return {
        ui: state.ui,
        toggleSidebar: () => dispatch({ type: 'TOGGLE_SIDEBAR' })
    };
}
```

### 3. **Portals for Modal Communication**

```javascript
function ModalProvider({ children }) {
    const [modals, setModals] = useState(new Map());

    const openModal = (id, component) => {
        setModals(prev => new Map(prev).set(id, component));
    };

    const closeModal = (id) => {
        setModals(prev => {
            const newModals = new Map(prev);
            newModals.delete(id);
            return newModals;
        });
    };

    const contextValue = {
        openModal,
        closeModal
    };

    return (
        <ModalContext.Provider value={contextValue}>
            {children}
            {Array.from(modals.entries()).map(([id, Component]) => (
                <Modal key={id} onClose={() => closeModal(id)}>
                    <Component />
                </Modal>
            ))}
        </ModalContext.Provider>
    );
}

// Usage in deeply nested components
function DeeplyNestedButton() {
    const { openModal } = useContext(ModalContext);

    const handleClick = () => {
        openModal('user-settings', UserSettingsModal);
    };

    return <button onClick={handleClick}>Open Settings</button>;
}
```

## Best Practices

### 1. **Choose the Right Tool**

- **Simple prop passing**: For 1-2 levels deep
- **Context API**: For theme, user auth, app settings
- **Redux/Zustand**: For complex state with multiple consumers
- **Custom hooks**: For reusable logic across components
- **Event emitters**: For cross-component communication without hierarchy

### 2. **Context Splitting**

```javascript
// ❌ Bad: Single large context
const AppContext = createContext();

// ✅ Good: Split contexts by domain
const UserContext = createContext();
const ThemeContext = createContext();
const NotificationContext = createContext();
```

### 3. **Performance Optimization**

```javascript
// Use memoization for context values
const UserContext = createContext();

function UserProvider({ children }) {
    const [user, setUser] = useState({ name: 'John' });

    const value = useMemo(() => ({
        user,
        setUser
    }), [user]);

    return (
        <UserContext.Provider value={value}>
            {children}
        </UserContext.Provider>
    );
}
```

### 4. **Error Boundaries**

```javascript
class ErrorBoundary extends Component {
    constructor(props) {
        super(props);
        this.state = { hasError: false };
    }

    static getDerivedStateFromError(error) {
        return { hasError: true };
    }

    componentDidCatch(error, errorInfo) {
        console.error('Context error:', error, errorInfo);
    }

    render() {
        if (this.state.hasError) {
            return <h1>Something went wrong.</h1>;
        }

        return this.props.children;
    }
}

// Wrap context providers with error boundaries
function App() {
    return (
        <ErrorBoundary>
            <UserProvider>
                <ThemeProvider>
                    <AppContent />
                </ThemeProvider>
            </UserProvider>
        </ErrorBoundary>
    );
}
```

### 5. **Testing Context Consumers**

```javascript
// Custom render function for testing
const renderWithProviders = (component, initialState = {}) => {
    const defaultState = {
        user: { name: 'Test User' },
        ...initialState
    };

    return render(
        <UserProvider initialState={defaultState}>
            {component}
        </UserProvider>
    );
};

// Test usage
test('renders user name', () => {
    renderWithProviders(<UserName />);
    expect(screen.getByText('Test User')).toBeInTheDocument();
});
```

## Real-World Examples

### 1. **E-commerce Product Management**

```javascript
// Context for cart management
const CartContext = createContext();

function CartProvider({ children }) {
    const [cart, setCart] = useState([]);

    const addToCart = (product) => {
        setCart(prev => [...prev, product]);
    };

    const removeFromCart = (productId) => {
        setCart(prev => prev.filter(item => item.id !== productId));
    };

    return (
        <CartContext.Provider value={{ cart, addToCart, removeFromCart }}>
            {children}
        </CartContext.Provider>
    );
}

// Usage in deeply nested components
function ProductCard({ product }) {
    const { addToCart } = useContext(CartContext);

    return (
        <div>
            <h3>{product.name}</h3>
            <button onClick={() => addToCart(product)}>
                Add to Cart
            </button>
        </div>
    );
}

function CartSummary() {
    const { cart, removeFromCart } = useContext(CartContext);

    return (
        <div>
            <h2>Cart ({cart.length} items)</h2>
            {cart.map(item => (
                <div key={item.id}>
                    {item.name}
                    <button onClick={() => removeFromCart(item.id)}>
                        Remove
                    </button>
                </div>
            ))}
        </div>
    );
}
```

### 2. **Dashboard with Multiple Widgets**

```javascript
const DashboardContext = createContext();

function DashboardProvider({ children }) {
    const [widgets, setWidgets] = useState([
        { id: 1, type: 'chart', visible: true },
        { id: 2, type: 'table', visible: true }
    ]);

    const toggleWidget = (id) => {
        setWidgets(prev => prev.map(widget =>
            widget.id === id
                ? { ...widget, visible: !widget.visible }
                : widget
        ));
    };

    return (
        <DashboardContext.Provider value={{ widgets, toggleWidget }}>
            {children}
        </DashboardContext.Provider>
    );
}

function WidgetControls() {
    const { widgets, toggleWidget } = useContext(DashboardContext);

    return (
        <div>
            {widgets.map(widget => (
                <label key={widget.id}>
                    <input
                        type="checkbox"
                        checked={widget.visible}
                        onChange={() => toggleWidget(widget.id)}
                    />
                    {widget.type}
                </label>
            ))}
        </div>
    );
}

function Dashboard() {
    const { widgets } = useContext(DashboardContext);

    return (
        <div>
            <WidgetControls />
            {widgets
                .filter(widget => widget.visible)
                .map(widget => (
                    <Widget key={widget.id} type={widget.type} />
                ))
            }
        </div>
    );
}
```

### Interview Tip:
*"For deeply nested component communication, start with Context API for simple cases, use Redux/Zustand for complex state management, and consider custom hooks for reusable logic. Always split contexts by domain and use memoization for performance."*