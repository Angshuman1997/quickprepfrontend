# How to handle authentication tokens in API calls?

## Question
How to handle authentication tokens in API calls?

## Answer
Store tokens securely and add them to request headers with "Bearer" prefix.

## Basic Example
```javascript
// Store tokens
const setTokens = (accessToken, refreshToken) => {
  localStorage.setItem('accessToken', accessToken);
  localStorage.setItem('refreshToken', refreshToken);
};

// Get token
const getAccessToken = () => {
  return localStorage.getItem('accessToken');
};

// API call with token
const apiCall = async (url) => {
  const token = getAccessToken();
  
  const response = await fetch(url, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  });
  
  return response.json();
};
```

## Token Refresh
```javascript
const refreshToken = async () => {
  const refreshToken = localStorage.getItem('refreshToken');
  
  const response = await fetch('/api/auth/refresh', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken })
  });
  
  const data = await response.json();
  setTokens(data.accessToken, data.refreshToken);
  
  return data.accessToken;
};
```

## In Redux
```javascript
const login = createAsyncThunk(
  'auth/login',
  async (credentials) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(credentials)
    });
    
    const data = await response.json();
    setTokens(data.accessToken, data.refreshToken);
    
    return data.user;
  }
);
```

## Interview Q&A

**Q: How do you handle authentication tokens?**

A: Store access tokens securely and include them in API request headers with "Bearer" prefix.

**Q: What are refresh tokens for?**

A: To get new access tokens when they expire without requiring user to login again.

**Q: Where should you store tokens?**

A: Access tokens in localStorage or memory, refresh tokens as httpOnly cookies for security.

### 2. **Memory-Based Storage for Sensitive Apps**

```typescript
// For highly sensitive applications, use memory storage
class MemoryTokenManager {
    private static accessToken: string | null = null;
    private static refreshToken: string | null = null;
    private static expiry: number | null = null;

    static setTokens(accessToken: string, refreshToken?: string, expiresIn?: number) {
        this.accessToken = accessToken;
        this.refreshToken = refreshToken || null;
        this.expiry = expiresIn ? Date.now() + (expiresIn * 1000) : null;
    }

    static getAccessToken(): string | null {
        if (this.isTokenExpired()) {
            this.clearTokens();
            return null;
        }
        return this.accessToken;
    }

    static getRefreshToken(): string | null {
        return this.refreshToken;
    }

    static isTokenExpired(): boolean {
        return this.expiry ? Date.now() >= this.expiry : false;
    }

    static clearTokens() {
        this.accessToken = null;
        this.refreshToken = null;
        this.expiry = null;
    }

    static isAuthenticated(): boolean {
        return !!(this.getAccessToken());
    }
}
```

## Axios Interceptors for Token Management

### 1. **Request Interceptor for Token Injection**

```typescript
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';

// Create axios instance
const apiClient: AxiosInstance = axios.create({
    baseURL: process.env.REACT_APP_API_BASE_URL,
    timeout: 10000,
});

// Request interceptor to add auth token
apiClient.interceptors.request.use(
    (config: AxiosRequestConfig): AxiosRequestConfig => {
        const token = TokenManager.getAccessToken();

        if (token) {
            config.headers = {
                ...config.headers,
                Authorization: `Bearer ${token}`,
            };
        }

        // Add request ID for tracking
        config.headers = {
            ...config.headers,
            'X-Request-ID': generateRequestId(),
        };

        return config;
    },
    (error) => Promise.reject(error)
);

// Utility function
function generateRequestId(): string {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
}
```

### 2. **Response Interceptor for Token Refresh**

```typescript
// Response interceptor to handle token refresh
apiClient.interceptors.response.use(
    (response: AxiosResponse) => response,
    async (error) => {
        const originalRequest = error.config;

        // Check if error is 401 and we haven't already tried to refresh
        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;

            try {
                const newTokens = await refreshAccessToken();

                if (newTokens) {
                    // Update stored tokens
                    TokenManager.setTokens(
                        newTokens.accessToken,
                        newTokens.refreshToken,
                        newTokens.expiresIn
                    );

                    // Retry the original request with new token
                    originalRequest.headers.Authorization = `Bearer ${newTokens.accessToken}`;
                    return apiClient(originalRequest);
                }
            } catch (refreshError) {
                // Refresh failed, redirect to login
                TokenManager.clearTokens();
                window.location.href = '/login';
                return Promise.reject(refreshError);
            }
        }

        return Promise.reject(error);
    }
);

// Token refresh function
async function refreshAccessToken() {
    try {
        const refreshToken = TokenManager.getRefreshToken();

        if (!refreshToken) {
            throw new Error('No refresh token available');
        }

        const response = await axios.post('/api/auth/refresh', {
            refreshToken,
        });

        return {
            accessToken: response.data.accessToken,
            refreshToken: response.data.refreshToken,
            expiresIn: response.data.expiresIn,
        };
    } catch (error) {
        console.error('Token refresh failed:', error);
        throw error;
    }
}
```

### 3. **Advanced Token Refresh with Queue**

```typescript
// Handle concurrent requests during token refresh
let isRefreshing = false;
let failedQueue: Array<{
    resolve: (token: string) => void;
    reject: (error: any) => void;
}> = [];

const processQueue = (error: any, token: string | null = null) => {
    failedQueue.forEach(({ resolve, reject }) => {
        if (error) {
            reject(error);
        } else {
            resolve(token!);
        }
    });

    failedQueue = [];
};

// Enhanced response interceptor
apiClient.interceptors.response.use(
    (response) => response,
    async (error) => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
            if (isRefreshing) {
                // If refresh is already in progress, queue the request
                return new Promise((resolve, reject) => {
                    failedQueue.push({
                        resolve: (token) => {
                            originalRequest.headers.Authorization = `Bearer ${token}`;
                            resolve(apiClient(originalRequest));
                        },
                        reject,
                    });
                });
            }

            originalRequest._retry = true;
            isRefreshing = true;

            try {
                const newTokens = await refreshAccessToken();

                if (newTokens) {
                    TokenManager.setTokens(
                        newTokens.accessToken,
                        newTokens.refreshToken,
                        newTokens.expiresIn
                    );

                    // Process queued requests
                    processQueue(null, newTokens.accessToken);

                    // Retry original request
                    originalRequest.headers.Authorization = `Bearer ${newTokens.accessToken}`;
                    return apiClient(originalRequest);
                }
            } catch (refreshError) {
                // Refresh failed, reject all queued requests
                processQueue(refreshError, null);
                TokenManager.clearTokens();
                window.location.href = '/login';
                return Promise.reject(refreshError);
            } finally {
                isRefreshing = false;
            }
        }

        return Promise.reject(error);
    }
);
```

## Redux Integration

### 1. **Authentication State Management**

```typescript
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

interface AuthState {
    user: User | null;
    isAuthenticated: boolean;
    loading: boolean;
    error: string | null;
}

const initialState: AuthState = {
    user: null,
    isAuthenticated: false,
    loading: false,
    error: null,
};

// Async thunk for login
export const loginUser = createAsyncThunk(
    'auth/login',
    async (credentials: LoginCredentials, { rejectWithValue }) => {
        try {
            const response = await apiClient.post('/auth/login', credentials);
            const { user, accessToken, refreshToken, expiresIn } = response.data;

            // Store tokens
            TokenManager.setTokens(accessToken, refreshToken, expiresIn);

            return user;
        } catch (error: any) {
            return rejectWithValue(error.response?.data?.message || 'Login failed');
        }
    }
);

// Async thunk for token refresh
export const refreshToken = createAsyncThunk(
    'auth/refresh',
    async (_, { rejectWithValue }) => {
        try {
            const newTokens = await refreshAccessToken();

            if (newTokens) {
                TokenManager.setTokens(
                    newTokens.accessToken,
                    newTokens.refreshToken,
                    newTokens.expiresIn
                );

                return newTokens;
            }
        } catch (error: any) {
            return rejectWithValue('Token refresh failed');
        }
    }
);

// Auth slice
const authSlice = createSlice({
    name: 'auth',
    initialState,
    reducers: {
        logout: (state) => {
            TokenManager.clearTokens();
            state.user = null;
            state.isAuthenticated = false;
        },
        setUser: (state, action: PayloadAction<User>) => {
            state.user = action.payload;
            state.isAuthenticated = true;
        },
        clearError: (state) => {
            state.error = null;
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(loginUser.pending, (state) => {
                state.loading = true;
                state.error = null;
            })
            .addCase(loginUser.fulfilled, (state, action) => {
                state.loading = false;
                state.user = action.payload;
                state.isAuthenticated = true;
            })
            .addCase(loginUser.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload as string;
            })
            .addCase(refreshToken.fulfilled, (state) => {
                state.isAuthenticated = true;
            })
            .addCase(refreshToken.rejected, (state) => {
                // Token refresh failed, user needs to login again
                TokenManager.clearTokens();
                state.user = null;
                state.isAuthenticated = false;
            });
    },
});

export const { logout, setUser, clearError } = authSlice.actions;
```

### 2. **Automatic Token Refresh in Redux**

```typescript
// Middleware to handle token refresh automatically
const tokenRefreshMiddleware = (store: any) => (next: any) => async (action: any) => {
    // Check if action requires authentication
    if (action.type?.endsWith('/pending') && action.meta?.requiresAuth) {
        const state = store.getState();

        // Check if token is expired or will expire soon (within 5 minutes)
        if (TokenManager.isTokenExpired() ||
            (TokenManager.getExpiry() && TokenManager.getExpiry()! - Date.now() < 5 * 60 * 1000)) {

            try {
                // Attempt to refresh token
                await store.dispatch(refreshToken());
            } catch (error) {
                // Refresh failed, continue with original action (will likely fail with 401)
                console.warn('Token refresh failed, proceeding with original action');
            }
        }
    }

    return next(action);
};

// Apply middleware
const store = configureStore({
    reducer: rootReducer,
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware().concat(tokenRefreshMiddleware),
});

// Usage in thunks
export const fetchUserData = createAsyncThunk(
    'user/fetchData',
    async (_, { rejectWithValue }) => {
        try {
            const response = await apiClient.get('/user/data');
            return response.data;
        } catch (error: any) {
            return rejectWithValue(error.response?.data?.message || 'Failed to fetch data');
        }
    },
    {
        // Mark action as requiring auth
        getPendingMeta: () => ({ requiresAuth: true }),
    }
);
```

## React Component Integration

### 1. **Authentication Context**

```typescript
import React, { createContext, useContext, useEffect, ReactNode } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { refreshToken, logout } from './authSlice';

interface AuthContextType {
    isAuthenticated: boolean;
    user: User | null;
    login: (credentials: LoginCredentials) => Promise<void>;
    logout: () => void;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
    const dispatch = useDispatch();
    const { isAuthenticated, user } = useSelector((state: any) => state.auth);

    useEffect(() => {
        // Check for existing tokens on app start
        if (TokenManager.isAuthenticated()) {
            // Try to refresh token to get fresh user data
            dispatch(refreshToken());
        }
    }, [dispatch]);

    const login = async (credentials: LoginCredentials) => {
        await dispatch(loginUser(credentials));
    };

    const handleLogout = () => {
        dispatch(logout());
    };

    const value = {
        isAuthenticated,
        user,
        login,
        logout: handleLogout,
    };

    return (
        <AuthContext.Provider value={value}>
            {children}
        </AuthContext.Provider>
    );
};

export const useAuth = (): AuthContextType => {
    const context = useContext(AuthContext);
    if (!context) {
        throw new Error('useAuth must be used within AuthProvider');
    }
    return context;
};
```

### 2. **Protected Route Component**

```typescript
import { useAuth } from './AuthContext';
import { Navigate, useLocation } from 'react-router-dom';

interface ProtectedRouteProps {
    children: ReactNode;
}

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({ children }) => {
    const { isAuthenticated } = useAuth();
    const location = useLocation();

    if (!isAuthenticated) {
        // Redirect to login with return URL
        return <Navigate to="/login" state={{ from: location }} replace />;
    }

    return <>{children}</>;
};

// Usage
function App() {
    return (
        <AuthProvider>
            <Routes>
                <Route path="/login" element={<LoginPage />} />
                <Route path="/dashboard" element={
                    <ProtectedRoute>
                        <Dashboard />
                    </ProtectedRoute>
                } />
            </Routes>
        </AuthProvider>
    );
}
```

### 3. **Login Component**

```typescript
import { useState } from 'react';
import { useAuth } from './AuthContext';
import { useNavigate, useLocation } from 'react-router-dom';

export const LoginPage: React.FC = () => {
    const [credentials, setCredentials] = useState({ email: '', password: '' });
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState('');

    const { login } = useAuth();
    const navigate = useNavigate();
    const location = useLocation();

    const from = location.state?.from?.pathname || '/dashboard';

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        setLoading(true);
        setError('');

        try {
            await login(credentials);
            navigate(from, { replace: true });
        } catch (error) {
            setError('Invalid credentials');
        } finally {
            setLoading(false);
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="email"
                value={credentials.email}
                onChange={(e) => setCredentials(prev => ({ ...prev, email: e.target.value }))}
                placeholder="Email"
                required
            />
            <input
                type="password"
                value={credentials.password}
                onChange={(e) => setCredentials(prev => ({ ...prev, password: e.target.value }))}
                placeholder="Password"
                required
            />
            {error && <div className="error">{error}</div>}
            <button type="submit" disabled={loading}>
                {loading ? 'Logging in...' : 'Login'}
            </button>
        </form>
    );
};
```

## Security Best Practices

### 1. **Token Security**

```typescript
// ✅ Good: Secure token handling
class SecureTokenManager {
    // Use httpOnly cookies for refresh tokens (server-side)
    // Store access tokens in memory or secure storage
    // Never store sensitive data in localStorage

    static setTokens(accessToken: string, refreshToken?: string) {
        // Store access token in memory (more secure than localStorage)
        sessionStorage.setItem('access_token', accessToken);

        // Refresh token should be httpOnly cookie set by server
        // Don't store it in JavaScript-accessible storage
    }

    static async validateToken(token: string): Promise<boolean> {
        try {
            // Validate token with server (optional)
            const response = await axios.post('/api/auth/validate', { token });
            return response.data.valid;
        } catch {
            return false;
        }
    }
}
```

### 2. **CSRF Protection**

```typescript
// Add CSRF token to requests
apiClient.interceptors.request.use((config) => {
    const csrfToken = getCsrfToken(); // From meta tag or cookie
    if (csrfToken) {
        config.headers['X-CSRF-Token'] = csrfToken;
    }
    return config;
});

function getCsrfToken(): string | null {
    // Get from meta tag
    const metaTag = document.querySelector('meta[name="csrf-token"]');
    return metaTag ? metaTag.getAttribute('content') : null;
}
```

### 3. **Secure Headers**

```typescript
// Set secure headers
const secureApiClient = axios.create({
    headers: {
        'X-Requested-With': 'XMLHttpRequest', // Helps prevent CSRF
        'X-Client-Version': process.env.REACT_APP_VERSION,
    },
    withCredentials: true, // Send cookies with requests
});
```

## Testing Token Management

```typescript
import { render, screen, waitFor } from '@testing-library/react';
import { AuthProvider, useAuth } from './AuthContext';

// Mock TokenManager
jest.mock('./TokenManager');

const TestComponent = () => {
    const { isAuthenticated, login } = useAuth();

    return (
        <div>
            <span>{isAuthenticated ? 'Authenticated' : 'Not authenticated'}</span>
            <button onClick={() => login({ email: 'test@test.com', password: 'password' })}>
                Login
            </button>
        </div>
    );
};

describe('AuthProvider', () => {
    beforeEach(() => {
        TokenManager.isAuthenticated.mockReturnValue(false);
        TokenManager.getAccessToken.mockReturnValue(null);
    });

    it('shows not authenticated initially', () => {
        render(
            <AuthProvider>
                <TestComponent />
            </AuthProvider>
        );

        expect(screen.getByText('Not authenticated')).toBeInTheDocument();
    });

    it('handles login success', async () => {
        TokenManager.setTokens.mockImplementation(() => {});
        // Mock successful login API call

        render(
            <AuthProvider>
                <TestComponent />
            </AuthProvider>
        );

        const loginButton = screen.getByText('Login');
        loginButton.click();

        await waitFor(() => {
            expect(screen.getByText('Authenticated')).toBeInTheDocument();
        });
    });
});
```

## Common Interview Questions

### Q: How do you handle token expiration in API calls?

**A:** I use response interceptors to catch 401 errors, then attempt to refresh the token automatically. If refresh succeeds, I retry the original request. If it fails, I redirect to login.

### Q: What's the difference between access tokens and refresh tokens?

**A:** Access tokens are short-lived and used for API authentication. Refresh tokens are long-lived and used to get new access tokens when they expire, improving security by limiting exposure of long-lived credentials.

### Q: How do you handle concurrent requests during token refresh?

**A:** I use a queue system where failed requests during refresh are stored and retried once the new token is available. This prevents multiple refresh attempts and ensures all requests use the new token.

### Q: Where should you store authentication tokens?

**A:** Access tokens can be stored in memory or secure HTTP-only cookies. Refresh tokens should always be HTTP-only cookies to prevent XSS attacks. Avoid localStorage for sensitive tokens.

## Summary

**Key Token Management Strategies:**
1. **Secure Storage**: Use memory or httpOnly cookies over localStorage
2. **Automatic Refresh**: Intercept 401 responses and refresh tokens
3. **Request Interceptors**: Automatically add tokens to requests
4. **Queue Management**: Handle concurrent requests during refresh
5. **Redux Integration**: Manage auth state with proper actions/reducers

**Security Best Practices:**
- Use short-lived access tokens
- Store refresh tokens securely (httpOnly cookies)
- Implement CSRF protection
- Validate tokens server-side
- Handle token expiration gracefully

**Interview Tip:** "I handle authentication tokens using Axios interceptors - request interceptors add Bearer tokens to API calls, response interceptors catch 401s and refresh tokens automatically. I store access tokens securely and use refresh tokens for seamless user experience without frequent logins."