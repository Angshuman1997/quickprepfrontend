# How to handle different types of API errors?

## Simple Answer
Handle API errors by categorizing them (network, validation, server, auth) and showing appropriate user messages. Always test error scenarios to ensure good user experience.

## Common API Error Types

### 1. Network Errors (No connection)
```javascript
// Handle when user is offline or server unreachable
const handleNetworkError = (error) => {
  if (!error.response) {
    return {
      message: "Check your internet connection and try again",
      action: "retry"
    };
  }
};
```

### 2. Authentication Errors (401)
```javascript
// Handle when user needs to log in
const handleAuthError = (error) => {
  if (error.response?.status === 401) {
    // Redirect to login or refresh token
    redirectToLogin();
    return {
      message: "Please log in to continue",
      action: "login"
    };
  }
};
```

### 3. Validation Errors (400/422)
```javascript
// Handle form validation errors from server
const handleValidationError = (error) => {
  if (error.response?.status === 400 || error.response?.status === 422) {
    const fieldErrors = error.response.data.errors;
    return {
      message: "Please fix the errors below",
      fieldErrors: fieldErrors, // { email: "Invalid format", password: "Too short" }
      action: "show_field_errors"
    };
  }
};
```

### 4. Not Found Errors (404)
```javascript
// Handle when resource doesn't exist
const handleNotFoundError = (error) => {
  if (error.response?.status === 404) {
    return {
      message: "The item you're looking for doesn't exist",
      action: "show_not_found_page"
    };
  }
};
```

### 5. Server Errors (500)
```javascript
// Handle server-side issues
const handleServerError = (error) => {
  if (error.response?.status >= 500) {
    return {
      message: "Something went wrong on our end. Please try again later",
      action: "show_generic_error"
    };
  }
};
```

### 6. Rate Limiting (429)
```javascript
// Handle too many requests
const handleRateLimitError = (error) => {
  if (error.response?.status === 429) {
    const retryAfter = error.response.data.retryAfter;
    return {
      message: `Too many requests. Try again in ${retryAfter} seconds`,
      action: "retry_later",
      retryAfter
    };
  }
};
```

## Complete Error Handler

```javascript
const handleApiError = (error) => {
  // Network error
  if (!error.response) {
    return {
      type: 'network',
      message: 'Connection failed. Check your internet.',
      userMessage: 'Please check your connection and try again'
    };
  }

  const { status, data } = error.response;

  // Categorize by status code
  if (status === 400 || status === 422) {
    return {
      type: 'validation',
      message: data.message || 'Invalid data',
      fieldErrors: data.errors || {},
      userMessage: 'Please check your input and try again'
    };
  }

  if (status === 401) {
    return {
      type: 'auth',
      message: 'Not authenticated',
      userMessage: 'Please log in to continue',
      action: 'redirect_to_login'
    };
  }

  if (status === 403) {
    return {
      type: 'permission',
      message: 'Access denied',
      userMessage: 'You don\'t have permission for this action'
    };
  }

  if (status === 404) {
    return {
      type: 'not_found',
      message: 'Resource not found',
      userMessage: 'The item you\'re looking for doesn\'t exist'
    };
  }

  if (status === 429) {
    return {
      type: 'rate_limit',
      message: 'Too many requests',
      userMessage: 'Please wait a moment before trying again'
    };
  }

  if (status >= 500) {
    return {
      type: 'server',
      message: 'Server error',
      userMessage: 'Something went wrong. Please try again later'
    };
  }

  // Unknown error
  return {
    type: 'unknown',
    message: 'Unexpected error',
    userMessage: 'An unexpected error occurred. Please try again'
  };
};
```

## Testing Error Handling

```javascript
describe('API Error Handling', () => {
  it('handles validation errors', () => {
    const error = {
      response: {
        status: 400,
        data: { errors: { email: 'Invalid format' } }
      }
    };

    const result = handleApiError(error);
    expect(result.type).toBe('validation');
    expect(result.fieldErrors.email).toBe('Invalid format');
  });

  it('handles network errors', () => {
    const error = { request: {} }; // No response

    const result = handleApiError(error);
    expect(result.type).toBe('network');
    expect(result.userMessage).toContain('connection');
  });

  it('handles auth errors', () => {
    const error = { response: { status: 401 } };

    const result = handleApiError(error);
    expect(result.type).toBe('auth');
    expect(result.action).toBe('redirect_to_login');
  });
});
```

## React Component Example

```javascript
const UserForm = () => {
  const [errors, setErrors] = useState({});
  const [generalError, setGeneralError] = useState('');

  const handleSubmit = async (userData) => {
    try {
      await api.createUser(userData);
      // Success
    } catch (error) {
      const errorInfo = handleApiError(error);

      if (errorInfo.type === 'validation') {
        setErrors(errorInfo.fieldErrors);
      } else {
        setGeneralError(errorInfo.userMessage);
      }
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" />
      {errors.email && <span>{errors.email}</span>}

      <input name="password" />
      {errors.password && <span>{errors.password}</span>}

      {generalError && <div>{generalError}</div>}

      <button type="submit">Submit</button>
    </form>
  );
};
```

## Interview Q&A

**Q: How do you handle API errors in your React app?**
A: I create a centralized error handler that categorizes errors by type (network, validation, auth, server) and returns user-friendly messages. For validation errors, I show field-specific messages. For auth errors, I redirect to login. For server errors, I show a generic message with retry option.

**Q: What's the difference between 400 and 422 status codes?**
A: Both indicate validation issues, but 400 is for general bad requests while 422 specifically means the data is syntactically correct but fails business rules validation.

**Q: How do you test error scenarios?**
A: I mock axios responses with different status codes and error structures. I test that the right error messages are shown and the correct actions are taken (like redirects for auth errors).

**Q: Should you retry failed requests automatically?**
A: Only for certain errors like network issues or 5xx server errors. Don't retry 4xx client errors as they won't succeed. Implement exponential backoff to avoid overwhelming the server.

## Best Practices
- **User-friendly messages**: Never show raw error messages to users
- **Categorize errors**: Different handling for different error types
- **Retry logic**: Smart retry for transient failures only
- **Loading states**: Show feedback during retries
- **Logging**: Log technical details for debugging while showing user-friendly messages
- **Fallback UI**: Show cached data or offline mode when APIs fail</content>
<parameter name="filePath">c:\Users\Angshuman\Desktop\MyScripts\QuickReadyInterview\QuickPrepFrontend\08-Testing\handle-different-api-errors-simple.md