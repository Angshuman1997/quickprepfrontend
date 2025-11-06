# What are the different types of functions in JavaScript?

## Question
What are the different types of functions in JavaScript?

## Answer

JavaScript has several ways to define functions, each with different characteristics and use cases.

## 1. **Function Declaration**
```javascript
// Function declaration - hoisted
function greet(name) {
    return `Hello, ${name}!`;
}

console.log(greet("John")); // "Hello, John!"
```

**Characteristics:**
- ✅ **Hoisted** - Can be called before declaration
- ✅ **Named** - Has a name property
- ❌ **Not suitable for conditional definition**

## 2. **Function Expression**
```javascript
// Anonymous function expression
const greet = function(name) {
    return `Hello, ${name}!`;
};

// Named function expression
const factorial = function fact(n) {
    return n <= 1 ? 1 : n * fact(n - 1);
};

console.log(greet("John")); // "Hello, John!"
console.log(factorial(5));  // 120
```

**Characteristics:**
- ❌ **Not hoisted** - Must be defined before use
- ✅ **Can be anonymous or named**
- ✅ **Can be assigned to variables**

## 3. **Arrow Functions (ES6+)**
```javascript
// Basic arrow function
const greet = (name) => `Hello, ${name}!`;

// With multiple parameters
const add = (a, b) => a + b;

// With function body
const calculate = (a, b) => {
    const sum = a + b;
    const product = a * b;
    return { sum, product };
};

// Single parameter (parentheses optional)
const square = x => x * x;

console.log(greet("John"));     // "Hello, John!"
console.log(add(5, 3));         // 8
console.log(calculate(4, 3));   // { sum: 7, product: 12 }
console.log(square(5));         // 25
```

**Characteristics:**
- ❌ **Not hoisted**
- ✅ **Lexical `this`** - Inherits `this` from parent scope
- ✅ **Concise syntax**
- ❌ **No `arguments` object**
- ❌ **Cannot be used as constructors**

## 4. **Method Definition (ES6+)**
```javascript
const calculator = {
    // Method definition syntax
    add(a, b) {
        return a + b;
    },

    multiply(a, b) {
        return a * b;
    },

    // Traditional way (same result)
    subtract: function(a, b) {
        return a - b;
    }
};

console.log(calculator.add(5, 3));      // 8
console.log(calculator.multiply(4, 3)); // 12
console.log(calculator.subtract(10, 3)); // 7
```

**Characteristics:**
- ✅ **Shorter syntax** than function expressions
- ✅ **Used in object literals and classes**

## 5. **Constructor Functions**
```javascript
// Constructor function (old way, before classes)
function Person(name, age) {
    this.name = name;
    this.age = age;

    this.greet = function() {
        return `Hi, I'm ${this.name}`;
    };
}

// Usage with 'new' keyword
const john = new Person("John", 30);
console.log(john.name);        // "John"
console.log(john.greet());     // "Hi, I'm John"
console.log(john instanceof Person); // true
```

**Characteristics:**
- ✅ **Creates objects** with `new` keyword
- ✅ **Capitalized by convention**
- ❌ **Verbose compared to classes**

## 6. **Generator Functions**
```javascript
// Generator function
function* numberGenerator() {
    yield 1;
    yield 2;
    yield 3;
}

// Usage
const generator = numberGenerator();
console.log(generator.next()); // { value: 1, done: false }
console.log(generator.next()); // { value: 2, done: false }
console.log(generator.next()); // { value: 3, done: false }
console.log(generator.next()); // { value: undefined, done: true }

// Infinite sequence generator
function* fibonacci() {
    let [prev, curr] = [0, 1];
    while (true) {
        yield curr;
        [prev, curr] = [curr, prev + curr];
    }
}

const fib = fibonacci();
console.log(fib.next().value); // 1
console.log(fib.next().value); // 1
console.log(fib.next().value); // 2
console.log(fib.next().value); // 3
console.log(fib.next().value); // 5
```

**Characteristics:**
- ✅ **Can pause and resume execution**
- ✅ **Returns iterator object**
- ✅ **Memory efficient for large sequences**

## 7. **Async Functions**
```javascript
// Async function declaration
async function fetchUser(id) {
    try {
        const response = await fetch(`/api/users/${id}`);
        const user = await response.json();
        return user;
    } catch (error) {
        console.error('Error:', error);
        throw error;
    }
}

// Async arrow function
const fetchPosts = async (userId) => {
    const response = await fetch(`/api/posts?userId=${userId}`);
    return response.json();
};

// Usage
fetchUser(123)
    .then(user => console.log(user))
    .catch(error => console.error(error));
```

**Characteristics:**
- ✅ **Returns a Promise**
- ✅ **Can use `await` inside**
- ✅ **Handles async operations elegantly**

## 8. **Immediately Invoked Function Expressions (IIFE)**
```javascript
// IIFE - executes immediately
const result = (function() {
    const privateVar = "secret";
    return privateVar.toUpperCase();
})();

console.log(result); // "SECRET"

// IIFE with parameters
const calculator = (function(a, b) {
    return {
        sum: a + b,
        product: a * b,
        difference: a - b
    };
})(10, 5);

console.log(calculator.sum); // 15
```

**Characteristics:**
- ✅ **Creates private scope**
- ✅ **Executes immediately**
- ✅ **Prevents global pollution**

## 9. **Class Methods (ES6+)**
```javascript
class Calculator {
    constructor() {
        this.memory = 0;
    }

    // Instance method
    add(a, b) {
        return a + b;
    }

    // Static method
    static multiply(a, b) {
        return a * b;
    }

    // Getter
    get currentMemory() {
        return this.memory;
    }

    // Setter
    set currentMemory(value) {
        this.memory = value;
    }
}

const calc = new Calculator();
console.log(calc.add(5, 3));        // 8
console.log(Calculator.multiply(4, 3)); // 12 (called on class, not instance)
calc.currentMemory = 42;
console.log(calc.currentMemory);    // 42
```

## Choosing the Right Function Type:

| Use Case | Recommended Function Type |
|----------|---------------------------|
| Simple utility functions | Arrow functions |
| Object methods | Method definition syntax |
| Constructors | Class syntax |
| Async operations | Async functions |
| Private scope | IIFE |
| Iterators | Generator functions |
| Event handlers | Arrow functions (lexical this) |
| Prototypal inheritance | Function constructors |

## Real React Examples:

```javascript
// Component with different function types
function UserList({ users, onUserSelect }) {
    // Arrow function for event handler (lexical this)
    const handleUserClick = (userId) => {
        onUserSelect(userId);
    };

    // Method definition in object
    const userActions = {
        edit(user) {
            console.log('Editing user:', user);
        },
        delete(userId) {
            console.log('Deleting user:', userId);
        }
    };

    // Async function for API calls
    const loadMoreUsers = async () => {
        try {
            const newUsers = await fetchUsers();
            // Update state...
        } catch (error) {
            console.error('Failed to load users:', error);
        }
    };

    return (
        <div>
            {users.map(user => (
                <div key={user.id} onClick={() => handleUserClick(user.id)}>
                    {user.name}
                </div>
            ))}
            <button onClick={loadMoreUsers}>Load More</button>
        </div>
    );
}

// Custom hook using different function types
function useLocalStorage(key, initialValue) {
    // Arrow function with useCallback
    const getStoredValue = useCallback(() => {
        try {
            const item = localStorage.getItem(key);
            return item ? JSON.parse(item) : initialValue;
        } catch (error) {
            console.error('Error reading localStorage:', error);
            return initialValue;
        }
    }, [key, initialValue]);

    const [value, setValue] = useState(getStoredValue);

    // Regular function for setter
    const setStoredValue = useCallback((newValue) => {
        try {
            setValue(newValue);
            localStorage.setItem(key, JSON.stringify(newValue));
        } catch (error) {
            console.error('Error writing to localStorage:', error);
        }
    }, [key]);

    return [value, setStoredValue];
}
```

### Interview Tip:
*"JavaScript has many ways to define functions because each serves different purposes. Arrow functions are great for concise callbacks, function declarations for hoisting, and async functions for handling promises. Choose based on your specific needs!"*