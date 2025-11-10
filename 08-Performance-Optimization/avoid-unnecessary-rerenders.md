# How to Avoid Unnecessary Re-renders in React

## Question
How do you avoid unnecessary re-renders in React?

## Answer

React re-renders components when their state or props change. But sometimes, components re-render even when nothing has actually changed in a way that affects what the user sees. This wastes performance. The goal is to prevent re-renders when the component's output stays the same.

## Why Re-renders Happen

React re-renders a component when:
- Its state changes (like with useState)
- Its props change (from parent component)
- Context values change
- Its parent re-renders
- You force it to update

Each re-render costs time because React has to:
- Check what changed (reconciliation)
- Run the component's code again
- Update the actual webpage (DOM)

## Main Ways to Prevent Unnecessary Re-renders

### 1. Use React.memo to Memoize Components

React.memo prevents a component from re-rendering if its props haven't changed.

**Simple Example:**
```typescript
// Without memo - re-renders every time parent does
function UserCard({ user, onClick }) {
    return <div onClick={onClick}>{user.name}</div>;
}

// With memo - only re-renders if props actually change
const UserCard = React.memo(function UserCard({ user, onClick }) {
    return <div onClick={onClick}>{user.name}</div>;
});
```

**When to use:** Wrap components that receive the same props often but don't need to update.

### 2. Use useCallback for Functions

Functions created in components get new references each render. This makes child components think props changed.

**Problem:**
```typescript
function Parent() {
    const [count, setCount] = useState(0);
    const handleClick = () => setCount(c => c + 1); // New function each render

    return <Child onClick={handleClick} />; // Child sees new prop each time
}

const Child = React.memo(() => <button onClick={onClick}>Click</button>);
```

**Solution:**
```typescript
function Parent() {
    const [count, setCount] = useState(0);
    const handleClick = useCallback(() => {
        setCount(c => c + 1);
    }, []); // Same function reference always

    return <Child onClick={handleClick} />;
}
```

**When to use:** For event handlers passed to memoized children.

### 3. Use useMemo for Expensive Calculations

useMemo remembers the result of a calculation and only re-runs it when dependencies change.

**Example:**
```typescript
function UserList({ users, filter }) {
    // Without memo - filters every render
    const filtered = users.filter(user =>
        user.name.includes(filter)
    );

    return <ul>{filtered.map(user => <li>{user.name}</li>)}</ul>;
}

// With memo - only filters when users or filter change
function UserList({ users, filter }) {
    const filtered = useMemo(() =>
        users.filter(user => user.name.includes(filter)),
        [users, filter]
    );

    return <ul>{filtered.map(user => <li>{user.name}</li>)}</ul>;
}
```

**When to use:** For computations that take time or happen often.

### 4. Split Large Components

Break big components into smaller ones. Only the changed parts re-render.

**Before:**
```typescript
function Dashboard({ data, user }) {
    return (
        <div>
            <Header user={user} />  {/* Re-renders when data changes */}
            <Table data={data} />   {/* Re-renders when user changes */}
        </div>
    );
}
```

**After:**
```typescript
const Header = React.memo(({ user }) => <h1>{user.name}</h1>);
const Table = React.memo(({ data }) => <table>{/* ... */}</table>);

function Dashboard({ data, user }) {
    return (
        <div>
            <Header user={user} />
            <Table data={data} />
        </div>
    );
}
```

### 5. Move State Closer to Where It's Used

Keep state in the component that needs it, not high up in the tree.

**Example:** If only a search box uses search state, put the state there instead of in App.

### 6. Optimize Context Usage

Context changes cause all consumers to re-render. Split contexts or memoize values.

**Split contexts:**
```typescript
const ThemeContext = createContext();
const UserContext = createContext();

function App() {
    return (
        <ThemeContext.Provider value={theme}>
            <UserContext.Provider value={user}>
                <Header /> {/* Only re-renders on theme change */}
                <Content /> {/* Only re-renders on user change */}
            </UserContext.Provider>
        </ThemeContext.Provider>
    );
}
```

## Advanced Tips

### Custom Hooks for Logic Reuse

Put stateful logic in custom hooks to avoid duplication.

### List Keys Matter

Use stable, unique keys for list items. Don't use array index if items can reorder.

### Debugging Tools

- React DevTools Profiler shows what re-renders and why
- Add console.log to see when components render
- Use a custom hook to track renders

## Common Questions

**Q: useMemo vs useCallback?**
- useMemo: Cache calculation results
- useCallback: Cache function references

**Q: When NOT to optimize?**
- For simple components that render fast
- When optimization costs more than the re-render

**Q: Most common cause of extra re-renders?**
- New function objects passed as props
- New object/array props instead of memoized ones

## Summary

Prevent unnecessary re-renders by:
1. Memoizing components with React.memo
2. Stabilizing functions with useCallback
3. Caching computations with useMemo
4. Splitting components
5. Moving state down
6. Optimizing context

Focus on components that actually slow down your app. Use React DevTools to find the real bottlenecks.