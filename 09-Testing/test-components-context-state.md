# How do you test components that use context/state?

## Simple Answer
Wrap components with the necessary providers in tests. For Redux, use `<Provider store={store}>`. For Context, create a wrapper component. For custom hooks, test them through components that use them.

## Testing React Context

```javascript
// ThemeContext.js
import { createContext, useContext } from 'react';

const ThemeContext = createContext();

export const ThemeProvider = ({ children, theme = 'light' }) => (
  <ThemeContext.Provider value={{ theme }}>
    {children}
  </ThemeContext.Provider>
);

export const useTheme = () => useContext(ThemeContext);
```

```javascript
// ThemedButton.js
import { useTheme } from './ThemeContext';

const ThemedButton = ({ children }) => {
  const { theme } = useTheme();
  return <button className={`btn-${theme}`}>{children}</button>;
};

export default ThemedButton;
```

```javascript
// ThemedButton.test.js
import { render, screen } from '@testing-library/react';
import { ThemeProvider } from './ThemeContext';
import ThemedButton from './ThemedButton';

const renderWithTheme = (component, theme = 'light') =>
  render(<ThemeProvider theme={theme}>{component}</ThemeProvider>);

describe('ThemedButton', () => {
  it('should apply light theme by default', () => {
    renderWithTheme(<ThemedButton>Click me</ThemedButton>);
    expect(screen.getByRole('button')).toHaveClass('btn-light');
  });

  it('should apply dark theme', () => {
    renderWithTheme(<ThemedButton>Click me</ThemedButton>, 'dark');
    expect(screen.getByRole('button')).toHaveClass('btn-dark');
  });
});
```

## Testing Redux Components

```javascript
// Counter.js
import { useSelector, useDispatch } from 'react-redux';
import { increment } from './counterSlice';

const Counter = () => {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();

  return (
    <div>
      <span data-testid="count">{count}</span>
      <button onClick={() => dispatch(increment())}>+</button>
    </div>
  );
};

export default Counter;
```

```javascript
// Counter.test.js
import { render, screen, fireEvent } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';
import Counter from './Counter';

const renderWithRedux = (component, initialState = {}) => {
  const store = configureStore({
    reducer: { counter: counterReducer },
    preloadedState: initialState
  });

  return render(<Provider store={store}>{component}</Provider>);
};

describe('Counter', () => {
  it('should display initial count', () => {
    renderWithRedux(<Counter />, { counter: { value: 5 } });
    expect(screen.getByTestId('count')).toHaveTextContent('5');
  });

  it('should increment count', () => {
    renderWithRedux(<Counter />);

    fireEvent.click(screen.getByText('+'));
    expect(screen.getByTestId('count')).toHaveTextContent('1');
  });
});
```

## Testing Zustand Store

```javascript
// useCounterStore.js
import { create } from 'zustand';

export const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 }))
}));
```

```javascript
// Counter.js
import { useCounterStore } from './useCounterStore';

const Counter = () => {
  const { count, increment } = useCounterStore();

  return (
    <div>
      <span data-testid="count">{count}</span>
      <button onClick={increment}>+</button>
    </div>
  );
};

export default Counter;
```

```javascript
// Counter.test.js
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from './Counter';

// Mock the store
jest.mock('./useCounterStore');

describe('Counter', () => {
  const mockUseCounterStore = require('./useCounterStore').useCounterStore;

  it('should display count from store', () => {
    mockUseCounterStore.mockReturnValue({
      count: 3,
      increment: jest.fn()
    });

    render(<Counter />);
    expect(screen.getByTestId('count')).toHaveTextContent('3');
  });

  it('should call increment on button click', () => {
    const mockIncrement = jest.fn();
    mockUseCounterStore.mockReturnValue({
      count: 0,
      increment: mockIncrement
    });

    render(<Counter />);

    fireEvent.click(screen.getByText('+'));
    expect(mockIncrement).toHaveBeenCalledTimes(1);
  });
});
```

## Testing useState Components

```javascript
// Toggle.js
import { useState } from 'react';

const Toggle = () => {
  const [isOn, setIsOn] = useState(false);

  return (
    <button onClick={() => setIsOn(!isOn)}>
      {isOn ? 'ON' : 'OFF'}
    </button>
  );
};

export default Toggle;
```

```javascript
// Toggle.test.js
import { render, screen, fireEvent } from '@testing-library/react';
import Toggle from './Toggle';

describe('Toggle', () => {
  it('should start OFF', () => {
    render(<Toggle />);
    expect(screen.getByRole('button')).toHaveTextContent('OFF');
  });

  it('should toggle to ON when clicked', () => {
    render(<Toggle />);

    const button = screen.getByRole('button');
    fireEvent.click(button);
    expect(button).toHaveTextContent('ON');

    fireEvent.click(button);
    expect(button).toHaveTextContent('OFF');
  });
});
```

## Testing Custom Hooks

```javascript
// useCounter.js
import { useState } from 'react';

export const useCounter = (initialValue = 0) => {
  const [count, setCount] = useState(initialValue);

  const increment = () => setCount(count + 1);
  const decrement = () => setCount(count - 1);

  return { count, increment, decrement };
};
```

```javascript
// useCounter.test.js
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('should initialize with default value', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('should initialize with custom value', () => {
    const { result } = renderHook(() => useCounter(5));
    expect(result.current.count).toBe(5);
  });

  it('should increment count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('should decrement count', () => {
    const { result } = renderHook(() => useCounter(2));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(1);
  });
});
```

## Interview Q&A

**Q: How do you test a component that uses React Context?**
A: I wrap the component with the Context Provider in the test. I create a helper function like `renderWithContext` that includes the provider, then test the component behavior with different context values.

**Q: How do you test Redux-connected components?**
A: I use the Redux Provider and create a test store with `configureStore`. I can pass initial state to test different scenarios, and check that actions are dispatched correctly when interacting with the component.

**Q: What's the difference between testing a component vs testing a custom hook?**
A: For components, I render them and interact with the DOM. For custom hooks, I use `renderHook` from React Testing Library, which gives me access to the hook's return value and allows me to test the hook logic directly.

**Q: How do you test components with multiple contexts?**
A: I create a wrapper component that includes all the necessary providers, or use a hierarchical structure of providers in the test setup. This ensures all contexts are available when testing the component.

## Best Practices
- **Provider wrappers**: Create reusable render functions with providers
- **Mock stores**: For Zustand/Redux, mock the store hooks for isolation
- **Initial state**: Test components with different initial states
- **State changes**: Verify state updates happen correctly
- **Custom hooks**: Use `renderHook` for testing hooks directly
- **Async effects**: Use `waitFor` for effects that update state asynchronously
- **Error states**: Test error boundaries and error handling</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\08-Testing\test-components-context-state-simple.md