# How to implement global error handling for APIs?

## Question
How to implement global error handling for APIs?

# How to implement global error handling for APIs?

## Question
How to implement global error handling for APIs?

## Answer

Global error handling for APIs provides a centralized way to handle errors across your application. This ensures consistent error handling, logging, and user experience.

## Axios Global Error Handling

### Setting up Axios Interceptors
```javascript
// api/client.js
import axios from 'axios';

// Create axios instance
const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request interceptor for auth
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    // Handle different error types
    return handleApiError(error);
  }
);

export default apiClient;
```

### Global Error Handler
```javascript
// api/errorHandler.js
import { toast } from 'react-toastify';

export const handleApiError = (error) => {
  // Network error
  if (!error.response) {
    toast.error('Network error. Please check your connection.');
    return Promise.reject({
      type: 'NETWORK_ERROR',
      message: 'Network connection failed'
    });
  }

  const { status, data } = error.response;

  // Handle different HTTP status codes
  switch (status) {
    case 400:
      handleValidationError(data);
      break;
    
    case 401:
      handleAuthenticationError();
      break;
    
    case 403:
      handleAuthorizationError();
      break;
    
    case 404:
      handleNotFoundError();
      break;
    
    case 429:
      handleRateLimitError(data);
      break;
    
    case 500:
    case 502:
    case 503:
    case 504:
      handleServerError();
      break;
    
    default:
      handleGenericError(status);
  }

  // Log error for debugging
  console.error('API Error:', {
    status,
    url: error.config?.url,
    message: data?.message || error.message
  });

  return Promise.reject(error);
};

const handleValidationError = (data) => {
  const errors = data.errors || {};
  const firstError = Object.values(errors)[0];
  toast.error(firstError || 'Please check your input');
};

const handleAuthenticationError = () => {
  toast.error('Session expired. Please login again.');
  localStorage.removeItem('authToken');
  // Redirect to login page
  window.location.href = '/login';
};

const handleAuthorizationError = () => {
  toast.error('You do not have permission to perform this action');
};

const handleNotFoundError = () => {
  toast.error('The requested resource was not found');
};

const handleRateLimitError = (data) => {
  const retryAfter = data.retryAfter || 60;
  toast.error(`Too many requests. Please wait ${retryAfter} seconds.`);
};

const handleServerError = () => {
  toast.error('Server error. Please try again later.');
};

const handleGenericError = (status) => {
  toast.error(`An error occurred (${status}). Please try again.`);
};
```

### Using the API Client
```javascript
// api/users.js
import apiClient from './client';

export const getUsers = () => apiClient.get('/users');
export const createUser = (userData) => apiClient.post('/users', userData);
export const updateUser = (id, userData) => apiClient.put(`/users/${id}`, userData);
export const deleteUser = (id) => apiClient.delete(`/users/${id}`);
```

## React Error Boundaries

### API Error Boundary
```javascript
// components/ApiErrorBoundary.js
import React from 'react';
import { toast } from 'react-toastify';

class ApiErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log error details
    console.error('API Error Boundary caught an error:', error, errorInfo);
    
    // Report to error tracking service
    if (window.Sentry) {
      window.Sentry.captureException(error);
    }
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <p>An unexpected error occurred while processing your request.</p>
          <button 
            onClick={() => this.setState({ hasError: false, error: null })}
          >
            Try Again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

export default ApiErrorBoundary;
```

### Using Error Boundary
```javascript
// App.js
import ApiErrorBoundary from './components/ApiErrorBoundary';

function App() {
  return (
    <ApiErrorBoundary>
      <UserManagement />
    </ApiErrorBoundary>
  );
}
```

## React Query Global Error Handling

### React Query Client Setup
```javascript
// api/queryClient.js
import { QueryClient } from 'react-query';
import { toast } from 'react-toastify';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        // Don't retry on 4xx errors
        if (error?.response?.status >= 400 && error?.response?.status < 500) {
          return false;
        }
        // Retry up to 3 times for other errors
        return failureCount < 3;
      },
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000)
    },
    mutations: {
      onError: (error) => {
        handleMutationError(error);
      }
    }
  }
});

// Global error handler for queries
queryClient.getQueryCache().subscribe((event) => {
  if (event?.type === 'error') {
    handleQueryError(event.error);
  }
});

const handleQueryError = (error) => {
  if (error?.response?.status === 401) {
    // Handle authentication globally
    handleAuthenticationError();
  } else if (error?.response?.status >= 500) {
    toast.error('Server error. Please try again later.');
  }
};

const handleMutationError = (error) => {
  if (error?.response?.status === 409) {
    toast.error('This action conflicts with existing data.');
  } else {
    toast.error('Failed to save changes. Please try again.');
  }
};

export default queryClient;
```

### Using React Query with Error Handling
```javascript
// components/UserList.js
import { useQuery, useMutation, useQueryClient } from 'react-query';
import { getUsers, createUser, updateUser, deleteUser } from '../api/users';

export const UserList = () => {
  const queryClient = useQueryClient();
  
  const { data: users, isLoading, error } = useQuery('users', getUsers, {
    onError: (error) => {
      // Specific error handling for this query
      if (error?.response?.status === 403) {
        toast.error('You do not have permission to view users');
      }
    }
  });

  const createMutation = useMutation(createUser, {
    onSuccess: () => {
      queryClient.invalidateQueries('users');
      toast.success('User created successfully');
    },
    onError: (error) => {
      // Mutation-specific error handling
      if (error?.response?.status === 409) {
        toast.error('User with this email already exists');
      }
    }
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error loading users</div>;

  return (
    <div>
      {/* User list */}
      <button onClick={() => createMutation.mutate({ name: 'New User' })}>
        Add User
      </button>
    </div>
  );
};
```

## Custom Hook for API Calls

### API Hook with Error Handling
```javascript
// hooks/useApi.js
import { useState, useCallback } from 'react';
import apiClient from '../api/client';
import { toast } from 'react-toastify';

export const useApi = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const makeRequest = useCallback(async (config) => {
    setLoading(true);
    setError(null);

    try {
      const response = await apiClient(config);
      return response.data;
    } catch (err) {
      setError(err);
      
      // Re-throw for component-specific handling
      throw err;
    } finally {
      setLoading(false);
    }
  }, []);

  const get = useCallback((url, config = {}) => 
    makeRequest({ ...config, method: 'get', url }), [makeRequest]);

  const post = useCallback((url, data, config = {}) => 
    makeRequest({ ...config, method: 'post', url, data }), [makeRequest]);

  const put = useCallback((url, data, config = {}) => 
    makeRequest({ ...config, method: 'put', url, data }), [makeRequest]);

  const del = useCallback((url, config = {}) => 
    makeRequest({ ...config, method: 'delete', url }), [makeRequest]);

  return { get, post, put, del, loading, error };
};
```

### Using the API Hook
```javascript
// components/UserForm.js
import { useState } from 'react';
import { useApi } from '../hooks/useApi';

export const UserForm = () => {
  const { post, loading } = useApi();
  const [formData, setFormData] = useState({ name: '', email: '' });

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await post('/users', formData);
      toast.success('User created successfully');
      setFormData({ name: '', email: '' });
    } catch (error) {
      // Error already handled globally by interceptor
      // Component-specific handling if needed
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Name"
        value={formData.name}
        onChange={(e) => setFormData({...formData, name: e.target.value})}
      />
      <input
        type="email"
        placeholder="Email"
        value={formData.email}
        onChange={(e) => setFormData({...formData, email: e.target.value})}
      />
      <button type="submit" disabled={loading}>
        {loading ? 'Creating...' : 'Create User'}
      </button>
    </form>
  );
};
```

## Testing Global Error Handling

### Testing Axios Interceptors
```javascript
// api/client.test.js
import axios from 'axios';
import apiClient from './client';
import * as errorHandler from './errorHandler';

jest.mock('./errorHandler');
jest.mock('axios');

describe('API Client', () => {
  it('should add authorization header to requests', async () => {
    const mockToken = 'mock-jwt-token';
    localStorage.setItem('authToken', mockToken);

    const mockResponse = { data: 'success' };
    axios.mockResolvedValueOnce(mockResponse);

    await apiClient.get('/test');

    expect(axios).toHaveBeenCalledWith({
      method: 'get',
      url: '/test',
      headers: expect.objectContaining({
        Authorization: `Bearer ${mockToken}`
      })
    });
  });

  it('should handle response errors globally', async () => {
    const mockError = new Error('API Error');
    mockError.response = { status: 500 };
    
    axios.mockRejectedValueOnce(mockError);
    
    errorHandler.handleApiError.mockResolvedValueOnce();

    await expect(apiClient.get('/test')).rejects.toThrow('API Error');
    expect(errorHandler.handleApiError).toHaveBeenCalledWith(mockError);
  });
});
```

### Testing Error Handler
```javascript
// api/errorHandler.test.js
import { handleApiError } from './errorHandler';
import { toast } from 'react-toastify';

jest.mock('react-toastify');

describe('handleApiError', () => {
  beforeEach(() => {
    toast.error.mockClear();
  });

  it('should handle network errors', async () => {
    const error = {
      request: {},
      message: 'Network Error'
    };

    await expect(handleApiError(error)).rejects.toEqual({
      type: 'NETWORK_ERROR',
      message: 'Network connection failed'
    });

    expect(toast.error).toHaveBeenCalledWith('Network error. Please check your connection.');
  });

  it('should handle 401 authentication errors', async () => {
    const error = {
      response: { status: 401 }
    };

    await expect(handleApiError(error)).rejects.toBe(error);

    expect(toast.error).toHaveBeenCalledWith('Session expired. Please login again.');
  });

  it('should handle 400 validation errors', async () => {
    const error = {
      response: {
        status: 400,
        data: {
          errors: { email: 'Invalid email' }
        }
      }
    };

    await expect(handleApiError(error)).rejects.toBe(error);

    expect(toast.error).toHaveBeenCalledWith('Invalid email');
  });

  it('should handle 500 server errors', async () => {
    const error = {
      response: { status: 500 }
    };

    await expect(handleApiError(error)).rejects.toBe(error);

    expect(toast.error).toHaveBeenCalledWith('Server error. Please try again later.');
  });
});
```

## Best Practices

1. **Centralized handling**: Use interceptors for global error handling
2. **User-friendly messages**: Translate technical errors to user language
3. **Logging**: Log errors for debugging while showing friendly messages
4. **Recovery actions**: Provide ways for users to recover from errors
5. **Retry logic**: Implement smart retry for transient errors
6. **Error boundaries**: Prevent app crashes from unhandled errors
7. **Testing**: Test all error scenarios thoroughly

## Interview Tips
- **Interceptors**: Axios interceptors for global request/response handling
- **Error categorization**: Different handling for network, auth, validation errors
- **User experience**: User-friendly error messages and recovery options
- **Logging**: Log errors for debugging without exposing to users
- **Retry logic**: Smart retry for transient failures
- **Error boundaries**: React error boundaries for graceful error handling
- **Testing**: Comprehensive testing of error scenarios