# How to Find Performance Issues in React

## Question
How do you identify performance issues in React?

## Answer
Finding performance problems in React apps is like being a detective. You look for clues that show where your app is slow or wasting resources. Here's how to spot the most common issues.

## Start with Browser Tools

### Chrome DevTools - Your First Stop

**How to check performance:**
1. Open your app in Chrome
2. Press F12 to open DevTools
3. Click the "Performance" tab
4. Click the record button (🔴)
5. Use your app normally for a few seconds
6. Click stop and look at the results

**What to look for:**
- **Red blocks** = Long tasks that freeze your app
- **Blue bars** = Time spent rendering/painting
- **Network tab** = Slow loading resources

**Common red flags:**
```javascript
// This code blocks the UI for too long
function slowFunction() {
    for (let i = 0; i < 1000000; i++) {
        // Heavy math that blocks scrolling/clicking
        Math.sqrt(i) * Math.sin(i);
    }
}

// This causes layout shifts (jumpy content)
function badResize() {
    const boxes = document.querySelectorAll('.box');
    boxes.forEach(box => {
        box.style.width = box.offsetWidth + 10 + 'px'; // Forces browser to recalculate
    });
}
```

### React DevTools Profiler

**Install it:**
```bash
npm install -D react-devtools
```

**How to use:**
```javascript
import { Profiler } from 'react';

function App() {
    const onRender = (id, phase, actualTime, baseTime) => {
        console.log(`${id} took ${actualTime}ms to render`);
    };

    return (
        <Profiler id="App" onRender={onRender}>
            <MyApp />
        </Profiler>
    );
}
```

**What it tells you:**
- Which components render slowly
- How many times components re-render
- What triggered each re-render

## Spot the Most Common Issues

### 1. Components Re-rendering Too Much

**Check if components re-render unnecessarily:**
```javascript
function useRenderCounter(name) {
    const count = useRef(0);
    count.current += 1;

    useEffect(() => {
        console.log(`${name} rendered ${count.current} times`);
    });

    return count.current;
}

function MyComponent() {
    const renders = useRenderCounter('MyComponent');
    return <div>Renders: {renders}</div>;
}
```

**Common causes:**

```javascript
// Problem: New function every render
function Parent() {
    const [count, setCount] = useState(0);

    // This creates a new function each time!
    const handleClick = () => setCount(c => c + 1);

    return <Child onClick={handleClick} />; // Child re-renders every time
}

// Problem: New object every render
function DataComponent() {
    const [items, setItems] = useState([]);

    // This creates a new array reference every render
    const processedItems = items.map(item => ({ ...item, extra: true }));

    return <List items={processedItems} />; // List re-renders every time
}
```

### 2. Slow Computations

**Time how long things take:**
```javascript
function useTimer(label) {
    const startTime = useRef(0);

    const start = () => startTime.current = performance.now();
    const end = () => {
        const time = performance.now() - startTime.current;
        console.log(`${label} took ${time.toFixed(2)}ms`);

        if (time > 16) { // Slower than 60fps
            console.warn('This is blocking the UI!');
        }
    };

    return { start, end };
}

function SlowComponent({ data }) {
    const timer = useTimer('SlowComponent');

    useEffect(() => {
        timer.start();
        const result = expensiveCalculation(data);
        timer.end();

        setResult(result);
    }, [data]);

    return <div>Result: {result}</div>;
}
```

**Common slow operations:**
```javascript
// Filtering large arrays on every render
function SearchList({ items, searchTerm }) {
    // This runs every time anything changes!
    const filtered = items.filter(item =>
        item.name.toLowerCase().includes(searchTerm.toLowerCase())
    );

    return <List items={filtered} />;
}

// Complex calculations in render
function Chart({ data }) {
    // This recalculates every render!
    const averages = data.reduce((acc, item) => {
        acc.sum += item.value;
        acc.count += 1;
        return acc;
    }, { sum: 0, count: 0 });

    const average = averages.sum / averages.count;

    return <div>Average: {average}</div>;
}
```

### 3. Memory Leaks

**Signs of memory leaks:**
- App gets slower over time
- Chrome memory usage keeps growing
- Components don't clean up properly

**Common causes:**
```javascript
// Timer not stopped
function TimerComponent() {
    const [count, setCount] = useState(0);

    useEffect(() => {
        const interval = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);

        // Forgot to clean up! Memory leak!
        // return () => clearInterval(interval);
    }, []);

    return <div>Count: {count}</div>;
}

// Event listener not removed
function ResizeComponent() {
    const [width, setWidth] = useState(window.innerWidth);

    useEffect(() => {
        const handleResize = () => setWidth(window.innerWidth);
        window.addEventListener('resize', handleResize);

        // Forgot to clean up! Memory leak!
        // return () => window.removeEventListener('resize', handleResize);
    }, []);

    return <div>Width: {width}</div>;
}
```

**Check for memory leaks:**
```javascript
// In Chrome DevTools:
1. Go to Memory tab
2. Click "Take snapshot"
3. Use your app for a while
4. Click "Take snapshot" again
5. Compare snapshots - look for growing object counts
```

## Quick Performance Checks

### Bundle Size - Is Your App Too Big?

**Check bundle size:**
```bash
# Install analyzer
npm install --save-dev webpack-bundle-analyzer

# Add to webpack config
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
    plugins: [new BundleAnalyzerPlugin()]
};
```

**Run it:**
```bash
npm run build
# Opens browser showing what's taking up space
```

**Red flags:**
- Bundle over 2MB for a simple app
- Large libraries you don't use much
- Multiple versions of same library

### Lighthouse Score

**How to run:**
1. Open Chrome DevTools
2. Go to "Lighthouse" tab
3. Check "Performance"
4. Click "Generate report"

**What the scores mean:**
- **90-100:** Good
- **50-89:** Needs work
- **0-49:** Poor

**Focus on:**
- **LCP (Largest Contentful Paint):** How fast main content loads
- **FID (First Input Delay):** How responsive clicks/taps are
- **CLS (Cumulative Layout Shift):** Does content jump around?

## Automated Checks

### Performance Tests

```javascript
// Check if components render fast enough
describe('Performance', () => {
    it('renders quickly', () => {
        const start = performance.now();

        render(<SlowComponent data={bigData} />);

        const time = performance.now() - start;
        expect(time).toBeLessThan(100); // Should render in < 100ms
    });

    it('doesnt re-render too much', () => {
        let renderCount = 0;
        const TestComponent = () => {
            renderCount++;
            return <div>test</div>;
        };

        const { rerender } = render(<TestComponent />);

        rerender(<TestComponent />);
        rerender(<TestComponent />);

        expect(renderCount).toBe(3); // Initial + 2 rerenders
    });
});
```

### Bundle Size Tests

```javascript
// Make sure bundle doesn't get too big
describe('Bundle Size', () => {
    it('stays under limit', () => {
        const stats = JSON.parse(fs.readFileSync('./build/stats.json'));

        const mainBundle = stats.assets.find(a => a.name.includes('main'));
        const sizeMB = mainBundle.size / (1024 * 1024);

        expect(sizeMB).toBeLessThan(2); // Keep under 2MB
    });
});
```

## Common Mistakes to Watch For

### 1. Creating Functions in Render

```javascript
// Bad - New function every render
function List({ items }) {
    return (
        <div>
            {items.map(item => (
                <li onClick={() => handleClick(item.id)}> {/* New function each time! */}
                    {item.name}
                </li>
            ))}
        </div>
    );
}

// Good - Stable function
function List({ items }) {
    const onItemClick = useCallback((id) => {
        handleClick(id);
    }, []);

    return (
        <div>
            {items.map(item => (
                <li onClick={() => onItemClick(item.id)}> {/* Same function */}
                    {item.name}
                </li>
            ))}
        </div>
    );
}
```

### 2. Missing useEffect Dependencies

```javascript
// Bad - Missing dependency
function DataFetcher({ userId }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        fetchData(userId).then(setData);
    }, []); // Forgot userId! Won't refetch when userId changes

    return <div>{data?.name}</div>;
}

// Good - Include all dependencies
function DataFetcher({ userId }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        fetchData(userId).then(setData);
    }, [userId]); // Now it refetches when userId changes

    return <div>{data?.name}</div>;
}
```

### 3. Unnecessary State

```javascript
// Bad - State for something you can calculate
function Counter() {
    const [count, setCount] = useState(0);
    const [doubled, setDoubled] = useState(0); // Don't need this!

    const increment = () => {
        setCount(c => c + 1);
        setDoubled(d => d + 2); // Have to keep in sync manually
    };

    return <div>Count: {count}, Doubled: {doubled}</div>;
}

// Good - Just calculate it
function Counter() {
    const [count, setCount] = useState(0);
    const doubled = count * 2; // Calculate when needed

    const increment = () => setCount(c => c + 1);

    return <div>Count: {count}, Doubled: {doubled}</div>;
}
```

## Performance Checklist

**Quick daily checks:**
- [ ] Run React DevTools profiler on new features
- [ ] Check Chrome Performance tab for long tasks
- [ ] Look at bundle size after adding dependencies

**Weekly reviews:**
- [ ] Run Lighthouse audit
- [ ] Check for memory leaks
- [ ] Review component re-render causes

**Monthly deep dives:**
- [ ] Full performance profiling
- [ ] Bundle analysis
- [ ] User experience testing

## Interview Answers

**"How do you find performance issues in React?"**
"I start with Chrome DevTools Performance tab to record user interactions and identify long tasks blocking the main thread. Then I use React DevTools Profiler to analyze component render times and unnecessary re-renders. I also run Lighthouse audits to check Core Web Vitals and bundle analyzers to spot large dependencies."

**"What's the most common performance issue you've seen?"**
"The most common is unnecessary re-renders caused by creating new functions or objects in the render function. I identify these with the React DevTools Profiler and fix them with useCallback, useMemo, and React.memo."

**"How do you measure component performance?"**
"I use the React DevTools Profiler for render duration, create custom hooks to track render counts and timing, and use performance.now() for specific operations. In production, I monitor with real user monitoring tools."

## Summary

**Top ways to find issues:**
1. **Chrome DevTools Performance** - Record and analyze interactions
2. **React DevTools Profiler** - Component render analysis
3. **Lighthouse** - Overall performance scores
4. **Bundle analyzer** - Find large dependencies
5. **Memory snapshots** - Detect leaks

**Most common problems:**
- Components re-rendering too much
- Slow calculations in render/effects
- Memory leaks from uncleaned subscriptions
- Large bundle sizes
- Layout-causing DOM queries

**Prevention:**
- Profile regularly during development
- Set performance budgets
- Write performance tests
- Monitor in production
- Review code for performance anti-patterns

**Prevention:**
- Profile regularly during development
- Set performance budgets
- Write performance tests
- Monitor in production
- Review code for performance anti-patterns