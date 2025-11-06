# Axios vs fetch comparison - when to use each?

## Question
Axios vs fetch comparison - when to use each?

## Answer

Axios and fetch are both popular HTTP client libraries for making API requests in JavaScript. While fetch is built into modern browsers, Axios is a third-party library. The choice between them depends on your project requirements, team preferences, and specific use cases.

## Core Differences

### 1. **API Design & Ease of Use**

**Fetch:**
```javascript
// Basic GET request
fetch('https://api.example.com/users')
    .then(response => {
        if (!response.ok) {
            throw new Error('Network response was not ok');
        }
        return response.json();
    })
    .then(data => console.log(data))
    .catch(error => console.error('Error:', error));

// POST request
fetch('https://api.example.com/users', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({ name: 'John', email: 'john@example.com' }),
})
    .then(response => response.json())
    .then(data => console.log(data))
    .catch(error => console.error('Error:', error));
```

**Axios:**
```javascript
// Basic GET request
axios.get('https://api.example.com/users')
    .then(response => console.log(response.data))
    .catch(error => console.error('Error:', error));

// POST request
axios.post('https://api.example.com/users', {
    name: 'John',
    email: 'john@example.com'
})
    .then(response => console.log(response.data))
    .catch(error => console.error('Error:', error));
```

**Key Differences:**
- **Fetch**: Requires manual JSON parsing and error checking
- **Axios**: Automatic JSON parsing and better error handling
- **Fetch**: Promise-based with manual response checking
- **Axios**: Returns data directly, handles errors more gracefully

### 2. **Error Handling**

**Fetch:**
```javascript
fetch('/api/users')
    .then(response => {
        if (!response.ok) {
            // Manual error checking required
            if (response.status === 404) {
                throw new Error('User not found');
            }
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        return response.json();
    })
    .catch(error => {
        // Handles both network errors and manual throws
        console.error('Error:', error);
    });
```

**Axios:**
```javascript
axios.get('/api/users')
    .then(response => {
        // Success: status codes 200-299
        console.log(response.data);
    })
    .catch(error => {
        if (error.response) {
            // Server responded with error status (4xx, 5xx)
            console.error('Response error:', error.response.status, error.response.data);
        } else if (error.request) {
            // Network error (no response received)
            console.error('Network error:', error.request);
        } else {
            // Other error
            console.error('Error:', error.message);
        }
    });
```

**Error Handling Comparison:**
- **Fetch**: Only rejects on network errors; HTTP errors (4xx, 5xx) are successful responses
- **Axios**: Rejects on both network errors and HTTP error status codes
- **Axios**: Provides structured error object with `error.response`, `error.request`, etc.

### 3. **Request/Response Interceptors**

**Axios (Built-in):**
```javascript
// Request interceptor
axios.interceptors.request.use(
    config => {
        // Add auth token
        config.headers.Authorization = `Bearer ${token}`;
        return config;
    },
    error => Promise.reject(error)
);

// Response interceptor
axios.interceptors.response.use(
    response => response,
    error => {
        if (error.response?.status === 401) {
            // Handle unauthorized
            redirectToLogin();
        }
        return Promise.reject(error);
    }
);
```

**Fetch (Manual Implementation):**
```javascript
// Custom wrapper function
const authenticatedFetch = (url, options = {}) => {
    const token = localStorage.getItem('token');

    return fetch(url, {
        ...options,
        headers: {
            ...options.headers,
            'Authorization': token ? `Bearer ${token}` : undefined,
        },
    })
    .then(response => {
        if (response.status === 401) {
            redirectToLogin();
            throw new Error('Unauthorized');
        }
        return response;
    });
};
```

**Interceptors Comparison:**
- **Axios**: Built-in interceptor system for global request/response handling
- **Fetch**: Requires manual wrapper functions or custom implementations
- **Axios**: Easier to add cross-cutting concerns (auth, logging, etc.)

### 4. **Configuration & Defaults**

**Axios:**
```javascript
// Create configured instance
const apiClient = axios.create({
    baseURL: 'https://api.example.com',
    timeout: 5000,
    headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
    },
});

// All requests use this configuration
apiClient.get('/users'); // https://api.example.com/users
```

**Fetch:**
```javascript
// Manual configuration for each request
const apiConfig = {
    baseURL: 'https://api.example.com',
    headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${token}`,
    },
};

const customFetch = (endpoint, options = {}) => {
    return fetch(`${apiConfig.baseURL}${endpoint}`, {
        ...options,
        headers: {
            ...apiConfig.headers,
            ...options.headers,
        },
    });
};
```

**Configuration Comparison:**
- **Axios**: Easy instance creation with defaults
- **Fetch**: Requires custom wrapper functions
- **Axios**: Better for complex applications with multiple API configurations

### 5. **Advanced Features**

| Feature | Axios | Fetch |
|---------|-------|-------|
| Request cancellation | ✅ (CancelToken) | ✅ (AbortController) |
| Timeout | ✅ | ✅ (AbortController) |
| Automatic JSON parsing | ✅ | ❌ |
| HTTP interceptors | ✅ | ❌ |
| Request progress | ✅ | ✅ (Modern browsers) |
| Automatic retries | ❌ (needs library) | ❌ (needs library) |
| CSRF protection | ✅ | ❌ |
| URL params serialization | ✅ | ❌ |

## Performance Comparison

### 1. **Bundle Size**

- **Axios**: ~50KB minified (larger bundle)
- **Fetch**: 0KB (built into browsers)
- **Consideration**: If bundle size is critical, fetch might be preferable

### 2. **Runtime Performance**

```javascript
// Axios handles JSON parsing automatically
axios.get('/api/data')
    .then(response => {
        // response.data is already parsed
        console.log(response.data);
    });

// Fetch requires manual parsing
fetch('/api/data')
    .then(response => response.json())
    .then(data => {
        console.log(data);
    });
```

**Performance Notes:**
- **Fetch**: Slightly faster for simple requests (no extra parsing)
- **Axios**: Better for JSON APIs (automatic parsing)
- **Real-world**: Performance difference is negligible for most applications

### 3. **Memory Usage**

- **Axios**: Slightly higher memory usage due to additional features
- **Fetch**: Lower memory footprint
- **Consideration**: Not significant for most web applications

## Browser Support

### Fetch Browser Support:
- ✅ Chrome 42+
- ✅ Firefox 39+
- ✅ Safari 10.1+
- ✅ Edge 14+
- ❌ IE 11 and below (requires polyfill)

### Axios Browser Support:
- ✅ All modern browsers
- ✅ IE 11+ (with polyfills)
- ✅ Node.js (server-side)

**Browser Support Notes:**
- **Fetch**: Requires polyfill for older browsers
- **Axios**: Works in all environments with broader support

## When to Use Each

### **Use Axios when:**

1. **Working with JSON APIs extensively**
   - Automatic JSON parsing
   - Better error handling for HTTP status codes

2. **Need advanced features**
   - Request/response interceptors
   - Request cancellation
   - Timeout handling
   - CSRF protection

3. **Building complex applications**
   - Multiple API configurations
   - Global error handling
   - Authentication management

4. **Team prefers promise-based APIs**
   - Cleaner syntax
   - Consistent error handling

5. **Server-side applications**
   - Works in Node.js
   - Better for isomorphic apps

### **Use Fetch when:**

1. **Bundle size is critical**
   - No additional dependencies
   - Smaller footprint

2. **Simple use cases**
   - Basic GET/POST requests
   - No complex error handling needed

3. **Modern browser-only applications**
   - No need for IE11 support
   - Can use modern Web APIs

4. **Need low-level control**
   - Direct access to Response object
   - Custom response handling

5. **Already using modern Web APIs**
   - Service Workers, Streams, etc.

## Migration Examples

### Migrating from Fetch to Axios:

```javascript
// Before (Fetch)
const getUsers = async () => {
    try {
        const response = await fetch('/api/users');
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        const data = await response.json();
        return data;
    } catch (error) {
        console.error('Error fetching users:', error);
        throw error;
    }
};

// After (Axios)
const getUsers = async () => {
    try {
        const response = await axios.get('/api/users');
        return response.data;
    } catch (error) {
        console.error('Error fetching users:', error);
        throw error;
    }
};
```

### Migrating from Axios to Fetch:

```javascript
// Before (Axios)
const createUser = async (userData) => {
    try {
        const response = await axios.post('/api/users', userData);
        return response.data;
    } catch (error) {
        if (error.response) {
            throw new Error(`HTTP ${error.response.status}: ${error.response.data.message}`);
        } else {
            throw error;
        }
    }
};

// After (Fetch)
const createUser = async (userData) => {
    try {
        const response = await fetch('/api/users', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(userData),
        });

        if (!response.ok) {
            const errorData = await response.json();
            throw new Error(`HTTP ${response.status}: ${errorData.message}`);
        }

        return await response.json();
    } catch (error) {
        console.error('Error creating user:', error);
        throw error;
    }
};
```

## Real-World Scenarios

### **E-commerce Application**

```javascript
// Axios - Better for complex API interactions
const apiClient = axios.create({
    baseURL: '/api',
    timeout: 10000,
});

apiClient.interceptors.request.use(config => {
    config.headers.Authorization = `Bearer ${getAuthToken()}`;
    return config;
});

const getProducts = (category) => apiClient.get(`/products?category=${category}`);
const addToCart = (productId) => apiClient.post('/cart', { productId });
const checkout = (orderData) => apiClient.post('/orders', orderData);
```

### **Real-time Dashboard**

```javascript
// Fetch - Better for simple, frequent requests
const fetchMetrics = () =>
    fetch('/api/metrics')
        .then(response => response.json())
        .catch(error => {
            console.error('Failed to fetch metrics:', error);
            return { error: true };
        });

// Update every 30 seconds
setInterval(fetchMetrics, 30000);
```

### **File Upload Application**

```javascript
// Axios - Better for progress tracking
const uploadFile = (file) => {
    const formData = new FormData();
    formData.append('file', file);

    return axios.post('/api/upload', formData, {
        onUploadProgress: (progressEvent) => {
            const percentCompleted = Math.round(
                (progressEvent.loaded * 100) / progressEvent.total
            );
            console.log(`Upload progress: ${percentCompleted}%`);
        },
    });
};
```

## Testing Considerations

### **Axios Testing:**
```javascript
import axios from 'axios';
import MockAdapter from 'axios-mock-adapter';

const mock = new MockAdapter(axios);
mock.onGet('/users').reply(200, [{ id: 1, name: 'John' }]);

// Test
axios.get('/users').then(response => {
    expect(response.data).toEqual([{ id: 1, name: 'John' }]);
});
```

### **Fetch Testing:**
```javascript
global.fetch = jest.fn(() =>
    Promise.resolve({
        ok: true,
        json: () => Promise.resolve([{ id: 1, name: 'John' }]),
    })
);

// Test
fetch('/users').then(response => {
    expect(response.ok).toBe(true);
    return response.json();
}).then(data => {
    expect(data).toEqual([{ id: 1, name: 'John' }]);
});
```

## Common Interview Questions

### Q: When would you choose Axios over fetch?

**A:** I'd choose Axios for complex applications needing interceptors, automatic JSON parsing, better error handling, and request cancellation. It's better for JSON APIs and when you need consistent configuration across requests.

### Q: Can fetch do everything Axios can do?

**A:** Yes, but with more boilerplate code. Fetch requires manual JSON parsing, error checking, and custom wrapper functions for features that Axios provides built-in.

### Q: What's the main advantage of fetch over Axios?

**A:** Fetch has zero bundle size impact since it's built into browsers, and it provides lower-level control over HTTP requests and responses.

### Q: How do you handle request cancellation?

**A:** In Axios, use `CancelToken`. In fetch, use `AbortController`. Both allow cancelling requests to prevent state updates on unmounted components.

### Q: Which one is better for server-side rendering?

**A:** Axios works better in Node.js environments and is more suitable for server-side rendering since it has consistent behavior across browser and server.

## Summary

**Choose Axios when:**
- Building complex applications with JSON APIs
- Need interceptors, cancellation, or advanced features
- Prefer cleaner, more declarative syntax
- Team is familiar with promise-based HTTP clients

**Choose Fetch when:**
- Bundle size is a major concern
- Need low-level control over requests
- Building simple applications with basic HTTP needs
- Targeting modern browsers only

**Migration Tip:** "Start with fetch for simple projects, but migrate to Axios as your application grows and needs more sophisticated HTTP client features like interceptors and automatic error handling."