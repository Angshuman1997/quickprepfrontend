# How do you test components that use context/state?

## Question
How do you test components that use context/state?

# How do you test components that use context/state?

## Question
How do you test components that use context/state?

## Answer

Testing React components that use Context or complex state management requires providing the proper context providers and mocking state as needed. Here are the main approaches:

## Testing Components with Context

### Basic Context Testing
```javascript
// ThemeContext.js
import React, { createContext, useContext } from 'react';

const ThemeContext = createContext();

export const ThemeProvider = ({ children, theme = 'light' }) => {
  return (
    <ThemeContext.Provider value={{ theme }}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
};
```

```javascript
// ThemedButton.js
import React from 'react';
import { useTheme } from './ThemeContext';

export const ThemedButton = ({ children, onClick }) => {
  const { theme } = useTheme();

  return (
    <button
      className={`btn btn-${theme}`}
      onClick={onClick}
    >
      {children}
    </button>
  );
};
```

```javascript
// ThemedButton.test.js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { ThemeProvider } from './ThemeContext';
import { ThemedButton } from './ThemedButton';

describe('ThemedButton', () => {
  const renderWithTheme = (component, theme = 'light') => {
    return render(
      <ThemeProvider theme={theme}>
        {component}
      </ThemeProvider>
    );
  };

  it('should render with light theme by default', () => {
    renderWithTheme(<ThemedButton>Click me</ThemedButton>);

    const button = screen.getByRole('button', { name: /click me/i });
    expect(button).toHaveClass('btn btn-light');
  });

  it('should render with dark theme', () => {
    renderWithTheme(<ThemedButton>Click me</ThemedButton>, 'dark');

    const button = screen.getByRole('button', { name: /click me/i });
    expect(button).toHaveClass('btn btn-dark');
  });

  it('should handle click events', () => {
    const handleClick = jest.fn();
    renderWithTheme(<ThemedButton onClick={handleClick}>Click me</ThemedButton>);

    const button = screen.getByRole('button', { name: /click me/i });
    fireEvent.click(button);

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('should throw error when used outside provider', () => {
    // Suppress console.error for this test
    const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});

    expect(() => render(<ThemedButton>Click me</ThemedButton>))
      .toThrow('useTheme must be used within ThemeProvider');

    consoleSpy.mockRestore();
  });
});
```

## Testing Components with Redux

### Redux Store Testing
```javascript
// store.js
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';

export const createStore = (preloadedState) => {
  return configureStore({
    reducer: {
      counter: counterReducer
    },
    preloadedState
  });
};
```

```javascript
// Counter.js
import React from 'react';
import { useSelector, useDispatch } from 'react-redux';
import { increment, decrement } from './counterSlice';

export const Counter = () => {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();

  return (
    <div>
      <div data-testid="count">{count}</div>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
    </div>
  );
};
```

```javascript
// Counter.test.js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { Provider } from 'react-redux';
import { createStore } from './store';
import { Counter } from './Counter';

const renderWithRedux = (
  component,
  initialState = { counter: { value: 0 } }
) => {
  const store = createStore(initialState);
  return {
    ...render(
      <Provider store={store}>
        {component}
      </Provider>
    ),
    store
  };
};

describe('Counter', () => {
  it('should display initial count', () => {
    renderWithRedux(<Counter />);
    expect(screen.getByTestId('count')).toHaveTextContent('0');
  });

  it('should display custom initial count', () => {
    renderWithRedux(<Counter />, { counter: { value: 5 } });
    expect(screen.getByTestId('count')).toHaveTextContent('5');
  });

  it('should increment count', () => {
    renderWithRedux(<Counter />);

    const incrementButton = screen.getByText('+');
    fireEvent.click(incrementButton);

    expect(screen.getByTestId('count')).toHaveTextContent('1');
  });

  it('should decrement count', () => {
    renderWithRedux(<Counter />, { counter: { value: 3 } });

    const decrementButton = screen.getByText('-');
    fireEvent.click(decrementButton);

    expect(screen.getByTestId('count')).toHaveTextContent('2');
  });

  it('should update store state correctly', () => {
    const { store } = renderWithRedux(<Counter />);

    const incrementButton = screen.getByText('+');
    fireEvent.click(incrementButton);
    fireEvent.click(incrementButton);

    expect(store.getState().counter.value).toBe(2);
  });
});
```

## Testing Components with Zustand

### Zustand Store Testing
```javascript
// useCounterStore.js
import { create } from 'zustand';

export const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 })
}));
```

```javascript
// CounterZustand.js
import React from 'react';
import { useCounterStore } from './useCounterStore';

export const CounterZustand = () => {
  const { count, increment, decrement, reset } = useCounterStore();

  return (
    <div>
      <div data-testid="count">{count}</div>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
};
```

```javascript
// CounterZustand.test.js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { CounterZustand } from './CounterZustand';

// Mock the store
jest.mock('./useCounterStore');

describe('CounterZustand', () => {
  const mockUseCounterStore = require('./useCounterStore').useCounterStore;

  beforeEach(() => {
    mockUseCounterStore.mockReturnValue({
      count: 0,
      increment: jest.fn(),
      decrement: jest.fn(),
      reset: jest.fn()
    });
  });

  it('should display count from store', () => {
    mockUseCounterStore.mockReturnValue({
      count: 5,
      increment: jest.fn(),
      decrement: jest.fn(),
      reset: jest.fn()
    });

    render(<CounterZustand />);
    expect(screen.getByTestId('count')).toHaveTextContent('5');
  });

  it('should call increment when + button is clicked', () => {
    const mockIncrement = jest.fn();
    mockUseCounterStore.mockReturnValue({
      count: 0,
      increment: mockIncrement,
      decrement: jest.fn(),
      reset: jest.fn()
    });

    render(<CounterZustand />);

    const incrementButton = screen.getByText('+');
    fireEvent.click(incrementButton);

    expect(mockIncrement).toHaveBeenCalledTimes(1);
  });

  it('should call decrement when - button is clicked', () => {
    const mockDecrement = jest.fn();
    mockUseCounterStore.mockReturnValue({
      count: 0,
      increment: jest.fn(),
      decrement: mockDecrement,
      reset: jest.fn()
    });

    render(<CounterZustand />);

    const decrementButton = screen.getByText('-');
    fireEvent.click(decrementButton);

    expect(mockDecrement).toHaveBeenCalledTimes(1);
  });

  it('should call reset when reset button is clicked', () => {
    const mockReset = jest.fn();
    mockUseCounterStore.mockReturnValue({
      count: 0,
      increment: jest.fn(),
      decrement: jest.fn(),
      reset: mockReset
    });

    render(<CounterZustand />);

    const resetButton = screen.getByText('Reset');
    fireEvent.click(resetButton);

    expect(mockReset).toHaveBeenCalledTimes(1);
  });
});
```

## Testing Components with useState

### Local State Testing
```javascript
// ToggleButton.js
import React, { useState } from 'react';

export const ToggleButton = ({ initialState = false }) => {
  const [isOn, setIsOn] = useState(initialState);

  return (
    <button
      data-testid="toggle-button"
      onClick={() => setIsOn(!isOn)}
    >
      {isOn ? 'ON' : 'OFF'}
    </button>
  );
};
```

```javascript
// ToggleButton.test.js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { ToggleButton } from './ToggleButton';

describe('ToggleButton', () => {
  it('should start in OFF state by default', () => {
    render(<ToggleButton />);
    expect(screen.getByTestId('toggle-button')).toHaveTextContent('OFF');
  });

  it('should start in ON state when initialState is true', () => {
    render(<ToggleButton initialState={true} />);
    expect(screen.getByTestId('toggle-button')).toHaveTextContent('ON');
  });

  it('should toggle from OFF to ON', () => {
    render(<ToggleButton />);

    const button = screen.getByTestId('toggle-button');
    expect(button).toHaveTextContent('OFF');

    fireEvent.click(button);
    expect(button).toHaveTextContent('ON');

    fireEvent.click(button);
    expect(button).toHaveTextContent('OFF');
  });
});
```

## Testing Components with useEffect

### Effect Testing
```javascript
// DataFetcher.js
import React, { useState, useEffect } from 'react';

export const DataFetcher = ({ url }) => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        const result = await response.json();
        setData(result);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [url]);

  if (loading) return <div data-testid="loading">Loading...</div>;
  if (error) return <div data-testid="error">{error}</div>;
  return <div data-testid="data">{JSON.stringify(data)}</div>;
};
```

```javascript
// DataFetcher.test.js
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import { DataFetcher } from './DataFetcher';

// Mock fetch
global.fetch = jest.fn();

describe('DataFetcher', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should show loading initially', () => {
    fetch.mockResolvedValueOnce({
      json: () => Promise.resolve({ message: 'Hello' })
    });

    render(<DataFetcher url="/api/data" />);
    expect(screen.getByTestId('loading')).toBeInTheDocument();
  });

  it('should fetch and display data', async () => {
    const mockData = { message: 'Hello World' };
    fetch.mockResolvedValueOnce({
      json: () => Promise.resolve(mockData)
    });

    render(<DataFetcher url="/api/data" />);

    await waitFor(() => {
      expect(screen.getByTestId('data')).toHaveTextContent(JSON.stringify(mockData));
    });

    expect(fetch).toHaveBeenCalledWith('/api/data');
  });

  it('should handle fetch errors', async () => {
    fetch.mockRejectedValueOnce(new Error('Network error'));

    render(<DataFetcher url="/api/data" />);

    await waitFor(() => {
      expect(screen.getByTestId('error')).toHaveTextContent('Network error');
    });
  });

  it('should refetch when url changes', async () => {
    const { rerender } = render(<DataFetcher url="/api/data1" />);

    fetch.mockResolvedValueOnce({
      json: () => Promise.resolve({ id: 1 })
    });

    await waitFor(() => {
      expect(fetch).toHaveBeenCalledWith('/api/data1');
    });

    // Change URL
    fetch.mockResolvedValueOnce({
      json: () => Promise.resolve({ id: 2 })
    });

    rerender(<DataFetcher url="/api/data2" />);

    await waitFor(() => {
      expect(fetch).toHaveBeenCalledWith('/api/data2');
    });
  });
});
```

## Testing Custom Hooks with Context

### Hook Testing with Providers
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
// AuthContext.js
import React, { createContext, useState } from 'react';

export const AuthContext = createContext();

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);

  const login = (userData) => setUser(userData);
  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};
```

```javascript
// useAuth.test.js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { AuthProvider } from './AuthContext';

// Test component that uses the hook
const TestAuthComponent = () => {
  const { user, login, logout } = useAuth();

  return (
    <div>
      <div data-testid="auth-status">
        {user ? `Logged in as ${user.name}` : 'Not logged in'}
      </div>
      <button onClick={() => login({ name: 'John' })}>Login</button>
      <button onClick={logout}>Logout</button>
    </div>
  );
};

describe('useAuth', () => {
  const renderWithAuth = (component) => {
    return render(
      <AuthProvider>
        {component}
      </AuthProvider>
    );
  };

  it('should start with no user', () => {
    renderWithAuth(<TestAuthComponent />);
    expect(screen.getByTestId('auth-status')).toHaveTextContent('Not logged in');
  });

  it('should login user', () => {
    renderWithAuth(<TestAuthComponent />);

    const loginButton = screen.getByText('Login');
    fireEvent.click(loginButton);

    expect(screen.getByTestId('auth-status')).toHaveTextContent('Logged in as John');
  });

  it('should logout user', () => {
    renderWithAuth(<TestAuthComponent />);

    // Login first
    fireEvent.click(screen.getByText('Login'));
    expect(screen.getByTestId('auth-status')).toHaveTextContent('Logged in as John');

    // Then logout
    fireEvent.click(screen.getByText('Logout'));
    expect(screen.getByTestId('auth-status')).toHaveTextContent('Not logged in');
  });
});
```

## Advanced Testing Patterns

### Testing Multiple Contexts
```javascript
// AppProviders.js
import React from 'react';
import { ThemeProvider } from './ThemeContext';
import { AuthProvider } from './AuthContext';
import { StoreProvider } from './StoreContext';

export const AppProviders = ({ children }) => {
  return (
    <ThemeProvider>
      <AuthProvider>
        <StoreProvider>
          {children}
        </StoreProvider>
      </AuthProvider>
    </ThemeProvider>
  );
};
```

```javascript
// ComponentWithMultipleContexts.test.js
import React from 'react';
import { render, screen } from '@testing-library/react';
import { AppProviders } from './AppProviders';
import { ComponentWithMultipleContexts } from './ComponentWithMultipleContexts';

const renderWithAllProviders = (component) => {
  return render(
    <AppProviders>
      {component}
    </AppProviders>
  );
};

describe('ComponentWithMultipleContexts', () => {
  it('should work with all required contexts', () => {
    renderWithAllProviders(<ComponentWithMultipleContexts />);

    // Test component behavior with all contexts available
    expect(screen.getByTestId('component')).toBeInTheDocument();
  });
});
```

## Best Practices

1. **Provider Wrappers**: Create reusable provider components for testing
2. **Mock State**: Use mocks for external state management
3. **Initial State**: Test components with different initial states
4. **State Changes**: Test how components respond to state updates
5. **Error Boundaries**: Test error states and error boundaries
6. **Async Operations**: Use `waitFor` for async state updates
7. **Cleanup**: Test that components clean up properly

## Interview Tips
- **Context**: Wrap components with providers in tests
- **Redux**: Use Provider with createStore for testing
- **Zustand**: Mock the store hook for isolation
- **useState**: Test state changes and initial values
- **useEffect**: Test side effects and async operations
- **Multiple contexts**: Create combined provider wrappers
- **Error handling**: Test error states and boundaries