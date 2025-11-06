# How to handle different types of API errors?

## Question
How to handle different types of API errors?

# How to handle different types of API errors?

## Question
How to handle different types of API errors?

## Answer

Handling different types of API errors is crucial for robust applications. Here are the main categories of API errors and how to handle them in both code and tests.

## Types of API Errors

### 1. HTTP Status Code Errors
```javascript
// ApiErrorHandler.js
export const handleApiError = (error) => {
  if (error.response) {
    // Server responded with error status
    const { status, data } = error.response;
    
    switch (status) {
      case 400:
        return {
          type: 'VALIDATION_ERROR',
          message: data.message || 'Invalid request data',
          details: data.errors
        };
      
      case 401:
        return {
          type: 'AUTHENTICATION_ERROR',
          message: 'Authentication required',
          action: 'redirect_to_login'
        };
      
      case 403:
        return {
          type: 'AUTHORIZATION_ERROR',
          message: 'Access denied',
          action: 'show_permission_denied'
        };
      
      case 404:
        return {
          type: 'NOT_FOUND_ERROR',
          message: 'Resource not found',
          action: 'show_not_found_page'
        };
      
      case 409:
        return {
          type: 'CONFLICT_ERROR',
          message: 'Resource conflict',
          details: data.conflicts
        };
      
      case 422:
        return {
          type: 'VALIDATION_ERROR',
          message: 'Validation failed',
          details: data.errors
        };
      
      case 429:
        return {
          type: 'RATE_LIMIT_ERROR',
          message: 'Too many requests',
          retryAfter: data.retryAfter
        };
      
      case 500:
        return {
          type: 'SERVER_ERROR',
          message: 'Internal server error',
          action: 'show_generic_error'
        };
      
      case 502:
      case 503:
      case 504:
        return {
          type: 'SERVICE_UNAVAILABLE',
          message: 'Service temporarily unavailable',
          action: 'retry_later'
        };
      
      default:
        return {
          type: 'UNKNOWN_ERROR',
          message: 'An unexpected error occurred',
          status
        };
    }
  } else if (error.request) {
    // Network error - no response received
    return {
      type: 'NETWORK_ERROR',
      message: 'Network connection failed',
      action: 'check_connection'
    };
  } else {
    // Other error
    return {
      type: 'CLIENT_ERROR',
      message: error.message || 'An unexpected error occurred'
    };
  }
};
```

### 2. Network Errors
```javascript
// NetworkErrorHandler.js
export const handleNetworkError = (error) => {
  // Check if it's a network error
  if (!error.response && error.request) {
    // Network timeout
    if (error.code === 'ECONNABORTED') {
      return {
        type: 'TIMEOUT_ERROR',
        message: 'Request timed out',
        action: 'retry'
      };
    }
    
    // Network unreachable
    if (error.code === 'ENOTFOUND' || error.code === 'ECONNREFUSED') {
      return {
        type: 'CONNECTION_ERROR',
        message: 'Unable to connect to server',
        action: 'check_network'
      };
    }
    
    // Generic network error
    return {
      type: 'NETWORK_ERROR',
      message: 'Network error occurred',
      action: 'retry'
    };
  }
  
  return null; // Not a network error
};
```

### 3. Validation Errors
```javascript
// ValidationErrorHandler.js
export const handleValidationError = (error) => {
  if (error.response?.status === 400 || error.response?.status === 422) {
    const validationErrors = error.response.data.errors || {};
    
    // Transform validation errors into user-friendly format
    const formattedErrors = {};
    
    Object.keys(validationErrors).forEach(field => {
      const errors = validationErrors[field];
      formattedErrors[field] = Array.isArray(errors) ? errors[0] : errors;
    });
    
    return {
      type: 'VALIDATION_ERROR',
      message: 'Please correct the following errors',
      errors: formattedErrors,
      action: 'show_field_errors'
    };
  }
  
  return null;
};
```

## Testing Error Handling

### Testing HTTP Status Errors
```javascript
// ApiErrorHandler.test.js
import axios from 'axios';
import { handleApiError } from './ApiErrorHandler';

jest.mock('axios');

describe('handleApiError', () => {
  it('should handle 400 validation error', () => {
    const error = {
      response: {
        status: 400,
        data: {
          message: 'Invalid input',
          errors: { email: 'Invalid email format' }
        }
      }
    };

    const result = handleApiError(error);

    expect(result).toEqual({
      type: 'VALIDATION_ERROR',
      message: 'Invalid input',
      details: { email: 'Invalid email format' }
    });
  });

  it('should handle 401 authentication error', () => {
    const error = {
      response: {
        status: 401,
        data: { message: 'Unauthorized' }
      }
    };

    const result = handleApiError(error);

    expect(result).toEqual({
      type: 'AUTHENTICATION_ERROR',
      message: 'Authentication required',
      action: 'redirect_to_login'
    });
  });

  it('should handle 404 not found error', () => {
    const error = {
      response: {
        status: 404,
        data: { message: 'User not found' }
      }
    };

    const result = handleApiError(error);

    expect(result).toEqual({
      type: 'NOT_FOUND_ERROR',
      message: 'Resource not found',
      action: 'show_not_found_page'
    });
  });

  it('should handle 429 rate limit error', () => {
    const error = {
      response: {
        status: 429,
        data: { 
          message: 'Too many requests',
          retryAfter: 60
        }
      }
    };

    const result = handleApiError(error);

    expect(result).toEqual({
      type: 'RATE_LIMIT_ERROR',
      message: 'Too many requests',
      retryAfter: 60
    });
  });

  it('should handle 500 server error', () => {
    const error = {
      response: {
        status: 500,
        data: { message: 'Internal server error' }
      }
    };

    const result = handleApiError(error);

    expect(result).toEqual({
      type: 'SERVER_ERROR',
      message: 'Internal server error',
      action: 'show_generic_error'
    });
  });
});
```

### Testing Network Errors
```javascript
// NetworkErrorHandler.test.js
import { handleNetworkError } from './NetworkErrorHandler';

describe('handleNetworkError', () => {
  it('should handle timeout error', () => {
    const error = {
      request: {},
      code: 'ECONNABORTED',
      message: 'Timeout of 5000ms exceeded'
    };

    const result = handleNetworkError(error);

    expect(result).toEqual({
      type: 'TIMEOUT_ERROR',
      message: 'Request timed out',
      action: 'retry'
    });
  });

  it('should handle connection refused error', () => {
    const error = {
      request: {},
      code: 'ECONNREFUSED',
      message: 'Connection refused'
    };

    const result = handleNetworkError(error);

    expect(result).toEqual({
      type: 'CONNECTION_ERROR',
      message: 'Unable to connect to server',
      action: 'check_network'
    });
  });

  it('should handle DNS resolution error', () => {
    const error = {
      request: {},
      code: 'ENOTFOUND',
      message: 'getaddrinfo ENOTFOUND api.example.com'
    };

    const result = handleNetworkError(error);

    expect(result).toEqual({
      type: 'CONNECTION_ERROR',
      message: 'Unable to connect to server',
      action: 'check_network'
    });
  });

  it('should return null for non-network errors', () => {
    const error = {
      response: { status: 400 }
    };

    const result = handleNetworkError(error);

    expect(result).toBe(null);
  });
});
```

### Testing Validation Errors
```javascript
// ValidationErrorHandler.test.js
import { handleValidationError } from './ValidationErrorHandler';

describe('handleValidationError', () => {
  it('should handle 400 validation error', () => {
    const error = {
      response: {
        status: 400,
        data: {
          errors: {
            email: 'Invalid email format',
            password: ['Too short', 'Must contain number']
          }
        }
      }
    };

    const result = handleValidationError(error);

    expect(result).toEqual({
      type: 'VALIDATION_ERROR',
      message: 'Please correct the following errors',
      errors: {
        email: 'Invalid email format',
        password: 'Too short' // Takes first error
      },
      action: 'show_field_errors'
    });
  });

  it('should handle 422 validation error', () => {
    const error = {
      response: {
        status: 422,
        data: {
          errors: {
            username: 'Username already taken'
          }
        }
      }
    };

    const result = handleValidationError(error);

    expect(result).toEqual({
      type: 'VALIDATION_ERROR',
      message: 'Please correct the following errors',
      errors: {
        username: 'Username already taken'
      },
      action: 'show_field_errors'
    });
  });

  it('should return null for non-validation errors', () => {
    const error = {
      response: {
        status: 500,
        data: { message: 'Server error' }
      }
    };

    const result = handleValidationError(error);

    expect(result).toBe(null);
  });
});
```

## Component Error Handling Tests

### Component with Error Handling
```javascript
// UserForm.js
import React, { useState } from 'react';
import axios from 'axios';
import { handleApiError } from './ApiErrorHandler';

export const UserForm = () => {
  const [formData, setFormData] = useState({ name: '', email: '' });
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [submitError, setSubmitError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    setErrors({});
    setSubmitError(null);

    try {
      await axios.post('/api/users', formData);
      // Success handling
    } catch (error) {
      const errorInfo = handleApiError(error);
      
      if (errorInfo.type === 'VALIDATION_ERROR') {
        setErrors(errorInfo.details || {});
      } else {
        setSubmitError(errorInfo.message);
      }
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          type="text"
          placeholder="Name"
          value={formData.name}
          onChange={(e) => setFormData({...formData, name: e.target.value})}
        />
        {errors.name && <span className="error">{errors.name}</span>}
      </div>
      
      <div>
        <input
          type="email"
          placeholder="Email"
          value={formData.email}
          onChange={(e) => setFormData({...formData, email: e.target.value})}
        />
        {errors.email && <span className="error">{errors.email}</span>}
      </div>
      
      {submitError && <div className="error">{submitError}</div>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
};
```

```javascript
// UserForm.test.js
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import axios from 'axios';
import { UserForm } from './UserForm';

jest.mock('axios');
const mockedAxios = axios;

describe('UserForm', () => {
  beforeEach(() => {
    mockedAxios.post.mockClear();
  });

  it('should submit form successfully', async () => {
    mockedAxios.post.mockResolvedValueOnce({ data: { id: 1 } });

    render(<UserForm />);

    fireEvent.change(screen.getByPlaceholderText('Name'), {
      target: { value: 'John Doe' }
    });
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'john@test.com' }
    });

    fireEvent.click(screen.getByText('Submit'));

    await waitFor(() => {
      expect(mockedAxios.post).toHaveBeenCalledWith('/api/users', {
        name: 'John Doe',
        email: 'john@test.com'
      });
    });
  });

  it('should display validation errors', async () => {
    mockedAxios.post.mockRejectedValueOnce({
      response: {
        status: 400,
        data: {
          errors: {
            email: 'Invalid email format',
            name: 'Name is required'
          }
        }
      }
    });

    render(<UserForm />);

    fireEvent.click(screen.getByText('Submit'));

    await waitFor(() => {
      expect(screen.getByText('Invalid email format')).toBeInTheDocument();
      expect(screen.getByText('Name is required')).toBeInTheDocument();
    });
  });

  it('should display authentication error', async () => {
    mockedAxios.post.mockRejectedValueOnce({
      response: {
        status: 401,
        data: { message: 'Unauthorized' }
      }
    });

    render(<UserForm />);

    fireEvent.click(screen.getByText('Submit'));

    await waitFor(() => {
      expect(screen.getByText('Authentication required')).toBeInTheDocument();
    });
  });

  it('should display network error', async () => {
    mockedAxios.post.mockRejectedValueOnce({
      request: {},
      message: 'Network Error'
    });

    render(<UserForm />);

    fireEvent.click(screen.getByText('Submit'));

    await waitFor(() => {
      expect(screen.getByText('Network connection failed')).toBeInTheDocument();
    });
  });

  it('should show loading state during submission', async () => {
    mockedAxios.post.mockResolvedValueOnce({ data: { id: 1 } });

    render(<UserForm />);

    const submitButton = screen.getByText('Submit');
    fireEvent.click(submitButton);

    expect(screen.getByText('Submitting...')).toBeInTheDocument();

    await waitFor(() => {
      expect(screen.getByText('Submit')).toBeInTheDocument();
    });
  });
});
```

## Error Recovery Strategies

### Retry Logic
```javascript
// RetryHandler.js
export const withRetry = async (fn, maxRetries = 3, delay = 1000) => {
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      // Don't retry on client errors (4xx)
      if (error.response?.status >= 400 && error.response?.status < 500) {
        throw error;
      }

      // Don't retry on the last attempt
      if (attempt === maxRetries) {
        throw error;
      }

      // Wait before retrying
      await new Promise(resolve => setTimeout(resolve, delay * attempt));
    }
  }

  throw lastError;
};
```

```javascript
// RetryHandler.test.js
import { withRetry } from './RetryHandler';

describe('withRetry', () => {
  it('should return result on first attempt', async () => {
    const mockFn = jest.fn().mockResolvedValue('success');

    const result = await withRetry(mockFn);

    expect(result).toBe('success');
    expect(mockFn).toHaveBeenCalledTimes(1);
  });

  it('should retry on network errors', async () => {
    const mockFn = jest.fn()
      .mockRejectedValueOnce({ request: {} }) // Network error
      .mockResolvedValueOnce('success');

    const result = await withRetry(mockFn, 2);

    expect(result).toBe('success');
    expect(mockFn).toHaveBeenCalledTimes(2);
  });

  it('should not retry on client errors', async () => {
    const mockFn = jest.fn().mockRejectedValue({
      response: { status: 400 }
    });

    await expect(withRetry(mockFn, 3)).rejects.toThrow();
    expect(mockFn).toHaveBeenCalledTimes(1);
  });

  it('should throw after max retries', async () => {
    const mockFn = jest.fn().mockRejectedValue(new Error('Persistent error'));

    await expect(withRetry(mockFn, 2)).rejects.toThrow('Persistent error');
    expect(mockFn).toHaveBeenCalledTimes(2);
  });
});
```

## Best Practices

1. **Categorize errors**: Different handling for different error types
2. **User-friendly messages**: Translate technical errors to user language
3. **Recovery actions**: Provide ways for users to recover from errors
4. **Logging**: Log errors for debugging while showing user-friendly messages
5. **Fallbacks**: Provide fallback content when APIs fail
6. **Retry logic**: Implement smart retry for transient errors
7. **Loading states**: Show appropriate loading indicators during retries

## Interview Tips
- **HTTP status codes**: Different handling for 4xx vs 5xx errors
- **Network errors**: Handle connection issues separately
- **Validation errors**: Show field-specific error messages
- **Retry logic**: Smart retry for transient failures
- **User experience**: User-friendly error messages
- **Error boundaries**: Prevent app crashes from errors
- **Testing**: Test all error scenarios thoroughly