# How to show loading spinners during API calls?

## Question
How to show loading spinners during API calls?

## Answer

Loading spinners provide visual feedback during asynchronous operations, improving user experience by indicating that something is happening. There are several patterns for implementing loading states in React applications.

## Basic Loading State

### Simple useState Approach

```javascript
import { useState } from 'react';

function DataComponent() {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const fetchData = async () => {
        setLoading(true);
        setError(null);

        try {
            const response = await fetch('/api/data');
            if (!response.ok) {
                throw new Error('Failed to fetch data');
            }
            const result = await response.json();
            setData(result);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    };

    return (
        <div>
            <button onClick={fetchData} disabled={loading}>
                {loading ? 'Loading...' : 'Fetch Data'}
            </button>

            {loading && <div className="spinner">Loading...</div>}

            {error && <div className="error">Error: {error}</div>}

            {data && !loading && (
                <div className="data">
                    <pre>{JSON.stringify(data, null, 2)}</pre>
                </div>
            )}
        </div>
    );
}
```

## Custom Loading Spinner Component

### Reusable Spinner Component

```javascript
// LoadingSpinner.js
import './LoadingSpinner.css';

function LoadingSpinner({ size = 'medium', color = 'primary' }) {
    return (
        <div className={`spinner ${size} ${color}`}>
            <div className="spinner-inner">
                <div className="spinner-circle"></div>
            </div>
        </div>
    );
}

export default LoadingSpinner;

// LoadingSpinner.css
.spinner {
    display: inline-block;
}

.spinner.small {
    width: 20px;
    height: 20px;
}

.spinner.medium {
    width: 40px;
    height: 40px;
}

.spinner.large {
    width: 60px;
    height: 60px;
}

.spinner-inner {
    width: 100%;
    height: 100%;
    position: relative;
}

.spinner-circle {
    width: 100%;
    height: 100%;
    border: 3px solid #f3f3f3;
    border-top: 3px solid #3498db;
    border-radius: 50%;
    animation: spin 1s linear infinite;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

.spinner.primary .spinner-circle {
    border-top-color: #007bff;
}

.spinner.secondary .spinner-circle {
    border-top-color: #6c757d;
}
```

### Usage with Custom Spinner

```javascript
import LoadingSpinner from './LoadingSpinner';

function DataComponent() {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const fetchData = async () => {
        setLoading(true);
        setError(null);

        try {
            const response = await fetch('/api/data');
            const result = await response.json();
            setData(result);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    };

    if (loading) {
        return (
            <div className="loading-container">
                <LoadingSpinner size="large" />
                <p>Loading data...</p>
            </div>
        );
    }

    if (error) {
        return (
            <div className="error-container">
                <p>Error: {error}</p>
                <button onClick={fetchData}>Try Again</button>
            </div>
        );
    }

    return (
        <div>
            <button onClick={fetchData}>Fetch Data</button>
            {data && <pre>{JSON.stringify(data, null, 2)}</pre>}
        </div>
    );
}
```

## Loading States with useEffect

### Automatic Loading on Mount

```javascript
function UserProfile({ userId }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    useEffect(() => {
        const fetchUser = async () => {
            try {
                const response = await fetch(`/api/users/${userId}`);
                if (!response.ok) {
                    throw new Error('User not found');
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
    }, [userId]);

    if (loading) {
        return (
            <div className="profile-loading">
                <LoadingSpinner />
                <p>Loading user profile...</p>
            </div>
        );
    }

    if (error) {
        return (
            <div className="profile-error">
                <h2>Error Loading Profile</h2>
                <p>{error}</p>
                <button onClick={() => window.location.reload()}>
                    Reload Page
                </button>
            </div>
        );
    }

    return (
        <div className="user-profile">
            <img src={user.avatar} alt={user.name} />
            <h1>{user.name}</h1>
            <p>{user.email}</p>
        </div>
    );
}
```

## Custom Hook for API Calls

### Reusable useApi Hook

```javascript
function useApi(url, options = {}) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);

    const execute = useCallback(async (overrideUrl = url, overrideOptions = {}) => {
        setLoading(true);
        setError(null);

        try {
            const response = await fetch(overrideUrl, {
                ...options,
                ...overrideOptions
            });

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }

            const result = await response.json();
            setData(result);
            return result;
        } catch (err) {
            setError(err.message);
            throw err;
        } finally {
            setLoading(false);
        }
    }, [url, options]);

    const refetch = useCallback(() => {
        return execute();
    }, [execute]);

    return {
        data,
        loading,
        error,
        execute,
        refetch
    };
}

// Usage
function ProductList() {
    const { data: products, loading, error, execute } = useApi('/api/products');

    useEffect(() => {
        execute();
    }, [execute]);

    if (loading) {
        return (
            <div className="products-loading">
                <LoadingSpinner size="large" />
                <p>Loading products...</p>
            </div>
        );
    }

    if (error) {
        return (
            <div className="products-error">
                <p>Failed to load products: {error}</p>
                <button onClick={() => execute()}>Retry</button>
            </div>
        );
    }

    return (
        <div className="products">
            {products?.map(product => (
                <div key={product.id} className="product">
                    <h3>{product.name}</h3>
                    <p>${product.price}</p>
                </div>
            ))}
        </div>
    );
}
```

## Skeleton Loading

### Content Placeholder

```javascript
// SkeletonLoader.js
function SkeletonLoader({ type = 'text', lines = 1 }) {
    if (type === 'card') {
        return (
            <div className="skeleton-card">
                <div className="skeleton-image"></div>
                <div className="skeleton-title"></div>
                <div className="skeleton-text"></div>
                <div className="skeleton-text short"></div>
            </div>
        );
    }

    return (
        <div className="skeleton-text-container">
            {Array.from({ length: lines }).map((_, index) => (
                <div
                    key={index}
                    className={`skeleton-text ${index === lines - 1 ? 'short' : ''}`}
                ></div>
            ))}
        </div>
    );
}

// SkeletonLoader.css
.skeleton-card {
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    padding: 16px;
    margin: 8px 0;
}

.skeleton-image {
    width: 100%;
    height: 200px;
    background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
    background-size: 200% 100%;
    animation: loading 1.5s infinite;
    border-radius: 4px;
    margin-bottom: 12px;
}

.skeleton-title {
    height: 24px;
    background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
    background-size: 200% 100%;
    animation: loading 1.5s infinite;
    border-radius: 4px;
    margin-bottom: 8px;
}

.skeleton-text {
    height: 16px;
    background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
    background-size: 200% 100%;
    animation: loading 1.5s infinite;
    border-radius: 4px;
    margin-bottom: 6px;
}

.skeleton-text.short {
    width: 60%;
}

@keyframes loading {
    0% {
        background-position: 200% 0;
    }
    100% {
        background-position: -200% 0;
    }
}
```

### Usage with Skeleton

```javascript
function ArticleList() {
    const { data: articles, loading, error } = useApi('/api/articles');

    useEffect(() => {
        // Simulate API call
        setTimeout(() => {
            // setArticles(mockArticles);
        }, 2000);
    }, []);

    if (error) {
        return <div className="error">Failed to load articles</div>;
    }

    return (
        <div className="articles">
            {loading ? (
                // Show skeleton while loading
                <>
                    <SkeletonLoader type="card" />
                    <SkeletonLoader type="card" />
                    <SkeletonLoader type="card" />
                </>
            ) : (
                // Show actual content when loaded
                articles?.map(article => (
                    <article key={article.id} className="article-card">
                        <img src={article.image} alt={article.title} />
                        <h2>{article.title}</h2>
                        <p>{article.excerpt}</p>
                        <div className="meta">
                            <span>{article.author}</span>
                            <span>{article.date}</span>
                        </div>
                    </article>
                ))
            )}
        </div>
    );
}
```

## Progressive Loading

### Load Content in Stages

```javascript
function ProgressiveLoader() {
    const [loadingStage, setLoadingStage] = useState('initial');
    const [data, setData] = useState({});

    useEffect(() => {
        const loadData = async () => {
            // Stage 1: Basic info
            setLoadingStage('basic');
            const basicInfo = await fetch('/api/basic-info').then(r => r.json());
            setData(prev => ({ ...prev, basic: basicInfo }));

            // Stage 2: Details
            setLoadingStage('details');
            const details = await fetch('/api/details').then(r => r.json());
            setData(prev => ({ ...prev, details }));

            // Stage 3: Related content
            setLoadingStage('related');
            const related = await fetch('/api/related').then(r => r.json());
            setData(prev => ({ ...prev, related }));

            setLoadingStage('complete');
        };

        loadData();
    }, []);

    const getLoadingMessage = () => {
        switch (loadingStage) {
            case 'initial': return 'Preparing...';
            case 'basic': return 'Loading basic information...';
            case 'details': return 'Loading details...';
            case 'related': return 'Loading related content...';
            default: return 'Complete';
        }
    };

    return (
        <div className="progressive-loader">
            <div className="loading-indicator">
                <LoadingSpinner />
                <p>{getLoadingMessage()}</p>
            </div>

            {/* Show partial content as it loads */}
            {data.basic && (
                <div className="basic-info">
                    <h1>{data.basic.title}</h1>
                    <p>{data.basic.description}</p>
                </div>
            )}

            {data.details && (
                <div className="details">
                    <h2>Details</h2>
                    <p>{data.details.content}</p>
                </div>
            )}

            {data.related && (
                <div className="related">
                    <h2>Related Content</h2>
                    {data.related.items.map(item => (
                        <div key={item.id}>{item.title}</div>
                    ))}
                </div>
            )}
        </div>
    );
}
```

## Button Loading States

### Loading Button Component

```javascript
function LoadingButton({
    children,
    loading,
    onClick,
    disabled,
    ...props
}) {
    const handleClick = async () => {
        if (loading || disabled) return;
        await onClick();
    };

    return (
        <button
            {...props}
            onClick={handleClick}
            disabled={loading || disabled}
            className={`loading-button ${loading ? 'loading' : ''}`}
        >
            {loading && <LoadingSpinner size="small" />}
            <span className={loading ? 'loading-text' : ''}>
                {loading ? 'Loading...' : children}
            </span>
        </button>
    );
}

// LoadingButton.css
.loading-button {
    position: relative;
    display: inline-flex;
    align-items: center;
    gap: 8px;
    padding: 8px 16px;
    border: 1px solid #007bff;
    background: #007bff;
    color: white;
    border-radius: 4px;
    cursor: pointer;
}

.loading-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
}

.loading-button.loading {
    background: #0056b3;
}

.loading-text {
    opacity: 0.8;
}
```

### Usage

```javascript
function ContactForm() {
    const [formData, setFormData] = useState({ name: '', email: '', message: '' });
    const [submitting, setSubmitting] = useState(false);

    const handleSubmit = async (e) => {
        e.preventDefault();
        setSubmitting(true);

        try {
            await submitContactForm(formData);
            alert('Message sent successfully!');
            setFormData({ name: '', email: '', message: '' });
        } catch (error) {
            alert('Failed to send message');
        } finally {
            setSubmitting(false);
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="text"
                placeholder="Name"
                value={formData.name}
                onChange={(e) => setFormData(prev => ({ ...prev, name: e.target.value }))}
                required
            />
            <input
                type="email"
                placeholder="Email"
                value={formData.email}
                onChange={(e) => setFormData(prev => ({ ...prev, email: e.target.value }))}
                required
            />
            <textarea
                placeholder="Message"
                value={formData.message}
                onChange={(e) => setFormData(prev => ({ ...prev, message: e.target.value }))}
                required
            />
            <LoadingButton
                type="submit"
                loading={submitting}
                disabled={!formData.name || !formData.email || !formData.message}
            >
                Send Message
            </LoadingButton>
        </form>
    );
}
```

## Global Loading State

### Context-based Loading Management

```javascript
// LoadingContext.js
const LoadingContext = createContext();

export function LoadingProvider({ children }) {
    const [loadingStates, setLoadingStates] = useState(new Map());

    const startLoading = useCallback((key) => {
        setLoadingStates(prev => new Map(prev).set(key, true));
    }, []);

    const stopLoading = useCallback((key) => {
        setLoadingStates(prev => {
            const newMap = new Map(prev);
            newMap.delete(key);
            return newMap;
        });
    }, []);

    const isLoading = useCallback((key) => {
        return loadingStates.has(key);
    }, [loadingStates]);

    const hasAnyLoading = loadingStates.size > 0;

    const value = {
        startLoading,
        stopLoading,
        isLoading,
        hasAnyLoading
    };

    return (
        <LoadingContext.Provider value={value}>
            {children}
        </LoadingContext.Provider>
    );
}

export function useLoading() {
    const context = useContext(LoadingContext);
    if (!context) {
        throw new Error('useLoading must be used within a LoadingProvider');
    }
    return context;
}

// GlobalLoadingIndicator.js
function GlobalLoadingIndicator() {
    const { hasAnyLoading } = useLoading();

    if (!hasAnyLoading) return null;

    return (
        <div className="global-loading-overlay">
            <div className="global-loading-content">
                <LoadingSpinner size="large" />
                <p>Loading...</p>
            </div>
        </div>
    );
}

// Usage
function App() {
    return (
        <LoadingProvider>
            <GlobalLoadingIndicator />
            <MainContent />
        </LoadingProvider>
    );
}

function DataFetcher() {
    const { startLoading, stopLoading, isLoading } = useLoading();

    const fetchData = async () => {
        startLoading('fetchData');
        try {
            await fetch('/api/data');
        } finally {
            stopLoading('fetchData');
        }
    };

    return (
        <div>
            <button onClick={fetchData} disabled={isLoading('fetchData')}>
                {isLoading('fetchData') ? 'Loading...' : 'Fetch Data'}
            </button>
        </div>
    );
}
```

## Real React Examples

### 1. **Complete Data Fetching Component**

```javascript
function UserDashboard() {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [loading, setLoading] = useState({
        user: true,
        posts: true
    });
    const [error, setError] = useState(null);

    useEffect(() => {
        const fetchUserData = async () => {
            try {
                const [userResponse, postsResponse] = await Promise.all([
                    fetch('/api/user'),
                    fetch('/api/user/posts')
                ]);

                const userData = await userResponse.json();
                const postsData = await postsResponse.json();

                setUser(userData);
                setPosts(postsData);
            } catch (err) {
                setError('Failed to load dashboard data');
            } finally {
                setLoading({ user: false, posts: false });
            }
        };

        fetchUserData();
    }, []);

    if (error) {
        return (
            <div className="dashboard-error">
                <h2>Dashboard Unavailable</h2>
                <p>{error}</p>
                <button onClick={() => window.location.reload()}>
                    Reload
                </button>
            </div>
        );
    }

    return (
        <div className="dashboard">
            <div className="user-section">
                {loading.user ? (
                    <div className="user-loading">
                        <LoadingSpinner />
                        <p>Loading user info...</p>
                    </div>
                ) : (
                    <div className="user-info">
                        <img src={user.avatar} alt={user.name} />
                        <h1>Welcome, {user.name}!</h1>
                    </div>
                )}
            </div>

            <div className="posts-section">
                <h2>Your Posts</h2>
                {loading.posts ? (
                    <div className="posts-loading">
                        <LoadingSpinner />
                        <p>Loading posts...</p>
                    </div>
                ) : (
                    <div className="posts-list">
                        {posts.map(post => (
                            <div key={post.id} className="post-card">
                                <h3>{post.title}</h3>
                                <p>{post.content}</p>
                                <small>{new Date(post.createdAt).toLocaleDateString()}</small>
                            </div>
                        ))}
                    </div>
                )}
            </div>
        </div>
    );
}
```

### 2. **Search with Loading States**

```javascript
function SearchComponent() {
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);
    const [hasSearched, setHasSearched] = useState(false);

    const search = useCallback(async (searchQuery) => {
        if (!searchQuery.trim()) {
            setResults([]);
            setHasSearched(false);
            return;
        }

        setLoading(true);
        setError(null);
        setHasSearched(true);

        try {
            const response = await fetch(`/api/search?q=${encodeURIComponent(searchQuery)}`);
            if (!response.ok) {
                throw new Error('Search failed');
            }
            const data = await response.json();
            setResults(data.results);
        } catch (err) {
            setError(err.message);
            setResults([]);
        } finally {
            setLoading(false);
        }
    }, []);

    // Debounced search
    useEffect(() => {
        const timeoutId = setTimeout(() => {
            search(query);
        }, 300);

        return () => clearTimeout(timeoutId);
    }, [query, search]);

    return (
        <div className="search-component">
            <div className="search-input">
                <input
                    type="text"
                    value={query}
                    onChange={(e) => setQuery(e.target.value)}
                    placeholder="Search..."
                />
                {loading && <LoadingSpinner size="small" />}
            </div>

            {error && (
                <div className="search-error">
                    <p>Search error: {error}</p>
                </div>
            )}

            {hasSearched && !loading && !error && (
                <div className="search-results">
                    {results.length === 0 ? (
                        <p>No results found</p>
                    ) : (
                        <ul>
                            {results.map(result => (
                                <li key={result.id}>
                                    <h3>{result.title}</h3>
                                    <p>{result.description}</p>
                                </li>
                            ))}
                        </ul>
                    )}
                </div>
            )}
        </div>
    );
}
```

### 3. **File Upload with Progress**

```javascript
function FileUpload() {
    const [uploading, setUploading] = useState(false);
    const [progress, setProgress] = useState(0);
    const [error, setError] = useState(null);

    const handleFileUpload = async (file) => {
        setUploading(true);
        setProgress(0);
        setError(null);

        try {
            const formData = new FormData();
            formData.append('file', file);

            const xhr = new XMLHttpRequest();

            xhr.upload.addEventListener('progress', (event) => {
                if (event.lengthComputable) {
                    const percentComplete = (event.loaded / event.total) * 100;
                    setProgress(percentComplete);
                }
            });

            const response = await new Promise((resolve, reject) => {
                xhr.addEventListener('load', () => {
                    if (xhr.status >= 200 && xhr.status < 300) {
                        resolve(xhr.response);
                    } else {
                        reject(new Error(`Upload failed: ${xhr.statusText}`));
                    }
                });

                xhr.addEventListener('error', () => {
                    reject(new Error('Upload failed'));
                });

                xhr.open('POST', '/api/upload');
                xhr.send(formData);
            });

            console.log('Upload successful:', response);
        } catch (err) {
            setError(err.message);
        } finally {
            setUploading(false);
            setProgress(0);
        }
    };

    return (
        <div className="file-upload">
            <input
                type="file"
                onChange={(e) => {
                    const file = e.target.files[0];
                    if (file) handleFileUpload(file);
                }}
                disabled={uploading}
            />

            {uploading && (
                <div className="upload-progress">
                    <LoadingSpinner />
                    <div className="progress-bar">
                        <div
                            className="progress-fill"
                            style={{ width: `${progress}%` }}
                        />
                    </div>
                    <p>Uploading... {Math.round(progress)}%</p>
                </div>
            )}

            {error && (
                <div className="upload-error">
                    <p>Upload failed: {error}</p>
                </div>
            )}
        </div>
    );
}
```

## Best Practices

### 1. **User Experience**
- Always show loading states for operations >100ms
- Provide clear feedback about what's loading
- Disable buttons during loading to prevent double-submission
- Show progress for long operations

### 2. **Performance**
- Use skeleton screens for better perceived performance
- Debounce search inputs to reduce API calls
- Cache results when appropriate
- Use optimistic updates for better UX

### 3. **Error Handling**
- Always handle loading and error states
- Provide retry mechanisms
- Show user-friendly error messages
- Log errors for debugging

### 4. **Accessibility**
- Use ARIA labels for loading states
- Don't hide content from screen readers
- Provide alternative text for spinners
- Ensure keyboard navigation works during loading

### Interview Tip:
*"Show loading spinners during API calls using state management (loading, error, data). Use custom spinner components, skeleton loaders for better UX, and handle different loading stages. Always provide user feedback and handle errors gracefully."*