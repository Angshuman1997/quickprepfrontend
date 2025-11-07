# How do you mock fetch or axios in tests?

## Simple Answer
Use Jest to mock HTTP requests. For axios, use `jest.mock('axios')` and mock the methods. For fetch, mock `global.fetch`. Always clear mocks between tests.

## Mocking Axios

```javascript
import axios from 'axios';
import { getUser } from './api';

// Mock axios module
jest.mock('axios');
const mockedAxios = axios;

describe('API calls', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
  });

  it('should fetch user successfully', async () => {
    const mockUser = { id: 1, name: 'John' };

    // Mock the response
    mockedAxios.get.mockResolvedValueOnce({
      data: mockUser
    });

    const result = await getUser(1);

    expect(mockedAxios.get).toHaveBeenCalledWith('/api/users/1');
    expect(result.data).toEqual(mockUser);
  });

  it('should handle errors', async () => {
    mockedAxios.get.mockRejectedValueOnce(new Error('Network error'));

    await expect(getUser(1)).rejects.toThrow('Network error');
  });
});
```

## Mocking Fetch

```javascript
// Mock fetch globally
global.fetch = jest.fn();

describe('fetch calls', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should fetch data successfully', async () => {
    const mockData = { id: 1, name: 'John' };

    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockData)
    });

    const response = await fetch('/api/users/1');
    const data = await response.json();

    expect(fetch).toHaveBeenCalledWith('/api/users/1');
    expect(data).toEqual(mockData);
  });

  it('should handle fetch errors', async () => {
    fetch.mockResolvedValueOnce({
      ok: false,
      status: 404
    });

    const response = await fetch('/api/users/999');
    expect(response.ok).toBe(false);
  });
});
```

## Testing Component with API Calls

```javascript
// UserComponent.js
import { useEffect, useState } from 'react';
import axios from 'axios';

const UserComponent = ({ userId }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    axios.get(`/api/users/${userId}`)
      .then(response => {
        setUser(response.data);
        setLoading(false);
      })
      .catch(error => {
        console.error('Error fetching user:', error);
        setLoading(false);
      });
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  return <div>{user?.name}</div>;
};

export default UserComponent;
```

```javascript
// UserComponent.test.js
import { render, screen, waitFor } from '@testing-library/react';
import axios from 'axios';
import UserComponent from './UserComponent';

jest.mock('axios');
const mockedAxios = axios;

describe('UserComponent', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
  });

  it('should display user name', async () => {
    const mockUser = { id: 1, name: 'John Doe' };

    mockedAxios.get.mockResolvedValueOnce({
      data: mockUser
    });

    render(<UserComponent userId={1} />);

    // Initially shows loading
    expect(screen.getByText('Loading...')).toBeInTheDocument();

    // Wait for API call to complete
    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    expect(mockedAxios.get).toHaveBeenCalledWith('/api/users/1');
  });

  it('should handle API errors', async () => {
    mockedAxios.get.mockRejectedValueOnce(new Error('API Error'));

    render(<UserComponent userId={1} />);

    await waitFor(() => {
      expect(screen.queryByText('Loading...')).not.toBeInTheDocument();
    });

    // Component should handle error gracefully
    expect(mockedAxios.get).toHaveBeenCalledWith('/api/users/1');
  });
});
```

## Mocking Different Response Types

```javascript
describe('Different error scenarios', () => {
  it('should handle 404 errors', async () => {
    const errorResponse = {
      response: {
        status: 404,
        data: { message: 'User not found' }
      }
    };

    mockedAxios.get.mockRejectedValueOnce(errorResponse);

    try {
      await getUser(999);
    } catch (error) {
      expect(error.response.status).toBe(404);
      expect(error.response.data.message).toBe('User not found');
    }
  });

  it('should handle network errors', async () => {
    const networkError = {
      request: {},
      message: 'Network Error'
    };

    mockedAxios.get.mockRejectedValueOnce(networkError);

    try {
      await getUser(1);
    } catch (error) {
      expect(error.request).toBeDefined();
      expect(error.message).toBe('Network Error');
    }
  });
});
```

## Using MSW (Mock Service Worker)

```javascript
// mocks/handlers.js
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(
      ctx.json({
        id: Number(id),
        name: 'John Doe'
      })
    );
  })
];

// mocks/server.js
import { setupServer } from 'msw/node';
import { handlers } from './handlers';
export const server = setupServer(...handlers);

// setupTests.js
import { server } from './mocks/server';
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

## Interview Q&A

**Q: How do you mock API calls in Jest tests?**
A: For axios, I use `jest.mock('axios')` and mock the specific methods like `get` or `post`. For fetch, I mock `global.fetch`. I always clear mocks between tests with `beforeEach` to avoid test interference.

**Q: What's the difference between mocking axios vs using MSW?**
A: Jest mocks replace the module entirely and are faster. MSW intercepts requests at the network level, making tests more realistic but slower. I use Jest mocks for unit tests and MSW for integration tests.

**Q: How do you test error scenarios in API calls?**
A: I mock axios or fetch to reject with different error structures - network errors (no response), 4xx errors (response with status), and 5xx errors. I test that the component handles each type appropriately.

**Q: Should you mock every API call in tests?**
A: For unit tests, yes - mock external dependencies to isolate the code being tested. For integration tests, you might use a real test API or MSW to test the full flow. The key is testing your code's behavior, not the external API.

## Best Practices
- **Clear mocks**: Always clear mocks between tests
- **Realistic responses**: Match the structure of real API responses
- **Error scenarios**: Test all error conditions (network, 4xx, 5xx)
- **Async testing**: Use `waitFor` for async operations
- **Descriptive mocks**: Name mock variables clearly
- **Isolation**: Mock external dependencies to test your code in isolation</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\08-Testing\mock-fetch-axios-tests-simple.md