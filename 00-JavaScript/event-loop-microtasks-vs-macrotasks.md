# Event Loop: Microtasks vs Macrotasks

## Simple Answer
Microtasks run before macrotasks. Promises execute before setTimeout!

## Example
```javascript
console.log('Start');

setTimeout(() => console.log('Timeout'), 0);

Promise.resolve().then(() => console.log('Promise'));

console.log('End');

// Output:
// Start
// End
// Promise  ← Microtask runs first!
// Timeout  ← Macrotask runs after
```

## What are they?

**Microtasks (higher priority):**
- Promise.then/catch/finally
- queueMicrotask()

**Macrotasks (lower priority):**
- setTimeout/setInterval
- DOM events
- I/O operations

## Interview Q&A

**Q: Which executes first: microtasks or macrotasks?**  
**A:** Microtasks always execute before macrotasks. Promises run before setTimeout.

**Q: Why does this matter?**  
**A:** It affects when your code runs. Promises resolve before timeouts, even with 0 delay.

**Q: What's a practical example?**  
**A:** State updates in React use microtasks, so they happen before setTimeout callbacks.