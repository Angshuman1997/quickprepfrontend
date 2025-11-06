# Common array manipulation methods and techniques

## Question
Common array manipulation methods and techniques

## Answer

JavaScript provides a rich set of built-in methods for array manipulation, along with various techniques for common operations. Arrays are mutable objects, and these methods help perform operations efficiently.

## 1. **Adding/Removing Elements**

### **push()** - Add to end
```javascript
const fruits = ['apple', 'banana'];
fruits.push('orange'); // Returns new length: 3
console.log(fruits); // ['apple', 'banana', 'orange']
```

### **pop()** - Remove from end
```javascript
const fruits = ['apple', 'banana', 'orange'];
const removed = fruits.pop(); // Returns removed element: 'orange'
console.log(fruits); // ['apple', 'banana']
```

### **unshift()** - Add to beginning
```javascript
const fruits = ['banana', 'orange'];
fruits.unshift('apple'); // Returns new length: 3
console.log(fruits); // ['apple', 'banana', 'orange']
```

### **shift()** - Remove from beginning
```javascript
const fruits = ['apple', 'banana', 'orange'];
const removed = fruits.shift(); // Returns removed element: 'apple'
console.log(fruits); // ['banana', 'orange']
```

### **splice()** - Add/Remove at specific position
```javascript
const fruits = ['apple', 'banana', 'orange', 'grape'];

// Remove 1 element at index 1
const removed = fruits.splice(1, 1); // Returns ['banana']
console.log(fruits); // ['apple', 'orange', 'grape']

// Add elements at index 1
fruits.splice(1, 0, 'banana', 'kiwi'); // Returns []
console.log(fruits); // ['apple', 'banana', 'kiwi', 'orange', 'grape']

// Replace elements
fruits.splice(2, 2, 'mango'); // Returns ['kiwi', 'orange']
console.log(fruits); // ['apple', 'banana', 'mango', 'grape']
```

## 2. **Creating New Arrays**

### **slice()** - Extract portion of array
```javascript
const fruits = ['apple', 'banana', 'orange', 'grape', 'kiwi'];

// Extract from index 1 to 3 (not including 3)
const citrus = fruits.slice(1, 3); // ['banana', 'orange']
console.log(fruits); // Original unchanged

// Extract from index 2 to end
const fromOrange = fruits.slice(2); // ['orange', 'grape', 'kiwi']

// Extract last 2 elements
const lastTwo = fruits.slice(-2); // ['grape', 'kiwi']
```

### **concat()** - Merge arrays
```javascript
const fruits = ['apple', 'banana'];
const vegetables = ['carrot', 'potato'];
const grains = ['rice', 'wheat'];

const food = fruits.concat(vegetables, grains);
// ['apple', 'banana', 'carrot', 'potato', 'rice', 'wheat']

// With spread operator (modern way)
const allFood = [...fruits, ...vegetables, ...grains];
```

### **Array.from()** - Create array from iterable
```javascript
// From string
const letters = Array.from('hello'); // ['h', 'e', 'l', 'l', 'o']

// From Set
const uniqueNumbers = Array.from(new Set([1, 2, 2, 3, 3, 3])); // [1, 2, 3]

// From NodeList
const buttons = Array.from(document.querySelectorAll('button'));

// With mapping function
const numbers = Array.from({ length: 5 }, (_, i) => i * 2); // [0, 2, 4, 6, 8]
```

## 3. **Searching and Finding**

### **indexOf()** - Find index of element
```javascript
const fruits = ['apple', 'banana', 'orange', 'banana'];

console.log(fruits.indexOf('banana')); // 1 (first occurrence)
console.log(fruits.indexOf('grape'));  // -1 (not found)
console.log(fruits.indexOf('banana', 2)); // 3 (search from index 2)
```

### **lastIndexOf()** - Find last index of element
```javascript
const fruits = ['apple', 'banana', 'orange', 'banana'];
console.log(fruits.lastIndexOf('banana')); // 3
```

### **includes()** - Check if element exists
```javascript
const fruits = ['apple', 'banana', 'orange'];
console.log(fruits.includes('banana')); // true
console.log(fruits.includes('grape'));  // false
```

### **find()** - Find first element matching condition
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
```

### **findIndex()** - Find index of first matching element
```javascript
const numbers = [10, 20, 30, 40, 50];
const index = numbers.findIndex(num => num > 25); // 2
```

## 4. **Transformation Methods**

### **map()** - Transform each element
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
```

### **filter()** - Filter elements based on condition
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
```

### **reduce()** - Reduce array to single value
```javascript
const numbers = [1, 2, 3, 4, 5];

// Sum all numbers
const sum = numbers.reduce((total, num) => total + num, 0); // 15

// Find maximum
const max = numbers.reduce((max, num) => num > max ? num : max, numbers[0]); // 5

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

// Flatten array
const nested = [[1, 2], [3, 4], [5, 6]];
const flattened = nested.reduce((flat, arr) => flat.concat(arr), []); // [1, 2, 3, 4, 5, 6]
```

## 5. **Sorting and Reversing**

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

## 7. **Advanced Techniques**

### **Chaining Methods**
```javascript
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

const result = numbers
    .filter(num => num % 2 === 0)     // [2, 4, 6, 8, 10]
    .map(num => num * 2)              // [4, 8, 12, 16, 20]
    .reduce((sum, num) => sum + num, 0); // 60

console.log(result); // 60
```

### **Spread Operator for Immutability**
```javascript
const original = [1, 2, 3];

// Add element immutably
const added = [...original, 4]; // [1, 2, 3, 4]

// Remove element immutably
const removed = original.filter(num => num !== 2); // [1, 3]

// Update element immutably
const updated = original.map(num => num === 2 ? 20 : num); // [1, 20, 3]
```

### **Array Destructuring**
```javascript
const [first, second, ...rest] = [1, 2, 3, 4, 5];
console.log(first);  // 1
console.log(second); // 2
console.log(rest);   // [3, 4, 5]

// Swap elements
let a = 1, b = 2;
[a, b] = [b, a];
console.log(a, b); // 2, 1
```

## Real React Examples:

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