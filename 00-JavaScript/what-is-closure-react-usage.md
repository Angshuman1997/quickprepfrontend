# What is a Closure?

## Simple Answer
A closure is a function that remembers variables from its outer scope even after the outer function finishes.

## Basic Example
```javascript
function outer() {
  let count = 0;

  function inner() {
    count++; // Can access 'count' from outer function
    return count;
  }

  return inner;
}

const counter = outer();
console.log(counter()); // 1
console.log(counter()); // 2
// 'count' is still accessible!
```

## In React
```javascript
function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(count + 1); // Closure captures 'count'
  };

  return <button onClick={increment}>Count: {count}</button>;
}
```

## Interview Q&A

**Q: What is a closure?**  
**A:** A closure is when a function remembers variables from where it was created, even after that outer function finishes.

**Q: Where do you use closures in React?**  
**A:** In event handlers, useEffect, and custom hooks - they capture state and props values.

**Q: What's a common closure problem in React?**  
**A:** Stale closures in useEffect where old state values are captured. Use functional updates to fix it.