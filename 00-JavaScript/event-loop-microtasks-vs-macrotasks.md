# In JavaScript's event loop, which executes first: microtasks or macrotasks?

## Question
In JavaScript's event loop, which executes first: microtasks or macrotasks?

## Answer

**Microtasks execute first** before any macrotasks. The event loop prioritizes the microtask queue over the macrotask queue.

### Event Loop Priority Order:
1. **Call Stack** (synchronous code)
2. **Microtask Queue** (higher priority)
3. **Macrotask Queue** (lower priority)

### Microtasks vs Macrotasks:

| Microtasks | Macrotasks |
|------------|------------|
| `Promise.then()` | `setTimeout()` |
| `Promise.catch()` | `setInterval()` |
| `Promise.finally()` | `setImmediate()` (Node.js) |
| `queueMicrotask()` | DOM events |
| `MutationObserver` | I/O operations |

### Execution Example:
```javascript
console.log('1: Start');

setTimeout(() => console.log('2: Macrotask'), 0);

Promise.resolve().then(() => console.log('3: Microtask'));

console.log('4: End');

// Output:
// 1: Start
// 4: End  
// 3: Microtask (executes first!)
// 2: Macrotask (executes after all microtasks)
```

### Complex Example:
```javascript
console.log('Start');

setTimeout(() => {
    console.log('Timeout 1');
    Promise.resolve().then(() => console.log('Promise inside timeout'));
}, 0);

Promise.resolve()
    .then(() => {
        console.log('Promise 1');
        return Promise.resolve();
    })
    .then(() => console.log('Promise 2'));

setTimeout(() => console.log('Timeout 2'), 0);

console.log('End');

// Output:
// Start
// End
// Promise 1
// Promise 2
// Timeout 1
// Promise inside timeout  
// Timeout 2
```

### Why This Matters in React:

**1. State Updates and Effects:**
```javascript
function Component() {
    const [count, setCount] = useState(0);
    
    const handleClick = () => {
        console.log('1: Click handler start');
        
        setCount(1); // Schedules microtask
        console.log('2: After setState');
        
        setTimeout(() => {
            console.log('4: Timeout - count is', count); // May be stale
        }, 0);
        
        Promise.resolve().then(() => {
            console.log('3: Promise - count is', count); // Executes before timeout
        });
    };
    
    return <button onClick={handleClick}>Click me</button>;
}
```

**2. API Calls and DOM Updates:**
```javascript
function SearchComponent() {
    const [results, setResults] = useState([]);
    
    const search = async (query) => {
        console.log('Search started');
        
        // This Promise resolves in microtask queue
        const data = await fetch(`/api/search?q=${query}`);
        setResults(data);
        
        // This runs after the Promise resolves (microtask)
        setTimeout(() => {
            console.log('DOM updated, scroll to top');
            window.scrollTo(0, 0);
        }, 0);
    };
}
```

### Visual Event Loop:
```
┌─────────────────┐
│   Call Stack    │ ← Executes immediately
└─────────────────┘
         ↓
┌─────────────────┐
│ Microtask Queue │ ← Higher priority (Promises)
└─────────────────┘
         ↓
┌─────────────────┐
│ Macrotask Queue │ ← Lower priority (setTimeout)
└─────────────────┘
```

### Important Rules:
1. **All microtasks** must complete before any macrotask runs
2. **New microtasks** can be added during microtask execution
3. **DOM rendering** happens between macrotasks, not between microtasks

### Interview Tip:
*"Remember: Promises beat timeouts! Microtasks always execute before macrotasks, which is why Promise.then() runs before setTimeout even with 0 delay."*