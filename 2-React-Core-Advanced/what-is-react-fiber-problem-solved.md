# What is React Fiber? What problem did it solve?

## Question
What is React Fiber? What problem did it solve?

## Answer

React Fiber is a complete rewrite of React's reconciliation algorithm that was introduced in React 16. It fundamentally changed how React handles rendering, updates, and scheduling of work. Fiber solved critical performance and architectural limitations of the previous Stack reconciler, enabling features like concurrent rendering, suspense, and better error boundaries.

## What is React Fiber?

Fiber is React's new reconciliation engine that reimplements the core algorithm of how React renders and updates components. It's essentially a rearchitecting of React's internals to make it more powerful and flexible.

### Key Characteristics

1. **Incremental Rendering**: Can pause, resume, and restart rendering work
2. **Priority-based Scheduling**: Different types of updates have different priorities
3. **Concurrent Rendering**: Can work on multiple tasks simultaneously
4. **Better Error Handling**: More granular error boundaries
5. **Time-slicing**: Can split work across multiple frames

## Problems with the Old Stack Reconciler

### 1. **Synchronous and Blocking**

The old reconciler worked synchronously and couldn't be interrupted:

```javascript
// Old Stack Reconciler behavior
function render() {
    // This would block the main thread until complete
    reconcileEntireTree(); // Could take hundreds of milliseconds
    commitChanges();       // Apply all changes at once
}

// If this took 100ms, the UI would freeze for 100ms
// No way to prioritize urgent updates (like user input)
```

**Problems:**
- UI would freeze during large updates
- No way to prioritize urgent updates over non-urgent ones
- Poor user experience on low-end devices
- Couldn't handle complex animations smoothly

### 2. **No Work Prioritization**

All updates were treated equally:

```javascript
// All these updates would be processed in order, regardless of urgency
setState({ count: count + 1 });        // User interaction - should be immediate
setState({ data: fetchResult });       // Network response - can wait
setTimeout(() => setState({ time: new Date() }), 1000); // Background update
```

### 3. **Error Handling Limitations**

Errors in one component could crash the entire app:

```javascript
// Old behavior: One error crashes everything
function App() {
    return (
        <div>
            <Header />
            <MainContent />  {/* If this throws, entire app crashes */}
            <Footer />
        </div>
    );
}
```

### 4. **No Incremental Updates**

Had to process entire component trees at once:

```javascript
// Had to reconcile entire subtrees even for small changes
function updateComponent(component) {
    // Recalculate entire virtual DOM subtree
    const newTree = renderComponent(component);
    // Diff entire trees
    const patches = diff(oldTree, newTree);
    // Apply all patches at once
    applyPatches(patches);
}
```

## How Fiber Solves These Problems

### 1. **Incremental and Interruptible Rendering**

Fiber can pause and resume work:

```javascript
// Fiber can break work into chunks
function* reconcileWork(unitOfWork) {
    while (unitOfWork) {
        // Do a small chunk of work
        const nextUnit = performUnitOfWork(unitOfWork);

        // Check if we should yield control back to browser
        if (shouldYield()) {
            // Save current work and yield
            yield nextUnit;
        }

        unitOfWork = nextUnit;
    }
}

// Can be interrupted and resumed
const workLoop = () => {
    while (workInProgress && !shouldYield()) {
        workInProgress = performUnitOfWork(workInProgress);
    }

    if (workInProgress) {
        // Schedule continuation for next frame
        requestIdleCallback(workLoop);
    }
};
```

### 2. **Priority-based Scheduling**

Different updates get different priorities:

```javascript
// Priority levels in Fiber
const priorities = {
    Immediate: 1,     // User blocking updates (typing, clicking)
    UserBlocking: 2,  // User interactions (drag, hover)
    Normal: 3,        // Network responses
    Low: 4,          // Background updates
    Idle: 5          // Non-urgent work
};

// Example: Typing gets immediate priority
<input onChange={handleChange} /> // Immediate priority

// Network response gets normal priority
useEffect(() => {
    fetchData().then(setData);
}, []); // Normal priority

// Background timer gets low priority
setInterval(updateTime, 1000); // Low priority
```

### 3. **Concurrent Rendering**

Can work on multiple versions of the UI simultaneously:

```javascript
// Can start rendering v2 while v1 is still committing
const currentTree = fiberTree;        // Currently displayed
const workInProgressTree = clone(currentTree); // Being built

// If high-priority update comes in while building workInProgressTree:
// 1. Pause current work
// 2. Switch to high-priority work
// 3. Resume previous work later or discard it
```

### 4. **Time Slicing**

Splits work across multiple frames:

```javascript
// Instead of 100ms blocking render:
Frame 1: Render components 1-10 (16ms)
Frame 2: Render components 11-20 (16ms)
Frame 3: Render components 21-30 (16ms)
// ... and so on

// User interactions can interrupt this process
```

## Fiber Architecture

### 1. **Fiber Data Structure**

Each Fiber node represents a component or DOM element:

```javascript
interface Fiber {
    // Component type (div, MyComponent, etc.)
    type: any;

    // Props and state
    props: any;
    stateNode: any; // DOM node or component instance

    // Links to other fibers
    child: Fiber | null;     // First child
    sibling: Fiber | null;   // Next sibling
    parent: Fiber | null;    // Parent fiber

    // Work scheduling
    priority: number;
    expirationTime: number;

    // Update queue
    updateQueue: UpdateQueue | null;

    // Effect list for commit phase
    effectTag: number;       // What type of change (update, delete, etc.)
    nextEffect: Fiber | null;

    // Alternate fiber for double buffering
    alternate: Fiber | null;
}
```

### 2. **Double Buffering**

Fiber uses two trees for smooth transitions:

```javascript
// Current tree: What's displayed on screen
const current = fiberRoot.current;

// Work-in-progress tree: Being built
const workInProgress = current.alternate || createFiber(current);

// Swap them during commit
fiberRoot.current = workInProgress;
workInProgress.alternate = current;
current.alternate = workInProgress;
```

### 3. **Phases of Rendering**

**Phase 1: Render/ Reconciliation Phase**
- Can be interrupted and resumed
- Builds work-in-progress tree
- Calculates which components need updates

**Phase 2: Commit Phase**
- Cannot be interrupted
- Applies changes to DOM
- Runs effects (useEffect, etc.)

```javascript
function workLoop() {
    // Phase 1: Render (can be interrupted)
    while (workInProgress && !shouldYield()) {
        workInProgress = performUnitOfWork(workInProgress);
    }

    // Phase 2: Commit (cannot be interrupted)
    if (!workInProgress) {
        commitRoot();
    }
}
```

## Benefits of Fiber

### 1. **Better User Experience**

```javascript
// Before Fiber: UI freezes during large updates
function LargeList({ items }) {
    return items.map(item => <HeavyComponent key={item.id} item={item} />);
}

// With Fiber: Updates are incremental
// - Initial render shows skeleton
// - Components load progressively
// - User can interact while loading
```

### 2. **Concurrent Features**

Enables features like Suspense:

```javascript
// Suspense works because Fiber can pause rendering
function App() {
    return (
        <Suspense fallback={<Loading />}>
            <SlowComponent />
        </Suspense>
    );
}

// Fiber can:
// 1. Start rendering SlowComponent
// 2. Hit a suspend point
// 3. Show fallback
// 4. Resume when data is ready
```

### 3. **Improved Error Boundaries**

```javascript
// Before: One error crashes whole app
class ErrorBoundary extends Component {
    componentDidCatch(error) {
        // Could only catch errors in commit phase
    }
}

// With Fiber: Can catch errors during render phase
class ErrorBoundary extends Component {
    componentDidCatch(error) {
        // Can catch errors from entire render tree
        // Can recover gracefully
    }
}
```

### 4. **Better Performance on Mobile**

```javascript
// Fiber adapts to device capabilities
const shouldYield = () => {
    const currentTime = performance.now();
    const timeElapsed = currentTime - frameStartTime;

    // Yield if we've used most of the frame time
    return timeElapsed > 16; // ~60fps
};
```

## Real-World Impact

### 1. **Smooth Animations**

```javascript
// Before Fiber: Animations could stutter during updates
function AnimatedList() {
    const [items, setItems] = useState(initialItems);

    // Adding items during animation would cause jank
    const addItem = () => setItems([...items, newItem]);
}

// With Fiber: Animations stay smooth
// Rendering work is time-sliced, animations continue smoothly
```

### 2. **Better Responsiveness**

```javascript
// Typing feels instant even with complex forms
function ComplexForm() {
    const [values, setValues] = useState({});
    const [errors, setErrors] = useState({});

    // Complex validation runs without blocking input
    const handleChange = (field, value) => {
        setValues(prev => ({ ...prev, [field]: value }));

        // Validation can be low priority
        setTimeout(() => {
            const error = validateField(field, value);
            setErrors(prev => ({ ...prev, [field]: error }));
        }, 0);
    };
}
```

### 3. **Progressive Loading**

```javascript
// Components can render partially
function Dashboard() {
    return (
        <div>
            <Header />           {/* Renders immediately */}
            <Suspense fallback={<ChartSkeleton />}>
                <Chart />        {/* Loads asynchronously */}
            </Suspense>
            <Suspense fallback={<TableSkeleton />}>
                <DataTable />    {/* Loads asynchronously */}
            </Suspense>
        </div>
    );
}
```

## Fiber Internals Deep Dive

### 1. **Work Units**

Fiber breaks work into small units:

```javascript
function performUnitOfWork(fiber) {
    // 1. Update component (if function component)
    if (fiber.type instanceof Function) {
        updateFunctionComponent(fiber);
    } else {
        // 2. Reconcile children (if host component)
        reconcileChildren(fiber);
    }

    // 3. Return next unit of work
    if (fiber.child) {
        return fiber.child;
    }

    let nextFiber = fiber;
    while (nextFiber) {
        if (nextFiber.sibling) {
            return nextFiber.sibling;
        }
        nextFiber = nextFiber.parent;
    }
}
```

### 2. **Effect List**

Tracks what needs to be committed:

```javascript
// During render phase, fibers are tagged with effects
const effectTags = {
    Placement: 1,    // New component
    Update: 2,       // Component updated
    Deletion: 3,     // Component removed
    // ... more tags
};

// Commit phase processes effect list
function commitWork(fiber) {
    switch (fiber.effectTag) {
        case Placement:
            placeFiber(fiber);
            break;
        case Update:
            updateFiber(fiber);
            break;
        case Deletion:
            deleteFiber(fiber);
            break;
    }
}
```

### 3. **Scheduling**

Fiber uses `requestIdleCallback` when available:

```javascript
// Schedule work with appropriate priority
function scheduleWork(fiber, priority) {
    const expirationTime = calculateExpirationTime(priority);

    // Add to priority queue
    priorityQueue.push({
        fiber,
        expirationTime,
        callback: () => performWork(fiber)
    });

    // Schedule work loop
    requestIdleCallback(workLoop, { timeout: 100 });
}
```

## Migration and Compatibility

### 1. **Backward Compatibility**

Fiber maintains API compatibility:

```javascript
// All existing React code works with Fiber
class OldComponent extends Component {
    // Still works exactly the same
}

function NewComponent() {
    // Hooks work with Fiber's new capabilities
}
```

### 2. **Gradual Adoption**

Fiber features were rolled out incrementally:

```javascript
// React 16.0: Basic Fiber
// React 16.3: createContext, forwardRef
// React 16.6: Suspense, memo
// React 16.8: Hooks
// React 17.0: Concurrent features
// React 18.0: Concurrent rendering, automatic batching
```

## Performance Optimizations

### 1. **Automatic Batching**

Fiber enables automatic batching of state updates:

```javascript
// Before: Multiple renders
setCount(c => c + 1);
setName('John');
setAge(25);

// With Fiber: Batched into single render
// All updates in event handler are batched
```

### 2. **Lazy Loading**

Better code splitting support:

```javascript
// Fiber enables better lazy loading
const LazyComponent = lazy(() => import('./HeavyComponent'));

function App() {
    return (
        <Suspense fallback={<div>Loading...</div>}>
            <LazyComponent />
        </Suspense>
    );
}
```

### 3. **Memoization Improvements**

Better support for React.memo and useMemo:

```javascript
// Fiber makes memoization more effective
const MemoizedComponent = memo(function Component({ data }) {
    // Only re-renders when data actually changes
    return <div>{data.value}</div>;
});
```

## Debugging Fiber

### 1. **React DevTools**

```javascript
// React DevTools shows Fiber tree structure
// Can inspect priorities, work progress, etc.
```

### 2. **Performance Profiling**

```javascript
// Profile component renders
<React.Profiler id="MyComponent" onRender={callback}>
    <MyComponent />
</React.Profiler>
```

### 3. **Debugging Concurrent Features**

```javascript
// StrictMode helps catch concurrent issues
<React.StrictMode>
    <App />
</React.StrictMode>
```

## Future of Fiber

Fiber enables React's future features:

- **Concurrent Rendering**: Multiple versions of UI
- **Suspense for Data Fetching**: Better loading states
- **Server Components**: Components that run on server
- **Concurrent Features**: startTransition, useDeferredValue

```javascript
// Future features enabled by Fiber
function App() {
    const [query, setQuery] = useState('');
    const deferredQuery = useDeferredValue(query);

    return (
        <div>
            <input value={query} onChange={e => setQuery(e.target.value)} />
            <SearchResults query={deferredQuery} /> {/* Updates with lower priority */}
        </div>
    );
}
```

### Interview Tip:
*"React Fiber is a complete rewrite of React's reconciliation algorithm that enables incremental, interruptible rendering and priority-based scheduling. It solved the problem of synchronous, blocking updates that caused UI freezes, enabling smooth user experiences even during complex updates."*