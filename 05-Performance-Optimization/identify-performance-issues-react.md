# How do you identify performance issues in React?

## Question
How do you identify performance issues in React?

## Answer

Identifying performance issues in React applications requires a systematic approach using browser developer tools, React-specific profiling tools, and performance monitoring techniques. The key is to measure, analyze, and optimize the most impactful bottlenecks.

## Browser Developer Tools

### 1. **Chrome DevTools Performance Tab**

**Recording a performance profile:**
1. Open Chrome DevTools (F12)
2. Go to Performance tab
3. Click record button
4. Perform user interactions
5. Stop recording and analyze

**Key metrics to analyze:**
- **Main Thread** - JavaScript execution time
- **Rendering** - Paint and composite operations
- **Interactions** - Event handling responsiveness
- **Network** - Resource loading times

**Common performance issues:**
```javascript
// Long tasks (>50ms) blocking main thread
function expensiveOperation() {
    for (let i = 0; i < 1000000; i++) {
        // Heavy computation
        Math.sqrt(i) * Math.sin(i);
    }
}

// Forced synchronous layout
function badLayout() {
    const elements = document.querySelectorAll('.item');
    elements.forEach(el => {
        el.style.width = el.offsetWidth + 10 + 'px'; // Forces layout
    });
}
```

### 2. **React DevTools Profiler**

**Installation and setup:**
```bash
npm install -D react-devtools
```

**Using the Profiler:**
```typescript
import { Profiler } from 'react';

function App() {
    const onRender = (
        id: string,
        phase: 'mount' | 'update',
        actualDuration: number,
        baseDuration: number,
        startTime: number,
        commitTime: number
    ) => {
        console.log(`${id} ${phase} - ${actualDuration}ms`);
    };

    return (
        <Profiler id="App" onRender={onRender}>
            <MyComponent />
        </Profiler>
    );
}
```

**Key profiler metrics:**
- **Render duration** - How long component takes to render
- **Commit time** - Time to apply changes to DOM
- **Interactions** - Which user actions triggered renders
- **Flame graph** - Visual representation of component hierarchy

### 3. **Lighthouse Performance Audit**

**Running Lighthouse:**
1. Open Chrome DevTools
2. Go to Lighthouse tab
3. Select Performance category
4. Click Generate Report

**Core Web Vitals:**
- **Largest Contentful Paint (LCP)** - Loading performance
- **First Input Delay (FID)** - Interactivity
- **Cumulative Layout Shift (CLS)** - Visual stability

**Lighthouse scores:**
- **0-49:** Poor
- **50-89:** Needs improvement
- **90-100:** Good

## React-Specific Performance Issues

### 1. **Unnecessary Re-renders**

**Identifying with useEffect:**
```typescript
function useRenderCount(componentName: string) {
    const count = useRef(0);
    count.current += 1;

    useEffect(() => {
        console.log(`${componentName} rendered ${count.current} times`);
    });

    return count.current;
}

function MyComponent() {
    const renderCount = useRenderCount('MyComponent');
    // ... component logic
}
```

**Common causes:**
```typescript
// 1. New function references on every render
function ParentComponent() {
    const [count, setCount] = useState(0);

    // New function every render - causes child re-renders
    const handleClick = () => setCount(c => c + 1);

    return <Child onClick={handleClick} />;
}

// 2. Object/array creation in render
function ComponentWithData() {
    const [data, setData] = useState([]);

    // New array reference every render
    const processedData = data.map(item => ({ ...item, processed: true }));

    return <DataList data={processedData} />;
}

// 3. Context value changes
const MyContext = createContext();

function App() {
    const [user, setUser] = useState({ name: 'John' });
    const [theme, setTheme] = useState('light');

    // New object every render - all consumers re-render
    const contextValue = { user, theme, setUser, setTheme };

    return (
        <MyContext.Provider value={contextValue}>
            <Component1 /> {/* Re-renders */}
            <Component2 /> {/* Re-renders */}
        </MyContext.Provider>
    );
}
```

### 2. **Expensive Computations**

**Identifying slow computations:**
```typescript
function usePerformanceMonitor(label: string) {
    const startTime = useRef(0);

    const start = useCallback(() => {
        startTime.current = performance.now();
    }, []);

    const end = useCallback(() => {
        const duration = performance.now() - startTime.current;
        console.log(`${label} took ${duration.toFixed(2)}ms`);

        if (duration > 16.67) { // More than one frame
            console.warn(`${label} is blocking the main thread!`);
        }
    }, [label]);

    return { start, end };
}

function ExpensiveComponent({ data }) {
    const { start, end } = usePerformanceMonitor('ExpensiveComponent');

    useEffect(() => {
        start();
        // Expensive operation
        const result = heavyComputation(data);
        end();

        setResult(result);
    }, [data, start, end]);
}
```

**Common expensive operations:**
```typescript
// 1. Large array operations
const filteredData = useMemo(() => {
    console.time('filtering');
    const result = data.filter(item => expensiveCondition(item));
    console.timeEnd('filtering');
    return result;
}, [data]);

// 2. Complex calculations in render
function ChartComponent({ data }) {
    // This runs on every render!
    const chartData = data.reduce((acc, item) => {
        acc.labels.push(item.label);
        acc.values.push(expensiveCalculation(item.value));
        return acc;
    }, { labels: [], values: [] });

    return <Chart data={chartData} />;
}

// 3. DOM queries in render
function Component() {
    // This causes layout thrashing
    const width = useRef(document.getElementById('container')?.offsetWidth);

    return <div>Width: {width.current}</div>;
}
```

### 3. **Memory Leaks**

**Identifying memory leaks:**
```typescript
// 1. Timers not cleaned up
function TimerComponent() {
    const [count, setCount] = useState(0);

    useEffect(() => {
        const interval = setInterval(() => {
            setCount(c => c + 1);
        }, 1000);

        // Missing cleanup - memory leak!
        // return () => clearInterval(interval);
    }, []);

    return <div>Count: {count}</div>;
}

// 2. Event listeners not removed
function EventComponent() {
    useEffect(() => {
        const handler = () => console.log('resize');

        window.addEventListener('resize', handler);
        // Missing cleanup - memory leak!
        // return () => window.removeEventListener('resize', handler);
    }, []);

    return <div>Component</div>;
}

// 3. Subscriptions not unsubscribed
function DataComponent() {
    useEffect(() => {
        const subscription = dataService.subscribe(handleData);
        // Missing cleanup - memory leak!
        // return () => subscription.unsubscribe();
    }, []);
}
```

**Memory leak detection:**
```typescript
// Use Chrome DevTools Memory tab
// 1. Take heap snapshot
// 2. Perform actions
// 3. Take another snapshot
// 4. Compare and look for growing object counts

// Or use performance.memory API
function logMemoryUsage() {
    if (performance.memory) {
        console.log('Memory usage:', {
            used: Math.round(performance.memory.usedJSHeapSize / 1048576) + 'MB',
            total: Math.round(performance.memory.totalJSHeapSize / 1048576) + 'MB',
            limit: Math.round(performance.memory.jsHeapSizeLimit / 1048576) + 'MB'
        });
    }
}
```

## Performance Monitoring Tools

### 1. **React Performance DevTool**

```typescript
// Custom hook for performance monitoring
function usePerformanceTracker(componentName: string) {
    const renderCount = useRef(0);
    const lastRenderTime = useRef(0);
    const totalRenderTime = useRef(0);

    renderCount.current += 1;

    useLayoutEffect(() => {
        const renderTime = performance.now() - lastRenderTime.current;
        totalRenderTime.current += renderTime;

        const averageRenderTime = totalRenderTime.current / renderCount.current;

        if (renderTime > 16.67) { // Longer than one frame
            console.warn(`${componentName} slow render: ${renderTime.toFixed(2)}ms`);
        }

        if (renderCount.current % 10 === 0) { // Log every 10 renders
            console.log(`${componentName} stats:`, {
                renders: renderCount.current,
                averageTime: averageRenderTime.toFixed(2) + 'ms',
                lastRender: renderTime.toFixed(2) + 'ms'
            });
        }

        lastRenderTime.current = performance.now();
    });

    return {
        renderCount: renderCount.current,
        averageRenderTime: totalRenderTime.current / renderCount.current
    };
}
```

### 2. **Bundle Size Analysis**

```javascript
// Use webpack-bundle-analyzer
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
    plugins: [
        new BundleAnalyzerPlugin({
            analyzerMode: 'static',
            reportFilename: 'bundle-report.html'
        })
    ]
};

// Or use source-map-explorer
// npx source-map-explorer build/static/js/*.js
```

### 3. **Runtime Performance Monitoring**

```typescript
// Monitor long tasks
const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
        if (entry.duration > 50) { // Long task
            console.warn('Long task detected:', {
                duration: entry.duration,
                startTime: entry.startTime,
                name: entry.name
            });
        }
    }
});

observer.observe({ entryTypes: ['longtask'] });

// Monitor layout shifts
const layoutObserver = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
        if (entry.value > 0.1) { // Significant layout shift
            console.warn('Layout shift detected:', {
                value: entry.value,
                sources: entry.sources
            });
        }
    }
});

layoutObserver.observe({ entryTypes: ['layout-shift'] });
```

## Automated Performance Testing

### 1. **Performance Regression Tests**

```javascript
// performance.test.js
describe('Performance Tests', () => {
    it('should render component within time limit', async () => {
        const startTime = performance.now();

        const { container } = render(<ExpensiveComponent data={largeDataSet} />);

        // Wait for render to complete
        await waitFor(() => {
            expect(container.querySelector('.result')).toBeInTheDocument();
        });

        const renderTime = performance.now() - startTime;
        expect(renderTime).toBeLessThan(100); // Should render in < 100ms
    });

    it('should not cause excessive re-renders', () => {
        const renderSpy = jest.fn();
        const TestComponent = () => {
            renderSpy();
            return <div>Test</div>;
        };

        const { rerender } = render(<TestComponent />);

        // Trigger multiple updates
        rerender(<TestComponent />);
        rerender(<TestComponent />);

        // Should only render 3 times (initial + 2 rerenders)
        expect(renderSpy).toHaveBeenCalledTimes(3);
    });
});
```

### 2. **Bundle Size Tests**

```javascript
// bundle-size.test.js
const fs = require('fs');
const path = require('path');

describe('Bundle Size', () => {
    it('should not exceed bundle size limit', () => {
        const stats = JSON.parse(
            fs.readFileSync('./build/static/js/webpack-stats.json')
        );

        const mainBundle = stats.assets.find(asset =>
            asset.name.includes('main') && asset.name.endsWith('.js')
        );

        const sizeInMB = mainBundle.size / (1024 * 1024);
        expect(sizeInMB).toBeLessThan(2); // Bundle should be < 2MB
    });

    it('should not have large unused dependencies', () => {
        // Check for known large libraries
        const packageJson = JSON.parse(
            fs.readFileSync('./package.json')
        );

        const largeDeps = ['lodash', 'moment', 'jquery'];
        const hasLargeDeps = largeDeps.some(dep =>
            packageJson.dependencies[dep] || packageJson.devDependencies[dep]
        );

        if (hasLargeDeps) {
            console.warn('Large dependencies detected. Consider tree-shaking or alternatives.');
        }
    });
});
```

## Common Performance Anti-Patterns

### 1. **Inline Function Definitions**

```typescript
// Bad - New function every render
function List({ items }) {
    return (
        <ul>
            {items.map(item => (
                <li key={item.id} onClick={() => handleClick(item.id)}>
                    {item.name}
                </li>
            ))}
        </ul>
    );
}

// Good - Stable function reference
function List({ items }) {
    const handleItemClick = useCallback((id) => {
        handleClick(id);
    }, []);

    return (
        <ul>
            {items.map(item => (
                <li key={item.id} onClick={() => handleItemClick(item.id)}>
                    {item.name}
                </li>
            ))}
        </ul>
    );
}
```

### 2. **Missing Dependencies in useEffect**

```typescript
// Bad - Missing dependency
function DataFetcher({ userId }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        fetchData(userId).then(setData);
    }, []); // Should include userId

    return <div>{data?.name}</div>;
}

// Good - Proper dependencies
function DataFetcher({ userId }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        fetchData(userId).then(setData);
    }, [userId]); // Include userId

    return <div>{data?.name}</div>;
}
```

### 3. **Unnecessary useState**

```typescript
// Bad - State for computed value
function Counter() {
    const [count, setCount] = useState(0);
    const [doubled, setDoubled] = useState(0); // Unnecessary

    const increment = () => {
        setCount(c => c + 1);
        setDoubled(d => d + 2); // Manual sync
    };

    return <div>Count: {count}, Doubled: {doubled}</div>;
}

// Good - Computed value
function Counter() {
    const [count, setCount] = useState(0);
    const doubled = count * 2; // Computed

    const increment = () => setCount(c => c + 1);

    return <div>Count: {count}, Doubled: {doubled}</div>;
}
```

## Performance Optimization Checklist

### 1. **Initial Assessment**

- [ ] Run Lighthouse performance audit
- [ ] Check bundle size with analyzer
- [ ] Profile with React DevTools
- [ ] Monitor Core Web Vitals
- [ ] Check for memory leaks

### 2. **Component-Level Optimizations**

- [ ] Use React.memo for expensive components
- [ ] Implement useMemo for expensive calculations
- [ ] Use useCallback for event handlers
- [ ] Avoid inline object/array creation
- [ ] Split large components

### 3. **Application-Level Optimizations**

- [ ] Implement code splitting
- [ ] Use lazy loading for routes
- [ ] Optimize images and assets
- [ ] Implement caching strategies
- [ ] Use service workers

### 4. **Monitoring and Maintenance**

- [ ] Set up performance budgets
- [ ] Implement error boundaries
- [ ] Monitor performance in production
- [ ] Regular performance audits
- [ ] Update dependencies

## Common Interview Questions

### Q: How do you identify performance issues in React?

**A:** I start with Chrome DevTools Performance tab to record interactions and identify long tasks. Then I use React DevTools Profiler to analyze component render times and unnecessary re-renders. I also run Lighthouse audits to check Core Web Vitals and use bundle analyzers to identify large dependencies.

### Q: What's the most common performance issue you've encountered?

**A:** The most common issue is unnecessary re-renders caused by new function references, object creation in render, or missing memoization. I identify these using the React DevTools Profiler and fix them with useCallback, useMemo, and React.memo.

### Q: How do you measure React component performance?

**A:** I use the React DevTools Profiler to measure render duration and identify slow components. I also create custom hooks that track render counts and timing, and use performance.now() to measure specific operations. For production, I monitor with tools like LogRocket or Sentry.

### Q: What tools do you use for performance monitoring?

**A:** Chrome DevTools for profiling, React DevTools for component analysis, Lighthouse for Core Web Vitals, webpack-bundle-analyzer for bundle size, and Performance Observer API for runtime monitoring. In production, I use RUM (Real User Monitoring) tools.

### Q: How do you prevent performance regressions?

**A:** I set up performance budgets in CI/CD, write performance tests that fail if components render too slowly, use bundle size limits, and implement regular performance audits. I also monitor key metrics in production and set up alerts for performance degradation.

## Summary

**Key Tools for Identification:**
1. **Chrome DevTools Performance** - Main thread analysis, long tasks
2. **React DevTools Profiler** - Component render analysis
3. **Lighthouse** - Core Web Vitals, overall performance score
4. **Bundle analyzers** - Identify large dependencies
5. **Performance Observer API** - Runtime monitoring

**Most Common Issues:**
- **Unnecessary re-renders** from function/object creation
- **Expensive computations** in render or effects
- **Memory leaks** from uncleaned subscriptions
- **Large bundle sizes** from unused dependencies
- **Layout thrashing** from DOM queries in render

**Prevention Strategies:**
- **Performance budgets** in build process
- **Regular profiling** during development
- **Performance tests** in CI/CD
- **Monitoring** in production
- **Code reviews** focusing on performance

**Interview Tip:** "I identify React performance issues by using Chrome DevTools Performance tab to record interactions and spot long tasks, React DevTools Profiler to analyze component render times, and Lighthouse for Core Web Vitals. The most common issues are unnecessary re-renders, which I fix with memoization techniques."