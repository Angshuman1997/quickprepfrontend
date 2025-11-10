# How to handle errors in React (simple and direct)

## Question
How to handle errors in React (simple and direct)

## Answer

There are three kinds of errors in React and each is handled differently:

1. **Rendering errors**: use Error Boundaries
2. **Event handler errors**: use try/catch inside the event handler  
3. **Async or API errors**: use try/catch around fetch or axios and show fallback UI

## 1. Rendering Errors - Error Boundaries

```javascript
// ErrorBoundary.jsx
import React from "react";

class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  componentDidCatch(error, info) {
    console.error(error, info);
  }

  render() {
    if (this.state.hasError) {
      return <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}

export default ErrorBoundary;
```

**Usage:**
```jsx
<ErrorBoundary>
  <Dashboard />
</ErrorBoundary>
```

## 2. Event Handler Errors - Try/Catch

```javascript
const handleClick = () => {
  try {
    riskyAction();
  } catch (err) {
    console.error(err);
    setErrorMessage("Action failed. Please try again.");
  }
};

const handleSubmit = async (event) => {
  event.preventDefault();
  try {
    await submitForm(formData);
    setSuccess(true);
  } catch (error) {
    setError("Submission failed");
  }
};
```

## 3. Async/API Errors - Try/Catch with State

```javascript
const [users, setUsers] = useState([]);
const [error, setError] = useState(null);
const [loading, setLoading] = useState(false);

async function loadUsers() {
  setLoading(true);
  setError(null);
  
  try {
    const res = await fetch("/api/users");
    if (!res.ok) throw new Error("Failed to load users");
    
    const data = await res.json();
    setUsers(data);
  } catch (err) {
    setError("Could not load users");
  } finally {
    setLoading(false);
  }
}

// In JSX
{error && <div className="error">{error}</div>}
{loading && <div>Loading...</div>}
{users.length > 0 && <UserList users={users} />}
```

## Real-World Example

```javascript
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadUser();
  }, [userId]);

  const loadUser = async () => {
    try {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) {
        throw new Error("User not found");
      }
      const userData = await response.json();
      setUser(userData);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  const handleUpdateProfile = async (formData) => {
    try {
      const response = await fetch(`/api/users/${userId}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData)
      });
      
      if (!response.ok) throw new Error("Update failed");
      
      const updatedUser = await response.json();
      setUser(updatedUser);
      alert("Profile updated successfully!");
    } catch (err) {
      alert("Failed to update profile");
    }
  };

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>User not found</div>;

  return (
    <div>
      <h1>{user.name}</h1>
      <UserForm user={user} onSubmit={handleUpdateProfile} />
    </div>
  );
}
```

## Quick Reference

| Error Type | Solution | Where to Use |
|-----------|----------|-------------|
| Rendering | Error Boundary | Wrap components that might crash during render |
| Event Handler | try/catch in handler | Button clicks, form submissions, user actions |
| Async/API | try/catch with state | Data fetching, API calls, async operations |

## Interview Tips
- **Always show user feedback**: Don't just log errors, show users what happened
- **Use loading states**: Show users when operations are in progress  
- **Provide recovery options**: Let users retry failed operations
- **Error boundaries don't catch event handler or async errors**: Use try/catch for those
- **Test error scenarios**: Make sure your error handling actually works

## Common Patterns

```javascript
// Custom hook for error handling
function useErrorHandler() {
  const [error, setError] = useState(null);
  
  const handleError = (error) => {
    console.error(error);
    setError(error.message);
  };
  
  const clearError = () => setError(null);
  
  return { error, handleError, clearError };
}

// Usage
function MyComponent() {
  const { error, handleError, clearError } = useErrorHandler();
  
  const doSomething = async () => {
    try {
      await riskyOperation();
    } catch (err) {
      handleError(err);
    }
  };
  
  return (
    <div>
      {error && (
        <div className="error">
          {error}
          <button onClick={clearError}>×</button>
        </div>
      )}
      <button onClick={doSomething}>Do Something</button>
    </div>
  );
}
```