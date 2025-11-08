# Async/Await, Promises, and Promise Chaining

## Question
What is async/await? How does it relate to Promises and Promise chaining? Provide detailed examples and explain when to use each approach.

## Detailed Answer

Asynchronous programming is fundamental to JavaScript, especially in modern web development. Understanding Promises, async/await, and Promise chaining is crucial for handling API calls, file operations, and any I/O-bound operations. This guide covers the evolution from callback hell to modern async patterns, with practical examples and best practices.

### 1. **Understanding Asynchronous JavaScript**

JavaScript is single-threaded but needs to handle asynchronous operations like:
- API calls (fetch, XMLHttpRequest)
- File system operations
- Database queries
- Timers (setTimeout, setInterval)
- User interactions

**Blocking vs Non-blocking:**
```javascript
// Blocking (synchronous) - BAD for user experience
console.log('Start');
const result = someLongRunningTask(); // Blocks execution
console.log('End');

// Non-blocking (asynchronous) - GOOD
console.log('Start');
someAsyncTask().then(() => {
    console.log('End');
});
console.log('This runs immediately');
```

### 2. **Promises - The Foundation**

Promises represent the eventual completion (or failure) of an asynchronous operation and its resulting value.

**Promise States:**
- **Pending**: Initial state, neither fulfilled nor rejected
- **Fulfilled**: Operation completed successfully
- **Rejected**: Operation failed

**Basic Promise Creation:**
```javascript
const promise = new Promise((resolve, reject) => {
    // Asynchronous operation
    setTimeout(() => {
        const success = Math.random() > 0.5;
        if (success) {
            resolve('Operation successful!');
        } else {
            reject(new Error('Operation failed!'));
        }
    }, 1000);
});

// Using the promise
promise
    .then(result => console.log(result))
    .catch(error => console.error(error));
```

**Promise Methods:**
```javascript
// Promise.resolve() - Create a resolved promise
const resolvedPromise = Promise.resolve('Immediate value');
const resolvedPromise2 = Promise.resolve(fetch('/api/data')); // Wraps existing promise

// Promise.reject() - Create a rejected promise
const rejectedPromise = Promise.reject(new Error('Something went wrong'));

// Promise.all() - Wait for all promises to resolve
const promises = [
    fetch('/api/users'),
    fetch('/api/posts'),
    fetch('/api/comments')
];

Promise.all(promises)
    .then(([users, posts, comments]) => {
        console.log('All data loaded:', { users, posts, comments });
    })
    .catch(error => {
        console.error('One of the requests failed:', error);
    });

// Promise.race() - First promise to settle wins
Promise.race([
    fetch('/api/fast-endpoint'),
    new Promise(resolve => setTimeout(resolve, 5000, 'Timeout'))
])
    .then(result => console.log('Fastest result:', result));

// Promise.allSettled() - Wait for all promises to settle (ES2020)
Promise.allSettled([
    Promise.resolve('Success'),
    Promise.reject('Error'),
    Promise.resolve('Another success')
])
    .then(results => {
        results.forEach((result, index) => {
            if (result.status === 'fulfilled') {
                console.log(`Promise ${index} fulfilled:`, result.value);
            } else {
                console.log(`Promise ${index} rejected:`, result.reason);
            }
        });
    });

// Promise.any() - First fulfilled promise wins (ES2021)
Promise.any([
    fetch('/api/primary'),
    fetch('/api/fallback'),
    fetch('/api/backup')
])
    .then(result => console.log('First successful result:', result))
    .catch(error => console.error('All promises rejected:', error));
```

### 3. **Promise Chaining**

Promise chaining allows sequential execution of asynchronous operations.

**Basic Chaining:**
```javascript
fetch('/api/user/123')
    .then(response => {
        if (!response.ok) {
            throw new Error('Network response was not ok');
        }
        return response.json();
    })
    .then(user => {
        console.log('User:', user);
        return fetch(`/api/user/${user.id}/posts`);
    })
    .then(response => response.json())
    .then(posts => {
        console.log('Posts:', posts);
        return posts.length;
    })
    .then(postCount => {
        console.log(`User has ${postCount} posts`);
    })
    .catch(error => {
        console.error('Error in chain:', error);
    })
    .finally(() => {
        console.log('Cleanup: hide loading spinner');
    });
```

**Chaining with Error Handling:**
```javascript
function getUserWithPosts(userId) {
    return fetch(`/api/user/${userId}`)
        .then(response => {
            if (!response.ok) {
                throw new Error(`HTTP error! status: ${response.status}`);
            }
            return response.json();
        })
        .then(user => {
            if (!user) {
                throw new Error('User not found');
            }
            return fetch(`/api/user/${user.id}/posts`)
                .then(response => response.json())
                .then(posts => ({
                    ...user,
                    posts: posts || []
                }));
        })
        .catch(error => {
            // Log error for debugging
            console.error('Error fetching user with posts:', error);
            // Return default value or re-throw
            throw error;
        });
}

// Usage
getUserWithPosts(123)
    .then(userWithPosts => {
        console.log('User with posts:', userWithPosts);
    })
    .catch(error => {
        // Handle error in calling code
        console.error('Failed to get user:', error);
    });
```

### 4. **Async/Await - Syntactic Sugar for Promises**

Async/await provides a more readable way to work with Promises, making asynchronous code look synchronous.

**Basic Syntax:**
```javascript
// Function declaration
async function getData() {
    // Function body
}

// Arrow function
const getData = async () => {
    // Function body
};

// Class method
class ApiService {
    async getData() {
        // Method body
    }
}
```

**Key Points:**
- `async` functions always return a Promise
- `await` can only be used inside `async` functions
- `await` pauses execution until the Promise resolves
- Use `try/catch` for error handling

**Basic Example:**
```javascript
async function getUserData(userId) {
    try {
        const response = await fetch(`/api/user/${userId}`);
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        const user = await response.json();
        return user;
    } catch (error) {
        console.error('Error fetching user:', error);
        throw error; // Re-throw to let caller handle
    }
}

// Usage
getUserData(123)
    .then(user => console.log('User:', user))
    .catch(error => console.error('Failed:', error));
```

**Sequential vs Parallel Execution:**
```javascript
// Sequential (one after another)
async function loadUserData() {
    try {
        const user = await fetch('/api/user').then(r => r.json());
        const posts = await fetch('/api/posts').then(r => r.json());
        const comments = await fetch('/api/comments').then(r => r.json());

        return { user, posts, comments };
    } catch (error) {
        console.error('Error loading data:', error);
    }
}

// Parallel (all at once) - MORE EFFICIENT
async function loadUserDataParallel() {
    try {
        const [userResponse, postsResponse, commentsResponse] = await Promise.all([
            fetch('/api/user'),
            fetch('/api/posts'),
            fetch('/api/comments')
        ]);

        const [user, posts, comments] = await Promise.all([
            userResponse.json(),
            postsResponse.json(),
            commentsResponse.json()
        ]);

        return { user, posts, comments };
    } catch (error) {
        console.error('Error loading data:', error);
    }
}
```

### 5. **Error Handling Patterns**

**With Promises:**
```javascript
function fetchWithRetry(url, maxRetries = 3) {
    return fetch(url)
        .then(response => {
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }
            return response.json();
        })
        .catch(error => {
            if (maxRetries > 0) {
                console.log(`Retrying... ${maxRetries} attempts left`);
                return new Promise(resolve => {
                    setTimeout(() => {
                        resolve(fetchWithRetry(url, maxRetries - 1));
                    }, 1000);
                });
            }
            throw error;
        });
}
```

**With Async/Await:**
```javascript
async function fetchWithRetry(url, maxRetries = 3) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            const response = await fetch(url);
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }
            return await response.json();
        } catch (error) {
            if (attempt === maxRetries) {
                throw error;
            }
            console.log(`Attempt ${attempt} failed, retrying...`);
            await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
        }
    }
}
```

**Advanced Error Handling:**
```javascript
class ApiError extends Error {
    constructor(message, status, data) {
        super(message);
        this.status = status;
        this.data = data;
        this.name = 'ApiError';
    }
}

async function apiCall(endpoint, options = {}) {
    try {
        const response = await fetch(endpoint, options);

        if (!response.ok) {
            let errorData;
            try {
                errorData = await response.json();
            } catch {
                errorData = { message: response.statusText };
            }

            throw new ApiError(
                `API call failed: ${response.statusText}`,
                response.status,
                errorData
            );
        }

        return await response.json();
    } catch (error) {
        if (error instanceof ApiError) {
            throw error; // Re-throw API errors
        }

        // Network errors, parsing errors, etc.
        throw new Error(`Network error: ${error.message}`);
    }
}
```

## Code Examples

### **Data Fetching Service (React)**
```javascript
import React, { useState, useEffect } from 'react';

// API service with error handling
class ApiService {
    async get(endpoint) {
        const response = await fetch(`/api${endpoint}`);
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        return response.json();
    }

    async post(endpoint, data) {
        const response = await fetch(`/api${endpoint}`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify(data),
        });
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        return response.json();
    }
}

const api = new ApiService();

function UserDashboard() {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        async function loadDashboardData() {
            try {
                setLoading(true);
                setError(null);

                // Load user and posts in parallel
                const [userData, postsData] = await Promise.all([
                    api.get('/user/profile'),
                    api.get('/user/posts')
                ]);

                setUser(userData);
                setPosts(postsData);
            } catch (err) {
                setError(err.message);
                console.error('Failed to load dashboard:', err);
            } finally {
                setLoading(false);
            }
        }

        loadDashboardData();
    }, []);

    const createPost = async (postData) => {
        try {
            const newPost = await api.post('/posts', postData);
            setPosts(prevPosts => [newPost, ...prevPosts]);
        } catch (err) {
            setError('Failed to create post');
            console.error('Post creation failed:', err);
        }
    };

    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error}</div>;

    return (
        <div>
            <h1>Welcome, {user?.name}!</h1>
            <div>
                <h2>Your Posts ({posts.length})</h2>
                {posts.map(post => (
                    <div key={post.id}>
                        <h3>{post.title}</h3>
                        <p>{post.content}</p>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

### **Custom Hook for API Calls**
```javascript
import { useState, useCallback } from 'react';

function useApi() {
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const execute = useCallback(async (asyncFn, ...args) => {
        try {
            setLoading(true);
            setError(null);
            const result = await asyncFn(...args);
            return result;
        } catch (err) {
            setError(err.message);
            throw err; // Re-throw so calling code can handle
        } finally {
            setLoading(false);
        }
    }, []);

    const resetError = useCallback(() => setError(null), []);

    return { execute, loading, error, resetError };
}

// Usage in component
function PostForm() {
    const { execute, loading, error } = useApi();
    const [title, setTitle] = useState('');
    const [content, setContent] = useState('');

    const handleSubmit = async (e) => {
        e.preventDefault();
        try {
            await execute(async () => {
                const response = await fetch('/api/posts', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ title, content }),
                });
                if (!response.ok) throw new Error('Failed to create post');
                return response.json();
            });
            // Success - reset form or show success message
            setTitle('');
            setContent('');
        } catch (err) {
            // Error already handled by hook
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            {error && <div className="error">{error}</div>}
            <input
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                placeholder="Post title"
                disabled={loading}
            />
            <textarea
                value={content}
                onChange={(e) => setContent(e.target.value)}
                placeholder="Post content"
                disabled={loading}
            />
            <button type="submit" disabled={loading}>
                {loading ? 'Creating...' : 'Create Post'}
            </button>
        </form>
    );
}
```

### **Complex Data Loading with Dependencies**
```javascript
async function loadUserProfile(userId) {
    // Step 1: Load basic user info
    const user = await fetch(`/api/users/${userId}`).then(r => r.json());

    // Step 2: Load related data in parallel
    const [posts, friends, settings] = await Promise.all([
        fetch(`/api/users/${userId}/posts`).then(r => r.json()),
        fetch(`/api/users/${userId}/friends`).then(r => r.json()),
        fetch(`/api/users/${userId}/settings`).then(r => r.json()),
    ]);

    // Step 3: Process and combine data
    const enrichedPosts = await Promise.all(
        posts.map(async (post) => {
            const comments = await fetch(`/api/posts/${post.id}/comments`).then(r => r.json());
            return { ...post, comments };
        })
    );

    return {
        ...user,
        posts: enrichedPosts,
        friends,
        settings,
    };
}

// Usage with error boundaries
function UserProfile({ userId }) {
    const [profile, setProfile] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        loadUserProfile(userId)
            .then(setProfile)
            .catch(setError)
            .finally(() => setLoading(false));
    }, [userId]);

    if (loading) return <div>Loading profile...</div>;
    if (error) return <div>Error: {error.message}</div>;

    return (
        <div>
            <h1>{profile.name}</h1>
            <p>Friends: {profile.friends.length}</p>
            <div>
                <h2>Posts</h2>
                {profile.posts.map(post => (
                    <div key={post.id}>
                        <h3>{post.title}</h3>
                        <p>{post.content}</p>
                        <small>{post.comments.length} comments</small>
                    </div>
                ))}
            </div>
        </div>
    );
}
```

## Performance Considerations

### **Promise vs Async/Await Performance**
- **No significant performance difference** - async/await is syntactic sugar
- **Memory**: Promises can be more memory-efficient for simple cases
- **Debugging**: Async/await provides better stack traces

### **Optimization Techniques**

**Concurrent vs Sequential Loading:**
```javascript
// BAD: Sequential loading (slower)
async function loadSlow() {
    const user = await fetch('/api/user');
    const posts = await fetch('/api/posts');
    const comments = await fetch('/api/comments');
}

// GOOD: Concurrent loading (faster)
async function loadFast() {
    const [user, posts, comments] = await Promise.all([
        fetch('/api/user'),
        fetch('/api/posts'),
        fetch('/api/comments')
    ]);
}
```

**Resource Management:**
```javascript
// Cleanup patterns
async function withTimeout(promise, timeoutMs) {
    const timeout = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), timeoutMs)
    );

    return Promise.race([promise, timeout]);
}

// AbortController for cancellation
async function fetchWithCancel(url, signal) {
    const response = await fetch(url, { signal });
    return response.json();
}

// Usage
const controller = new AbortController();
const signal = controller.signal;

fetchWithCancel('/api/data', signal)
    .then(data => console.log(data))
    .catch(err => {
        if (err.name === 'AbortError') {
            console.log('Request was cancelled');
        }
    });

// Cancel the request
controller.abort();
```

### **Common Pitfalls**

1. **Forgetting await:**
```javascript
// BUG: Missing await
async function bug() {
    const result = fetch('/api/data'); // Returns Promise, not data
    console.log(result); // Promise {<pending>}
}

// FIX: Add await
async function fix() {
    const result = await fetch('/api/data');
    console.log(result); // Response object
}
```

2. **Multiple awaits in loop:**
```javascript
// BAD: Sequential requests
async function bad() {
    const userIds = [1, 2, 3, 4, 5];
    const users = [];

    for (const id of userIds) {
        const user = await fetch(`/api/user/${id}`).then(r => r.json());
        users.push(user);
    }
}

// GOOD: Concurrent requests
async function good() {
    const userIds = [1, 2, 3, 4, 5];
    const userPromises = userIds.map(id =>
        fetch(`/api/user/${id}`).then(r => r.json())
    );
    const users = await Promise.all(userPromises);
}
```

3. **Unhandled promise rejections:**
```javascript
// BAD: Unhandled rejection
async function bad() {
    fetch('/api/data').then(data => {
        throw new Error('Something went wrong');
    });
    // Error is silently ignored
}

// GOOD: Proper error handling
async function good() {
    try {
        const data = await fetch('/api/data').then(r => r.json());
        // Handle data
    } catch (error) {
        console.error('Error:', error);
    }
}
```

## Common Patterns and Best Practices

### **API Service Layer**
```javascript
class HttpClient {
    constructor(baseURL = '') {
        this.baseURL = baseURL;
    }

    async request(endpoint, options = {}) {
        const url = `${this.baseURL}${endpoint}`;
        const config = {
            headers: {
                'Content-Type': 'application/json',
                ...options.headers,
            },
            ...options,
        };

        try {
            const response = await fetch(url, config);

            if (!response.ok) {
                const error = await response.text();
                throw new Error(`HTTP ${response.status}: ${error}`);
            }

            return await response.json();
        } catch (error) {
            console.error(`Request failed: ${url}`, error);
            throw error;
        }
    }

    get(endpoint) {
        return this.request(endpoint);
    }

    post(endpoint, data) {
        return this.request(endpoint, {
            method: 'POST',
            body: JSON.stringify(data),
        });
    }

    put(endpoint, data) {
        return this.request(endpoint, {
            method: 'PUT',
            body: JSON.stringify(data),
        });
    }

    delete(endpoint) {
        return this.request(endpoint, {
            method: 'DELETE',
        });
    }
}

// Usage
const api = new HttpClient('/api');
const users = await api.get('/users');
const newUser = await api.post('/users', { name: 'John', email: 'john@example.com' });
```

### **Retry and Circuit Breaker Patterns**
```javascript
class ResilientApi {
    constructor(maxRetries = 3, timeoutMs = 5000) {
        this.maxRetries = maxRetries;
        this.timeoutMs = timeoutMs;
        this.failures = 0;
        this.lastFailureTime = 0;
        this.circuitOpen = false;
    }

    async call(endpoint, options = {}) {
        if (this.circuitOpen) {
            if (Date.now() - this.lastFailureTime > 60000) { // 1 minute timeout
                this.circuitOpen = false;
                this.failures = 0;
            } else {
                throw new Error('Circuit breaker is open');
            }
        }

        for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
            try {
                const result = await withTimeout(
                    fetch(endpoint, options).then(r => {
                        if (!r.ok) throw new Error(`HTTP ${r.status}`);
                        return r.json();
                    }),
                    this.timeoutMs
                );

                // Success - reset circuit breaker
                this.failures = 0;
                return result;

            } catch (error) {
                console.warn(`Attempt ${attempt} failed:`, error.message);

                if (attempt === this.maxRetries) {
                    this.failures++;
                    this.lastFailureTime = Date.now();

                    if (this.failures >= 5) {
                        this.circuitOpen = true;
                    }

                    throw error;
                }

                // Exponential backoff
                await new Promise(resolve =>
                    setTimeout(resolve, Math.pow(2, attempt) * 1000)
                );
            }
        }
    }
}

function withTimeout(promise, timeoutMs) {
    return Promise.race([
        promise,
        new Promise((_, reject) =>
            setTimeout(() => reject(new Error('Timeout')), timeoutMs)
        )
    ]);
}
```

## Interview Tips

**Key Concepts to Master:**
- **Promises**: States (pending, fulfilled, rejected), chaining, error handling
- **Async/Await**: Syntax, error handling with try/catch, execution flow
- **Promise Methods**: `all()`, `race()`, `allSettled()`, `any()`
- **Performance**: Concurrent vs sequential execution

**Common Interview Questions:**
1. "What's the difference between async/await and Promises?"
2. "How do you handle errors in async/await?"
3. "When would you use Promise.all() vs sequential awaits?"
4. "How do you implement retry logic for failed API calls?"
5. "What's the difference between Promise.race() and Promise.any()?"

**Pro Tips:**
- Always use `try/catch` with async/await
- Prefer `Promise.all()` for concurrent operations
- Use `Promise.allSettled()` when you need all results regardless of failures
- Implement proper error handling and loading states in React
- Consider timeout and cancellation for long-running operations

**Red Flags in Interviews:**
- Forgetting that async functions return Promises
- Not handling errors properly
- Making unnecessary sequential API calls
- Not understanding Promise chaining vs async/await