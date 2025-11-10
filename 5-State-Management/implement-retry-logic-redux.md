# How to implement retry logic in Redux?

## Question
How to implement retry logic in Redux?

## Answer
Retry failed API calls automatically with delays between attempts.

## Basic Example
```javascript
const fetchData = createAsyncThunk(
  'data/fetch',
  async (_, { rejectWithValue }) => {
    const maxRetries = 3;
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        const response = await fetch('/api/data');
        if (!response.ok) throw new Error('API Error');
        return await response.json();
      } catch (error) {
        if (attempt === maxRetries) {
          return rejectWithValue(error.message);
        }
        // Wait 1 second before retry
        await new Promise(resolve => setTimeout(resolve, 1000));
      }
    }
  }
);

const dataSlice = createSlice({
  name: 'data',
  initialState: { items: [], loading: false, error: null },
  extraReducers: (builder) => {
    builder
      .addCase(fetchData.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchData.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchData.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      });
  }
});
```

## In Component
```javascript
function DataList() {
  const dispatch = useDispatch();
  const { items, loading, error } = useSelector(state => state.data);

  useEffect(() => {
    dispatch(fetchData());
  }, [dispatch]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <ul>
      {items.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}
```

## Interview Q&A

**Q: What is retry logic?**

A: Automatically retry failed API calls to handle temporary network issues.

**Q: When should you retry API calls?**

A: On network errors or server errors (5xx), but not on client errors (4xx).

**Q: How do you implement retry in Redux?**

A: Use createAsyncThunk with a loop that catches errors and retries with delays.

### 2. **Component with Retry UI**

```typescript
import { useDispatch, useSelector } from 'react-redux';
import { fetchDataWithRetry, resetRetryCount } from './apiSlice';

function DataComponent() {
    const dispatch = useDispatch();
    const { data, loading, error, retryCount } = useSelector((state: RootState) => state.api);

    const handleRetry = () => {
        dispatch(resetRetryCount());
        dispatch(fetchDataWithRetry());
    };

    React.useEffect(() => {
        dispatch(fetchDataWithRetry());
    }, [dispatch]);

    if (loading) {
        return (
            <div>
                <div>Loading... {retryCount > 1 && `(Attempt ${retryCount})`}</div>
                {retryCount > 1 && <div>Retrying...</div>}
            </div>
        );
    }

    if (error) {
        return (
            <div>
                <div>Error: {error}</div>
                <button onClick={handleRetry}>Retry</button>
            </div>
        );
    }

    return <div>Data: {JSON.stringify(data)}</div>;
}
```

## Advanced Retry Patterns

### 1. **Exponential Backoff with Jitter**

```typescript
// Utility functions for retry logic
const calculateDelay = (attempt: number, baseDelay: number = 1000): number => {
    // Exponential backoff: baseDelay * 2^attempt
    const exponentialDelay = baseDelay * Math.pow(2, attempt);

    // Add jitter to prevent thundering herd
    const jitter = Math.random() * 0.1 * exponentialDelay;

    return Math.min(exponentialDelay + jitter, 30000); // Cap at 30 seconds
};

const isRetryableError = (error: any): boolean => {
    // Retry on network errors, 5xx server errors, and specific 4xx errors
    if (error.name === 'TypeError' && error.message.includes('fetch')) {
        return true; // Network error
    }

    if (error.message?.includes('HTTP')) {
        const statusCode = parseInt(error.message.split(' ')[1]);
        return statusCode >= 500 || statusCode === 429 || statusCode === 408;
    }

    return false;
};

// Enhanced async thunk with exponential backoff
export const fetchDataWithBackoff = createAsyncThunk(
    'api/fetchDataBackoff',
    async (_, { rejectWithValue }) => {
        const maxRetries = 5;
        let lastError: any = null;

        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                const controller = new AbortController();
                const timeoutId = setTimeout(() => controller.abort(), 10000); // 10s timeout

                const response = await fetch('/api/data', {
                    signal: controller.signal,
                });

                clearTimeout(timeoutId);

                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}`);
                }

                return await response.json();
            } catch (error) {
                lastError = error;

                if (!isRetryableError(error) || attempt === maxRetries) {
                    // Non-retryable error or max retries reached
                    return rejectWithValue({
                        message: attempt === maxRetries ? 'Max retries exceeded' : 'Non-retryable error',
                        attempts: attempt + 1,
                        lastError: error.message,
                        retryable: isRetryableError(error),
                    });
                }

                // Wait with exponential backoff
                const delay = calculateDelay(attempt);
                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }
    }
);
```

### 2. **Circuit Breaker Pattern**

```typescript
interface CircuitBreakerState {
    state: 'closed' | 'open' | 'half-open';
    failureCount: number;
    lastFailureTime: number;
    successCount: number;
}

class CircuitBreaker {
    private state: CircuitBreakerState = {
        state: 'closed',
        failureCount: 0,
        lastFailureTime: 0,
        successCount: 0,
    };

    private readonly failureThreshold = 5;
    private readonly recoveryTimeout = 60000; // 1 minute
    private readonly successThreshold = 3;

    async execute<T>(operation: () => Promise<T>): Promise<T> {
        if (this.state.state === 'open') {
            if (Date.now() - this.state.lastFailureTime > this.recoveryTimeout) {
                this.state.state = 'half-open';
                this.state.successCount = 0;
            } else {
                throw new Error('Circuit breaker is open');
            }
        }

        try {
            const result = await operation();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }

    private onSuccess() {
        this.state.failureCount = 0;

        if (this.state.state === 'half-open') {
            this.state.successCount++;
            if (this.state.successCount >= this.successThreshold) {
                this.state.state = 'closed';
            }
        }
    }

    private onFailure() {
        this.state.failureCount++;
        this.state.lastFailureTime = Date.now();

        if (this.state.failureCount >= this.failureThreshold) {
            this.state.state = 'open';
        }
    }

    getState() {
        return { ...this.state };
    }
}

// Redux slice with circuit breaker
interface ApiState {
    data: any;
    loading: boolean;
    error: string | null;
    circuitBreaker: CircuitBreakerState;
}

const circuitBreaker = new CircuitBreaker();

export const fetchDataWithCircuitBreaker = createAsyncThunk(
    'api/fetchDataCircuit',
    async (_, { rejectWithValue }) => {
        try {
            const result = await circuitBreaker.execute(async () => {
                const response = await fetch('/api/data');
                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}`);
                }
                return response.json();
            });

            return result;
        } catch (error) {
            return rejectWithValue({
                message: error.message,
                circuitBreakerState: circuitBreaker.getState(),
            });
        }
    }
);

const apiSlice = createSlice({
    name: 'api',
    initialState: {
        data: null,
        loading: false,
        error: null,
        circuitBreaker: circuitBreaker.getState(),
    } as ApiState,
    extraReducers: (builder) => {
        builder
            .addCase(fetchDataWithCircuitBreaker.pending, (state) => {
                state.loading = true;
                state.error = null;
                state.circuitBreaker = circuitBreaker.getState();
            })
            .addCase(fetchDataWithCircuitBreaker.fulfilled, (state, action) => {
                state.loading = false;
                state.data = action.payload;
                state.circuitBreaker = circuitBreaker.getState();
            })
            .addCase(fetchDataWithCircuitBreaker.rejected, (state, action) => {
                state.loading = false;
                state.error = action.payload.message;
                state.circuitBreaker = circuitBreaker.getState();
            });
    },
});
```

### 3. **Retry with Redux Middleware**

```typescript
// Retry middleware
const retryMiddleware = (store: any) => (next: any) => async (action: any) => {
    if (action.type?.endsWith('/rejected') && action.meta?.retryable !== false) {
        const originalAction = action.meta.originalAction;

        if (originalAction && shouldRetry(action)) {
            const retryCount = action.meta.retryCount || 0;
            const maxRetries = 3;

            if (retryCount < maxRetries) {
                const delay = calculateDelay(retryCount);

                setTimeout(() => {
                    store.dispatch({
                        ...originalAction,
                        meta: {
                            ...originalAction.meta,
                            retryCount: retryCount + 1,
                        },
                    });
                }, delay);

                return; // Don't pass the rejected action to reducers
            }
        }
    }

    return next(action);
};

const shouldRetry = (action: any): boolean => {
    const error = action.payload;

    // Retry on network errors and 5xx status codes
    if (error?.name === 'TypeError') return true;
    if (error?.message?.includes('HTTP 5')) return true;
    if (error?.message?.includes('HTTP 429')) return true; // Rate limited

    return false;
};

// Enhanced async thunk that works with retry middleware
export const fetchDataWithMiddleware = createAsyncThunk(
    'api/fetchDataMiddleware',
    async (_, { rejectWithValue, requestId }) => {
        try {
            const response = await fetch('/api/data');
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}`);
            }
            return await response.json();
        } catch (error) {
            return rejectWithValue({
                message: error.message,
                retryable: shouldRetry({ payload: error }),
                requestId,
            });
        }
    }
);

// Store configuration with retry middleware
const store = configureStore({
    reducer: {
        api: apiSlice.reducer,
    },
    middleware: (getDefaultMiddleware) =>
        getDefaultMiddleware().concat(retryMiddleware),
});
```

## Retry Strategies for Different Scenarios

### 1. **Idempotent Operations**

```typescript
// For operations that can be safely retried (GET, PUT, DELETE)
export const fetchUserProfile = createAsyncThunk(
    'user/fetchProfile',
    async (userId: string, { rejectWithValue }) => {
        const maxRetries = 3;

        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                const response = await fetch(`/api/users/${userId}`);

                if (response.status === 404) {
                    // Don't retry 404s - user doesn't exist
                    return rejectWithValue('User not found');
                }

                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}`);
                }

                return await response.json();
            } catch (error) {
                if (!isRetryableError(error) || attempt === maxRetries) {
                    return rejectWithValue(error.message);
                }

                await new Promise(resolve =>
                    setTimeout(resolve, calculateDelay(attempt))
                );
            }
        }
    }
);
```

### 2. **Non-Idempotent Operations**

```typescript
// For operations that shouldn't be blindly retried (POST)
export const createOrder = createAsyncThunk(
    'order/create',
    async (orderData: any, { rejectWithValue }) => {
        try {
            const response = await fetch('/api/orders', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(orderData),
            });

            if (!response.ok) {
                // For POST operations, be more conservative with retries
                if (response.status >= 500) {
                    // Only retry server errors
                    throw new Error(`HTTP ${response.status}`);
                }

                // Don't retry client errors for POST
                return rejectWithValue(`Failed to create order: ${response.status}`);
            }

            return await response.json();
        } catch (error) {
            // Only retry network errors for POST operations
            if (error.name === 'TypeError' && error.message.includes('fetch')) {
                throw error; // Will be caught by retry logic
            }

            return rejectWithValue('Network error');
        }
    }
);
```

### 3. **Conditional Retry Based on Response**

```typescript
export const updateUser = createAsyncThunk(
    'user/update',
    async ({ userId, updates }: { userId: string; updates: any }, { rejectWithValue }) => {
        const maxRetries = 3;

        for (let attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                const response = await fetch(`/api/users/${userId}`, {
                    method: 'PATCH',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(updates),
                });

                if (response.status === 409) {
                    // Conflict - don't retry
                    const conflictData = await response.json();
                    return rejectWithValue({
                        type: 'conflict',
                        message: 'Data was modified by another user',
                        conflictData,
                    });
                }

                if (response.status === 422) {
                    // Validation error - don't retry
                    const validationErrors = await response.json();
                    return rejectWithValue({
                        type: 'validation',
                        message: 'Invalid data',
                        errors: validationErrors,
                    });
                }

                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}`);
                }

                return await response.json();
            } catch (error) {
                if (!isRetryableError(error) || attempt === maxRetries) {
                    return rejectWithValue(error.message);
                }

                await new Promise(resolve =>
                    setTimeout(resolve, calculateDelay(attempt))
                );
            }
        }
    }
);
```

## Component Patterns for Retry Logic

### 1. **Retry Hook**

```typescript
function useRetryableAction<T>(
    asyncThunk: any,
    options: {
        maxRetries?: number;
        retryDelay?: number;
        onRetry?: (attempt: number) => void;
    } = {}
) {
    const dispatch = useDispatch();
    const [retryState, setRetryState] = useState({
        isRetrying: false,
        attempt: 0,
        canRetry: true,
    });

    const execute = useCallback(async (payload?: T) => {
        setRetryState({ isRetrying: true, attempt: 0, canRetry: true });

        for (let attempt = 0; attempt <= (options.maxRetries || 3); attempt++) {
            try {
                setRetryState(prev => ({ ...prev, attempt: attempt + 1 }));

                if (attempt > 0) {
                    options.onRetry?.(attempt);
                }

                const result = await dispatch(asyncThunk(payload)).unwrap();
                setRetryState({ isRetrying: false, attempt: 0, canRetry: true });
                return result;
            } catch (error) {
                if (attempt === (options.maxRetries || 3)) {
                    setRetryState({ isRetrying: false, attempt: 0, canRetry: false });
                    throw error;
                }

                const delay = options.retryDelay || calculateDelay(attempt);
                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }
    }, [dispatch, asyncThunk, options]);

    return { execute, ...retryState };
}

// Usage
function UserProfile({ userId }: { userId: string }) {
    const { execute, isRetrying, attempt, canRetry } = useRetryableAction(
        fetchUserProfile,
        {
            maxRetries: 3,
            onRetry: (attempt) => console.log(`Retrying... attempt ${attempt}`),
        }
    );

    const handleFetch = () => execute(userId);

    return (
        <div>
            <button onClick={handleFetch} disabled={isRetrying}>
                {isRetrying ? `Retrying... (${attempt})` : 'Fetch Profile'}
            </button>
            {!canRetry && <div>Failed after multiple attempts</div>}
        </div>
    );
}
```

### 2. **Retry Button Component**

```typescript
interface RetryButtonProps {
    onRetry: () => void;
    loading: boolean;
    error: string | null;
    retryCount: number;
    maxRetries?: number;
    children: React.ReactNode;
}

function RetryButton({
    onRetry,
    loading,
    error,
    retryCount,
    maxRetries = 3,
    children,
}: RetryButtonProps) {
    const canRetry = error && retryCount < maxRetries;

    if (!canRetry) {
        return <>{children}</>;
    }

    return (
        <div>
            {children}
            <div className="error-message">
                {error}
                <button
                    onClick={onRetry}
                    disabled={loading}
                    className="retry-button"
                >
                    {loading ? 'Retrying...' : `Retry (${retryCount}/${maxRetries})`}
                </button>
            </div>
        </div>
    );
}

// Usage
function DataDisplay() {
    const dispatch = useDispatch();
    const { data, loading, error, retryCount } = useSelector((state: RootState) => state.api);

    const handleRetry = () => {
        dispatch(fetchDataWithRetry());
    };

    return (
        <RetryButton
            onRetry={handleRetry}
            loading={loading}
            error={error}
            retryCount={retryCount}
        >
            <div>{data ? JSON.stringify(data) : 'No data'}</div>
        </RetryButton>
    );
}
```

## Best Practices

### 1. **Retry Configuration**

```typescript
// Centralized retry configuration
const RETRY_CONFIG = {
    maxRetries: 3,
    baseDelay: 1000,
    maxDelay: 30000,
    backoffFactor: 2,
    jitter: true,
} as const;

const calculateRetryDelay = (attempt: number): number => {
    const delay = RETRY_CONFIG.baseDelay * Math.pow(RETRY_CONFIG.backoffFactor, attempt);

    if (RETRY_CONFIG.jitter) {
        const jitterAmount = delay * 0.1 * Math.random();
        return Math.min(delay + jitterAmount, RETRY_CONFIG.maxDelay);
    }

    return Math.min(delay, RETRY_CONFIG.maxDelay);
};
```

### 2. **Error Classification**

```typescript
enum ErrorType {
    NETWORK = 'network',
    SERVER = 'server',
    CLIENT = 'client',
    TIMEOUT = 'timeout',
    RATE_LIMIT = 'rate_limit',
}

const classifyError = (error: any): ErrorType => {
    if (error.name === 'AbortError') return ErrorType.TIMEOUT;
    if (error.name === 'TypeError' && error.message.includes('fetch')) return ErrorType.NETWORK;

    if (error.message?.includes('HTTP')) {
        const status = parseInt(error.message.split(' ')[1]);

        if (status === 429) return ErrorType.RATE_LIMIT;
        if (status >= 500) return ErrorType.SERVER;
        if (status >= 400) return ErrorType.CLIENT;
    }

    return ErrorType.NETWORK;
};

const shouldRetryError = (errorType: ErrorType): boolean => {
    switch (errorType) {
        case ErrorType.NETWORK:
        case ErrorType.SERVER:
        case ErrorType.TIMEOUT:
            return true;
        case ErrorType.RATE_LIMIT:
            return true; // With longer delay
        case ErrorType.CLIENT:
            return false;
        default:
            return false;
    }
};
```

### 3. **Retry Analytics and Monitoring**

```typescript
interface RetryMetrics {
    operation: string;
    attempts: number;
    totalDelay: number;
    success: boolean;
    errorType: ErrorType;
    timestamp: number;
}

const retryMetrics: RetryMetrics[] = [];

const trackRetry = (metrics: RetryMetrics) => {
    retryMetrics.push(metrics);

    // Send to analytics service
    if (typeof window !== 'undefined' && (window as any).gtag) {
        (window as any).gtag('event', 'retry_attempt', {
            operation: metrics.operation,
            attempts: metrics.attempts,
            success: metrics.success,
        });
    }
};

// Enhanced thunk with metrics
export const fetchDataWithMetrics = createAsyncThunk(
    'api/fetchDataMetrics',
    async (_, { rejectWithValue }) => {
        const startTime = Date.now();
        let attempts = 0;
        let totalDelay = 0;

        while (attempts <= RETRY_CONFIG.maxRetries) {
            attempts++;

            try {
                const response = await fetch('/api/data');
                if (!response.ok) throw new Error(`HTTP ${response.status}`);

                const data = await response.json();

                // Track successful retry
                trackRetry({
                    operation: 'fetchData',
                    attempts,
                    totalDelay,
                    success: true,
                    errorType: ErrorType.NETWORK,
                    timestamp: Date.now(),
                });

                return data;
            } catch (error) {
                const errorType = classifyError(error);

                if (!shouldRetryError(errorType) || attempts > RETRY_CONFIG.maxRetries) {
                    // Track failed retry
                    trackRetry({
                        operation: 'fetchData',
                        attempts,
                        totalDelay,
                        success: false,
                        errorType,
                        timestamp: Date.now(),
                    });

                    return rejectWithValue({
                        message: error.message,
                        attempts,
                        errorType,
                    });
                }

                const delay = calculateRetryDelay(attempts - 1);
                totalDelay += delay;
                await new Promise(resolve => setTimeout(resolve, delay));
            }
        }
    }
);
```

### 4. **Testing Retry Logic**

```typescript
describe('retry logic', () => {
    it('retries on network errors', async () => {
        let attempts = 0;
        const mockFetch = jest.fn(() => {
            attempts++;
            if (attempts < 3) {
                return Promise.reject(new TypeError('Failed to fetch'));
            }
            return Promise.resolve({ ok: true, json: () => Promise.resolve({ data: 'success' }) });
        });

        global.fetch = mockFetch;

        const result = await store.dispatch(fetchDataWithRetry()).unwrap();

        expect(result.data).toBe('success');
        expect(mockFetch).toHaveBeenCalledTimes(3);
    });

    it('gives up after max retries', async () => {
        const mockFetch = jest.fn(() =>
            Promise.reject(new TypeError('Failed to fetch'))
        );

        global.fetch = mockFetch;

        await expect(
            store.dispatch(fetchDataWithRetry()).unwrap()
        ).rejects.toMatchObject({
            message: 'Failed after multiple attempts',
        });

        expect(mockFetch).toHaveBeenCalledTimes(4); // initial + 3 retries
    });

    it('does not retry on 4xx errors', async () => {
        const mockFetch = jest.fn(() =>
            Promise.reject(new Error('HTTP 400'))
        );

        global.fetch = mockFetch;

        await expect(
            store.dispatch(fetchDataWithRetry()).unwrap()
        ).rejects.toMatchObject({
            message: 'Non-retryable error',
        });

        expect(mockFetch).toHaveBeenCalledTimes(1); // no retries
    });
});
```

## Common Interview Questions

### Q: How do you implement retry logic in Redux applications?

**A:** I use `createAsyncThunk` with a retry loop that catches errors and retries with exponential backoff. I classify errors to determine what's retryable (network errors, 5xx responses) vs not (4xx responses, validation errors). For complex scenarios, I implement circuit breakers to prevent cascading failures.

### Q: What's the difference between different retry strategies?

**A:** Fixed delay retries wait the same time between attempts. Exponential backoff increases the delay exponentially to reduce server load. Jitter adds randomness to prevent thundering herd problems. Circuit breakers temporarily stop retrying when failure rates are high.

### Q: When should you not implement retry logic?

**A:** Don't retry on client errors (4xx status codes), authentication failures, or validation errors. Also avoid retrying non-idempotent operations like POST requests that create resources, as this could lead to duplicate data.

### Q: How do you handle retry logic in production?

**A:** I implement proper monitoring and metrics to track retry success rates, implement circuit breakers to prevent system overload, and use different retry strategies based on error types. I also ensure retries don't cause UI blocking and provide user feedback about retry attempts.

## Summary

**Key Retry Patterns:**
1. **Basic Retry**: Simple loop with fixed delay
2. **Exponential Backoff**: Increasing delays with jitter
3. **Circuit Breaker**: Prevent cascading failures
4. **Conditional Retry**: Based on error classification
5. **Middleware-Based**: Automatic retries at the Redux level

**Best Practices:**
- Classify errors to determine retry eligibility
- Use exponential backoff with jitter
- Implement circuit breakers for resilience
- Track metrics and monitor retry success
- Provide user feedback during retries
- Test retry logic thoroughly

**Interview Tip:** "I implement retry logic in Redux using `createAsyncThunk` with a retry loop that uses exponential backoff. I classify errors to only retry network and server errors, not client errors. For production systems, I add circuit breakers and monitoring to ensure system stability."