# What will be the output order in this case: Promise vs setTimeout?

## Question
What will be the output order in this case: Promise vs setTimeout?

## Answer

**Promises always execute before setTimeout**, even with setTimeout delay of 0, because Promises use the microtask queue which has higher priority.

### Example 1 - Basic Comparison:
```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');

// Output:
// 1
// 4
// 3 (Promise - microtask)
// 2 (setTimeout - macrotask)
```

### Example 2 - Multiple Promises and Timeouts:
```javascript
setTimeout(() => console.log('timeout1'), 0);
setTimeout(() => console.log('timeout2'), 0);

Promise.resolve().then(() => console.log('promise1'));
Promise.resolve().then(() => console.log('promise2'));

console.log('sync');

// Output:
// sync
// promise1
// promise2  
// timeout1
// timeout2
```

### Example 3 - Nested Scenarios:
```javascript
console.log('start');

setTimeout(() => {
    console.log('timeout');
    Promise.resolve().then(() => console.log('promise inside timeout'));
}, 0);

Promise.resolve()
    .then(() => {
        console.log('promise1');
        setTimeout(() => console.log('timeout inside promise'), 0);
    })
    .then(() => console.log('promise2'));

console.log('end');

// Output:
// start
// end
// promise1
// promise2
// timeout
// promise inside timeout
// timeout inside promise
```

### Example 4 - Real React Scenario:
```javascript
function handleSubmit() {
    console.log('Submit started');
    
    // Synchronous
    validateForm();
    console.log('Validation done');
    
    // Macrotask
    setTimeout(() => {
        console.log('Show success message');
    }, 0);
    
    // Microtask
    submitToAPI()
        .then(response => {
            console.log('API response received');
            updateState(response);
        })
        .catch(error => {
            console.log('API error');
        });
    
    console.log('Submit function ended');
}

// Output order:
// Submit started
// Validation done
// Submit function ended
// API response received (or API error)
// Show success message
```

### Example 5 - Complex Chaining:
```javascript
Promise.resolve()
    .then(() => {
        console.log('Promise 1');
        setTimeout(() => console.log('Timeout 1'), 0);
        return 'result1';
    })
    .then((result) => {
        console.log('Promise 2:', result);
        return Promise.resolve('result2');
    })
    .then((result) => {
        console.log('Promise 3:', result);
    });

setTimeout(() => {
    console.log('Timeout 2');
    Promise.resolve().then(() => console.log('Promise inside timeout'));
}, 0);

// Output:
// Promise 1
// Promise 2: result1
// Promise 3: result2
// Timeout 1
// Timeout 2
// Promise inside timeout
```

### Memory Trick:
```
"Promises are impatient, timeouts are patient"
Microtasks (Promises) = VIP queue 🎫
Macrotasks (setTimeout) = Regular queue 🎟️
```

### Why This Matters:
1. **State updates** might not be immediate in timeouts
2. **API responses** will process before timer callbacks
3. **UI updates** happen between macrotasks, not microtasks
4. **Performance** - too many microtasks can block rendering

### Interview Tip:
*"When explaining this, draw the event loop queues on paper. Show that microtasks drain completely before any macrotask runs. This visual helps interviewers understand your grasp of asynchronous JavaScript."*