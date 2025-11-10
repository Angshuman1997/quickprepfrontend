# How to create custom Redux middleware?

## Question
How to create custom Redux middleware?

## Answer
Middleware intercepts actions before they reach the reducer. Use it for logging, async operations, or cross-cutting concerns.

## Basic Structure
```javascript
const myMiddleware = store => next => action => {
  // Do something before action reaches reducer
  console.log('Action:', action);

  // Call next to pass action to next middleware/reducer
  const result = next(action);

  // Do something after
  console.log('New state:', store.getState());

  return result;
};
```

## Logger Example
```javascript
const logger = ({ getState }) => next => action => {
  console.log('Before:', getState());
  const result = next(action);
  console.log('After:', getState());
  return result;
};
```

## Async Example (Thunk)
```javascript
const thunk = ({ dispatch, getState }) => next => action => {
  if (typeof action === 'function') {
    // If action is a function, call it with dispatch and getState
    return action(dispatch, getState);
  }
  return next(action);
};
```

## Usage
```javascript
import { createStore, applyMiddleware } from 'redux';

const store = createStore(
  rootReducer,
  applyMiddleware(logger, thunk)
);
```

## Interview Q&A

**Q: What is Redux middleware?**

A: Code that runs between dispatching an action and reaching the reducer. It can intercept, modify, or add behavior to actions.

**Q: When would you create custom middleware?**

A: For logging actions, handling async operations, error handling, or any cross-cutting concerns that affect multiple actions.

**Q: How does middleware work?**

A: Each middleware receives store, next function, and action. It can do work before/after calling next(action) to continue the chain.
    console.log('Action:', action);

    const result = next(action); // Call next middleware/reducer

    console.log('Next State:', getState());
    console.groupEnd();

    return result;
};

// Usage
import { createStore, applyMiddleware } from 'redux';
import rootReducer from './reducers';

const store = createStore(
    rootReducer,
    applyMiddleware(loggerMiddleware)
);
```

### 2. **Error Handling Middleware**

```typescript
// Middleware to catch and handle errors in actions
const errorHandlerMiddleware = ({ dispatch }) => next => action => {
    try {
        // Check if action is a function (thunk) or plain object
        if (typeof action === 'function') {
            // For thunks, wrap in try-catch
            return next(action);
        }

        return next(action);
    } catch (error) {
        console.error('Action error:', error);

        // Dispatch error action
        dispatch({
            type: 'ERROR_OCCURRED',
            payload: {
                message: error.message,
                action: action.type,
                stack: error.stack,
            },
        });

        // Re-throw to let other middleware handle it
        throw error;
    }
};
```

### 3. **Analytics Middleware**

```typescript
// Middleware to track user actions for analytics
const analyticsMiddleware = ({ getState }) => next => action => {
    // Track specific actions
    const actionsToTrack = [
        'USER_LOGIN',
        'USER_LOGOUT',
        'PRODUCT_PURCHASED',
        'PAGE_VIEWED',
    ];

    if (actionsToTrack.includes(action.type)) {
        // Send to analytics service
        analytics.track(action.type, {
            userId: getState().auth.user?.id,
            timestamp: new Date().toISOString(),
            metadata: action.payload,
        });
    }

    return next(action);
};
```

## Advanced Middleware Patterns

### 1. **Async Middleware (Thunk-like)**

```typescript
// Custom thunk middleware
const thunkMiddleware = ({ dispatch, getState }) => next => action => {
    // If action is a function, call it with dispatch and getState
    if (typeof action === 'function') {
        return action(dispatch, getState);
    }

    // Otherwise, pass to next middleware
    return next(action);
};

// Usage
const asyncAction = (userId) => async (dispatch, getState) => {
    dispatch({ type: 'FETCH_USER_START', payload: userId });

    try {
        const response = await fetch(`/api/users/${userId}`);
        const user = await response.json();

        dispatch({ type: 'FETCH_USER_SUCCESS', payload: user });
    } catch (error) {
        dispatch({ type: 'FETCH_USER_ERROR', payload: error.message });
    }
};

// Dispatch async action
store.dispatch(asyncAction(123));
```

### 2. **Retry Middleware**

```typescript
// Middleware to automatically retry failed actions
const retryMiddleware = ({ dispatch }) => next => action => {
    // Check if action should be retried
    if (action.meta?.retry) {
        const { maxRetries = 3, retryDelay = 1000 } = action.meta.retry;

        let attempts = 0;

        const executeWithRetry = async () => {
            try {
                attempts++;
                const result = await next(action);

                // If successful, return result
                return result;
            } catch (error) {
                if (attempts < maxRetries) {
                    console.log(`Retrying action ${action.type} (attempt ${attempts})`);

                    // Wait before retrying
                    await new Promise(resolve => setTimeout(resolve, retryDelay));

                    return executeWithRetry();
                } else {
                    // Max retries reached, dispatch failure action
                    dispatch({
                        type: `${action.type}_FAILED`,
                        payload: error,
                        meta: { originalAction: action },
                    });

                    throw error;
                }
            }
        };

        return executeWithRetry();
    }

    return next(action);
};

// Usage
const fetchDataAction = {
    type: 'FETCH_DATA',
    payload: { url: '/api/data' },
    meta: {
        retry: {
            maxRetries: 3,
            retryDelay: 2000,
        },
    },
};

store.dispatch(fetchDataAction);
```

### 3. **Caching Middleware**

```typescript
// Middleware to cache API responses
const cacheMiddleware = ({ getState, dispatch }) => next => action => {
    // Only cache specific actions
    if (action.type === 'API_REQUEST' && action.payload?.cache) {
        const { url, method = 'GET' } = action.payload;
        const cacheKey = `${method}_${url}`;
        const state = getState();

        // Check if cached
        if (state.cache[cacheKey]) {
            const { data, timestamp } = state.cache[cacheKey];
            const isExpired = Date.now() - timestamp > 5 * 60 * 1000; // 5 minutes

            if (!isExpired) {
                // Return cached data
                dispatch({
                    type: 'API_REQUEST_SUCCESS',
                    payload: data,
                    meta: { cached: true },
                });
                return;
            }
        }

        // Not cached or expired, proceed with request
        const result = next(action);

        // Cache successful responses
        if (result && result.then) {
            result.then(response => {
                if (response.type === 'API_REQUEST_SUCCESS') {
                    dispatch({
                        type: 'CACHE_SET',
                        payload: {
                            key: cacheKey,
                            data: response.payload,
                            timestamp: Date.now(),
                        },
                    });
                }
            });
        }

        return result;
    }

    return next(action);
};
```

### 4. **Validation Middleware**

```typescript
// Middleware to validate actions before they reach the reducer
const validationMiddleware = () => next => action => {
    // Define validation schemas
    const actionValidators = {
        'USER_UPDATE': (payload) => {
            if (!payload.id) throw new Error('User ID is required');
            if (!payload.name || payload.name.length < 2) {
                throw new Error('Name must be at least 2 characters');
            }
            if (payload.email && !payload.email.includes('@')) {
                throw new Error('Invalid email format');
            }
        },
        'PRODUCT_CREATE': (payload) => {
            if (!payload.name) throw new Error('Product name is required');
            if (!payload.price || payload.price <= 0) {
                throw new Error('Price must be greater than 0');
            }
        },
    };

    // Validate action if validator exists
    const validator = actionValidators[action.type];
    if (validator) {
        try {
            validator(action.payload);
        } catch (error) {
            console.error(`Validation failed for ${action.type}:`, error.message);

            // Dispatch validation error action
            return next({
                type: 'VALIDATION_ERROR',
                payload: {
                    action: action.type,
                    error: error.message,
                    originalPayload: action.payload,
                },
            });
        }
    }

    return next(action);
};
```

### 5. **Debounce Middleware**

```typescript
// Middleware to debounce rapid-fire actions
const debounceMiddleware = () => {
    const timeouts = new Map();

    return () => next => action => {
        // Check if action should be debounced
        if (action.meta?.debounce) {
            const { key, delay = 300 } = action.meta.debounce;

            // Clear existing timeout for this key
            if (timeouts.has(key)) {
                clearTimeout(timeouts.get(key));
            }

            // Set new timeout
            const timeout = setTimeout(() => {
                timeouts.delete(key);
                next(action);
            }, delay);

            timeouts.set(key, timeout);

            return;
        }

        return next(action);
    };
};

// Usage
const searchAction = {
    type: 'SEARCH_USERS',
    payload: { query: 'john' },
    meta: {
        debounce: {
            key: 'search',
            delay: 500,
        },
    },
};

store.dispatch(searchAction); // Will be debounced
```

## Real-World Examples

### 1. **API Middleware with Loading States**

```typescript
// Comprehensive API middleware
const apiMiddleware = ({ dispatch }) => next => async action => {
    // Check if action is an API action
    if (action.type && action.type.endsWith('_REQUEST')) {
        const baseType = action.type.replace('_REQUEST', '');
        const { url, method = 'GET', data, onSuccess, onError } = action.payload;

        // Dispatch loading action
        dispatch({ type: `${baseType}_LOADING` });

        try {
            const response = await fetch(url, {
                method,
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': `Bearer ${getAuthToken()}`,
                },
                body: data ? JSON.stringify(data) : undefined,
            });

            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }

            const result = await response.json();

            // Dispatch success action
            dispatch({
                type: `${baseType}_SUCCESS`,
                payload: result,
            });

            // Call success callback if provided
            if (onSuccess) {
                onSuccess(result);
            }

        } catch (error) {
            // Dispatch error action
            dispatch({
                type: `${baseType}_ERROR`,
                payload: error.message,
            });

            // Call error callback if provided
            if (onError) {
                onError(error);
            }
        }

        return;
    }

    return next(action);
};

// Usage
dispatch({
    type: 'FETCH_USERS_REQUEST',
    payload: {
        url: '/api/users',
        onSuccess: (users) => console.log('Users loaded:', users),
        onError: (error) => console.error('Failed to load users:', error),
    },
});
```

### 2. **Notification Middleware**

```typescript
// Middleware to show notifications for specific actions
const notificationMiddleware = () => next => action => {
    const result = next(action);

    // Define notification rules
    const notificationRules = {
        'USER_LOGIN_SUCCESS': {
            type: 'success',
            message: 'Welcome back!',
        },
        'USER_LOGOUT': {
            type: 'info',
            message: 'You have been logged out',
        },
        'SAVE_SUCCESS': {
            type: 'success',
            message: 'Changes saved successfully',
        },
        'DELETE_SUCCESS': {
            type: 'success',
            message: 'Item deleted successfully',
        },
        'ERROR': {
            type: 'error',
            message: action.payload?.message || 'An error occurred',
        },
    };

    const rule = notificationRules[action.type];
    if (rule) {
        // Show notification (using your notification library)
        showNotification({
            type: rule.type,
            message: rule.message,
            duration: rule.type === 'error' ? 5000 : 3000,
        });
    }

    return result;
};
```

### 3. **Undo/Redo Middleware**

```typescript
// Middleware to support undo/redo functionality
const undoRedoMiddleware = ({ getState, dispatch }) => {
    let history = [];
    let currentIndex = -1;

    return next => action => {
        // Skip undo/redo actions from history
        if (action.type === 'UNDO' || action.type === 'REDO') {
            return next(action);
        }

        // Save current state before action
        const prevState = getState();

        // Execute action
        const result = next(action);

        // Save action to history
        const nextState = getState();
        const historyEntry = {
            action,
            prevState,
            nextState,
        };

        // Remove any history after current index (for new branch)
        history = history.slice(0, currentIndex + 1);
        history.push(historyEntry);
        currentIndex = history.length - 1;

        return result;
    };
};

// Add undo/redo actions
const undoAction = () => (dispatch, getState) => {
    if (currentIndex > 0) {
        currentIndex--;
        const entry = history[currentIndex];
        dispatch({
            type: 'UNDO',
            payload: entry.prevState,
        });
    }
};

const redoAction = () => (dispatch, getState) => {
    if (currentIndex < history.length - 1) {
        currentIndex++;
        const entry = history[currentIndex];
        dispatch({
            type: 'REDO',
            payload: entry.nextState,
        });
    }
};
```

## Testing Custom Middleware

```typescript
import { createStore, applyMiddleware } from 'redux';
import loggerMiddleware from './loggerMiddleware';

describe('loggerMiddleware', () => {
    let store;
    let consoleSpy;

    beforeEach(() => {
        consoleSpy = jest.spyOn(console, 'log').mockImplementation(() => {});
        const reducer = (state = { count: 0 }, action) => {
            switch (action.type) {
                case 'INCREMENT':
                    return { count: state.count + 1 };
                default:
                    return state;
            }
        };

        store = createStore(reducer, applyMiddleware(loggerMiddleware));
    });

    afterEach(() => {
        consoleSpy.mockRestore();
    });

    it('logs actions and state changes', () => {
        store.dispatch({ type: 'INCREMENT' });

        expect(consoleSpy).toHaveBeenCalledWith('Action: INCREMENT');
        expect(consoleSpy).toHaveBeenCalledWith('Previous State:', { count: 0 });
        expect(consoleSpy).toHaveBeenCalledWith('Next State:', { count: 1 });
    });
});
```

## Best Practices

### 1. **Middleware Ordering**

```typescript
// Apply middleware in correct order
const store = createStore(
    rootReducer,
    applyMiddleware(
        // Early middleware (logging, validation)
        loggerMiddleware,
        validationMiddleware,

        // Async middleware
        thunkMiddleware,
        apiMiddleware,

        // Late middleware (notifications, analytics)
        notificationMiddleware,
        analyticsMiddleware,
    )
);
```

### 2. **Error Handling**

```typescript
// Always handle errors gracefully
const safeMiddleware = () => next => action => {
    try {
        return next(action);
    } catch (error) {
        console.error('Middleware error:', error);
        // Don't break the middleware chain
        return action;
    }
};
```

### 3. **Performance Considerations**

```typescript
// Avoid expensive operations in middleware
const optimizedMiddleware = () => next => action => {
    // ✅ Good: Simple checks
    if (action.type === 'EXPENSIVE_ACTION' && process.env.NODE_ENV === 'development') {
        console.time('Action processing');
        const result = next(action);
        console.timeEnd('Action processing');
        return result;
    }

    // ❌ Avoid: Expensive operations on every action
    // console.log(JSON.stringify(getState(), null, 2));

    return next(action);
};
```

### 4. **TypeScript Support**

```typescript
// Typed middleware
interface MiddlewareAPI {
    dispatch: Dispatch;
    getState: () => RootState;
}

type Middleware = (
    api: MiddlewareAPI
) => (
    next: Dispatch
) => (action: AnyAction) => any;

// Typed custom middleware
const typedMiddleware: Middleware = ({ dispatch, getState }) => next => action => {
    // TypeScript knows the types
    const state = getState();
    // ...
};
```

## Common Interview Questions

### Q: What is Redux middleware and why would you create custom middleware?

**A:** Redux middleware intercepts actions between dispatch and reducer, allowing cross-cutting concerns like logging, async operations, and validation. I'd create custom middleware for app-specific needs like API handling, caching, or analytics tracking.

### Q: How does middleware differ from reducers?

**A:** Middleware can modify actions, dispatch new actions, or prevent actions from reaching reducers. Reducers must be pure functions that only update state based on current state and action.

### Q: What's the order of middleware execution?

**A:** Middleware executes in the order applied with `applyMiddleware()`. Each calls `next(action)` to pass to the next middleware, ending with the reducer.

### Q: Can middleware prevent actions from reaching the reducer?

**A:** Yes, by not calling `next(action)`. This is useful for validation middleware that stops invalid actions or caching middleware that returns early with cached data.

## Summary

**Key Concepts:**
- Middleware intercepts actions before reducers
- Each middleware receives `{ dispatch, getState }`, `next`, and `action`
- Call `next(action)` to continue the chain
- Can dispatch new actions or modify existing ones

**Common Use Cases:**
- Logging and debugging
- Async operations (thunks, sagas)
- API calls and error handling
- Validation and sanitization
- Analytics and monitoring
- Caching and optimization

**Best Practices:**
- Keep middleware focused on single concerns
- Handle errors gracefully
- Consider performance impact
- Test middleware thoroughly
- Use TypeScript for type safety

**Interview Tip:** "Redux middleware lets me intercept actions for cross-cutting concerns. I create custom middleware for logging, API calls, validation, and analytics. Each middleware receives store API, next function, and action, allowing me to modify actions or dispatch new ones before reaching reducers."