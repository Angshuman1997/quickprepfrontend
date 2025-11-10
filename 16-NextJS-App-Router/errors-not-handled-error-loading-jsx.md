# Which errors are not handled by error.jsx and loading.jsx

## Question
Which errors are not handled by error.jsx and loading.jsx

## Answer

`error.jsx` and `loading.jsx` only handle **rendering errors** and **server-side loading states**. They do NOT handle event handler errors and client-side async errors, which must be handled manually with try/catch.

## What error.jsx Does NOT Catch

### 1. Event Handler Errors

```javascript
// ❌ NOT caught by error.jsx
function Component() {
  const handleClick = () => {
    throw new Error("Button failed"); // error.jsx will NOT catch this
  };

  return <button onClick={handleClick}>Click</button>;
}
```

**Solution: Manual try/catch**

```javascript
// ✅ Correct way to handle event handler errors
function Component() {
  const [error, setError] = useState(null);

  const handleClick = () => {
    try {
      throw new Error("Button failed");
    } catch (err) {
      console.error(err);
      setError("Action failed. Please try again.");
    }
  };

  return (
    <div>
      {error && <div className="error">{error}</div>}
      <button onClick={handleClick}>Click</button>
    </div>
  );
}
```

### 2. Async Errors in Event Handlers

```javascript
// ❌ NOT caught by error.jsx
function Component() {
  const handleAsyncAction = async () => {
    // These errors won't be caught by error.jsx
    const response = await fetch('/api/data');
    throw new Error("Async error in event handler");
  };

  return <button onClick={handleAsyncAction}>Async Action</button>;
}
```

**Solution: Try/catch with state**

```javascript
// ✅ Correct way to handle async event handler errors
function Component() {
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);

  const handleAsyncAction = async () => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('/api/data');
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
      const data = await response.json();
      // Handle success
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      {error && <div className="error">{error}</div>}
      <button onClick={handleAsyncAction} disabled={loading}>
        {loading ? 'Loading...' : 'Async Action'}
      </button>
    </div>
  );
}
```

### 3. Client-Side API Calls

```javascript
// ❌ NOT caught by error.jsx  
function UserProfile() {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // This error won't be caught by error.jsx
    fetch('/api/user')
      .then(res => {
        if (!res.ok) throw new Error('Failed to fetch user');
        return res.json();
      })
      .then(setUser)
      .catch(err => {
        // Must handle error here manually
        console.error(err);
      });
  }, []);

  return <div>{user?.name}</div>;
}
```

**Solution: Proper error state management**

```javascript
// ✅ Correct way to handle client-side API errors
function UserProfile() {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchUser = async () => {
      try {
        const response = await fetch('/api/user');
        if (!response.ok) {
          throw new Error(`Failed to fetch user: ${response.status}`);
        }
        const userData = await response.json();
        setUser(userData);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
  }, []);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>No user found</div>;

  return <div>Welcome, {user.name}!</div>;
}
```

## What loading.jsx Does NOT Cover

### 1. Client-Side Loading States

```javascript
// ❌ loading.jsx won't show for client-side loading
function ProductList() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(false);

  const fetchProducts = async () => {
    setLoading(true); // Must manage this manually
    try {
      const response = await fetch('/api/products');
      const data = await response.json();
      setProducts(data);
    } catch (error) {
      // Handle error
    } finally {
      setLoading(false); // Must manage this manually
    }
  };

  return (
    <div>
      {loading && <div>Loading products...</div>}
      <button onClick={fetchProducts}>Load Products</button>
      {products.map(product => (
        <div key={product.id}>{product.name}</div>
      ))}
    </div>
  );
}
```

### 2. Form Submission Loading

```javascript
// ❌ loading.jsx won't show during form submission
function ContactForm() {
  const [isSubmitting, setIsSubmitting] = useState(false); // Manual state

  const handleSubmit = async (event) => {
    event.preventDefault();
    setIsSubmitting(true); // Manual loading state
    
    try {
      await fetch('/api/contact', {
        method: 'POST',
        body: new FormData(event.target)
      });
      alert('Message sent!');
    } catch (error) {
      alert('Failed to send message');
    } finally {
      setIsSubmitting(false); // Manual loading state
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" required />
      <textarea name="message" required></textarea>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Sending...' : 'Send Message'}
      </button>
    </form>
  );
}
```

## Real-World Examples

### 1. E-commerce Cart Component

```javascript
function ShoppingCart() {
  const [items, setItems] = useState([]);
  const [error, setError] = useState(null);
  const [updating, setUpdating] = useState(false);

  // Add item to cart - NOT handled by error.jsx/loading.jsx
  const addToCart = async (productId) => {
    setUpdating(true);
    setError(null);
    
    try {
      const response = await fetch('/api/cart/add', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ productId })
      });
      
      if (!response.ok) {
        throw new Error('Failed to add item to cart');
      }
      
      const updatedCart = await response.json();
      setItems(updatedCart.items);
    } catch (err) {
      setError('Could not add item to cart. Please try again.');
    } finally {
      setUpdating(false);
    }
  };

  // Remove item - NOT handled by error.jsx/loading.jsx  
  const removeFromCart = async (itemId) => {
    try {
      await fetch(`/api/cart/remove/${itemId}`, { method: 'DELETE' });
      setItems(items.filter(item => item.id !== itemId));
    } catch (err) {
      setError('Could not remove item. Please try again.');
    }
  };

  return (
    <div className="cart">
      {error && (
        <div className="error">
          {error}
          <button onClick={() => setError(null)}>×</button>
        </div>
      )}
      
      {updating && <div className="loading">Updating cart...</div>}
      
      {items.map(item => (
        <div key={item.id} className="cart-item">
          <span>{item.name}</span>
          <button onClick={() => removeFromCart(item.id)}>
            Remove
          </button>
        </div>
      ))}
    </div>
  );
}
```

### 2. Search Component

```javascript
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [searching, setSearching] = useState(false);
  const [error, setError] = useState(null);

  // Search action - NOT handled by error.jsx/loading.jsx
  const handleSearch = async (searchTerm) => {
    if (!searchTerm.trim()) return;
    
    setSearching(true);
    setError(null);
    
    try {
      const response = await fetch(`/api/search?q=${encodeURIComponent(searchTerm)}`);
      
      if (!response.ok) {
        throw new Error('Search request failed');
      }
      
      const data = await response.json();
      setResults(data.results);
    } catch (err) {
      setError('Search failed. Please try again.');
      setResults([]);
    } finally {
      setSearching(false);
    }
  };

  // Debounced search
  useEffect(() => {
    const timeoutId = setTimeout(() => {
      if (query) handleSearch(query);
    }, 300);
    
    return () => clearTimeout(timeoutId);
  }, [query]);

  return (
    <div className="search">
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      
      {searching && <div className="searching">Searching...</div>}
      {error && <div className="error">{error}</div>}
      
      <div className="results">
        {results.map(result => (
          <div key={result.id} className="result">
            {result.title}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 3. File Upload Component

```javascript
function FileUpload() {
  const [uploading, setUploading] = useState(false);
  const [error, setError] = useState(null);
  const [progress, setProgress] = useState(0);

  // File upload - NOT handled by error.jsx/loading.jsx
  const handleFileUpload = async (event) => {
    const file = event.target.files[0];
    if (!file) return;

    setUploading(true);
    setError(null);
    setProgress(0);

    const formData = new FormData();
    formData.append('file', file);

    try {
      const response = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
        // Monitor upload progress
        onUploadProgress: (progressEvent) => {
          const percentCompleted = Math.round(
            (progressEvent.loaded * 100) / progressEvent.total
          );
          setProgress(percentCompleted);
        }
      });

      if (!response.ok) {
        throw new Error(`Upload failed: ${response.status}`);
      }

      const result = await response.json();
      alert('File uploaded successfully!');
    } catch (err) {
      setError('Upload failed. Please try again.');
    } finally {
      setUploading(false);
      setProgress(0);
    }
  };

  return (
    <div className="file-upload">
      <input 
        type="file" 
        onChange={handleFileUpload} 
        disabled={uploading}
      />
      
      {uploading && (
        <div className="upload-progress">
          <div className="progress-bar">
            <div 
              className="progress-fill" 
              style={{ width: `${progress}%` }}
            ></div>
          </div>
          <span>{progress}%</span>
        </div>
      )}
      
      {error && <div className="error">{error}</div>}
    </div>
  );
}
```

## Error Handling Patterns

### 1. Custom Error Hook

```javascript
// hooks/useErrorHandler.js
function useErrorHandler() {
  const [error, setError] = useState(null);
  
  const handleError = useCallback((error) => {
    console.error(error);
    setError(error.message);
  }, []);
  
  const clearError = useCallback(() => {
    setError(null);
  }, []);
  
  return { error, handleError, clearError };
}

// Usage
function MyComponent() {
  const { error, handleError, clearError } = useErrorHandler();
  
  const riskyAction = async () => {
    try {
      await fetch('/api/risky');
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
      <button onClick={riskyAction}>Do Something</button>
    </div>
  );
}
```

### 2. Global Error Context

```javascript
// context/ErrorContext.js
const ErrorContext = createContext();

export function ErrorProvider({ children }) {
  const [errors, setErrors] = useState([]);
  
  const addError = (error) => {
    const id = Date.now();
    setErrors(prev => [...prev, { id, message: error.message }]);
    
    // Auto-remove after 5 seconds
    setTimeout(() => {
      setErrors(prev => prev.filter(e => e.id !== id));
    }, 5000);
  };
  
  const removeError = (id) => {
    setErrors(prev => prev.filter(e => e.id !== id));
  };
  
  return (
    <ErrorContext.Provider value={{ errors, addError, removeError }}>
      {children}
      <ErrorDisplay errors={errors} onRemove={removeError} />
    </ErrorContext.Provider>
  );
}

// Usage in components
function SomeComponent() {
  const { addError } = useContext(ErrorContext);
  
  const handleAction = async () => {
    try {
      await riskyOperation();
    } catch (error) {
      addError(error);
    }
  };
  
  return <button onClick={handleAction}>Action</button>;
}
```

## Quick Reference

### What error.jsx DOES catch:
- ✅ Rendering errors during component lifecycle
- ✅ Errors in Server Components during data fetching  
- ✅ Errors thrown in page.jsx during SSR

### What error.jsx does NOT catch:
- ❌ Event handler errors (onClick, onSubmit, etc.)
- ❌ Async errors in useEffect
- ❌ Promise rejections in event handlers
- ❌ Client-side API call errors

### What loading.jsx DOES show:
- ✅ Server-side data fetching in page.jsx
- ✅ Navigation between routes
- ✅ Suspense boundaries resolving

### What loading.jsx does NOT show:
- ❌ Client-side API calls in useEffect
- ❌ Form submission loading
- ❌ Button click loading states
- ❌ File upload progress

## Interview Tips

- **Key distinction**: error.jsx/loading.jsx only handle **rendering/navigation**, not **user interactions**
- **Manual handling required**: Event handlers and client-side async operations need try/catch
- **State management**: Use useState for loading/error states in client components
- **User feedback**: Always show loading states and error messages for user actions
- **Error boundaries vs try/catch**: Know when to use each approach
- **Progressive enhancement**: Server-side loading/error handling + client-side interaction handling