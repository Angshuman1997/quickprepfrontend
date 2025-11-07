# How to implement global error handling for APIs?

## Simple Answer
Implement global error handling using Axios interceptors to catch all API errors in one place. Categorize errors by type (network, auth, validation, server) and show appropriate user messages.

## Axios Interceptor Setup

```javascript
import axios from 'axios';
import { toast } from 'react-toastify';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 10000
});

// Request interceptor - add auth token
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

// Response interceptor - handle errors globally
apiClient.interceptors.response.use(
  (response) => response,
  (error) => handleApiError(error)
);

export default apiClient;
```

## Global Error Handler

```javascript
const handleApiError = (error) => {
  // Network error (no response)
  if (!error.response) {
    toast.error('Network error. Please check your connection.');
    return Promise.reject({
      type: 'NETWORK_ERROR',
      message: 'Network connection failed'
    });
  }

  const { status, data } = error.response;

  switch (status) {
    case 400:
    case 422:
      // Validation errors
      const errors = data.errors || {};
      const firstError = Object.values(errors)[0];
      toast.error(firstError || 'Please check your input');
      break;

    case 401:
      // Authentication error
      toast.error('Session expired. Please login again.');
      localStorage.removeItem('authToken');
      window.location.href = '/login';
      break;

    case 403:
      // Permission error
      toast.error('You do not have permission for this action');
      break;

    case 404:
      // Not found
      toast.error('The requested item was not found');
      break;

    case 429:
      // Rate limiting
      toast.error('Too many requests. Please wait a moment.');
      break;

    case 500:
    case 502:
    case 503:
    case 504:
      // Server errors
      toast.error('Server error. Please try again later.');
      break;

    default:
      toast.error(`An error occurred (${status}). Please try again.`);
  }

  // Log for debugging
  console.error('API Error:', { status, url: error.config?.url, message: data?.message });

  return Promise.reject(error);
};
```

## Using the API Client

```javascript
// api/users.js
import apiClient from './client';

export const getUsers = () => apiClient.get('/users');
export const createUser = (userData) => apiClient.post('/users', userData);
export const updateUser = (id, userData) => apiClient.put(`/users/${id}`, userData);
export const deleteUser = (id) => apiClient.delete(`/users/${id}`);
```

## React Error Boundary

```javascript
import React from 'react';

class ApiErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Could send to error tracking service like Sentry
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Something went wrong</h2>
          <p>An unexpected error occurred.</p>
          <button onClick={() => this.setState({ hasError: false })}>
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

## Custom Hook for API Calls

```javascript
import { useState, useCallback } from 'react';
import apiClient from '../api/client';

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
      throw err; // Let global handler deal with it
    } finally {
      setLoading(false);
    }
  }, []);

  const get = (url) => makeRequest({ method: 'get', url });
  const post = (url, data) => makeRequest({ method: 'post', url, data });
  const put = (url, data) => makeRequest({ method: 'put', url, data });
  const del = (url) => makeRequest({ method: 'delete', url });

  return { get, post, put, del, loading, error };
};
```

## Testing Error Handling

```javascript
describe('Global Error Handler', () => {
  it('handles network errors', () => {
    const error = { request: {} }; // No response
    handleApiError(error);
    expect(toast.error).toHaveBeenCalledWith('Network error. Please check your connection.');
  });

  it('handles auth errors', () => {
    const error = { response: { status: 401 } };
    handleApiError(error);
    expect(toast.error).toHaveBeenCalledWith('Session expired. Please login again.');
  });

  it('handles validation errors', () => {
    const error = {
      response: {
        status: 400,
        data: { errors: { email: 'Invalid format' } }
      }
    };
    handleApiError(error);
    expect(toast.error).toHaveBeenCalledWith('Invalid format');
  });
});
```

## Interview Q&A

**Q: How do you implement global error handling in a React app?**
A: I use Axios interceptors to catch all API errors globally. In the response interceptor, I check the error type and show appropriate user messages. For auth errors, I redirect to login. For validation errors, I show field-specific messages. For server errors, I show a generic message with retry option.

**Q: What's the difference between error boundaries and global error handling?**
A: Error boundaries catch JavaScript errors in the React component tree to prevent app crashes. Global error handling (via interceptors) catches API errors specifically. Both are important for good user experience.

**Q: How do you test error handling?**
A: I mock axios responses with different status codes and test that the right error messages are displayed. I also test that the correct actions are taken, like redirects for auth errors or field error display for validation errors.

**Q: Should you retry failed requests automatically?**
A: Yes for network errors and 5xx server errors, but not for 4xx client errors. I implement exponential backoff to avoid overwhelming the server.

## Best Practices
- **Centralized**: Handle all API errors in one place
- **User-friendly**: Show clear, actionable error messages
- **Logging**: Log technical details for debugging
- **Recovery**: Provide retry options where appropriate
- **Categorization**: Different handling for different error types
- **Testing**: Test all error scenarios thoroughly
- **Boundaries**: Use error boundaries to prevent crashes</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\08-Testing\implement-global-error-handling-apis-simple.md