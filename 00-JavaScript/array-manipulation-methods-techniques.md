# Common Array Manipulation Methods and Techniques

## Question
What are the common array manipulation methods and techniques in JavaScript? Provide detailed examples and explain when to use each method.

## Detailed Answer

JavaScript arrays are fundamental data structures that provide powerful methods for manipulation, transformation, and iteration. Understanding these methods is crucial for efficient data handling in both vanilla JavaScript and React applications. This guide covers the most important array methods, their use cases, performance considerations, and practical examples.

### 1. **Mutation Methods (Modify Original Array)**

#### **Adding/Removing Elements**

**`push()`** - Adds elements to the end of an array
```javascript
const fruits = ['apple', 'banana'];
fruits.push('orange'); // Returns new length: 3
console.log(fruits); // ['apple', 'banana', 'orange']

// Add multiple elements
fruits.push('grape', 'kiwi'); // Returns 5
```

**`pop()`** - Removes the last element
```javascript
const fruits = ['apple', 'banana', 'orange'];
const removed = fruits.pop(); // Returns 'orange'
console.log(fruits); // ['apple', 'banana']
```

**`unshift()`** - Adds elements to the beginning
```javascript
const fruits = ['banana', 'orange'];
fruits.unshift('apple'); // Returns new length: 3
console.log(fruits); // ['apple', 'banana', 'orange']
```

**`shift()`** - Removes the first element
```javascript
const fruits = ['apple', 'banana', 'orange'];
const removed = fruits.shift(); // Returns 'apple'
console.log(fruits); // ['banana', 'orange']
```

**`splice()`** - Add/remove elements at specific positions
```javascript
const fruits = ['apple', 'banana', 'orange', 'grape'];

// Remove 1 element at index 1
const removed = fruits.splice(1, 1); // Returns ['banana']
console.log(fruits); // ['apple', 'orange', 'grape']

// Add elements at index 1 (no removal)
fruits.splice(1, 0, 'banana', 'kiwi'); // Returns []
console.log(fruits); // ['apple', 'banana', 'kiwi', 'orange', 'grape']

// Replace elements (remove 2, add 1)
fruits.splice(2, 2, 'mango'); // Returns ['kiwi', 'orange']
console.log(fruits); // ['apple', 'banana', 'mango', 'grape']
```

### 2. **Accessor Methods (Return New Arrays or Values)**

#### **Creating New Arrays**

**`slice()`** - Extracts a portion of an array without modifying the original
```javascript
const fruits = ['apple', 'banana', 'orange', 'grape', 'kiwi'];

// Extract from index 1 to 3 (not including 3)
const citrus = fruits.slice(1, 3); // ['banana', 'orange']
console.log(fruits); // Original unchanged: ['apple', 'banana', 'orange', 'grape', 'kiwi']

// Extract from index 2 to end
const fromOrange = fruits.slice(2); // ['orange', 'grape', 'kiwi']

// Extract last 2 elements
const lastTwo = fruits.slice(-2); // ['grape', 'kiwi']

// Shallow copy entire array
const copy = fruits.slice(); // ['apple', 'banana', 'orange', 'grape', 'kiwi']
```

**`concat()`** - Merges arrays into a new array
```javascript
const fruits = ['apple', 'banana'];
const vegetables = ['carrot', 'potato'];
const grains = ['rice', 'wheat'];

const food = fruits.concat(vegetables, grains);
// ['apple', 'banana', 'carrot', 'potato', 'rice', 'wheat']

// Modern approach with spread operator
const allFood = [...fruits, ...vegetables, ...grains];
```

**`Array.from()`** - Creates arrays from iterable objects
```javascript
// From string
const letters = Array.from('hello'); // ['h', 'e', 'l', 'l', 'o']

// From Set (removes duplicates)
const uniqueNumbers = Array.from(new Set([1, 2, 2, 3, 3, 3])); // [1, 2, 3]

// From NodeList (DOM elements)
const buttons = Array.from(document.querySelectorAll('button'));

// With mapping function
const squares = Array.from({ length: 5 }, (_, i) => (i + 1) ** 2); // [1, 4, 9, 16, 25]

// From arguments object
function sum() {
    return Array.from(arguments).reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4); // 10
```

#### **Searching and Finding**

**`indexOf()`** - Finds the first index of an element
```javascript
const fruits = ['apple', 'banana', 'orange', 'banana'];

console.log(fruits.indexOf('banana')); // 1 (first occurrence)
console.log(fruits.indexOf('grape'));  // -1 (not found)
console.log(fruits.indexOf('banana', 2)); // 3 (search from index 2)
```

**`lastIndexOf()`** - Finds the last index of an element
```javascript
const fruits = ['apple', 'banana', 'orange', 'banana'];
console.log(fruits.lastIndexOf('banana')); // 3
```

**`includes()`** - Checks if an element exists (ES2016)
```javascript
const fruits = ['apple', 'banana', 'orange'];
console.log(fruits.includes('banana')); // true
console.log(fruits.includes('grape'));  // false

// Case-sensitive
console.log(['Apple', 'Banana'].includes('apple')); // false
```

**`find()`** - Returns the first element that matches a condition
```javascript
const users = [
    { id: 1, name: 'Alice', age: 25 },
    { id: 2, name: 'Bob', age: 30 },
    { id: 3, name: 'Charlie', age: 35 }
];

const userOver30 = users.find(user => user.age > 30);
console.log(userOver30); // { id: 3, name: 'Charlie', age: 35 }

const userWithId2 = users.find(user => user.id === 2);
console.log(userWithId2); // { id: 2, name: 'Bob', age: 30 }

// Returns undefined if not found
const notFound = users.find(user => user.age > 50); // undefined
```

**`findIndex()`** - Returns the index of the first matching element
```javascript
const numbers = [10, 20, 30, 40, 50];
const index = numbers.findIndex(num => num > 25); // 2
### 3. **Transformation Methods**

**`map()`** - Transforms each element and returns a new array
```javascript
const numbers = [1, 2, 3, 4, 5];

// Double each number
const doubled = numbers.map(num => num * 2); // [2, 4, 6, 8, 10]

// Transform objects
const users = [
    { firstName: 'John', lastName: 'Doe' },
    { firstName: 'Jane', lastName: 'Smith' }
];

const fullNames = users.map(user => `${user.firstName} ${user.lastName}`);
// ['John Doe', 'Jane Smith']

// With index parameter
const indexed = numbers.map((num, index) => `${index}: ${num}`);
// ['0: 1', '1: 2', '2: 3', '3: 4', '4: 5']

// Complex transformation
const products = [
    { name: 'Laptop', price: 1000 },
    { name: 'Mouse', price: 50 }
];

const productsWithTax = products.map(product => ({
    ...product,
    priceWithTax: product.price * 1.1
}));
```

**`filter()`** - Creates a new array with elements that pass a test
```javascript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// Get even numbers
const evenNumbers = numbers.filter(num => num % 2 === 0); // [2, 4, 6, 8, 10]

// Get numbers greater than 5
const greaterThan5 = numbers.filter(num => num > 5); // [6, 7, 8, 9, 10]

// Filter objects
const products = [
    { name: 'Laptop', price: 1000, inStock: true },
    { name: 'Mouse', price: 50, inStock: false },
    { name: 'Keyboard', price: 80, inStock: true }
];

const availableProducts = products.filter(product => product.inStock);
// [{ name: 'Laptop', price: 1000, inStock: true }, { name: 'Keyboard', price: 80, inStock: true }]

const expensiveProducts = products.filter(product => product.price > 100);
// [{ name: 'Laptop', price: 1000, inStock: true }]

// Remove falsy values
const mixed = [0, 1, false, 2, '', 3, null, undefined, 4];
const truthy = mixed.filter(Boolean); // [1, 2, 3, 4]
```

**`reduce()`** - Reduces an array to a single value
```javascript
const numbers = [1, 2, 3, 4, 5];

// Sum all numbers
const sum = numbers.reduce((total, num) => total + num, 0); // 15

// Find maximum
const max = numbers.reduce((max, num) => num > max ? num : max, numbers[0]); // 5

// Count occurrences
const fruits = ['apple', 'banana', 'apple', 'orange', 'banana', 'apple'];
const count = fruits.reduce((acc, fruit) => {
    acc[fruit] = (acc[fruit] || 0) + 1;
    return acc;
}, {}); // { apple: 3, banana: 2, orange: 1 }

// Group objects by property
const people = [
    { name: 'Alice', department: 'Engineering' },
    { name: 'Bob', department: 'Sales' },
    { name: 'Charlie', department: 'Engineering' },
    { name: 'Diana', department: 'Sales' }
];

const byDepartment = people.reduce((groups, person) => {
    const dept = person.department;
    if (!groups[dept]) groups[dept] = [];
    groups[dept].push(person);
    return groups;
}, {});

// Result:
// {
//   Engineering: [{ name: 'Alice', department: 'Engineering' }, { name: 'Charlie', department: 'Engineering' }],
//   Sales: [{ name: 'Bob', department: 'Sales' }, { name: 'Diana', department: 'Sales' }]
// }

// Flatten nested arrays
const nested = [[1, 2], [3, 4], [5, 6]];
const flattened = nested.reduce((flat, arr) => flat.concat(arr), []); // [1, 2, 3, 4, 5, 6]

// Running total
const runningTotal = numbers.reduce((acc, num, index) => {
    acc.push((acc[index - 1] || 0) + num);
    return acc;
}, []); // [1, 3, 6, 10, 15]
```

### 4. **Sorting and Ordering**
```

### 4. **Sorting and Ordering**

**`sort()`** - Sorts array elements (mutates original array)
```javascript
const numbers = [3, 1, 4, 1, 5, 9, 2, 6];

// Sort numbers (ascending by default)
numbers.sort(); // [1, 1, 2, 3, 4, 5, 6, 9]

// Sort in descending order
numbers.sort((a, b) => b - a); // [9, 6, 5, 4, 3, 2, 1, 1]

// Sort strings (lexicographic order)
const fruits = ['banana', 'Apple', 'orange', 'grape'];
fruits.sort(); // ['Apple', 'banana', 'grape', 'orange'] (case-sensitive)

// Case-insensitive sort
fruits.sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()));

// Sort objects by property
const users = [
    { name: 'Alice', age: 25 },
    { name: 'Bob', age: 30 },
    { name: 'Charlie', age: 20 }
];

users.sort((a, b) => a.age - b.age);
// [{ name: 'Charlie', age: 20 }, { name: 'Alice', age: 25 }, { name: 'Bob', age: 30 }]

// Sort by multiple criteria
users.sort((a, b) => {
    if (a.age !== b.age) return a.age - b.age;
    return a.name.localeCompare(b.name);
});
```

**`reverse()`** - Reverses the order of elements
```javascript
const numbers = [1, 2, 3, 4, 5];
numbers.reverse(); // [5, 4, 3, 2, 1]

// Reverse string
const str = 'hello';
const reversed = str.split('').reverse().join(''); // 'olleh'
```

### 5. **Iteration Methods**

**`forEach()`** - Executes a function for each element (no return value)
```javascript
const numbers = [1, 2, 3, 4, 5];

// Log each number
numbers.forEach(num => console.log(num)); // 1, 2, 3, 4, 5

// With index and array parameters
numbers.forEach((num, index, array) => {
    console.log(`Index ${index}: ${num} (array length: ${array.length})`);
});

// Side effects (logging, DOM manipulation)
const elements = document.querySelectorAll('.item');
elements.forEach((el, index) => {
    el.textContent = `Item ${index + 1}`;
});

// Modify external state
let sum = 0;
numbers.forEach(num => sum += num);
console.log(sum); // 15
```

**`some()`** - Tests if at least one element passes the condition
```javascript
const numbers = [1, 2, 3, 4, 5];

const hasEven = numbers.some(num => num % 2 === 0); // true
const hasNegative = numbers.some(num => num < 0);   // false
const hasGreaterThan10 = numbers.some(num => num > 10); // false

// Check object properties
const users = [
    { name: 'Alice', active: false },
    { name: 'Bob', active: true },
    { name: 'Charlie', active: false }
];

const hasActiveUser = users.some(user => user.active); // true
```

**`every()`** - Tests if all elements pass the condition
```javascript
const numbers = [2, 4, 6, 8, 10];

const allEven = numbers.every(num => num % 2 === 0); // true
const allPositive = numbers.every(num => num > 0);   // true
const allGreaterThan5 = numbers.every(num => num > 5); // false

// Validate form fields
const fields = [
    { name: 'email', value: 'user@example.com', required: true },
    { name: 'name', value: 'John', required: true },
    { name: 'age', value: '', required: false }
];

const allRequiredFilled = fields
    .filter(field => field.required)
    .every(field => field.value.trim() !== '');
```

### **sort()** - Sort array elements
```javascript
const numbers = [3, 1, 4, 1, 5, 9, 2, 6];

// Sort numbers (mutates original array)
numbers.sort(); // [1, 1, 2, 3, 4, 5, 6, 9]

// Sort in descending order
numbers.sort((a, b) => b - a); // [9, 6, 5, 4, 3, 2, 1, 1]

// Sort strings
const fruits = ['banana', 'Apple', 'orange', 'grape'];
fruits.sort(); // ['Apple', 'banana', 'grape', 'orange'] (case-sensitive)

// Case-insensitive sort
fruits.sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()));

// Sort objects
const users = [
    { name: 'Alice', age: 25 },
    { name: 'Bob', age: 30 },
    { name: 'Charlie', age: 20 }
];

users.sort((a, b) => a.age - b.age);
// [{ name: 'Charlie', age: 20 }, { name: 'Alice', age: 25 }, { name: 'Bob', age: 30 }]
```

### **reverse()** - Reverse array order
```javascript
const numbers = [1, 2, 3, 4, 5];
numbers.reverse(); // [5, 4, 3, 2, 1]
```

## 6. **Iteration Methods**

### **forEach()** - Execute function for each element
```javascript
const numbers = [1, 2, 3, 4, 5];

// Log each number
numbers.forEach(num => console.log(num)); // 1, 2, 3, 4, 5

// With index
numbers.forEach((num, index) => {
    console.log(`Index ${index}: ${num}`);
});

// Modify original array
const doubled = [];
numbers.forEach(num => doubled.push(num * 2)); // [2, 4, 6, 8, 10]
```

### **some()** - Test if at least one element passes condition
```javascript
const numbers = [1, 2, 3, 4, 5];

const hasEven = numbers.some(num => num % 2 === 0); // true
const hasNegative = numbers.some(num => num < 0);   // false
const hasGreaterThan10 = numbers.some(num => num > 10); // false
```

### **every()** - Test if all elements pass condition
```javascript
const numbers = [2, 4, 6, 8, 10];

const allEven = numbers.every(num => num % 2 === 0); // true
const allPositive = numbers.every(num => num > 0);   // true
const allGreaterThan5 = numbers.every(num => num > 5); // false
```

## Code Examples

### **Todo List Management (React)**
```javascript
import React, { useState } from 'react';

function TodoApp() {
    const [todos, setTodos] = useState([
        { id: 1, text: 'Learn React', completed: false },
        { id: 2, text: 'Build app', completed: true },
        { id: 3, text: 'Deploy app', completed: false }
    ]);

    // Add todo
    const addTodo = (text) => {
        const newTodo = {
            id: Date.now(),
            text,
            completed: false
        };
        setTodos(prevTodos => [...prevTodos, newTodo]);
    };

    // Toggle todo
    const toggleTodo = (id) => {
        setTodos(prevTodos =>
            prevTodos.map(todo =>
                todo.id === id ? { ...todo, completed: !todo.completed } : todo
            )
        );
    };

    // Delete todo
    const deleteTodo = (id) => {
        setTodos(prevTodos => prevTodos.filter(todo => todo.id !== id));
    };

    // Get statistics
    const completedCount = todos.filter(todo => todo.completed).length;
    const pendingCount = todos.filter(todo => !todo.completed).length;

    return (
        <div>
            <h2>Todos: {completedCount} completed, {pendingCount} pending</h2>
            <button onClick={() => addTodo('New Task')}>Add Todo</button>
            <ul>
                {todos.map(todo => (
                    <li key={todo.id}>
                        <input
                            type="checkbox"
                            checked={todo.completed}
                            onChange={() => toggleTodo(todo.id)}
                        />
                        <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
                            {todo.text}
                        </span>
                        <button onClick={() => deleteTodo(todo.id)}>Delete</button>
                    </li>
                ))}
            </ul>
        </div>
    );
}
```

### **Shopping Cart with Array Methods**
```javascript
import React, { useState } from 'react';

function ShoppingCart() {
    const [cart, setCart] = useState([
        { id: 1, name: 'Laptop', price: 1000, quantity: 1 },
        { id: 2, name: 'Mouse', price: 50, quantity: 2 }
    ]);

    // Add item to cart
    const addItem = (item) => {
        setCart(prevCart => {
            const existingItem = prevCart.find(cartItem => cartItem.id === item.id);
            if (existingItem) {
                return prevCart.map(cartItem =>
                    cartItem.id === item.id
                        ? { ...cartItem, quantity: cartItem.quantity + 1 }
                        : cartItem
                );
            }
            return [...prevCart, { ...item, quantity: 1 }];
        });
    };

    // Remove item from cart
    const removeItem = (id) => {
        setCart(prevCart => prevCart.filter(item => item.id !== id));
    };

    // Update quantity
    const updateQuantity = (id, quantity) => {
        if (quantity <= 0) {
            removeItem(id);
            return;
        }
        setCart(prevCart =>
            prevCart.map(item =>
                item.id === id ? { ...item, quantity } : item
            )
        );
    };

    // Calculate totals
    const total = cart.reduce((sum, item) => sum + (item.price * item.quantity), 0);
    const totalItems = cart.reduce((sum, item) => sum + item.quantity, 0);

    return (
        <div>
            <h2>Shopping Cart ({totalItems} items)</h2>
            <p>Total: ${total.toFixed(2)}</p>
            <ul>
                {cart.map(item => (
                    <li key={item.id}>
                        {item.name} - ${item.price} x {item.quantity}
                        <button onClick={() => updateQuantity(item.id, item.quantity + 1)}>+</button>
                        <button onClick={() => updateQuantity(item.id, item.quantity - 1)}>-</button>
                        <button onClick={() => removeItem(item.id)}>Remove</button>
                    </li>
                ))}
            </ul>
        </div>
    );
}
```

### **Data Table with Sorting and Filtering**
```javascript
import React, { useState, useMemo } from 'react';

function DataTable({ data }) {
    const [sortConfig, setSortConfig] = useState({ key: null, direction: 'asc' });
    const [filterText, setFilterText] = useState('');

    // Sort data
    const sortedData = useMemo(() => {
        if (!sortConfig.key) return data;

        return [...data].sort((a, b) => {
            const aValue = a[sortConfig.key];
            const bValue = b[sortConfig.key];

            if (aValue < bValue) {
                return sortConfig.direction === 'asc' ? -1 : 1;
            }
            if (aValue > bValue) {
                return sortConfig.direction === 'asc' ? 1 : -1;
            }
            return 0;
        });
    }, [data, sortConfig]);

    // Filter data
    const filteredData = useMemo(() => {
        if (!filterText) return sortedData;
        return sortedData.filter(item =>
            Object.values(item).some(value =>
                String(value).toLowerCase().includes(filterText.toLowerCase())
            )
        );
    }, [sortedData, filterText]);

    // Handle sort
    const handleSort = (key) => {
        setSortConfig(prevConfig => ({
            key,
            direction: prevConfig.key === key && prevConfig.direction === 'asc' ? 'desc' : 'asc'
        }));
    };

    return (
        <div>
            <input
                type="text"
                placeholder="Search..."
                value={filterText}
                onChange={(e) => setFilterText(e.target.value)}
                style={{ marginBottom: '10px', padding: '5px' }}
            />
            <table border="1" style={{ width: '100%', borderCollapse: 'collapse' }}>
                <thead>
                    <tr>
                        {data.length > 0 && Object.keys(data[0]).map(key => (
                            <th
                                key={key}
                                onClick={() => handleSort(key)}
                                style={{ cursor: 'pointer', padding: '10px' }}
                            >
                                {key} {sortConfig.key === key ? (sortConfig.direction === 'asc' ? '↑' : '↓') : ''}
                            </th>
                        ))}
                    </tr>
                </thead>
                <tbody>
                    {filteredData.map((item, index) => (
                        <tr key={index}>
                            {Object.values(item).map((value, i) => (
                                <td key={i} style={{ padding: '10px' }}>{value}</td>
                            ))}
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
}

// Usage example
const sampleData = [
    { name: 'Alice', age: 25, city: 'New York' },
    { name: 'Bob', age: 30, city: 'San Francisco' },
    { name: 'Charlie', age: 20, city: 'Chicago' }
];

function App() {
    return <DataTable data={sampleData} />;
}
```

## Performance Considerations

### **Mutation vs Immutability**
- **Mutation methods** (`push`, `pop`, `splice`) modify the original array and are faster for small arrays
- **Accessor methods** (`map`, `filter`, `slice`) create new arrays and are preferred for immutability in React
- For large arrays (>1000 elements), consider the performance impact of creating new arrays

### **Method Selection Guidelines**
- Use `forEach` when you need side effects but don't need a return value
- Use `map` when transforming each element
- Use `filter` when selecting a subset of elements
- Use `find`/`findIndex` when you need the first match (stops iteration early)
- Use `some`/`every` for boolean checks
- Use `reduce` for complex aggregations or transformations

### **Memory and Performance Tips**
- **Chaining**: `array.map().filter().reduce()` creates intermediate arrays - consider combining operations
- **Large datasets**: For 10,000+ elements, consider pagination or virtualization
- **Primitive vs Objects**: Methods are generally fast, but complex operations on large object arrays can be expensive
- **Spread operator**: `[...array]` creates shallow copies - deep cloning requires libraries like Lodash

## Common Patterns and Best Practices

### **Functional Programming Patterns**
```javascript
// Compose functions
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);

// Example: process user data
const processUsers = compose(
    users => users.filter(user => user.active),
    users => users.map(user => ({ ...user, fullName: `${user.first} ${user.last}` })),
    users => users.sort((a, b) => a.age - b.age)
);

// Remove duplicates
const unique = array => array.filter((item, index) => array.indexOf(item) === index);
const uniqueBy = (array, key) => array.filter((item, index, arr) =>
    arr.findIndex(i => i[key] === item[key]) === index
);

// Group by multiple criteria
const groupBy = (array, keyFn) =>
    array.reduce((groups, item) => {
        const key = keyFn(item);
        (groups[key] = groups[key] || []).push(item);
        return groups;
    }, {});

// Flatten deeply nested arrays
const flattenDeep = array =>
    array.reduce((flat, item) =>
        Array.isArray(item) ? [...flat, ...flattenDeep(item)] : [...flat, item], []
    );

// Chunk array into smaller arrays
const chunk = (array, size) =>
    array.reduce((chunks, item, index) => {
        if (index % size === 0) chunks.push([]);
        chunks[chunks.length - 1].push(item);
        return chunks;
    }, []);
```

### **React-Specific Patterns**
```javascript
// Update nested state immutably
const updateNestedState = (state, path, value) => {
    const keys = path.split('.');
    const lastKey = keys.pop();
    const lastObj = keys.reduce((obj, key) => obj[key] = obj[key] || {}, state);
    lastObj[lastKey] = value;
    return { ...state };
};

// Optimistic updates
const optimisticUpdate = (items, updatedItem) =>
    items.map(item => item.id === updatedItem.id ? { ...item, ...updatedItem } : item);

// Handle loading states
const withLoadingState = async (asyncFn, setLoading) => {
    setLoading(true);
    try {
        const result = await asyncFn();
        return result;
    } finally {
        setLoading(false);
    }
};
```

## Interview Tips

**Key Differences to Remember:**
- `map()` vs `forEach()`: `map()` returns new array, `forEach()` doesn't return anything
- `filter()` vs `find()`: `filter()` returns all matches, `find()` returns first match or undefined
- `slice()` vs `splice()`: `slice()` doesn't mutate, `splice()` does
- `some()` vs `every()`: `some()` needs one truthy, `every()` needs all truthy

**Performance Questions:**
- When dealing with large arrays, prefer methods that stop early (`find`, `some`, `every`)
- For React apps, prefer immutable patterns to avoid unnecessary re-renders
- Consider memory usage when chaining multiple array methods

**Common Interview Questions:**
1. "How would you remove duplicates from an array?"
2. "What's the difference between `map()` and `forEach()`?"
3. "How do you flatten a nested array?"
4. "How would you group objects by a property?"
5. "What's the most efficient way to check if an array contains a value?"

**Pro Tip:** Master array methods by implementing them yourself, then understand their native implementations. This shows deep understanding during interviews.

### 1. **Managing Todo List State**
```javascript
function TodoApp() {
    const [todos, setTodos] = useState([
        { id: 1, text: 'Learn React', completed: false },
        { id: 2, text: 'Build app', completed: true },
        { id: 3, text: 'Deploy app', completed: false }
    ]);

    // Add todo
    const addTodo = (text) => {
        const newTodo = {
            id: Date.now(),
            text,
            completed: false
        };
        setTodos([...todos, newTodo]);
    };

    // Toggle todo
    const toggleTodo = (id) => {
        setTodos(todos.map(todo =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo
        ));
    };

    // Delete todo
    const deleteTodo = (id) => {
        setTodos(todos.filter(todo => todo.id !== id));
    };

    // Get completed todos
    const completedTodos = todos.filter(todo => todo.completed);

    // Get pending todos
    const pendingTodos = todos.filter(todo => !todo.completed);

    return (
        <div>
            <h2>Completed: {completedTodos.length}</h2>
            <h2>Pending: {pendingTodos.length}</h2>
            {/* Todo list rendering */}
        </div>
    );
}
```

### 2. **Shopping Cart with Array Methods**
```javascript
function ShoppingCart() {
    const [cart, setCart] = useState([
        { id: 1, name: 'Laptop', price: 1000, quantity: 1 },
        { id: 2, name: 'Mouse', price: 50, quantity: 2 }
    ]);

    // Add item to cart
    const addItem = (item) => {
        setCart(prevCart => {
            const existingItem = prevCart.find(cartItem => cartItem.id === item.id);
            if (existingItem) {
                return prevCart.map(cartItem =>
                    cartItem.id === item.id
                        ? { ...cartItem, quantity: cartItem.quantity + 1 }
                        : cartItem
                );
            }
            return [...prevCart, { ...item, quantity: 1 }];
        });
    };

    // Remove item from cart
    const removeItem = (id) => {
        setCart(prevCart => prevCart.filter(item => item.id !== id));
    };

    // Update quantity
    const updateQuantity = (id, quantity) => {
        if (quantity <= 0) {
            removeItem(id);
            return;
        }
        setCart(prevCart =>
            prevCart.map(item =>
                item.id === id ? { ...item, quantity } : item
            )
        );
    };

    // Calculate total
    const total = cart.reduce((sum, item) => sum + (item.price * item.quantity), 0);

    // Get total items count
    const totalItems = cart.reduce((sum, item) => sum + item.quantity, 0);

    return (
        <div>
            <h2>Cart ({totalItems} items)</h2>
            <p>Total: ${total}</p>
            {/* Cart items rendering */}
        </div>
    );
}
```

### 3. **Data Table with Sorting and Filtering**
```javascript
function DataTable({ data }) {
    const [sortConfig, setSortConfig] = useState({ key: null, direction: 'asc' });
    const [filterText, setFilterText] = useState('');

    // Sort data
    const sortedData = useMemo(() => {
        if (!sortConfig.key) return data;

        return [...data].sort((a, b) => {
            if (a[sortConfig.key] < b[sortConfig.key]) {
                return sortConfig.direction === 'asc' ? -1 : 1;
            }
            if (a[sortConfig.key] > b[sortConfig.key]) {
                return sortConfig.direction === 'asc' ? 1 : -1;
            }
            return 0;
        });
    }, [data, sortConfig]);

    // Filter data
    const filteredData = useMemo(() => {
        if (!filterText) return sortedData;
        return sortedData.filter(item =>
            Object.values(item).some(value =>
                String(value).toLowerCase().includes(filterText.toLowerCase())
            )
        );
    }, [sortedData, filterText]);

    // Handle sort
    const handleSort = (key) => {
        setSortConfig(prevConfig => ({
            key,
            direction: prevConfig.key === key && prevConfig.direction === 'asc' ? 'desc' : 'asc'
        }));
    };

    return (
        <div>
            <input
                type="text"
                placeholder="Search..."
                value={filterText}
                onChange={(e) => setFilterText(e.target.value)}
            />
            <table>
                <thead>
                    <tr>
                        {Object.keys(data[0] || {}).map(key => (
                            <th key={key} onClick={() => handleSort(key)}>
                                {key} {sortConfig.key === key ? (sortConfig.direction === 'asc' ? '↑' : '↓') : ''}
                            </th>
                        ))}
                    </tr>
                </thead>
                <tbody>
                    {filteredData.map((item, index) => (
                        <tr key={index}>
                            {Object.values(item).map((value, i) => (
                                <td key={i}>{value}</td>
                            ))}
                        </tr>
                    ))}
                </tbody>
            </table>
        </div>
    );
}
```

## Performance Considerations:

- **`map()`**, **`filter()`**, **`reduce()`** create new arrays - use sparingly for large datasets
- **`forEach()`** is faster than **`map()`** when you don't need a new array
- **`find()`** stops at first match - more efficient than **`filter()`** for single results
- **Spread operator** creates shallow copies - deep cloning requires other methods
- **Array methods** are generally optimized by JavaScript engines

## Common Patterns:

1. **Transform and Filter**: `data.map(transform).filter(condition)`
2. **Find and Update**: `array.map(item => item.id === id ? updatedItem : item)`
3. **Remove by Index**: `array.filter((_, index) => index !== removeIndex)`
4. **Group by Property**: `array.reduce((groups, item) => (groups[item.key] = [...(groups[item.key] || []), item], groups), {})`
5. **Unique Values**: `array.filter((item, index) => array.indexOf(item) === index)`

### Interview Tip:
*"JavaScript arrays have mutation methods (push, pop, splice) that change the original array and accessor methods (map, filter, reduce) that return new arrays. Choose based on whether you need immutability. Method chaining with map/filter/reduce is powerful for data transformations."*