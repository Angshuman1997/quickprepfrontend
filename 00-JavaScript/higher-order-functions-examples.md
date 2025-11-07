# Higher-Order Functions

## Simple Answer
Functions that take other functions as arguments or return functions.

## Examples

**Array methods:**
```javascript
const numbers = [1, 2, 3, 4, 5];

numbers.forEach(num => console.log(num));     // Takes function
const doubled = numbers.map(num => num * 2);  // Takes function, returns array
const evens = numbers.filter(num => num % 2 === 0); // Takes function, returns array
```

**Custom higher-order function:**
```javascript
function greetUser(greetingFunction, name) {
  return greetingFunction(name);
}

const sayHello = name => `Hello, ${name}!`;
const sayHi = name => `Hi, ${name}!`;

console.log(greetUser(sayHello, 'Alice')); // "Hello, Alice!"
console.log(greetUser(sayHi, 'Bob'));     // "Hi, Bob!"
```

## Functions that return functions:
```javascript
function createMultiplier(multiplier) {
  return function(number) {
    return number * multiplier;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5)); // 10
console.log(triple(5)); // 15
```

## Interview Q&A

**Q: What is a higher-order function?**  
**A:** A function that takes other functions as arguments or returns a function.

**Q: Give examples of higher-order functions.**  
**A:** Array methods like map, filter, forEach. Also functions like setTimeout that take callbacks.

**Q: Why are they useful?**  
**A:** They enable functional programming patterns, code reuse, and abstraction.

// find - takes a function to find first matching element
const firstEven = numbers.find(num => num % 2 === 0); // 2

// some - takes a function to test if any element passes
const hasEven = numbers.some(num => num % 2 === 0); // true

// every - takes a function to test if all elements pass
const allEven = numbers.every(num => num % 2 === 0); // false
```

### Custom Higher-Order Functions:
```javascript
// A function that takes a function as argument
function executeOperation(operation, a, b) {
    return operation(a, b);
}

const add = (a, b) => a + b;
const multiply = (a, b) => a * b;

console.log(executeOperation(add, 5, 3));      // 8
console.log(executeOperation(multiply, 5, 3)); // 15
```

## 2. **Functions that Return Functions**

### Function Factory:
```javascript
// A function that returns a function
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

### Currying Example:
```javascript
// Currying: breaking down a function that takes multiple arguments
function add(a) {
    return function(b) {
        return function(c) {
            return a + b + c;
        };
    };
}

const add5 = add(5);        // Returns a function
const add5And3 = add5(3);   // Returns a function
const result = add5And3(2); // Returns 10

// Or more concisely:
const result2 = add(5)(3)(2); // 10
```

### Event Handler Factory:
```javascript
function createEventHandler(eventType) {
    return function(element) {
        element.addEventListener(eventType, (e) => {
            console.log(`${eventType} event on ${element.tagName}`);
        });
    };
}

const handleClick = createEventHandler('click');
const handleHover = createEventHandler('mouseover');

// Later...
const button = document.querySelector('button');
handleClick(button);  // Adds click listener
handleHover(button);  // Adds hover listener
```

## 3. **Functions that Do Both (Take and Return Functions)**

### Advanced Composition:
```javascript
function compose(...functions) {
    return function(initialValue) {
        return functions.reduceRight((value, func) => func(value), initialValue);
    };
}

const add2 = x => x + 2;
const multiply3 = x => x * 3;
const square = x => x * x;

const complexOperation = compose(square, multiply3, add2);
console.log(complexOperation(5)); // ((5 + 2) * 3)² = 49
```

### Middleware Pattern:
```javascript
function createLogger(prefix) {
    return function(next) {
        return function(action) {
            console.log(`${prefix}: ${action.type}`);
            return next(action);
        };
    };
}

function createThunkMiddleware() {
    return function(next) {
        return function(action) {
            if (typeof action === 'function') {
                return action(next);
            }
            return next(action);
        };
    };
}
```

## Real React Examples:

### 1. **useCallback (Returns a Memoized Function):**
```javascript
function TodoList({ todos, onToggle }) {
    // useCallback returns a memoized function
    const handleToggle = useCallback((id) => {
        onToggle(id);
    }, [onToggle]);

    return (
        <ul>
            {todos.map(todo => (
                <li key={todo.id}>
                    <input
                        type="checkbox"
                        checked={todo.completed}
                        onChange={() => handleToggle(todo.id)} // Function passed as prop
                    />
                    {todo.text}
                </li>
            ))}
        </ul>
    );
}
```

### 2. **Custom Hook that Returns Functions:**
```javascript
function useForm(initialValues) {
    const [values, setValues] = useState(initialValues);

    // Returns a function (higher-order function)
    const handleChange = useCallback((fieldName) => {
        return (event) => {
            setValues(prev => ({
                ...prev,
                [fieldName]: event.target.value
            }));
        };
    }, []);

    // Returns multiple functions
    const reset = useCallback(() => {
        setValues(initialValues);
    }, [initialValues]);

    return { values, handleChange, reset };
}

// Usage
function LoginForm() {
    const { values, handleChange, reset } = useForm({
        email: '',
        password: ''
    });

    return (
        <form>
            <input
                value={values.email}
                onChange={handleChange('email')} // Function returned by handleChange
            />
            <input
                type="password"
                value={values.password}
                onChange={handleChange('password')} // Another function
            />
            <button type="button" onClick={reset}>Reset</button>
        </form>
    );
}
```

### 3. **Event Handlers with Closures:**
```javascript
function useTimer() {
    const [time, setTime] = useState(0);
    const [isRunning, setIsRunning] = useState(false);

    // Returns functions that close over state
    const start = useCallback(() => {
        setIsRunning(true);
    }, []);

    const stop = useCallback(() => {
        setIsRunning(false);
    }, []);

    const reset = useCallback(() => {
        setTime(0);
        setIsRunning(false);
    }, []);

    useEffect(() => {
        if (!isRunning) return;

        const interval = setInterval(() => {
            setTime(t => t + 1); // Function passed to setInterval
        }, 1000);

        return () => clearInterval(interval);
    }, [isRunning]);

    return { time, isRunning, start, stop, reset };
}
```

### 4. **Data Fetching with Callbacks:**
```javascript
function useApi(endpoint) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    // Takes a callback function as parameter
    const fetchData = useCallback(async (onSuccess, onError) => {
        try {
            setLoading(true);
            const response = await fetch(endpoint);
            const result = await response.json();

            setData(result);
            if (onSuccess) onSuccess(result); // Execute callback
        } catch (err) {
            setError(err);
            if (onError) onError(err); // Execute callback
        } finally {
            setLoading(false);
        }
    }, [endpoint]);

    return { data, loading, error, fetchData };
}

// Usage
function UserProfile({ userId }) {
    const { data: user, loading, fetchData } = useApi(`/api/users/${userId}`);

    useEffect(() => {
        fetchData(
            (userData) => console.log('User loaded:', userData), // Success callback
            (error) => console.error('Failed to load user:', error) // Error callback
        );
    }, [fetchData]);

    if (loading) return <div>Loading...</div>;
    return <div>{user?.name}</div>;
}
```

## Benefits of Higher-Order Functions:

1. **Reusability** - Create generic functions that work with any function
2. **Abstraction** - Hide complex logic behind simple interfaces
3. **Composition** - Combine simple functions into complex operations
4. **Flexibility** - Allow users to customize behavior
5. **Functional Programming** - Enable declarative programming style

## Common Higher-Order Functions in JavaScript:

- **Array methods**: `map`, `filter`, `reduce`, `forEach`, `find`, `some`, `every`
- **Promise methods**: `then`, `catch`, `finally`
- **Function methods**: `call`, `apply`, `bind`
- **React hooks**: `useCallback`, `useMemo`, `useEffect`

### Interview Tip:
*"Higher-order functions are the backbone of functional programming in JavaScript. They make code more modular and reusable. Array methods like map and filter are classic examples, but React hooks like useCallback also follow this pattern."*