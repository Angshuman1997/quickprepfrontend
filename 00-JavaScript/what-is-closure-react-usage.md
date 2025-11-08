# Closures in JavaScript: Complete Guide with React Examples

## Question
What are closures in JavaScript? How do they work, and what are their practical applications, especially in React development? Provide detailed examples and explain common pitfalls.

## Detailed Answer

Closures are one of JavaScript's most powerful and fundamental concepts. A closure is a function that has access to variables from its outer (enclosing) scope, even after the outer function has finished executing. This "remembers" behavior enables powerful patterns like data privacy, function factories, and stateful functions. Understanding closures is crucial for React development, where they're used extensively in hooks, event handlers, and component logic.

### 1. **Closure Fundamentals**

#### **Basic Definition**
A closure is created when a function is defined inside another function and the inner function references variables from the outer function's scope. Even after the outer function returns, the inner function maintains access to those variables.

#### **Lexical Scoping**
JavaScript uses lexical (static) scoping - a function's scope is determined by where it's defined, not where it's called.

```javascript
function outer() {
    const outerVar = 'I am from outer scope';

    function inner() {
        console.log(outerVar); // Can access outerVar
    }

    return inner;
}

const closureFunc = outer(); // outer() finishes execution here
closureFunc(); // Still prints "I am from outer scope"
```

#### **Closure Creation Process**
```javascript
function createCounter() {
    let count = 0; // This variable is "closed over"

    return {
        increment: function() {
            count++; // Closure captures 'count'
            return count;
        },
        decrement: function() {
            count--; // Same 'count' variable
            return count;
        },
        getCount: function() {
            return count; // Access to shared state
        }
    };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.getCount());  // 2
```

### 2. **Advanced Closure Patterns**

#### **Function Factory Pattern**
```javascript
function createMultiplier(multiplier) {
    return function(number) {
        return number * multiplier;
    };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15
```

#### **Private Variables with Closures**
```javascript
function createBankAccount(initialBalance) {
    let balance = initialBalance; // Private variable

    return {
        deposit: function(amount) {
            if (amount > 0) {
                balance += amount;
                return balance;
            }
        },
        withdraw: function(amount) {
            if (amount > 0 && amount <= balance) {
                balance -= amount;
                return balance;
            }
        },
        getBalance: function() {
            return balance;
        }
        // No direct access to 'balance' variable!
    };
}

const account = createBankAccount(1000);
console.log(account.getBalance()); // 1000
account.deposit(500);
console.log(account.getBalance()); // 1500
// account.balance = 999999; // This won't work!
```

#### **Memoization with Closures**
```javascript
function createMemoizedFunction(fn) {
    const cache = {}; // Private cache

    return function(...args) {
        const key = JSON.stringify(args);
        if (cache[key]) {
            console.log('Returning cached result');
            return cache[key];
        }

        const result = fn.apply(this, args);
        cache[key] = result;
        return result;
    };
}

function expensiveCalculation(n) {
    console.log(`Computing fibonacci for ${n}`);
    if (n <= 1) return n;
    return expensiveCalculation(n - 1) + expensiveCalculation(n - 2);
}

const memoizedFib = createMemoizedFunction(expensiveCalculation);
console.log(memoizedFib(10)); // Computes and caches
console.log(memoizedFib(10)); // Returns cached result
```

#### **Partial Application**
```javascript
function partial(fn, ...fixedArgs) {
    return function(...remainingArgs) {
        return fn.apply(this, [...fixedArgs, ...remainingArgs]);
    };
}

function greet(greeting, name, punctuation) {
    return `${greeting}, ${name}${punctuation}`;
}

const sayHello = partial(greet, 'Hello');
const sayHelloExcitedly = partial(sayHello, '!', '!');

console.log(sayHello('Alice'));           // "Hello, Aliceundefined"
console.log(sayHelloExcitedly('Bob'));    // "Hello, Bob!!"
```

### 3. **Closures in React**

#### **Event Handlers and State Capture**
```javascript
function TodoList({ todos, onToggle }) {
    return (
        <ul>
            {todos.map((todo, index) => (
                <li key={todo.id}>
                    {/* Closure captures 'todo.id' and 'onToggle' */}
                    <input
                        type="checkbox"
                        checked={todo.completed}
                        onChange={() => onToggle(todo.id)} // Closure!
                    />
                    <span style={{
                        textDecoration: todo.completed ? 'line-through' : 'none'
                    }}>
                        {todo.text}
                    </span>
                </li>
            ))}
        </ul>
    );
}
```

#### **useEffect and Stale Closures**
```javascript
function DataFetcher({ userId }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        // This closure captures 'userId'
        fetch(`/api/user/${userId}`)
            .then(response => response.json())
            .then(data => setData(data));
    }, [userId]); // userId is a dependency

    return <div>{data ? data.name : 'Loading...'}</div>;
}

// Problem: Stale closure
function Counter() {
    const [count, setCount] = useState(0);

    useEffect(() => {
        const interval = setInterval(() => {
            console.log(count); // Always logs initial 'count' (0)
            setCount(count + 1); // Uses stale 'count' value
        }, 1000);

        return () => clearInterval(interval);
    }, []); // Empty dependency array - closure captures initial count

    return <div>Count: {count}</div>;
}

// Solution: Functional updates
function FixedCounter() {
    const [count, setCount] = useState(0);

    useEffect(() => {
        const interval = setInterval(() => {
            setCount(prevCount => prevCount + 1); // No stale closure issue
        }, 1000);

        return () => clearInterval(interval);
    }, []); // Empty dependency array is now safe

    return <div>Count: {count}</div>;
}
```

#### **Custom Hooks with Closures**
```javascript
function useLocalStorage(key, initialValue) {
    // Closure captures 'key'
    const [storedValue, setStoredValue] = useState(() => {
        try {
            const item = window.localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch (error) {
            console.error(error);
            return initialValue;
        }
    });

    // Closure captures 'key' and 'setStoredValue'
    const setValue = useCallback((value) => {
        try {
            const valueToStore = value instanceof Function ? value(storedValue) : value;
            setStoredValue(valueToStore);
            window.localStorage.setItem(key, JSON.stringify(valueToStore));
        } catch (error) {
            console.error(error);
        }
    }, [key, storedValue]);

    return [storedValue, setValue];
}

// Usage
function App() {
    const [name, setName] = useLocalStorage('name', 'Anonymous');

    return (
        <input
            value={name}
            onChange={(e) => setName(e.target.value)}
        />
    );
}
```

#### **useCallback and useMemo with Closures**
```javascript
function DataList({ items, filter }) {
    // Closure captures 'filter'
    const filteredItems = useMemo(() => {
        console.log('Filtering items...');
        return items.filter(item =>
            item.name.toLowerCase().includes(filter.toLowerCase())
        );
    }, [items, filter]); // Dependencies

    // Closure captures 'filteredItems'
    const handleItemClick = useCallback((item) => {
        console.log('Clicked:', item.name);
        // Access to current filteredItems via closure
    }, [filteredItems]);

    return (
        <ul>
            {filteredItems.map(item => (
                <li key={item.id} onClick={() => handleItemClick(item)}>
                    {item.name}
                </li>
            ))}
        </ul>
    );
}
```

### 4. **Common Closure Pitfalls and Solutions**

#### **Problem 1: Loop Variable Capture**
```javascript
// Problem: All handlers log the same value
function createHandlers() {
    const handlers = [];
    for (var i = 0; i < 3; i++) {
        handlers.push(() => console.log(i)); // All closures capture same 'i'
    }
    return handlers;
}

const handlers = createHandlers();
handlers[0](); // 3
handlers[1](); // 3
handlers[2](); // 3

// Solution 1: Use let (block scope)
function createHandlersFixed() {
    const handlers = [];
    for (let i = 0; i < 3; i++) { // let creates new binding each iteration
        handlers.push(() => console.log(i));
    }
    return handlers;
}

// Solution 2: IIFE (Immediately Invoked Function Expression)
function createHandlersIIFE() {
    const handlers = [];
    for (var i = 0; i < 3; i++) {
        handlers.push(((index) => () => console.log(index))(i));
    }
    return handlers;
}
```

#### **Problem 2: Memory Leaks**
```javascript
// Problem: Closures can prevent garbage collection
function createLeak() {
    const largeObject = { data: new Array(1000000).fill('data') };

    return function() {
        console.log(largeObject.data.length); // Keeps largeObject alive
    };
}

const leakyFunction = createLeak();
// largeObject cannot be garbage collected while leakyFunction exists

// Solution: Clean up references
function createNoLeak() {
    let largeObject = { data: new Array(1000000).fill('data') };

    const cleanup = () => {
        largeObject = null; // Allow garbage collection
    };

    const useData = () => {
        if (largeObject) {
            console.log(largeObject.data.length);
        }
    };

    return { useData, cleanup };
}
```

#### **Problem 3: this Context in Closures**
```javascript
const obj = {
    name: 'Object',
    methods: [],

    createMethods() {
        for (let i = 0; i < 3; i++) {
            // Arrow function inherits 'this' from createMethods
            this.methods.push(() => console.log(`${this.name}: ${i}`));
        }
    }
};

obj.createMethods();
obj.methods[0](); // "Object: 0"
obj.methods[1](); // "Object: 1"
obj.methods[2](); // "Object: 2"
```

### 5. **Advanced Closure Applications**

#### **Module Pattern**
```javascript
const CalculatorModule = (function() {
    // Private variables
    let memory = 0;
    let history = [];

    // Private functions
    function validateNumber(num) {
        return typeof num === 'number' && !isNaN(num);
    }

    function addToHistory(operation, result) {
        history.push({ operation, result, timestamp: Date.now() });
    }

    // Public API
    return {
        add(a, b) {
            if (!validateNumber(a) || !validateNumber(b)) return null;
            const result = a + b;
            addToHistory(`${a} + ${b}`, result);
            return result;
        },

        getHistory() {
            return [...history]; // Return copy to prevent external modification
        },

        clearHistory() {
            history = [];
        },

        getMemory() {
            return memory;
        },

        setMemory(value) {
            if (validateNumber(value)) {
                memory = value;
            }
        }
    };
})();

console.log(CalculatorModule.add(5, 3)); // 8
console.log(CalculatorModule.getHistory()); // [{ operation: "5 + 3", result: 8, ... }]
```

#### **Currying with Closures**
```javascript
function curry(fn) {
    return function curried(...args) {
        if (args.length >= fn.length) {
            return fn.apply(this, args);
        } else {
            return function(...moreArgs) {
                return curried.apply(this, [...args, ...moreArgs]);
            };
        }
    };
}

function multiply(a, b, c) {
    return a * b * c;
}

const curriedMultiply = curry(multiply);
const multiplyBy2 = curriedMultiply(2);
const multiplyBy2Then3 = multiplyBy2(3);

console.log(multiplyBy2Then3(4)); // 24
console.log(curriedMultiply(2)(3)(4)); // 24
```

#### **Debounce with Closures**
```javascript
function debounce(func, delay) {
    let timeoutId;

    return function(...args) {
        const context = this;

        clearTimeout(timeoutId);
        timeoutId = setTimeout(() => {
            func.apply(context, args);
        }, delay);
    };
}

// Usage in React
function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);

    const debouncedSearch = useMemo(
        () => debounce(async (searchQuery) => {
            const response = await fetch(`/api/search?q=${searchQuery}`);
            const data = await response.json();
            setResults(data);
        }, 300),
        []
    );

    const handleChange = (e) => {
        const value = e.target.value;
        setQuery(value);
        debouncedSearch(value);
    };

    return (
        <div>
            <input value={query} onChange={handleChange} />
            <ul>
                {results.map(item => (
                    <li key={item.id}>{item.name}</li>
                ))}
            </ul>
        </div>
    );
}
```

### 6. **Performance Considerations**

#### **Memory Usage**
- Closures prevent garbage collection of captured variables
- Long-lived closures can cause memory leaks
- Use weak references or cleanup functions when possible

#### **Performance Impact**
```javascript
// Inefficient: Creates new closure on every render
function InefficientComponent({ items }) {
    return (
        <ul>
            {items.map(item => (
                <li key={item.id}>
                    {/* New function created on every render */}
                    <button onClick={() => console.log(item.id)}>
                        {item.name}
                    </button>
                </li>
            ))}
        </ul>
    );
}

// Efficient: Extract function outside component
function EfficientComponent({ items }) {
    const handleClick = useCallback((id) => {
        console.log(id);
    }, []); // Stable reference

    return (
        <ul>
            {items.map(item => (
                <li key={item.id}>
                    <button onClick={() => handleClick(item.id)}>
                        {item.name}
                    </button>
                </li>
            ))}
        </ul>
    );
}
```

#### **Closure vs Class Comparison**
```javascript
// Closure approach (functional)
function createTimer() {
    let startTime = null;

    return {
        start() {
            startTime = Date.now();
        },
        stop() {
            if (startTime) {
                const elapsed = Date.now() - startTime;
                console.log(`Elapsed: ${elapsed}ms`);
                startTime = null;
            }
        }
    };
}

// Class approach
class Timer {
    constructor() {
        this.startTime = null;
    }

    start() {
        this.startTime = Date.now();
    }

    stop() {
        if (this.startTime) {
            const elapsed = Date.now() - this.startTime;
            console.log(`Elapsed: ${elapsed}ms`);
            this.startTime = null;
        }
    }
}

// Both achieve similar results, but closures can be more memory efficient
```

## Code Examples

### **Complete React Todo App with Closures**
```javascript
import React, { useState, useCallback, useMemo } from 'react';

function useTodoManager() {
    const [todos, setTodos] = useState([]);

    // Closure captures setTodos
    const addTodo = useCallback((text) => {
        const newTodo = {
            id: Date.now(),
            text,
            completed: false
        };
        setTodos(prevTodos => [...prevTodos, newTodo]);
    }, []);

    // Closure captures setTodos
    const toggleTodo = useCallback((id) => {
        setTodos(prevTodos =>
            prevTodos.map(todo =>
                todo.id === id ? { ...todo, completed: !todo.completed } : todo
            )
        );
    }, []);

    // Closure captures setTodos
    const deleteTodo = useCallback((id) => {
        setTodos(prevTodos => prevTodos.filter(todo => todo.id !== id));
    }, []);

    // Closure captures todos
    const completedCount = useMemo(() => {
        return todos.filter(todo => todo.completed).length;
    }, [todos]);

    return {
        todos,
        addTodo,
        toggleTodo,
        deleteTodo,
        completedCount,
        totalCount: todos.length
    };
}

function TodoApp() {
    const {
        todos,
        addTodo,
        toggleTodo,
        deleteTodo,
        completedCount,
        totalCount
    } = useTodoManager();

    const [inputValue, setInputValue] = useState('');

    const handleSubmit = (e) => {
        e.preventDefault();
        if (inputValue.trim()) {
            addTodo(inputValue.trim());
            setInputValue('');
        }
    };

    return (
        <div className="todo-app">
            <h1>Todo App with Closures</h1>

            <form onSubmit={handleSubmit}>
                <input
                    type="text"
                    value={inputValue}
                    onChange={(e) => setInputValue(e.target.value)}
                    placeholder="Add a new todo..."
                />
                <button type="submit">Add</button>
            </form>

            <div className="stats">
                {completedCount} of {totalCount} completed
            </div>

            <ul className="todo-list">
                {todos.map(todo => (
                    <li key={todo.id} className={todo.completed ? 'completed' : ''}>
                        <input
                            type="checkbox"
                            checked={todo.completed}
                            onChange={() => toggleTodo(todo.id)} // Closure!
                        />
                        <span>{todo.text}</span>
                        <button onClick={() => deleteTodo(todo.id)}>Delete</button>
                    </li>
                ))}
            </ul>
        </div>
    );
}

export default TodoApp;
```

### **Custom Hook with Complex Closures**
```javascript
import { useState, useEffect, useCallback, useRef } from 'react';

function useAsyncOperation(operation, dependencies = []) {
    const [state, setState] = useState({
        loading: false,
        data: null,
        error: null
    });

    const abortControllerRef = useRef(null);

    // Closure captures operation and dependencies
    const execute = useCallback(async (...args) => {
        // Cancel previous operation
        if (abortControllerRef.current) {
            abortControllerRef.current.abort();
        }

        abortControllerRef.current = new AbortController();

        setState({ loading: true, data: null, error: null });

        try {
            // Closure captures abortControllerRef
            const result = await operation(abortControllerRef.current.signal, ...args);
            setState({ loading: false, data: result, error: null });
            return result;
        } catch (error) {
            if (error.name !== 'AbortError') {
                setState({ loading: false, data: null, error });
            }
        }
    }, dependencies); // Dependencies for operation function

    // Cleanup on unmount
    useEffect(() => {
        return () => {
            if (abortControllerRef.current) {
                abortControllerRef.current.abort();
            }
        };
    }, []);

    return { ...state, execute };
}

// Usage
function UserProfile({ userId }) {
    const fetchUser = useCallback(async (signal, id) => {
        const response = await fetch(`/api/users/${id}`, { signal });
        if (!response.ok) throw new Error('Failed to fetch user');
        return response.json();
    }, []);

    const { loading, data: user, error, execute } = useAsyncOperation(fetchUser, []);

    useEffect(() => {
        if (userId) {
            execute(userId);
        }
    }, [userId, execute]);

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;
    if (!user) return <div>No user selected</div>;

    return (
        <div>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
}
```

## Interview Tips

**Key Concepts to Master:**
- **Closure Creation**: Function inside function accessing outer variables
- **Lexical Scoping**: Scope determined by definition location
- **Variable Capture**: Variables are captured by reference, not value
- **Memory Implications**: Closures prevent garbage collection
- **React Applications**: Event handlers, hooks, stale closure problems

**Common Interview Questions:**
1. "What is a closure and how does it work?"
2. "Can you give an example of a closure in JavaScript?"
3. "What's the difference between closure and scope?"
4. "How do closures relate to memory leaks?"
5. "How do you handle stale closures in React useEffect?"
6. "What's the module pattern and how does it use closures?"

**Pro Tips:**
- **Always consider timing**: Closures capture variables at creation time
- **Watch for memory leaks**: Long-lived closures can prevent GC
- **Use functional updates**: In React, prefer `setState(prev => prev + 1)` over `setState(count + 1)`
- **Debug with console**: Log captured variables to understand closure behavior
- **Consider alternatives**: Sometimes classes or external state management is clearer

**Red Flags in Interviews:**
- Confusing closures with scope
- Not understanding variable capture timing
- Missing memory leak considerations
- Not knowing React closure pitfalls (stale closures)
- Thinking closures copy variable values (they capture references)

**Advanced Interview Questions:**
- "How would you implement private variables without classes?"
- "What's the difference between closures in arrow functions vs regular functions?"
- "How do closures work with asynchronous code?"
- "Can you implement currying using closures?"
- "How does the module pattern leverage closures for encapsulation?"