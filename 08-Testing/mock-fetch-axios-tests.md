# How do you mock fetch or axios in tests?

## Question
How do you mock fetch or axios in tests?

# How do you mock fetch or axios in tests?

## Question
How do you mock fetch or axios in tests?

## Answer

Mocking HTTP requests is essential for testing components that make API calls. Here are the main approaches for mocking `fetch` and `axios` in Jest tests.

## Mocking Fetch API

### Global Fetch Mock
```javascript
// UserService.js
export const fetchUser = async (userId) => {
  const response = await fetch(`/api/users/${userId}`);
  if (!response.ok) {
    throw new Error('Failed to fetch user');
  }
  return response.json();
};
```

```javascript
// UserService.test.js
import { fetchUser } from './UserService';

// Mock fetch globally
global.fetch = jest.fn();

describe('fetchUser', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should fetch user successfully', async () => {
    const mockUser = { id: 1, name: 'John' };

    // Mock successful response
    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockUser)
    });

    const result = await fetchUser(1);

    expect(fetch).toHaveBeenCalledWith('/api/users/1');
    expect(result).toEqual(mockUser);
  });

  it('should handle fetch errors', async () => {
    // Mock failed response
    fetch.mockResolvedValueOnce({
      ok: false
    });

    await expect(fetchUser(1)).rejects.toThrow('Failed to fetch user');
  });

  it('should handle network errors', async () => {
    // Mock network error
    fetch.mockRejectedValueOnce(new Error('Network error'));

    await expect(fetchUser(1)).rejects.toThrow('Network error');
  });
});
```

### Manual Mock in __mocks__ Directory
```javascript
// __mocks__/node-fetch.js (if using node-fetch)
module.exports = jest.fn();

// Or for browser fetch
// __mocks__/whatwg-fetch.js
global.fetch = jest.fn();
```

### Mock Implementation with jest.mock
```javascript
// UserService.test.js
import { fetchUser } from './UserService';

// Mock the entire module
jest.mock('./UserService', () => ({
  fetchUser: jest.fn()
}));

describe('fetchUser', () => {
  const mockFetchUser = require('./UserService').fetchUser;

  it('should be called with correct parameters', async () => {
    mockFetchUser.mockResolvedValueOnce({ id: 1, name: 'John' });

    const result = await fetchUser(1);

    expect(mockFetchUser).toHaveBeenCalledWith(1);
    expect(result).toEqual({ id: 1, name: 'John' });
  });
});
```

## Mocking Axios

### Basic Axios Mock
```javascript
// ApiService.js
import axios from 'axios';

export const getUser = (userId) => {
  return axios.get(`/api/users/${userId}`);
};

export const createUser = (userData) => {
  return axios.post('/api/users', userData);
};
```

```javascript
// ApiService.test.js
import axios from 'axios';
import { getUser, createUser } from './ApiService';

// Mock axios
jest.mock('axios');
const mockedAxios = axios;

describe('ApiService', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
    mockedAxios.post.mockClear();
  });

  describe('getUser', () => {
    it('should fetch user successfully', async () => {
      const mockUser = { id: 1, name: 'John' };

      mockedAxios.get.mockResolvedValueOnce({
        data: mockUser
      });

      const result = await getUser(1);

      expect(mockedAxios.get).toHaveBeenCalledWith('/api/users/1');
      expect(result.data).toEqual(mockUser);
    });

    it('should handle axios errors', async () => {
      mockedAxios.get.mockRejectedValueOnce(new Error('Network error'));

      await expect(getUser(1)).rejects.toThrow('Network error');
    });
  });

  describe('createUser', () => {
    it('should create user successfully', async () => {
      const userData = { name: 'John', email: 'john@test.com' };
      const createdUser = { id: 1, ...userData };

      mockedAxios.post.mockResolvedValueOnce({
        data: createdUser
      });

      const result = await createUser(userData);

      expect(mockedAxios.post).toHaveBeenCalledWith('/api/users', userData);
      expect(result.data).toEqual(createdUser);
    });
  });
});
```

### Axios Mock with Different Response Types
```javascript
// ApiService.test.js
describe('ApiService with different responses', () => {
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

  it('should handle timeout', async () => {
    mockedAxios.get.mockRejectedValueOnce({
      code: 'ECONNABORTED',
      message: 'Timeout'
    });

    try {
      await getUser(1);
    } catch (error) {
      expect(error.code).toBe('ECONNABORTED');
      expect(error.message).toBe('Timeout');
    }
  });
});
```

## Advanced Mocking Patterns

### Mock Axios Instance
```javascript
// api.js
import axios from 'axios';

const api = axios.create({
  baseURL: '/api',
  timeout: 5000
});

export const getPosts = () => api.get('/posts');
export const createPost = (post) => api.post('/posts', post);
```

```javascript
// api.test.js
import axios from 'axios';
import { getPosts, createPost } from './api';

// Mock axios module
jest.mock('axios');
const mockedAxios = axios;

// Mock the create method
mockedAxios.create = jest.fn(() => mockedAxios);

describe('API functions', () => {
  beforeEach(() => {
    mockedAxios.get.mockClear();
    mockedAxios.post.mockClear();
  });

  it('should create axios instance with correct config', () => {
    require('./api'); // This will call axios.create

    expect(mockedAxios.create).toHaveBeenCalledWith({
      baseURL: '/api',
      timeout: 5000
    });
  });

  it('should get posts', async () => {
    const mockPosts = [{ id: 1, title: 'Post 1' }];

    mockedAxios.get.mockResolvedValueOnce({
      data: mockPosts
    });

    const result = await getPosts();

    expect(mockedAxios.get).toHaveBeenCalledWith('/posts');
    expect(result.data).toEqual(mockPosts);
  });
});
```

### Mock with Interceptors
```javascript
// apiWithInterceptors.js
import axios from 'axios';

const api = axios.create();

api.interceptors.request.use((config) => {
  config.headers.Authorization = `Bearer ${localStorage.getItem('token')}`;
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle unauthorized
      localStorage.removeItem('token');
    }
    return Promise.reject(error);
  }
);

export const getData = () => api.get('/data');
```

```javascript
// apiWithInterceptors.test.js
import axios from 'axios';
import { getData } from './apiWithInterceptors';

jest.mock('axios');
const mockedAxios = axios;

describe('API with interceptors', () => {
  beforeEach(() => {
    mockedAxios.create.mockReturnValue(mockedAxios);
    mockedAxios.get.mockClear();
  });

  it('should add authorization header', async () => {
    // Mock localStorage
    Object.defineProperty(window, 'localStorage', {
      value: {
        getItem: jest.fn(() => 'mock-token')
      },
      writable: true
    });

    mockedAxios.get.mockResolvedValueOnce({ data: 'success' });

    await getData();

    expect(mockedAxios.get).toHaveBeenCalledWith('/data');
    // Note: Testing interceptors requires more complex mocking
  });
});
```

## Using MSW (Mock Service Worker)

### MSW Setup for Fetch
```javascript
// src/mocks/handlers.js
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    return res(
      ctx.json({
        id: Number(id),
        name: 'John Doe',
        email: 'john@test.com'
      })
    );
  }),

  rest.post('/api/users', (req, res, ctx) => {
    return res(
      ctx.status(201),
      ctx.json({
        id: 3,
        ...req.body
      })
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

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

```javascript
// Component.test.js with MSW
import React from 'react';
import { render, screen, waitFor } from '@testing-library/react';
import { UserProfile } from './UserProfile';

describe('UserProfile with MSW', () => {
  it('should display user data', async () => {
    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
      expect(screen.getByText('john@test.com')).toBeInTheDocument();
    });
  });
});
```

## Testing Custom Hooks with API Calls

### Hook with API Call
```javascript
// useUser.js
import { useState, useEffect } from 'react';

export const useUser = (userId) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(response => {
        if (!response.ok) throw new Error('Failed to fetch');
        return response.json();
      })
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err);
        setLoading(false);
      });
  }, [userId]);

  return { user, loading, error };
};
```

```javascript
// useUser.test.js
import { renderHook, waitFor } from '@testing-library/react';
import { useUser } from './useUser';

// Mock fetch
global.fetch = jest.fn();

describe('useUser', () => {
  beforeEach(() => {
    fetch.mockClear();
  });

  it('should fetch user data', async () => {
    const mockUser = { id: 1, name: 'John' };

    fetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockUser)
    });

    const { result } = renderHook(() => useUser(1));

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.user).toEqual(mockUser);
    expect(result.current.error).toBe(null);
  });

  it('should handle errors', async () => {
    fetch.mockResolvedValueOnce({
      ok: false
    });

    const { result } = renderHook(() => useUser(1));

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.user).toBe(null);
    expect(result.current.error).toEqual(new Error('Failed to fetch'));
  });
});
```

## Best Practices

### 1. Clear Mocks Between Tests
```javascript
beforeEach(() => {
  jest.clearAllMocks();
  // or specifically
  fetch.mockClear();
  axios.get.mockClear();
});
```

### 2. Use Descriptive Mock Names
```javascript
const mockUserResponse = {
  ok: true,
  json: () => Promise.resolve({ id: 1, name: 'John' })
};

fetch.mockResolvedValueOnce(mockUserResponse);
```

### 3. Test Error Scenarios
```javascript
it('should handle network errors', async () => {
  fetch.mockRejectedValueOnce(new Error('Network error'));
  // Test error handling
});
```

### 4. Mock Response Structure
```javascript
// Match real API response structure
const mockResponse = {
  data: { /* actual data */ },
  status: 200,
  headers: { 'content-type': 'application/json' }
};
```

### 5. Avoid Over-Mocking
```javascript
// Don't mock everything - test real integration where possible
// Use integration tests for real API calls with test database
```

## Interview Tips
- **Jest.mock**: Use for module-level mocking
- **Global mocks**: For fetch in browser environment
- **MSW**: More realistic API mocking
- **Clear mocks**: Always clear between tests
- **Error scenarios**: Test all error conditions
- **Response structure**: Match real API responses
- **Async testing**: Use waitFor for async operations