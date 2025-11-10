# How do you test a custom hook?

## Simple Answer
Create a test component that uses the hook, then test the component's behavior. Use `renderHook` from `@testing-library/react-hooks` for direct hook testing. Mock any external dependencies like APIs or timers.

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

## Method 1: Test Component Pattern

```javascript
// useCounter.test.js
import { render, screen, fireEvent } from '@testing-library/react';
import { useCounter } from './useCounter';

// Create a test component that uses the hook
const TestCounter = ({ initialValue }) => {
  const { count, increment, decrement } = useCounter(initialValue);

  return (
    <div>
      <span data-testid="count">{count}</span>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
};

describe('useCounter', () => {
  it('should initialize with default value', () => {
    render(<TestCounter />);
    expect(screen.getByTestId('count')).toHaveTextContent('0');
  });

  it('should initialize with custom value', () => {
    render(<TestCounter initialValue={5} />);
    expect(screen.getByTestId('count')).toHaveTextContent('5');
  });

  it('should increment count', () => {
    render(<TestCounter />);

    fireEvent.click(screen.getByText('+'));
    expect(screen.getByTestId('count')).toHaveTextContent('1');
  });

  it('should decrement count', () => {
    render(<TestCounter initialValue={3} />);

    fireEvent.click(screen.getByText('-'));
    expect(screen.getByTestId('count')).toHaveTextContent('2');
  });
});
```

## Method 2: renderHook (React Hooks Testing Library)

```javascript
// useCounter.test.js
import { renderHook, act } from '@testing-library/react-hooks';
import { useCounter } from './useCounter';

describe('useCounter with renderHook', () => {
  it('should initialize with default value', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('should increment count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('should decrement count', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(4);
  });
});
```

## Testing Hooks with API Calls

```javascript
// useUser.js
import { useState, useEffect } from 'react';

export const useUser = (userId) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]);

  return { user, loading };
};
```

```javascript
// useUser.test.js
import { render, screen, waitFor } from '@testing-library/react';
import { useUser } from './useUser';

// Mock fetch
global.fetch = jest.fn();

const TestUser = ({ userId }) => {
  const { user, loading } = useUser(userId);

  if (loading) return <div>Loading...</div>;
  return <div data-testid="user">{user?.name}</div>;
};

describe('useUser', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should fetch user data', async () => {
    const mockUser = { name: 'John' };
    fetch.mockResolvedValueOnce({
      json: () => Promise.resolve(mockUser)
    });

    render(<TestUser userId={1} />);

    await waitFor(() => {
      expect(screen.getByTestId('user')).toHaveTextContent('John');
    });

    expect(fetch).toHaveBeenCalledWith('/api/users/1');
  });
});
```

## Testing Hooks with Context

```javascript
// useTheme.js
import { useContext } from 'react';
import { ThemeContext } from './ThemeContext';

export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  return context;
};
```

```javascript
// useTheme.test.js
import { render, screen } from '@testing-library/react';
import { ThemeProvider } from './ThemeContext';

const TestTheme = () => {
  const { theme } = useTheme();
  return <div data-testid="theme">{theme}</div>;
};

describe('useTheme', () => {
  it('should throw error without provider', () => {
    expect(() => render(<TestTheme />)).toThrow();
  });

  it('should work with provider', () => {
    render(
      <ThemeProvider theme="dark">
        <TestTheme />
      </ThemeProvider>
    );

    expect(screen.getByTestId('theme')).toHaveTextContent('dark');
  });
});
```

## Testing Async Hooks

```javascript
// useDebounce.js
import { useState, useEffect } from 'react';

export const useDebounce = (value, delay) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
};
```

```javascript
// useDebounce.test.js
import { render, screen, act } from '@testing-library/react';
import { useDebounce } from './useDebounce';

const TestDebounce = ({ value, delay }) => {
  const debounced = useDebounce(value, delay);
  return <div data-testid="debounced">{debounced}</div>;
};

describe('useDebounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it('should debounce value changes', () => {
    const { rerender } = render(<TestDebounce value="hello" delay={500} />);

    expect(screen.getByTestId('debounced')).toHaveTextContent('hello');

    // Change value
    rerender(<TestDebounce value="world" delay={500} />);

    // Should still show old value
    expect(screen.getByTestId('debounced')).toHaveTextContent('hello');

    // Fast-forward time
    act(() => {
      jest.advanceTimersByTime(500);
    });

    expect(screen.getByTestId('debounced')).toHaveTextContent('world');
  });
});
```

## Interview Q&A

**Q: How do you test a custom React hook?**
A: I create a test component that uses the hook and test the component's behavior. Alternatively, I can use `renderHook` from the React Hooks Testing Library to test the hook directly. I mock any external dependencies like API calls or timers.

**Q: What's the difference between testing a hook vs testing a component?**
A: For components, I test the rendered output and user interactions. For hooks, I test the logic and state management. Since hooks can't be called outside components, I either wrap them in test components or use specialized testing utilities.

**Q: How do you test async behavior in hooks?**
A: For API calls, I mock fetch/axios and use `waitFor` to wait for async operations. For timers, I use Jest's fake timers with `jest.useFakeTimers()` and `jest.advanceTimersByTime()`. I wrap timer advances in `act()` to ensure state updates are processed.

**Q: Should you test implementation details of hooks?**
A: No, focus on the public API and behavior. Test what the hook returns and how it behaves, not internal implementation details like which useState calls it makes.

## Best Practices
- **Test component pattern**: Most common approach for hook testing
- **renderHook**: For direct hook testing without wrapper components
- **Mock dependencies**: API calls, timers, DOM APIs
- **Async testing**: Use `waitFor` and `act()` for async operations
- **Fake timers**: Use Jest fake timers for time-based hooks
- **Context providers**: Wrap test components with necessary providers
- **Behavior over implementation**: Test what the hook does, not how it does it</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\08-Testing\test-custom-hook-simple.md