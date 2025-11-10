# Hoisting in JavaScript: var, let, const, and Temporal Dead Zone

## Question
What is hoisting in JavaScript? Explain the differences between var, let, and const hoisting behavior, including the Temporal Dead Zone. Provide detailed examples and discuss implications for modern JavaScript development.

## Detailed Answer

Hoisting is JavaScript's behavior of moving variable and function declarations to the top of their scope during the compilation phase, before the code executes. This fundamental concept affects how variables and functions are accessed and can lead to bugs if not understood properly. The introduction of let and const in ES6 changed hoisting behavior significantly, introducing the "Temporal Dead Zone" (TDZ) and making JavaScript safer and more predictable.

### 1. **Hoisting Fundamentals**

#### **What is Hoisting?**
Hoisting is a JavaScript mechanism where variable and function declarations are moved to the top of their containing scope during the compilation phase. This allows you to use variables and functions before they're declared in the code.

**Important**: Only declarations are hoisted, not initializations.

#### **Compilation vs Execution Phases**
```javascript
// What you write:
console.log(x);
var x = 5;

// What JavaScript does during compilation:
var x;           // Declaration hoisted
console.log(x);  // undefined
x = 5;          // Initialization stays in place
```

### 2. **Variable Hoisting: var vs let vs const**

#### **var Hoisting**
- **Declarations**: Hoisted and initialized with `undefined`
- **Scope**: Function scope
- **Re-declaration**: Allowed
- **Can be problematic**: Leads to unexpected `undefined` values

```javascript
// Example 1: Basic var hoisting
console.log(name); // undefined (hoisted)
var name = 'Alice';
console.log(name); // 'Alice'

// Example 2: var in function scope
function example() {
    console.log(age); // undefined
    if (true) {
        var age = 25;
    }
    console.log(age); // 25 (function scope)
}
example();

// Example 3: Re-declaration allowed
var count = 1;
var count = 2; // No error
console.log(count); // 2
```

#### **let Hoisting**
- **Declarations**: Hoisted but NOT initialized
- **Scope**: Block scope
- **Temporal Dead Zone**: Cannot access until declared
- **No re-declaration**: In same scope

```javascript
// Example 1: let hoisting with TDZ
console.log(firstName); // ReferenceError: Cannot access 'firstName' before initialization
let firstName = 'Bob';

// Example 2: let in block scope
function example() {
    if (true) {
        console.log(lastName); // ReferenceError (TDZ)
        let lastName = 'Smith';
    }
}

// Example 3: No re-declaration
let score = 100;
// let score = 200; // SyntaxError: Identifier 'score' has already been declared
```

#### **const Hoisting**
- **Declarations**: Hoisted but NOT initialized
- **Scope**: Block scope
- **Temporal Dead Zone**: Cannot access until declared
- **Must be initialized**: At declaration time
- **Immutable**: Cannot reassign

```javascript
// Example 1: const hoisting with TDZ
console.log(PI); // ReferenceError: Cannot access 'PI' before initialization
const PI = 3.14159;

// Example 2: Must initialize
// const MAX_SIZE; // SyntaxError: Missing initializer in const declaration

// Example 3: Cannot reassign
const API_URL = 'https://api.example.com';
// API_URL = 'https://new-api.example.com'; // TypeError: Assignment to constant variable
```

### 3. **Temporal Dead Zone (TDZ)**

#### **What is the TDZ?**
The Temporal Dead Zone is the period between when a variable is hoisted (declared) and when it's initialized. During this time, accessing the variable throws a `ReferenceError`.

```javascript
// TDZ starts here (at the beginning of scope)
console.log(myVar); // ReferenceError - in TDZ
let myVar = 'Hello'; // TDZ ends here
console.log(myVar); // 'Hello' - accessible now
```

#### **TDZ Examples**
```javascript
// Example 1: Function parameter TDZ
function test(param = console.log(param)) { // ReferenceError during parameter initialization
    // param is in TDZ during default parameter evaluation
}
test(); // ReferenceError

// Example 2: Class hoisting
console.log(MyClass); // ReferenceError
class MyClass {}
console.log(MyClass); // [class MyClass]

// Example 3: Complex TDZ scenarios
{
    // TDZ for all variables starts here
    const func = () => console.log(a, b, c); // ReferenceError for all

    let a = 1;
    let b = 2;
    let c = 3;

    func(); // Now works: 1 2 3
}
```

### 4. **Function Hoisting**

#### **Function Declaration Hoisting**
Function declarations are fully hoisted - both declaration and implementation.

```javascript
// Example 1: Function declaration hoisting
greet(); // "Hello, World!"

function greet() {
    console.log("Hello, World!");
}

// Example 2: Function declarations override variables
console.log(typeof myFunc); // "function"
var myFunc;

function myFunc() {
    console.log("I'm a function");
}

myFunc(); // "I'm a function"
```

#### **Function Expression Hoisting**
Only the variable declaration is hoisted, not the function assignment.

```javascript
// Example 1: Function expression with var
console.log(typeof sayHi); // "undefined"
sayHi(); // TypeError: sayHi is not a function

var sayHi = function() {
    console.log("Hi!");
};

// Example 2: Function expression with let (TDZ)
console.log(typeof sayBye); // ReferenceError
let sayBye = function() {
    console.log("Bye!");
};
```

#### **Arrow Function Hoisting**
Same rules as function expressions.

```javascript
// Arrow function with var
console.log(typeof arrowFunc); // "undefined"
var arrowFunc = () => console.log("Arrow!");

// Arrow function with let (TDZ)
console.log(typeof arrowFunc2); // ReferenceError
let arrowFunc2 = () => console.log("Arrow 2!");
```

### 5. **Advanced Hoisting Scenarios**

#### **Hoisting in Loops**
```javascript
// Problem with var in loops
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100); // Prints 3, 3, 3
}

// Solution with let (block scope)
for (let j = 0; j < 3; j++) {
    setTimeout(() => console.log(j), 100); // Prints 0, 1, 2
}

// IIFE solution for var
for (var k = 0; k < 3; k++) {
    ((index) => {
        setTimeout(() => console.log(index), 100);
    })(k); // Prints 0, 1, 2
}
```

#### **Hoisting with Destructuring**
```javascript
// Object destructuring
console.log(a, b); // ReferenceError
let {a, b} = {a: 1, b: 2};

// Array destructuring
console.log(x, y); // ReferenceError
let [x, y] = [1, 2];

// Default parameters and TDZ
function test(a = b, b = 2) { // ReferenceError: b is not defined
    return a + b;
}
```

#### **Hoisting in Classes**
```javascript
// Class declarations are not hoisted
console.log(Person); // ReferenceError
class Person {
    constructor(name) {
        this.name = name;
    }
}

// Class expressions
console.log(typeof Employee); // "undefined"
var Employee = class {
    constructor(name) {
        this.name = name;
    }
};
```

#### **Module Hoisting**
```javascript
// In ES6 modules, hoisting works differently
// Imports are hoisted
console.log(moment); // ReferenceError (not hoisted like var)

import moment from 'moment';

// Exports are not hoisted
export { myFunction }; // ReferenceError if myFunction not declared yet

function myFunction() {
    return 'Hello';
}
```

### 6. **React and Framework Implications**

#### **useState and Hoisting**
```javascript
function Counter() {
    // This works because useState declarations are hoisted within the component
    const [count, setCount] = useState(0);

    const increment = () => {
        setCount(count + 1);
    };

    // But this would cause issues:
    // console.log(count); // ReferenceError if used before declaration
    // let [count, setCount] = useState(0);

    return (
        <button onClick={increment}>
            Count: {count}
        </button>
    );
}
```

#### **useEffect and Variable Hoisting**
```javascript
function DataFetcher({ userId }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        let isSubscribed = true; // Hoisted within effect scope

        fetch(`/api/user/${userId}`)
            .then(response => response.json())
            .then(data => {
                if (isSubscribed) {
                    setData(data);
                }
            });

        return () => {
            isSubscribed = false; // Cleanup function
        };
    }, [userId]);

    return <div>{data ? data.name : 'Loading...'}</div>;
}
```

#### **Custom Hooks and Hoisting**
```javascript
function useCustomHook(initialValue) {
    // Variables are hoisted within the hook scope
    const [value, setValue] = useState(initialValue);

    // This would work because useCallback is declared after
    const updateValue = useCallback((newValue) => {
        setValue(newValue);
    }, []);

    return [value, updateValue];
}

// Usage
function MyComponent() {
    const [count, updateCount] = useCustomHook(0);

    return (
        <button onClick={() => updateCount(count + 1)}>
            Count: {count}
        </button>
    );
}
```

### 7. **Common Pitfalls and Best Practices**

#### **Pitfall 1: Accidental Global Variables**
```javascript
// Without 'use strict', this creates a global variable
function test() {
    myVar = 'global'; // No var/let/const - becomes global!
}
test();
console.log(myVar); // 'global' (pollutes global scope)
```

#### **Pitfall 2: TDZ in Conditional Blocks**
```javascript
// This can be confusing
if (true) {
    console.log(typeof myVar); // "undefined" (hoisted)
    var myVar = 'hello';
}

if (true) {
    console.log(typeof myLet); // ReferenceError (TDZ)
    let myLet = 'hello';
}
```

#### **Best Practices**
```javascript
// 1. Always declare variables at the top of scope
function goodFunction() {
    let result, i;

    // Use variables here
    result = calculate();
    for (i = 0; i < 10; i++) {
        // loop logic
    }
}

// 2. Use const by default, let when needed, avoid var
const API_KEY = 'abc123';
let userName = 'Alice';

// 3. Be aware of TDZ in modern code
function component() {
    // Declare hooks at the top
    const [state, setState] = useState(null);

    // TDZ ends here for state
    console.log(state); // Safe to use
}

// 4. Use 'use strict' to catch hoisting issues
'use strict';
globalVar = 'test'; // ReferenceError in strict mode
```

### 8. **Performance and Debugging**

#### **Performance Considerations**
- **var**: Slightly faster in some engines (older optimization)
- **let/const**: Modern engines optimize equally well
- **TDZ**: Helps catch bugs early, preventing runtime errors
- **Block scoping**: Better for garbage collection and memory management

#### **Debugging Hoisting Issues**
```javascript
// Debug helper for TDZ
function debugVariable(name, value) {
    try {
        console.log(`${name}:`, value);
    } catch (error) {
        console.log(`${name}:`, error.message);
    }
}

// Usage
debugVariable('myVar', myVar); // Shows if in TDZ or undefined
let myVar = 'test';
debugVariable('myVar', myVar); // Shows actual value
```

#### **Linting Rules**
```javascript
// ESLint rules for hoisting
{
    "rules": {
        "no-use-before-define": ["error", { "variables": true, "functions": false }],
        "no-var": "error",
        "prefer-const": "error"
    }
}
```

## Code Examples

### **Complete Hoisting Demonstration**
```javascript
console.log('=== Hoisting Demonstration ===');

// 1. var hoisting
console.log('1. var hoisting:');
console.log('before declaration:', typeof a); // "undefined"
var a = 'var variable';
console.log('after declaration:', a); // "var variable"

// 2. let hoisting (TDZ)
console.log('\n2. let hoisting (TDZ):');
try {
    console.log('before declaration:', b); // ReferenceError
} catch (e) {
    console.log('before declaration:', e.message);
}
let b = 'let variable';
console.log('after declaration:', b); // "let variable"

// 3. const hoisting (TDZ)
console.log('\n3. const hoisting (TDZ):');
try {
    console.log('before declaration:', c); // ReferenceError
} catch (e) {
    console.log('before declaration:', e.message);
}
const c = 'const variable';
console.log('after declaration:', c); // "const variable"

// 4. Function hoisting
console.log('\n4. Function hoisting:');
console.log('before declaration:', typeof greet); // "function"
greet(); // "Hello from function declaration!"

function greet() {
    console.log('Hello from function declaration!');
}

// 5. Function expression hoisting
console.log('\n5. Function expression hoisting:');
console.log('before declaration:', typeof sayHi); // "undefined"
try {
    sayHi(); // TypeError
} catch (e) {
    console.log('calling before declaration:', e.message);
}
var sayHi = function() {
    console.log('Hello from function expression!');
};
sayHi(); // Works now

console.log('\n=== End Demonstration ===');
```

### **React Component with Hoisting Awareness**
```javascript
import React, { useState, useEffect, useCallback } from 'react';

function TodoApp() {
    // All hooks declared at the top (good practice)
    const [todos, setTodos] = useState([]);
    const [filter, setFilter] = useState('all');

    // useCallback uses hoisting within the component scope
    const addTodo = useCallback((text) => {
        const newTodo = {
            id: Date.now(),
            text,
            completed: false
        };
        setTodos(prevTodos => [...prevTodos, newTodo]);
    }, []);

    const toggleTodo = useCallback((id) => {
        setTodos(prevTodos =>
            prevTodos.map(todo =>
                todo.id === id ? { ...todo, completed: !todo.completed } : todo
            )
        );
    }, []);

    // useEffect demonstrates proper dependency management
    useEffect(() => {
        // Variables declared within effect scope are hoisted here
        let isMounted = true;

        // Simulate API call
        const loadTodos = async () => {
            try {
                // In real app, this would be an API call
                const savedTodos = JSON.parse(localStorage.getItem('todos') || '[]');
                if (isMounted) {
                    setTodos(savedTodos);
                }
            } catch (error) {
                console.error('Failed to load todos:', error);
            }
        };

        loadTodos();

        // Cleanup function
        return () => {
            isMounted = false;
        };
    }, []); // Empty dependency array - effect runs once

    // Save todos whenever they change
    useEffect(() => {
        localStorage.setItem('todos', JSON.stringify(todos));
    }, [todos]);

    // Computed values using proper scoping
    const filteredTodos = React.useMemo(() => {
        switch (filter) {
            case 'active':
                return todos.filter(todo => !todo.completed);
            case 'completed':
                return todos.filter(todo => todo.completed);
            default:
                return todos;
        }
    }, [todos, filter]);

    const activeCount = todos.filter(todo => !todo.completed).length;

    return (
        <div className="todo-app">
            <h1>Todo App (Hoisting Aware)</h1>

            <div className="add-todo">
                <input
                    type="text"
                    placeholder="Add a new todo..."
                    onKeyPress={(e) => {
                        if (e.key === 'Enter' && e.target.value.trim()) {
                            addTodo(e.target.value.trim());
                            e.target.value = '';
                        }
                    }}
                />
            </div>

            <div className="filters">
                <button
                    className={filter === 'all' ? 'active' : ''}
                    onClick={() => setFilter('all')}
                >
                    All ({todos.length})
                </button>
                <button
                    className={filter === 'active' ? 'active' : ''}
                    onClick={() => setFilter('active')}
                >
                    Active ({activeCount})
                </button>
                <button
                    className={filter === 'completed' ? 'active' : ''}
                    onClick={() => setFilter('completed')}
                >
                    Completed ({todos.length - activeCount})
                </button>
            </div>

            <ul className="todo-list">
                {filteredTodos.map(todo => (
                    <li
                        key={todo.id}
                        className={todo.completed ? 'completed' : ''}
                        onClick={() => toggleTodo(todo.id)}
                    >
                        <input
                            type="checkbox"
                            checked={todo.completed}
                            onChange={() => {}} // Handled by li onClick
                        />
                        <span>{todo.text}</span>
                    </li>
                ))}
            </ul>
        </div>
    );
}

export default TodoApp;
```

### **Hoisting Edge Cases**
```javascript
// Edge Case 1: Variable hoisting in catch blocks
try {
    console.log(errorVar); // ReferenceError (TDZ)
    let errorVar = 'test';
} catch (error) {
    console.log('Caught:', error.message);
}

// Edge Case 2: Function parameters and hoisting
function testParams(a = b, b = 3) {
    return a + b;
}
// ReferenceError: b is not defined (during parameter default evaluation)

// Edge Case 3: Class field hoisting
class TestClass {
    // Class fields are not hoisted like var
    constructor() {
        console.log(this.field); // undefined (not ReferenceError)
    }
}

const instance = new TestClass();
instance.field = 'value';

// Edge Case 4: Generator function hoisting
console.log(typeof myGenerator); // "function"
function* myGenerator() {
    yield 1;
    yield 2;
}

// Edge Case 5: Async function hoisting
console.log(typeof myAsyncFunc); // "function"
async function myAsyncFunc() {
    return await Promise.resolve('done');
}
```

## Interview Tips

**Key Concepts to Master:**
- **Hoisting**: Declarations moved to top of scope
- **var**: Hoisted + initialized as `undefined`
- **let/const**: Hoisted but in TDZ until initialized
- **TDZ**: Time between hoisting and initialization
- **Function hoisting**: Full hoisting for declarations

**Common Interview Questions:**
1. "What is hoisting in JavaScript?"
2. "What's the difference between var, let, and const hoisting?"
3. "What is the Temporal Dead Zone?"
4. "Why can't I use let/const variables before declaration?"
5. "How does hoisting work with function declarations vs expressions?"

**Pro Tips:**
- **Always declare variables at scope top**: Prevents confusion
- **Use const by default**: Forces thinking about mutability
- **Be aware of TDZ in interviews**: Common gotcha questions
- **Understand scoping differences**: var=function scope, let/const=block scope
- **Debug with typeof**: Shows hoisting behavior differences

**Red Flags in Interviews:**
- Thinking var variables are not hoisted
- Not understanding TDZ causes ReferenceError
- Confusing hoisting with execution order
- Not knowing function declaration vs expression differences
- Thinking let/const variables are completely unhoisted

**Advanced Interview Questions:**
- "How does hoisting work in ES6 modules?"
- "What's the difference between hoisting in strict vs non-strict mode?"
- "How do default parameters interact with TDZ?"
- "Can you give an example of hoisting causing a bug?"
- "How does hoisting affect performance in JavaScript engines?"