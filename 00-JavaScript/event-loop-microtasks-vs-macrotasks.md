# Event Loop: Microtasks vs Macrotasks

## Question
How does JavaScript's event loop work? What are microtasks and macrotasks, and how do they differ in execution order? Provide detailed examples and explain their impact on asynchronous programming.

## Detailed Answer

JavaScript's event loop is the core mechanism that enables asynchronous programming in a single-threaded environment. Understanding microtasks vs macrotasks is crucial for predicting execution order, debugging timing issues, and writing performant asynchronous code. This guide covers the event loop architecture, task prioritization, and practical implications for modern JavaScript development.

### 1. **JavaScript Runtime Architecture**

JavaScript runs in a single-threaded environment but handles asynchronous operations through the event loop. The key components are:

- **Call Stack**: Executes synchronous code
- **Web APIs**: Browser/Node.js APIs (setTimeout, fetch, DOM events)
- **Task Queues**: 
  - **Macrotask Queue** (or Task Queue)
  - **Microtask Queue**
- **Event Loop**: Orchestrates execution between queues

**Visual Representation:**
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────┐
│   Call Stack    │    │  Web APIs/Node   │    │ Task Queues │
│                 │    │                  │    │             │
│ console.log()   │    │ setTimeout()     │    │ Macrotasks  │
│ function()      │    │ fetch()          │    │ Microtasks  │
│                 │    │ DOM events       │    │             │
└─────────────────┘    └──────────────────┘    └─────────────┘
         │                        │                    │
         └────────────────────────┼────────────────────┘
                                  │
                         ┌─────────────┐
                         │ Event Loop  │
                         │             │
                         │ 1. Check    │
                         │    Stack    │
                         │ 2. Micro-   │
                         │    tasks    │
                         │ 3. Macro-   │
                         │    tasks    │
                         │ 4. Render   │
                         └─────────────┘
```

### 2. **Task Types and Priorities**

#### **Macrotasks (Lower Priority)**
Macrotasks are scheduled for execution in the next iteration of the event loop. They include:

- `setTimeout()`, `setInterval()`
- `setImmediate()` (Node.js)
- I/O operations (file reading, network requests)
- DOM events (click, scroll, etc.)
- `requestAnimationFrame()`
- `MessageChannel`

#### **Microtasks (Higher Priority)**
Microtasks execute immediately after the current task completes, before the next macrotask. They include:

- `Promise.then()`, `Promise.catch()`, `Promise.finally()`
- `async/await` (which uses Promises internally)
- `queueMicrotask()`
- MutationObserver callbacks
- `process.nextTick()` (Node.js)

**Priority Order:**
1. **Synchronous code** (call stack)
2. **Microtasks** (Promise callbacks, async/await)
3. **Macrotasks** (setTimeout, DOM events)
4. **Rendering** (browser repaints/reflows)

### 3. **Event Loop Execution Flow**

The event loop follows this algorithm:

1. **Execute synchronous code** until call stack is empty
2. **Process all microtasks** in the queue
3. **Process one macrotask** from the queue
4. **Check for rendering updates** (browser only)
5. **Repeat**

**Key Insight:** Microtasks can starve macrotasks if they're continuously added to the queue.

### 4. **Basic Examples**

#### **Classic Microtask vs Macrotask**
```javascript
console.log('Start');

setTimeout(() => {
    console.log('Timeout (Macrotask)');
}, 0);

Promise.resolve().then(() => {
    console.log('Promise (Microtask)');
});

console.log('End');

// Output:
// Start
// End
// Promise (Microtask)  ← Executes first!
// Timeout (Macrotask)  ← Executes after
```

#### **Multiple Tasks**
```javascript
console.log('Script start');

setTimeout(() => {
    console.log('setTimeout 1');
}, 0);

Promise.resolve().then(() => {
    console.log('Promise 1');
});

setTimeout(() => {
    console.log('setTimeout 2');
}, 0);

Promise.resolve().then(() => {
    console.log('Promise 2');
});

console.log('Script end');

// Output:
// Script start
// Script end
// Promise 1
// Promise 2
// setTimeout 1
// setTimeout 2
```

#### **Nested Tasks**
```javascript
console.log('Start');

setTimeout(() => {
    console.log('Timeout 1');
    Promise.resolve().then(() => {
        console.log('Promise inside timeout');
    });
}, 0);

Promise.resolve().then(() => {
    console.log('Promise 1');
    setTimeout(() => {
        console.log('Timeout inside promise');
    }, 0);
});

console.log('End');

// Output:
// Start
// End
// Promise 1
// Timeout 1
// Promise inside timeout
// Timeout inside promise
```

### 5. **Advanced Patterns and Edge Cases**

#### **Microtask Starvation**
```javascript
// WARNING: This can freeze the browser!
function infiniteMicrotasks() {
    Promise.resolve().then(() => {
        console.log('Microtask');
        infiniteMicrotasks(); // Adds another microtask
    });
}

// setTimeout will never execute!
setTimeout(() => console.log('This never runs'), 0);
infiniteMicrotasks();
```

#### **queueMicrotask() Usage**
```javascript
console.log('Start');

// Direct microtask
queueMicrotask(() => {
    console.log('Queued microtask');
});

// Promise microtask
Promise.resolve().then(() => {
    console.log('Promise microtask');
});

console.log('End');

// Output:
// Start
// End
// Queued microtask
// Promise microtask
```

#### **Complex Nesting**
```javascript
console.log('Start');

setTimeout(() => {
    console.log('Timeout 1');
    Promise.resolve().then(() => {
        console.log('Promise in timeout 1');
        setTimeout(() => {
            console.log('Timeout in promise');
        }, 0);
    });
}, 0);

Promise.resolve().then(() => {
    console.log('Promise 1');
    setTimeout(() => {
        console.log('Timeout in promise 1');
        Promise.resolve().then(() => {
            console.log('Promise in timeout in promise');
        });
    }, 0);
});

console.log('End');

// Output:
// Start
// End
// Promise 1
// Timeout 1
// Promise in timeout 1
// Timeout in promise 1
// Timeout in promise
// Promise in timeout in promise
```

### 6. **React and Framework Implications**

#### **State Updates and Re-renders**
```javascript
function Counter() {
    const [count, setCount] = useState(0);

    const handleClick = () => {
        console.log('Before setState:', count);

        setCount(count + 1); // Microtask - state update
        console.log('After setState:', count); // Still old value!

        setTimeout(() => {
            console.log('In setTimeout:', count); // Still old value!
        }, 0);

        Promise.resolve().then(() => {
            console.log('In Promise:', count); // Still old value!
        });
    };

    // State updates happen in microtasks
    useEffect(() => {
        console.log('Effect runs with new count:', count);
    }, [count]);

    return (
        <button onClick={handleClick}>
            Count: {count}
        </button>
    );
}
```

#### **useEffect Timing**
```javascript
function DataFetcher() {
    const [data, setData] = useState(null);

    useEffect(() => {
        console.log('Effect runs');

        fetch('/api/data')
            .then(response => response.json())
            .then(data => {
                console.log('Data fetched');
                setData(data);
            });

        setTimeout(() => {
            console.log('Timeout in effect');
        }, 0);

        Promise.resolve().then(() => {
            console.log('Promise in effect');
        });
    }, []);

    // Output order:
    // Effect runs
    // Promise in effect  ← Microtask
    // Timeout in effect  ← Macrotask
    // Data fetched       ← Network macrotask
}
```

#### **Concurrent Mode Considerations**
```javascript
// In React 18+ Concurrent Mode, rendering can be interrupted
function ConcurrentComponent() {
    const [count, setCount] = useState(0);

    const handleExpensiveClick = () => {
        setCount(c => c + 1);

        // Expensive synchronous work
        const start = Date.now();
        while (Date.now() - start < 100) {} // Block for 100ms

        setCount(c => c + 1);
    };

    // In Concurrent Mode:
    // - Initial render might be interrupted
    // - State updates are batched as microtasks
    // - UI updates happen after microtasks complete

    return <button onClick={handleExpensiveClick}>Count: {count}</button>;
}
```

### 7. **Performance and Debugging**

#### **Performance Implications**
```javascript
// BAD: Microtask spam
function badPerformance() {
    for (let i = 0; i < 1000; i++) {
        Promise.resolve().then(() => {
            // Do something
        });
    }
    // All 1000 microtasks execute before any macrotasks
}

// GOOD: Batch microtasks
function goodPerformance() {
    const tasks = [];
    for (let i = 0; i < 1000; i++) {
        tasks.push(/* task data */);
    }

    // Single microtask handles all work
    Promise.resolve().then(() => {
        tasks.forEach(task => {
            // Process task
        });
    });
}
```

#### **Debugging Event Loop Issues**
```javascript
// Debug helper
function logWithStack(message) {
    console.log(message, new Error().stack);
}

// Track execution order
console.log('=== Start ===');

setTimeout(() => logWithStack('Macrotask: setTimeout'), 0);

Promise.resolve().then(() => logWithStack('Microtask: Promise'));

queueMicrotask(() => logWithStack('Microtask: queueMicrotask'));

requestAnimationFrame(() => logWithStack('Macrotask: rAF'));

console.log('=== End synchronous code ===');
```

#### **Testing Event Loop Behavior**
```javascript
// Helper to flush microtasks
function flushMicrotasks() {
    return new Promise(resolve => {
        queueMicrotask(resolve);
    });
}

// Helper to flush macrotasks
function flushMacrotasks() {
    return new Promise(resolve => {
        setTimeout(resolve, 0);
    });
}

// Test function
async function testEventLoop() {
    console.log('1. Synchronous');

    Promise.resolve().then(() => console.log('2. Microtask'));

    setTimeout(() => console.log('3. Macrotask'), 0);

    await flushMicrotasks();
    console.log('4. After microtasks flushed');

    await flushMacrotasks();
    console.log('5. After macrotasks flushed');
}

// Output:
// 1. Synchronous
// 2. Microtask
// 4. After microtasks flushed
// 3. Macrotask
// 5. After macrotasks flushed
```

## Code Examples

### **Custom Event Loop Simulator**
```javascript
class EventLoopSimulator {
    constructor() {
        this.callStack = [];
        this.microtaskQueue = [];
        this.macrotaskQueue = [];
        this.isRunning = false;
    }

    // Simulate synchronous execution
    executeSync(code) {
        console.log(`Executing: ${code}`);
        this.callStack.push(code);
        // Simulate execution
        this.callStack.pop();
    }

    // Add microtask
    addMicrotask(task) {
        this.microtaskQueue.push(task);
    }

    // Add macrotask
    addMacrotask(task) {
        this.macrotaskQueue.push(task);
    }

    // Run event loop
    async run() {
        this.isRunning = true;

        while (this.isRunning) {
            // Process all microtasks first
            while (this.microtaskQueue.length > 0) {
                const task = this.microtaskQueue.shift();
                console.log(`Microtask: ${task}`);
                await this.simulateDelay(10); // Simulate async execution
            }

            // Process one macrotask
            if (this.macrotaskQueue.length > 0) {
                const task = this.macrotaskQueue.shift();
                console.log(`Macrotask: ${task}`);
                await this.simulateDelay(10);
            }

            // Simulate rendering
            console.log('Render cycle');

            // Stop after processing all tasks
            if (this.microtaskQueue.length === 0 && this.macrotaskQueue.length === 0) {
                this.isRunning = false;
            }
        }
    }

    simulateDelay(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Usage
const simulator = new EventLoopSimulator();

simulator.executeSync('console.log("Start")');

simulator.addMacrotask('setTimeout callback');
simulator.addMicrotask('Promise.then callback');
simulator.addMicrotask('Another Promise.then');

simulator.executeSync('console.log("End")');

simulator.run();
```

### **Real-World React Hook with Event Loop Awareness**
```javascript
import { useState, useEffect, useCallback, useRef } from 'react';

// Custom hook that batches state updates
function useBatchedState(initialState) {
    const [state, setState] = useState(initialState);
    const batchedUpdates = useRef([]);

    const batchUpdate = useCallback((update) => {
        batchedUpdates.current.push(update);

        // Schedule batched update as microtask
        queueMicrotask(() => {
            if (batchedUpdates.current.length > 0) {
                setState(currentState => {
                    let newState = currentState;
                    batchedUpdates.current.forEach(updateFn => {
                        newState = updateFn(newState);
                    });
                    batchedUpdates.current = [];
                    return newState;
                });
            }
        });
    }, []);

    return [state, batchUpdate];
}

// Usage in component
function BatchUpdateComponent() {
    const [items, batchUpdate] = useBatchedState([]);

    const addMultipleItems = () => {
        // All these updates get batched into a single render
        batchUpdate(items => [...items, 'Item 1']);
        batchUpdate(items => [...items, 'Item 2']);
        batchUpdate(items => [...items, 'Item 3']);

        // Only one re-render occurs after all microtasks complete
    };

    return (
        <div>
            <button onClick={addMultipleItems}>Add Items</button>
            <ul>
                {items.map((item, index) => (
                    <li key={index}>{item}</li>
                ))}
            </ul>
        </div>
    );
}
```

### **Async Iterator with Event Loop Control**
```javascript
// Custom async iterator that yields control to event loop
function createAsyncIterator(items, delay = 0) {
    return {
        [Symbol.asyncIterator]() {
            let index = 0;
            return {
                async next() {
                    if (index >= items.length) {
                        return { done: true };
                    }

                    // Yield control to event loop
                    if (delay > 0) {
                        await new Promise(resolve => setTimeout(resolve, delay));
                    } else {
                        await Promise.resolve(); // Microtask yield
                    }

                    const value = items[index++];
                    return { value, done: false };
                }
            };
        }
    };
}

// Usage
async function processItems() {
    const items = ['A', 'B', 'C', 'D', 'E'];

    // Process with microtask yielding (fast)
    console.log('=== Microtask yielding ===');
    for await (const item of createAsyncIterator(items)) {
        console.log(`Processing ${item}`);
        // Other microtasks can run here
    }

    // Process with macrotask yielding (slower but allows rendering)
    console.log('=== Macrotask yielding ===');
    for await (const item of createAsyncIterator(items, 10)) {
        console.log(`Processing ${item}`);
        // Browser can render between iterations
    }
}

processItems();
```

## Performance Considerations

### **Microtask vs Macrotask Performance**
- **Microtasks**: Faster execution, no delay, but can block rendering
- **Macrotasks**: Allow rendering between executions, better for UI responsiveness
- **Trade-off**: Use microtasks for performance, macrotasks for responsiveness

### **Memory and Starvation Issues**
```javascript
// Memory leak potential with microtasks
function memoryLeak() {
    const elements = document.querySelectorAll('*');

    elements.forEach(element => {
        Promise.resolve().then(() => {
            // This creates thousands of microtasks
            // All execute before next macrotask (like user input)
            element.style.color = 'red';
        });
    });
}

// Better approach
function betterPerformance() {
    const elements = document.querySelectorAll('*');

    // Single microtask handles all updates
    queueMicrotask(() => {
        elements.forEach(element => {
            element.style.color = 'red';
        });
    });
}
```

### **Browser Rendering and Event Loop**
- **Rendering happens after microtasks complete**
- **Use `requestAnimationFrame`** for visual updates
- **Avoid microtask loops** that prevent rendering
- **Consider `requestIdleCallback`** for non-urgent work

### **Node.js vs Browser Differences**
```javascript
// Node.js specific
process.nextTick(() => {
    console.log('nextTick (microtask in Node.js)');
});

// Browser specific
requestAnimationFrame(() => {
    console.log('rAF (macrotask in browser)');
});

// Cross-platform
queueMicrotask(() => {
    console.log('queueMicrotask (works everywhere)');
});
```

## Common Patterns and Best Practices

### **Deferring Work**
```javascript
// Defer to next macrotask (allows rendering)
function deferToNextTick(callback) {
    setTimeout(callback, 0);
}

// Defer to next microtask (immediate but after current stack)
function deferToMicrotask(callback) {
    queueMicrotask(callback);
}

// Defer to next frame (browser only)
function deferToNextFrame(callback) {
    requestAnimationFrame(callback);
}
```

### **Batching Updates**
```javascript
class UpdateBatcher {
    constructor() {
        this.updates = [];
        this.scheduled = false;
    }

    add(update) {
        this.updates.push(update);

        if (!this.scheduled) {
            this.scheduled = true;
            queueMicrotask(() => {
                this.flush();
            });
        }
    }

    flush() {
        const updates = this.updates;
        this.updates = [];
        this.scheduled = false;

        updates.forEach(update => update());
    }
}

// Usage
const batcher = new UpdateBatcher();

function updateUI() {
    batcher.add(() => console.log('Update 1'));
    batcher.add(() => console.log('Update 2'));
    batcher.add(() => console.log('Update 3'));
    // All updates batched into single microtask
}
```

### **Event Loop Aware Scheduling**
```javascript
class SmartScheduler {
    constructor() {
        this.microtaskQueue = [];
        this.macrotaskQueue = [];
        this.isProcessing = false;
    }

    // High priority (microtask)
    scheduleHigh(callback) {
        this.microtaskQueue.push(callback);
        this.process();
    }

    // Low priority (macrotask)
    scheduleLow(callback) {
        this.macrotaskQueue.push(callback);
        this.process();
    }

    async process() {
        if (this.isProcessing) return;
        this.isProcessing = true;

        // Process all microtasks first
        while (this.microtaskQueue.length > 0) {
            const task = this.microtaskQueue.shift();
            await Promise.resolve(); // Yield control
            task();
        }

        // Process macrotasks with yielding
        while (this.macrotaskQueue.length > 0) {
            const task = this.macrotaskQueue.shift();
            await new Promise(resolve => setTimeout(resolve, 0));
            task();
        }

        this.isProcessing = false;
    }
}

// Usage
const scheduler = new SmartScheduler();

// High priority tasks (microtasks)
scheduler.scheduleHigh(() => console.log('Critical update'));

// Low priority tasks (macrotasks)
scheduler.scheduleLow(() => console.log('Background task'));
```

## Interview Tips

**Key Concepts to Master:**
- **Event Loop Phases**: Call stack → Microtasks → Macrotasks → Render
- **Task Types**: Microtasks (Promises) vs Macrotasks (setTimeout)
- **Execution Order**: Synchronous → Microtasks → Macrotasks
- **Performance Impact**: Microtask starvation, rendering blocks

**Common Interview Questions:**
1. "Why do Promises execute before setTimeout with 0 delay?"
2. "What's the difference between microtasks and macrotasks?"
3. "How does the event loop work in JavaScript?"
4. "Can you give an example of microtask starvation?"
5. "How does this affect React state updates?"

**Pro Tips:**
- **Always consider rendering**: Microtasks can block UI updates
- **Use appropriate timing**: Microtasks for immediate work, macrotasks for deferred work
- **Debug with logging**: Track execution order to understand timing issues
- **Test event loop behavior**: Use queueMicrotask and setTimeout for testing
- **Consider frameworks**: React's concurrent features change event loop behavior

**Red Flags in Interviews:**
- Confusing microtasks with macrotasks
- Not understanding rendering implications
- Thinking setTimeout(fn, 0) executes immediately
- Not considering performance impact of microtask loops

**Advanced Interview Questions:**
- "How would you implement a custom setTimeout using Promises?"
- "What's the difference between process.nextTick and setImmediate in Node.js?"
- "How does React's concurrent mode affect the event loop?"
- "Can you implement requestAnimationFrame using setTimeout?"