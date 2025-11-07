# How do you test a component that makes API calls?

## Simple Answer
Mock the API calls using Jest. For axios, use `jest.mock('axios')`. For fetch, mock `global.fetch`. Test loading states, success, and error scenarios using `waitFor`.

## Testing Component with Axios

```javascript
// UserProfile.js
import { useEffect, useState } from 'react';
import axios from 'axios';

const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    axios.get(`/api/users/${userId}`)
      .then(response => {
        setUser(response.data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  return <div>{user?.name}</div>;
};

export default UserProfile;
```

```javascript
// UserProfile.test.js
import { render, screen, waitFor } from '@testing-library/react';
import axios from 'axios';
import UserProfile from './UserProfile';

jest.mock('axios');
const mockedAxios = axios;

describe('UserProfile', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
  });

  it('should show loading initially', () => {
    mockedAxios.get.mockResolvedValueOnce({
      data: { name: 'John' }
    });

    render(<UserProfile userId={1} />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  it('should fetch and display user data', async () => {
    const mockUser = { name: 'John Doe', email: 'john@test.com' };

    mockedAxios.get.mockResolvedValueOnce({
      data: mockUser
    });

    render(<UserProfile userId={1} />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    expect(mockedAxios.get).toHaveBeenCalledWith('/api/users/1');
  });

  it('should handle API errors', async () => {
    mockedAxios.get.mockRejectedValueOnce(new Error('API Error'));

    render(<UserProfile userId={1} />);

    await waitFor(() => {
      expect(screen.getByText('Error: API Error')).toBeInTheDocument();
    });
  });
});
```

## Testing Component with Fetch

```javascript
// ProductList.js
import { useEffect, useState } from 'react';

const ProductList = () => {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/products')
      .then(response => response.json())
      .then(data => {
        setProducts(data);
        setLoading(false);
      })
      .catch(error => {
        console.error('Error:', error);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading products...</div>;

  return (
    <div>
      {products.map(product => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
};

export default ProductList;
```

```javascript
// ProductList.test.js
import { render, screen, waitFor } from '@testing-library/react';
import ProductList from './ProductList';

// Mock fetch globally
global.fetch = jest.fn();

describe('ProductList', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should fetch and display products', async () => {
    const mockProducts = [
      { id: 1, name: 'Product A' },
      { id: 2, name: 'Product B' }
    ];

    fetch.mockResolvedValueOnce({
      json: () => Promise.resolve(mockProducts)
    });

    render(<ProductList />);

    await waitFor(() => {
      expect(screen.getByText('Product A')).toBeInTheDocument();
      expect(screen.getByText('Product B')).toBeInTheDocument();
    });

    expect(fetch).toHaveBeenCalledWith('/api/products');
  });

  it('should handle fetch errors', async () => {
    fetch.mockRejectedValueOnce(new Error('Network error'));

    // Mock console.error to avoid test output noise
    const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});

    render(<ProductList />);

    await waitFor(() => {
      expect(screen.queryByText('Loading products...')).not.toBeInTheDocument();
    });

    expect(consoleSpy).toHaveBeenCalledWith('Error:', expect.any(Error));

    consoleSpy.mockRestore();
  });
});
```

## Testing with React Query

```javascript
// UserList.js
import { useQuery } from 'react-query';
import axios from 'axios';

const fetchUsers = () => axios.get('/api/users').then(res => res.data);

const UserList = () => {
  const { data: users, isLoading, error } = useQuery('users', fetchUsers);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading users</div>;

  return (
    <div>
      {users?.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
    </div>
  );
};

export default UserList;
```

```javascript
// UserList.test.js
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from 'react-query';
import axios from 'axios';
import UserList from './UserList';

jest.mock('axios');
const mockedAxios = axios;

// Create wrapper with QueryClient
const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } }
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

  it('should display users', async () => {
    const mockUsers = [{ id: 1, name: 'John' }];

    mockedAxios.get.mockResolvedValueOnce({
      data: mockUsers
    });

    render(<UserList />, { wrapper: createWrapper() });

    await waitFor(() => {
      expect(screen.getByText('John')).toBeInTheDocument();
    });
  });

  it('should handle errors', async () => {
    mockedAxios.get.mockRejectedValueOnce(new Error('API Error'));

    render(<UserList />, { wrapper: createWrapper() });

    await waitFor(() => {
      expect(screen.getByText('Error loading users')).toBeInTheDocument();
    });
  });
});
```

## Testing Different Error Types

```javascript
describe('Error handling', () => {
  it('should handle 404 errors', async () => {
    mockedAxios.get.mockRejectedValueOnce({
      response: { status: 404, data: { message: 'Not found' } }
    });

    render(<UserProfile userId={999} />);

    await waitFor(() => {
      expect(screen.getByText('Error: Request failed with status code 404')).toBeInTheDocument();
    });
  });

  it('should handle network errors', async () => {
    mockedAxios.get.mockRejectedValueOnce({
      request: {}, // No response received
      message: 'Network Error'
    });

    render(<UserProfile userId={1} />);

    await waitFor(() => {
      expect(screen.getByText('Error: Network Error')).toBeInTheDocument();
    });
  });
});
```

## Interview Q&A

**Q: How do you test a React component that makes API calls?**
A: I mock the HTTP library (axios or fetch) using Jest mocks. I test the loading state initially, then use `waitFor` to wait for the async operation to complete and check that the correct data is displayed. I also test error scenarios by mocking rejected promises.

**Q: What's the difference between testing with mocked axios vs MSW?**
A: Jest mocks replace the module entirely and are faster for unit tests. MSW intercepts requests at the network level, making tests more realistic but slower. I use Jest mocks for most component tests and MSW for integration tests.

**Q: How do you test loading and error states?**
A: For loading states, I check that a loading indicator is shown initially. For error states, I mock the API to reject and verify that error messages are displayed. I use `waitFor` to wait for state changes since API calls are asynchronous.

**Q: Should you make real API calls in tests?**
A: No, never make real API calls in tests. They make tests slow, unreliable, and dependent on external services. Always mock HTTP requests to ensure tests are fast, reliable, and isolated.

## Best Practices
- **Mock HTTP calls**: Never make real API calls in tests
- **Clear mocks**: Reset mocks between tests with `beforeEach`
- **Test all states**: Loading, success, and error scenarios
- **Async testing**: Use `waitFor` for asynchronous operations
- **Error scenarios**: Test network errors, 4xx, and 5xx responses
- **Loading indicators**: Verify loading states are shown appropriately
- **Data display**: Test that API data is rendered correctly</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\08-Testing\test-component-api-calls-simple.md