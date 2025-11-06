# How do you test a component that makes API calls?

## Question
How do you test a component that makes API calls?

# How do you test a component that makes API calls?

## Question
How do you test a component that makes API calls?

## Answer

Testing components that make API calls requires mocking the HTTP requests to avoid actual network calls and ensure predictable test results. Here are the main approaches using different testing libraries.

## Testing with Fetch API

### Component with Fetch
```javascript
// UserProfile.js
import React, { useState, useEffect } from 'react';

export const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchUser = async () => {
      try {
        setLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) {
          throw new Error('Failed to fetch user');
        }
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

  if (loading) return <div data-testid="loading">Loading...</div>;
  if (error) return <div data-testid="error">{error}</div>;
  if (!user) return <div data-testid="no-user">No user found</div>;

  return (
    <div data-testid="user-profile">
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
};
```

```javascript
// UserProfile.test.js
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import { UserProfile } from './UserProfile';

// Mock fetch globally
global.fetch = jest.fn();

describe('UserProfile', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should show loading initially', () => {
    // Mock successful response
    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ name: 'John Doe', email: 'john@test.com' })
    });

    render(<UserProfile userId="123" />);
    expect(screen.getByTestId('loading')).toBeInTheDocument();
  });

  it('should fetch and display user data', async () => {
    const mockUser = { name: 'John Doe', email: 'john@test.com' };

    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockUser)
    });

    render(<UserProfile userId="123" />);

    // Wait for the user data to be displayed
    await waitFor(() => {
      expect(screen.getByTestId('user-profile')).toBeInTheDocument();
    });

    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@test.com')).toBeInTheDocument();
    expect(fetch).toHaveBeenCalledWith('/api/users/123');
  });

  it('should handle API errors', async () => {
    fetch.mockResolvedValueOnce({
      ok: false
    });

    render(<UserProfile userId="123" />);

    await waitFor(() => {
      expect(screen.getByTestId('error')).toBeInTheDocument();
    });

    expect(screen.getByText('Failed to fetch user')).toBeInTheDocument();
  });

  it('should handle network errors', async () => {
    fetch.mockRejectedValueOnce(new Error('Network error'));

    render(<UserProfile userId="123" />);

    await waitFor(() => {
      expect(screen.getByTestId('error')).toBeInTheDocument();
    });

    expect(screen.getByText('Network error')).toBeInTheDocument();
  });

  it('should not fetch when userId is not provided', () => {
    render(<UserProfile />);
    expect(screen.getByTestId('no-user')).toBeInTheDocument();
    expect(fetch).not.toHaveBeenCalled();
  });

  it('should refetch when userId changes', async () => {
    const { rerender } = render(<UserProfile userId="123" />);

    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ name: 'John', email: 'john@test.com' })
    });

    await waitFor(() => {
      expect(screen.getByText('John')).toBeInTheDocument();
    });

    // Change userId
    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve({ name: 'Jane', email: 'jane@test.com' })
    });

    rerender(<UserProfile userId="456" />);

    await waitFor(() => {
      expect(screen.getByText('Jane')).toBeInTheDocument();
    });

    expect(fetch).toHaveBeenCalledTimes(2);
    expect(fetch).toHaveBeenLastCalledWith('/api/users/456');
  });
});
```

## Testing with Axios

### Component with Axios
```javascript
// ProductList.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';

export const ProductList = () => {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchProducts = async () => {
      try {
        setLoading(true);
        const response = await axios.get('/api/products');
        setProducts(response.data);
      } catch (err) {
        setError(err.response?.data?.message || err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchProducts();
  }, []);

  if (loading) return <div data-testid="loading">Loading products...</div>;
  if (error) return <div data-testid="error">{error}</div>;

  return (
    <div data-testid="product-list">
      {products.map(product => (
        <div key={product.id} data-testid={`product-${product.id}`}>
          <h3>{product.name}</h3>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
};
```

```javascript
// ProductList.test.js
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import axios from 'axios';
import { ProductList } from './ProductList';

// Mock axios
jest.mock('axios');
const mockedAxios = axios;

describe('ProductList', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
  });

  it('should show loading initially', () => {
    mockedAxios.get.mockResolvedValueOnce({
      data: []
    });

    render(<ProductList />);
    expect(screen.getByTestId('loading')).toBeInTheDocument();
  });

  it('should fetch and display products', async () => {
    const mockProducts = [
      { id: 1, name: 'Product A', price: 10.99 },
      { id: 2, name: 'Product B', price: 15.99 }
    ];

    mockedAxios.get.mockResolvedValueOnce({
      data: mockProducts
    });

    render(<ProductList />);

    await waitFor(() => {
      expect(screen.getByTestId('product-list')).toBeInTheDocument();
    });

    expect(screen.getByText('Product A')).toBeInTheDocument();
    expect(screen.getByText('$10.99')).toBeInTheDocument();
    expect(screen.getByText('Product B')).toBeInTheDocument();
    expect(screen.getByText('$15.99')).toBeInTheDocument();

    expect(mockedAxios.get).toHaveBeenCalledWith('/api/products');
  });

  it('should handle axios errors with response', async () => {
    const errorMessage = 'Server error';
    mockedAxios.get.mockRejectedValueOnce({
      response: {
        data: { message: errorMessage }
      }
    });

    render(<ProductList />);

    await waitFor(() => {
      expect(screen.getByTestId('error')).toBeInTheDocument();
    });

    expect(screen.getByText(errorMessage)).toBeInTheDocument();
  });

  it('should handle network errors', async () => {
    mockedAxios.get.mockRejectedValueOnce(new Error('Network error'));

    render(<ProductList />);

    await waitFor(() => {
      expect(screen.getByTestId('error')).toBeInTheDocument();
    });

    expect(screen.getByText('Network error')).toBeInTheDocument();
  });

  it('should handle empty product list', async () => {
    mockedAxios.get.mockResolvedValueOnce({
      data: []
    });

    render(<ProductList />);

    await waitFor(() => {
      expect(screen.getByTestId('product-list')).toBeInTheDocument();
    });

    // Should render empty list without errors
    expect(screen.queryByTestId(/product-/)).toBeNull();
  });
});
```

## Testing with React Query/SWR

### Component with React Query
```javascript
// UserList.js
import React from 'react';
import { useQuery } from 'react-query';
import axios from 'axios';

const fetchUsers = () => axios.get('/api/users').then(res => res.data);

export const UserList = () => {
  const { data: users, isLoading, error, refetch } = useQuery('users', fetchUsers);

  if (isLoading) return <div data-testid="loading">Loading users...</div>;
  if (error) return <div data-testid="error">Error: {error.message}</div>;

  return (
    <div>
      <button onClick={refetch} data-testid="refetch">Refresh</button>
      <div data-testid="user-list">
        {users?.map(user => (
          <div key={user.id} data-testid={`user-${user.id}`}>
            {user.name}
          </div>
        ))}
      </div>
    </div>
  );
};
```

```javascript
// UserList.test.js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from 'react-query';
import axios from 'axios';
import { UserList } from './UserList';

// Mock axios
jest.mock('axios');
const mockedAxios = axios;

// Create a custom render function with QueryClient
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false, // Disable retries for testing
      },
    },
  });

  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
};

describe('UserList', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
  });

  it('should show loading state', () => {
    mockedAxios.get.mockResolvedValueOnce({
      data: []
    });

    render(<UserList />, { wrapper: createWrapper() });
    expect(screen.getByTestId('loading')).toBeInTheDocument();
  });

  it('should fetch and display users', async () => {
    const mockUsers = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' }
    ];

    mockedAxios.get.mockResolvedValueOnce({
      data: mockUsers
    });

    render(<UserList />, { wrapper: createWrapper() });

    await waitFor(() => {
      expect(screen.getByTestId('user-list')).toBeInTheDocument();
    });

    expect(screen.getByText('John')).toBeInTheDocument();
    expect(screen.getByText('Jane')).toBeInTheDocument();
  });

  it('should handle errors', async () => {
    mockedAxios.get.mockRejectedValueOnce(new Error('API Error'));

    render(<UserList />, { wrapper: createWrapper() });

    await waitFor(() => {
      expect(screen.getByTestId('error')).toBeInTheDocument();
    });

    expect(screen.getByText('Error: API Error')).toBeInTheDocument();
  });

  it('should refetch when refresh button is clicked', async () => {
    const mockUsers = [{ id: 1, name: 'John' }];
    mockedAxios.get.mockResolvedValue({ data: mockUsers });

    render(<UserList />, { wrapper: createWrapper() });

    await waitFor(() => {
      expect(screen.getByText('John')).toBeInTheDocument();
    });

    // Click refresh
    fireEvent.click(screen.getByTestId('refetch'));

    // Should call API again
    await waitFor(() => {
      expect(mockedAxios.get).toHaveBeenCalledTimes(2);
    });
  });
});
```

## Testing with MSW (Mock Service Worker)

### MSW Setup
```javascript
// src/mocks/handlers.js
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(
      ctx.json({
        id,
        name: 'John Doe',
        email: 'john@test.com'
      })
    );
  }),

  rest.get('/api/users', (req, res, ctx) => {
    return res(
      ctx.json([
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' }
      ])
    );
  }),

  rest.post('/api/users', (req, res, ctx) => {
    return res(
      ctx.status(201),
      ctx.json({ id: 3, ...req.body })
    );
  })
];
```

```javascript
// src/mocks/server.js
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```javascript
// src/setupTests.js
import { server } from './mocks/server';

// Establish API mocking before all tests
beforeAll(() => server.listen());

// Reset any request handlers that we may add during the tests
afterEach(() => server.resetHandlers());

// Clean up after all tests are done
afterAll(() => server.close());
```

```javascript
// UserProfile.test.js with MSW
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import { UserProfile } from './UserProfile';
import { server } from '../mocks/server';
import { rest } from 'msw';

describe('UserProfile with MSW', () => {
  it('should fetch and display user data', async () => {
    render(<UserProfile userId="123" />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
      expect(screen.getByText('john@test.com')).toBeInTheDocument();
    });
  });

  it('should handle server errors', async () => {
    server.use(
      rest.get('/api/users/:id', (req, res, ctx) => {
        return res(ctx.status(500));
      })
    );

    render(<UserProfile userId="123" />);

    await waitFor(() => {
      expect(screen.getByTestId('error')).toBeInTheDocument();
    });
  });
});
```

## Testing API Error Handling

### Component with Error Handling
```javascript
// ApiErrorHandler.js
import React, { useState } from 'react';
import axios from 'axios';

export const ApiErrorHandler = () => {
  const [result, setResult] = useState('');
  const [error, setError] = useState('');

  const handleApiCall = async (endpoint) => {
    try {
      setError('');
      const response = await axios.get(`/api/${endpoint}`);
      setResult(`Success: ${response.data.message}`);
    } catch (err) {
      if (err.response) {
        // Server responded with error status
        switch (err.response.status) {
          case 400:
            setError('Bad Request: Please check your input');
            break;
          case 401:
            setError('Unauthorized: Please login');
            break;
          case 403:
            setError('Forbidden: You do not have permission');
            break;
          case 404:
            setError('Not Found: Resource does not exist');
            break;
          case 500:
            setError('Server Error: Please try again later');
            break;
          default:
            setError(`Error: ${err.response.status}`);
        }
      } else if (err.request) {
        // Network error
        setError('Network Error: Please check your connection');
      } else {
        // Other error
        setError(`Error: ${err.message}`);
      }
    }
  };

  return (
    <div>
      <button onClick={() => handleApiCall('success')}>Success Call</button>
      <button onClick={() => handleApiCall('error/400')}>400 Error</button>
      <button onClick={() => handleApiCall('error/500')}>500 Error</button>
      <div data-testid="result">{result}</div>
      <div data-testid="error">{error}</div>
    </div>
  );
};
```

```javascript
// ApiErrorHandler.test.js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import axios from 'axios';
import { ApiErrorHandler } from './ApiErrorHandler';

jest.mock('axios');
const mockedAxios = axios;

describe('ApiErrorHandler', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
  });

  it('should handle successful API call', async () => {
    mockedAxios.get.mockResolvedValueOnce({
      data: { message: 'Success!' }
    });

    render(<ApiErrorHandler />);

    fireEvent.click(screen.getByText('Success Call'));

    await waitFor(() => {
      expect(screen.getByTestId('result')).toHaveTextContent('Success: Success!');
    });

    expect(screen.getByTestId('error')).toHaveTextContent('');
  });

  it('should handle 400 Bad Request error', async () => {
    mockedAxios.get.mockRejectedValueOnce({
      response: { status: 400 }
    });

    render(<ApiErrorHandler />);

    fireEvent.click(screen.getByText('400 Error'));

    await waitFor(() => {
      expect(screen.getByTestId('error')).toHaveTextContent('Bad Request: Please check your input');
    });
  });

  it('should handle 500 Server Error', async () => {
    mockedAxios.get.mockRejectedValueOnce({
      response: { status: 500 }
    });

    render(<ApiErrorHandler />);

    fireEvent.click(screen.getByText('500 Error'));

    await waitFor(() => {
      expect(screen.getByTestId('error')).toHaveTextContent('Server Error: Please try again later');
    });
  });

  it('should handle network errors', async () => {
    mockedAxios.get.mockRejectedValueOnce({
      request: {}
    });

    render(<ApiErrorHandler />);

    fireEvent.click(screen.getByText('Success Call'));

    await waitFor(() => {
      expect(screen.getByTestId('error')).toHaveTextContent('Network Error: Please check your connection');
    });
  });

  it('should handle other errors', async () => {
    mockedAxios.get.mockRejectedValueOnce(new Error('Custom error'));

    render(<ApiErrorHandler />);

    fireEvent.click(screen.getByText('Success Call'));

    await waitFor(() => {
      expect(screen.getByTestId('error')).toHaveTextContent('Error: Custom error');
    });
  });
});
```

## Best Practices

1. **Mock HTTP calls**: Never make real API calls in tests
2. **Test all states**: Loading, success, error states
3. **Mock different responses**: Success, various error codes, network errors
4. **Test error handling**: Different error scenarios
5. **Use descriptive test names**: What behavior is being tested
6. **Clean up mocks**: Reset mocks between tests
7. **Test async behavior**: Use `waitFor` for async operations

## Interview Tips
- **Mock APIs**: Use jest.mock or MSW to avoid real calls
- **Test states**: Loading, success, error states
- **Error handling**: Test different error scenarios
- **Async testing**: Use waitFor for async operations
- **Network errors**: Test both response and request errors
- **Loading states**: Verify loading indicators
- **Data fetching**: Test successful data retrieval