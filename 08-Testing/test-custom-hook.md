# How do you test a custom hook?

## Question
How do you test a custom hook?

# How do you test a custom hook?

## Question
How do you test a custom hook?

## Answer

Testing custom hooks in React requires a different approach than testing components because hooks can't be called directly outside of React components. The standard approach is to create a test component that uses the hook and then test the component's behavior.

## Testing Custom Hooks

### Basic Hook Testing Pattern
```javascript
// useCounter.js
import { useState, useCallback } from 'react';

export const useCounter = (initialValue = 0) => {
  const [count, setCount] = useState(initialValue);

  const increment = useCallback(() => {
    setCount(prev => prev + 1);
  }, []);

  const decrement = useCallback(() => {
    setCount(prev => prev - 1);
  }, []);

  const reset = useCallback(() => {
    setCount(initialValue);
  }, [initialValue]);

  return { count, increment, decrement, reset };
};
```

```javascript
// useCounter.test.js
import { render, screen, fireEvent } from '@testing-library/react';
import { useCounter } from './useCounter';

// Test component that uses the hook
const TestComponent = ({ initialValue }) => {
  const { count, increment, decrement, reset } = useCounter(initialValue);

  return (
    <div>
      <div data-testid="count">{count}</div>
      <button onClick={increment}>Increment</button>
      <button onClick={decrement}>Decrement</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
};

describe('useCounter', () => {
  it('should initialize with default value', () => {
    render(<TestComponent />);
    expect(screen.getByTestId('count')).toHaveTextContent('0');
  });

  it('should initialize with custom value', () => {
    render(<TestComponent initialValue={5} />);
    expect(screen.getByTestId('count')).toHaveTextContent('5');
  });

  it('should increment count', () => {
    render(<TestComponent />);
    const incrementButton = screen.getByText('Increment');

    fireEvent.click(incrementButton);
    expect(screen.getByTestId('count')).toHaveTextContent('1');

    fireEvent.click(incrementButton);
    expect(screen.getByTestId('count')).toHaveTextContent('2');
  });

  it('should decrement count', () => {
    render(<TestComponent initialValue={5} />);
    const decrementButton = screen.getByText('Decrement');

    fireEvent.click(decrementButton);
    expect(screen.getByTestId('count')).toHaveTextContent('4');
  });

  it('should reset count', () => {
    render(<TestComponent initialValue={10} />);
    const incrementButton = screen.getByText('Increment');
    const resetButton = screen.getByText('Reset');

    fireEvent.click(incrementButton);
    fireEvent.click(incrementButton);
    expect(screen.getByTestId('count')).toHaveTextContent('12');

    fireEvent.click(resetButton);
    expect(screen.getByTestId('count')).toHaveTextContent('10');
  });
});
```

## Advanced Hook Testing

### Hook with API Calls
```javascript
// useUserData.js
import { useState, useEffect } from 'react';

export const useUserData = (userId) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      try {
        setLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) throw new Error('Failed to fetch user');
        const userData = await response.json();
        setUser(userData);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    if (userId) {
      fetchUser();
    }
  }, [userId]);

  return { user, loading, error };
};
```

```javascript
// useUserData.test.js
import { render, screen, waitFor } from '@testing-library/react';
import { useUserData } from './useUserData';

// Mock fetch
global.fetch = jest.fn();

const TestComponent = ({ userId }) => {
  const { user, loading, error } = useUserData(userId);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>No user</div>;

  return (
    <div>
      <div data-testid="user-name">{user.name}</div>
      <div data-testid="user-email">{user.email}</div>
    </div>
  );
};

describe('useUserData', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should show loading initially', () => {
    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ name: 'John', email: 'john@test.com' })
    });

    render(<TestComponent userId="123" />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  it('should fetch and display user data', async () => {
    const mockUser = { name: 'John Doe', email: 'john@test.com' };
    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockUser)
    });

    render(<TestComponent userId="123" />);

    await waitFor(() => {
      expect(screen.getByTestId('user-name')).toHaveTextContent('John Doe');
      expect(screen.getByTestId('user-email')).toHaveTextContent('john@test.com');
    });

    expect(fetch).toHaveBeenCalledWith('/api/users/123');
  });

  it('should handle API errors', async () => {
    fetch.mockResolvedValueOnce({
      ok: false
    });

    render(<TestComponent userId="123" />);

    await waitFor(() => {
      expect(screen.getByText('Error: Failed to fetch user')).toBeInTheDocument();
    });
  });

  it('should not fetch if no userId', () => {
    render(<TestComponent />);
    expect(screen.getByText('No user')).toBeInTheDocument();
    expect(fetch).not.toHaveBeenCalled();
  });
});
```

## Testing Hooks with Context

### Hook Using Context
```javascript
// useAuth.js
import { useContext } from 'react';
import { AuthContext } from './AuthContext';

export const useAuth = () => {
  const context = useContext(AuthContext);

  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }

  return context;
};
```

```javascript
// useAuth.test.js
import { render, screen } from '@testing-library/react';
import { useAuth } from './useAuth';
import { AuthProvider } from './AuthContext';

const TestComponent = () => {
  const { user, login, logout } = useAuth();

  return (
    <div>
      <div data-testid="user-status">
        {user ? `Logged in as ${user.name}` : 'Not logged in'}
      </div>
      <button onClick={() => login({ name: 'John' })}>Login</button>
      <button onClick={logout}>Logout</button>
    </div>
  );
};

describe('useAuth', () => {
  it('should throw error when used outside provider', () => {
    // Mock console.error to avoid noise
    const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});

    expect(() => render(<TestComponent />)).toThrow(
      'useAuth must be used within AuthProvider'
    );

    consoleSpy.mockRestore();
  });

  it('should work within AuthProvider', () => {
    render(
      <AuthProvider>
        <TestComponent />
      </AuthProvider>
    );

    expect(screen.getByTestId('user-status')).toHaveTextContent('Not logged in');
  });

  it('should handle login/logout', () => {
    render(
      <AuthProvider>
        <TestComponent />
      </AuthProvider>
    );

    const loginButton = screen.getByText('Login');
    const logoutButton = screen.getByText('Logout');

    // Login
    fireEvent.click(loginButton);
    expect(screen.getByTestId('user-status')).toHaveTextContent('Logged in as John');

    // Logout
    fireEvent.click(logoutButton);
    expect(screen.getByTestId('user-status')).toHaveTextContent('Not logged in');
  });
});
```

## Testing Async Hooks

### Hook with Debouncing
```javascript
// useDebounce.js
import { useState, useEffect } from 'react';

export const useDebounce = (value, delay) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [value, delay]);

  return debouncedValue;
};
```

```javascript
// useDebounce.test.js
import { render, screen, act } from '@testing-library/react';
import { useDebounce } from './useDebounce';

const TestComponent = ({ value, delay }) => {
  const debouncedValue = useDebounce(value, delay);
  return <div data-testid="debounced">{debouncedValue}</div>;
};

describe('useDebounce', () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.runOnlyPendingTimers();
    jest.useRealTimers();
  });

  it('should return initial value immediately', () => {
    render(<TestComponent value="hello" delay={500} />);
    expect(screen.getByTestId('debounced')).toHaveTextContent('hello');
  });

  it('should debounce value changes', () => {
    const { rerender } = render(<TestComponent value="hello" delay={500} />);

    // Change value
    rerender(<TestComponent value="world" delay={500} />);

    // Should still show old value immediately
    expect(screen.getByTestId('debounced')).toHaveTextContent('hello');

    // Fast-forward time
    act(() => {
      jest.advanceTimersByTime(500);
    });

    // Now should show new value
    expect(screen.getByTestId('debounced')).toHaveTextContent('world');
  });

  it('should cancel previous timeout on value change', () => {
    const { rerender } = render(<TestComponent value="first" delay={500} />);

    // Change value before timeout
    rerender(<TestComponent value="second" delay={500} />);

    act(() => {
      jest.advanceTimersByTime(300); // Not enough time for first timeout
    });

    // Should still show first value
    expect(screen.getByTestId('debounced')).toHaveTextContent('first');

    // Wait for second timeout
    act(() => {
      jest.advanceTimersByTime(200);
    });

    expect(screen.getByTestId('debounced')).toHaveTextContent('second');
  });
});
```

## Testing Hook Callbacks

### Hook with Event Listeners
```javascript
// useWindowSize.js
import { useState, useEffect } from 'react';

export const useWindowSize = () => {
  const [size, setSize] = useState({
    width: typeof window !== 'undefined' ? window.innerWidth : 0,
    height: typeof window !== 'undefined' ? window.innerHeight : 0
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight
      });
    };

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
};
```

```javascript
// useWindowSize.test.js
import { render, screen, fireEvent, act } from '@testing-library/react';
import { useWindowSize } from './useWindowSize';

const TestComponent = () => {
  const { width, height } = useWindowSize();
  return (
    <div>
      <div data-testid="width">{width}</div>
      <div data-testid="height">{height}</div>
    </div>
  );
};

describe('useWindowSize', () => {
  beforeEach(() => {
    // Mock window dimensions
    Object.defineProperty(window, 'innerWidth', {
      writable: true,
      value: 1024
    });
    Object.defineProperty(window, 'innerHeight', {
      writable: true,
      value: 768
    });
  });

  it('should return initial window size', () => {
    render(<TestComponent />);
    expect(screen.getByTestId('width')).toHaveTextContent('1024');
    expect(screen.getByTestId('height')).toHaveTextContent('768');
  });

  it('should update on window resize', () => {
    render(<TestComponent />);

    // Simulate window resize
    act(() => {
      window.innerWidth = 800;
      window.innerHeight = 600;
      fireEvent(window, new Event('resize'));
    });

    expect(screen.getByTestId('width')).toHaveTextContent('800');
    expect(screen.getByTestId('height')).toHaveTextContent('600');
  });

  it('should clean up event listener', () => {
    const addEventListenerSpy = jest.spyOn(window, 'addEventListener');
    const removeEventListenerSpy = jest.spyOn(window, 'removeEventListener');

    const { unmount } = render(<TestComponent />);

    expect(addEventListenerSpy).toHaveBeenCalledWith('resize', expect.any(Function));

    unmount();

    expect(removeEventListenerSpy).toHaveBeenCalledWith('resize', expect.any(Function));
  });
});
```

## Best Practices

1. **Test component pattern**: Always create a test component that uses the hook
2. **Mock external dependencies**: API calls, timers, DOM APIs
3. **Test all states**: Loading, success, error states
4. **Test side effects**: Event listeners, timers, API calls
5. **Cleanup**: Test that cleanup functions work properly
6. **Async operations**: Use `waitFor` and proper async testing
7. **Error boundaries**: Test error scenarios

## Common Testing Libraries

### React Testing Library
```javascript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
```

### Testing Library Hooks (Alternative)
```javascript
import { renderHook } from '@testing-library/react-hooks';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('should increment counter', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });
});
```

## Interview Tips
- **Test component**: Create component that uses hook to test it
- **RTL**: Use React Testing Library for component testing
- **Mock dependencies**: API calls, timers, DOM APIs
- **Async testing**: Use `waitFor` for async operations
- **Side effects**: Test event listeners, timers, cleanup
- **Error states**: Test error scenarios and boundaries
- **Multiple renders**: Test hook behavior across re-renders