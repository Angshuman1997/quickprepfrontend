# What is a closure? Where do you use closures in React?

## Question
What is a closure? Where do you use closures in React?

## Answer

**A closure** is a function that has access to variables from its outer (enclosing) scope even after the outer function has finished executing. It "closes over" these variables.

### Basic Closure Example:
```javascript
function outerFunction(x) {
    // Outer scope variable
    
    function innerFunction(y) {
        // Inner function has access to 'x' from outer scope
        return x + y;
    }
    
    return innerFunction;
}

const addFive = outerFunction(5);
console.log(addFive(3)); // 8 - still has access to 'x' (5)
```

### Practical JavaScript Example:
```javascript
function createCounter() {
    let count = 0; // Private variable
    
    return {
        increment: () => ++count,
        decrement: () => --count,
        getValue: () => count
    };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getValue());  // 2
// 'count' is not directly accessible from outside
```

## Closures in React:

### 1. **Event Handlers with State:**
```javascript
function Counter() {
    const [count, setCount] = useState(0);
    
    // This function closes over 'count' and 'setCount'
    const handleClick = () => {
        setCount(count + 1); // Accesses current count value
    };
    
    return <button onClick={handleClick}>Count: {count}</button>;
}
```

### 2. **useEffect Dependencies:**
```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    
    useEffect(() => {
        // This function closes over 'userId'
        async function fetchUser() {
            const userData = await getUserById(userId);
            setUser(userData);
        }
        
        fetchUser();
    }, [userId]); // Closure captures userId
    
    return <div>{user?.name}</div>;
}
```

### 3. **Custom Hooks:**
```javascript
function useLocalStorage(key, initialValue) {
    const [value, setValue] = useState(() => {
        // Closure captures 'key' and 'initialValue'
        const saved = localStorage.getItem(key);
        return saved ? JSON.parse(saved) : initialValue;
    });
    
    const updateValue = (newValue) => {
        setValue(newValue);
        localStorage.setItem(key, JSON.stringify(newValue)); // Closes over 'key'
    };
    
    return [value, updateValue];
}
```

### 4. **Callback Functions with Dynamic Values:**
```javascript
function TodoList({ todos }) {
    const handleDelete = (todoId) => {
        // Returns a function that closes over 'todoId'
        return () => {
            deleteTodo(todoId);
        };
    };
    
    return (
        <ul>
            {todos.map(todo => (
                <li key={todo.id}>
                    {todo.text}
                    <button onClick={handleDelete(todo.id)}>Delete</button>
                </li>
            ))}
        </ul>
    );
}
```

### 5. **useCallback Hook:**
```javascript
function ExpensiveComponent({ data, filter }) {
    // Memoized function that closes over 'filter'
    const filterData = useCallback(() => {
        return data.filter(item => item.category === filter);
    }, [data, filter]); // Dependencies define closure scope
    
    return <DataGrid data={filterData()} />;
}
```

### Common Closure Pitfall in React:
```javascript
// WRONG - Stale closure
function Counter() {
    const [count, setCount] = useState(0);
    
    useEffect(() => {
        const interval = setInterval(() => {
            setCount(count + 1); // Always uses initial count (0)
        }, 1000);
        
        return () => clearInterval(interval);
    }, []); // Missing dependency
}

// CORRECT - Fresh closure
function Counter() {
    const [count, setCount] = useState(0);
    
    useEffect(() => {
        const interval = setInterval(() => {
            setCount(prev => prev + 1); // Uses callback for fresh value
        }, 1000);
        
        return () => clearInterval(interval);
    }, []); // Now safe with functional update
}
```

### Interview Tip:
*"Closures are everywhere in React! They help us access values in event handlers, useEffect, and custom hooks. The key is understanding dependency arrays and avoiding stale closures."*