# What is an Error Boundary? Can it catch errors inside event handlers?

## Question
What is an Error Boundary? Can it catch errors inside event handlers?

## Answer

Error boundaries are React components that catch JavaScript errors anywhere in their child component tree, log those errors, and display a fallback UI instead of crashing the entire component tree.

## What is an Error Boundary?

Error boundaries are special React components that catch errors during rendering, in lifecycle methods, and in constructors of the entire subtree below them.

### Key Characteristics:
- **Catch errors in subtree**: Errors in any descendant component
- **Display fallback UI**: Show alternative content when errors occur
- **Prevent crashes**: Stop error propagation to parent components
- **Logging capability**: Can log error information for debugging

### Error Boundary Implementation:

```javascript
import React from 'react';

class ErrorBoundary extends React.Component {
    constructor(props) {
        super(props);
        this.state = { hasError: false, error: null, errorInfo: null };
    }

    static getDerivedStateFromError(error) {
        // Update state so the next render will show the fallback UI
        return { hasError: true };
    }

    componentDidCatch(error, errorInfo) {
        // Log the error to an error reporting service
        console.error('Error caught by boundary:', error, errorInfo);

        this.setState({
            error: error,
            errorInfo: errorInfo
        });

        // You can also log to external services here
        // logErrorToService(error, errorInfo);
    }

    render() {
        if (this.state.hasError) {
            // Fallback UI
            return (
                <div className="error-boundary">
                    <h2>Something went wrong.</h2>
                    <details style={{ whiteSpace: 'pre-wrap' }}>
                        {this.state.error && this.state.error.toString()}
                        <br />
                        {this.state.errorInfo.componentStack}
                    </details>
                    <button onClick={() => this.setState({ hasError: false })}>
                        Try again
                    </button>
                </div>
            );
        }

        return this.props.children;
    }
}
```

## Can Error Boundaries Catch Errors in Event Handlers?

**No, error boundaries cannot catch errors that occur inside event handlers.**

### Why Event Handler Errors Aren't Caught:

Error boundaries only catch errors during:
- Render phase
- Component lifecycle methods
- Constructor

Event handlers run outside the render cycle and are not part of the component tree rendering process.

### Event Handler Error Behavior:

```javascript
function ProblematicComponent() {
    const handleClick = () => {
        // This error will NOT be caught by error boundaries
        throw new Error('Error in event handler!');
    };

    const handleAsyncError = async () => {
        try {
            // This will throw an unhandled promise rejection
            await fetch('/api/fail');
        } catch (error) {
            // Error boundaries won't catch this either
            throw error;
        }
    };

    return (
        <div>
            <button onClick={handleClick}>Click me (will crash)</button>
            <button onClick={handleAsyncError}>Async error</button>
        </div>
    );
}

// Wrapping with ErrorBoundary won't help for event handler errors
function App() {
    return (
        <ErrorBoundary>
            <ProblematicComponent />
        </ErrorBoundary>
    );
}
```

## How to Handle Event Handler Errors

### 1. **Try-Catch in Event Handlers**

```javascript
function SafeComponent() {
    const handleClick = () => {
        try {
            // Risky operation
            riskyFunction();
        } catch (error) {
            console.error('Error in event handler:', error);
            // Handle error gracefully
            alert('Something went wrong. Please try again.');
        }
    };

    return (
        <button onClick={handleClick}>Safe Click</button>
    );
}
```

### 2. **Error Boundary for Event Handler Errors**

Create a custom hook that uses error boundaries for event handlers:

```javascript
import { useCallback, useState } from 'react';

function useErrorHandler() {
    const [error, setError] = useState(null);

    const handleError = useCallback((error, errorInfo) => {
        console.error('Handled error:', error, errorInfo);
        setError(error);
    }, []);

    const resetError = useCallback(() => {
        setError(null);
    }, []);

    const withErrorHandling = useCallback((fn) => {
        return (...args) => {
            try {
                const result = fn(...args);
                // Handle promises
                if (result && typeof result.catch === 'function') {
                    return result.catch(error => {
                        handleError(error);
                        throw error; // Re-throw if needed
                    });
                }
                return result;
            } catch (error) {
                handleError(error);
                throw error; // Re-throw if needed
            }
        };
    }, [handleError]);

    return { error, resetError, withErrorHandling };
}

function ComponentWithErrorHandling() {
    const { error, resetError, withErrorHandling } = useErrorHandler();

    const handleClick = withErrorHandling(() => {
        // This error will be caught and handled
        throw new Error('Event handler error!');
    });

    const handleAsyncClick = withErrorHandling(async () => {
        const response = await fetch('/api/data');
        if (!response.ok) {
            throw new Error('API call failed');
        }
        return response.json();
    });

    if (error) {
        return (
            <div className="error-ui">
                <h3>An error occurred</h3>
                <p>{error.message}</p>
                <button onClick={resetError}>Try again</button>
            </div>
        );
    }

    return (
        <div>
            <button onClick={handleClick}>Test Error</button>
            <button onClick={handleAsyncClick}>Test Async Error</button>
        </div>
    );
}
```

### 3. **Global Error Handler for Unhandled Errors**

```javascript
// Set up global error handlers
window.addEventListener('error', (event) => {
    console.error('Global error:', event.error);
    // Log to error reporting service
});

window.addEventListener('unhandledrejection', (event) => {
    console.error('Unhandled promise rejection:', event.reason);
    // Log to error reporting service
});

// For React 18+ with createRoot
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
    <React.StrictMode>
        <App />
    </React.StrictMode>
);
```

## Real React Examples

### 1. **Complete Error Boundary with Logging**

```javascript
import React from 'react';
import * as Sentry from '@sentry/react'; // Error reporting service

class ErrorBoundary extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
            hasError: false,
            errorId: null,
            retryCount: 0
        };
    }

    static getDerivedStateFromError(error) {
        return { hasError: true };
    }

    componentDidCatch(error, errorInfo) {
        const errorId = `error_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;

        this.setState({ errorId });

        // Log to Sentry or other error reporting service
        Sentry.captureException(error, {
            contexts: {
                react: {
                    componentStack: errorInfo.componentStack,
                },
            },
            tags: {
                errorId,
                component: this.props.componentName || 'Unknown'
            }
        });

        // Also log to console in development
        if (process.env.NODE_ENV === 'development') {
            console.error('Error Boundary caught an error:', error, errorInfo);
        }
    }

    handleRetry = () => {
        this.setState(prevState => ({
            hasError: false,
            errorId: null,
            retryCount: prevState.retryCount + 1
        }));
    };

    render() {
        if (this.state.hasError) {
            const { FallbackComponent, fallbackProps } = this.props;

            if (FallbackComponent) {
                return (
                    <FallbackComponent
                        errorId={this.state.errorId}
                        retryCount={this.state.retryCount}
                        onRetry={this.handleRetry}
                        {...fallbackProps}
                    />
                );
            }

            return (
                <div className="error-boundary-fallback">
                    <div className="error-icon">⚠️</div>
                    <h2>Oops! Something went wrong</h2>
                    <p>We apologize for the inconvenience. Our team has been notified.</p>

                    {this.state.errorId && (
                        <p className="error-id">Error ID: {this.state.errorId}</p>
                    )}

                    <div className="error-actions">
                        <button onClick={this.handleRetry} className="retry-btn">
                            Try Again
                        </button>
                        <button onClick={() => window.location.reload()} className="reload-btn">
                            Reload Page
                        </button>
                    </div>

                    {process.env.NODE_ENV === 'development' && (
                        <details className="error-details">
                            <summary>Error Details (Development)</summary>
                            <pre>{this.state.error?.toString()}</pre>
                            <pre>{this.state.errorInfo?.componentStack}</pre>
                        </details>
                    )}
                </div>
            );
        }

        return this.props.children;
    }
}

// Usage with custom fallback
function CustomErrorFallback({ errorId, retryCount, onRetry }) {
    return (
        <div className="custom-error">
            <h3>Component Error</h3>
            <p>This component encountered an error and couldn't render properly.</p>
            {retryCount > 0 && <p>Retry attempts: {retryCount}</p>}
            <button onClick={onRetry}>Retry</button>
        </div>
    );
}

function App() {
    return (
        <ErrorBoundary componentName="App">
            <Header />
            <ErrorBoundary
                componentName="MainContent"
                FallbackComponent={CustomErrorFallback}
                fallbackProps={{ showDetails: true }}
            >
                <MainContent />
            </ErrorBoundary>
            <Footer />
        </ErrorBoundary>
    );
}
```

### 2. **Error Boundary for Data Fetching**

```javascript
function DataComponent() {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        const fetchData = async () => {
            try {
                const response = await fetch('/api/data');
                if (!response.ok) {
                    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
                }
                const result = await response.json();
                setData(result);
            } catch (error) {
                // This error will be caught by ErrorBoundary
                throw error;
            } finally {
                setLoading(false);
            }
        };

        fetchData();
    }, []);

    if (loading) return <div>Loading...</div>;

    // This will cause an error if data is null
    return <div>{data.nonexistentProperty.value}</div>;
}

// Error boundary will catch the rendering error
function App() {
    return (
        <ErrorBoundary>
            <DataComponent />
        </ErrorBoundary>
    );
}
```

### 3. **Handling Event Handler Errors in Forms**

```javascript
function ContactForm() {
    const [formData, setFormData] = useState({ name: '', email: '', message: '' });
    const [submitError, setSubmitError] = useState(null);
    const [isSubmitting, setIsSubmitting] = useState(false);

    const handleSubmit = async (event) => {
        event.preventDefault();
        setIsSubmitting(true);
        setSubmitError(null);

        try {
            // Simulate form submission
            const response = await fetch('/api/contact', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(formData)
            });

            if (!response.ok) {
                throw new Error('Failed to send message');
            }

            alert('Message sent successfully!');
            setFormData({ name: '', email: '', message: '' });
        } catch (error) {
            // Handle error gracefully - this won't be caught by error boundaries
            console.error('Form submission error:', error);
            setSubmitError(error.message);
        } finally {
            setIsSubmitting(false);
        }
    };

    const handleInputChange = (event) => {
        const { name, value } = event.target;
        setFormData(prev => ({ ...prev, [name]: value }));

        // Clear submit error when user starts typing
        if (submitError) {
            setSubmitError(null);
        }
    };

    return (
        <form onSubmit={handleSubmit}>
            <div>
                <input
                    type="text"
                    name="name"
                    value={formData.name}
                    onChange={handleInputChange}
                    placeholder="Name"
                    required
                />
            </div>

            <div>
                <input
                    type="email"
                    name="email"
                    value={formData.email}
                    onChange={handleInputChange}
                    placeholder="Email"
                    required
                />
            </div>

            <div>
                <textarea
                    name="message"
                    value={formData.message}
                    onChange={handleInputChange}
                    placeholder="Message"
                    required
                />
            </div>

            {submitError && (
                <div className="error-message">
                    Error: {submitError}
                </div>
            )}

            <button type="submit" disabled={isSubmitting}>
                {isSubmitting ? 'Sending...' : 'Send Message'}
            </button>
        </form>
    );
}
```

### 4. **Error Boundary with Recovery**

```javascript
function ResilientComponent({ data }) {
    // Component that might fail based on data
    if (!data || data.error) {
        throw new Error('Invalid data provided');
    }

    return <div>Data: {JSON.stringify(data)}</div>;
}

class RecoverableErrorBoundary extends React.Component {
    constructor(props) {
        super(props);
        this.state = { hasError: false, errorCount: 0 };
    }

    static getDerivedStateFromError(error) {
        return { hasError: true };
    }

    componentDidCatch(error, errorInfo) {
        this.setState(prevState => ({
            errorCount: prevState.errorCount + 1
        }));

        // Log error
        console.error('Component error:', error, errorInfo);
    }

    handleRecover = () => {
        this.setState({ hasError: false });
    };

    render() {
        if (this.state.hasError) {
            const { maxRetries = 3 } = this.props;

            if (this.state.errorCount >= maxRetries) {
                return (
                    <div className="permanent-error">
                        <h3>Component failed permanently</h3>
                        <p>This component has failed too many times. Please refresh the page.</p>
                        <button onClick={() => window.location.reload()}>
                            Refresh Page
                        </button>
                    </div>
                );
            }

            return (
                <div className="error-recovery">
                    <h3>Component temporarily unavailable</h3>
                    <p>Attempt {this.state.errorCount} of {maxRetries}</p>
                    <button onClick={this.handleRecover}>
                        Try Again
                    </button>
                </div>
            );
        }

        return this.props.children;
    }
}

// Usage
function App() {
    const [data, setData] = useState(null);

    useEffect(() => {
        // Simulate data that might be invalid
        setData({ error: true }); // This will cause the component to error
    }, []);

    return (
        <RecoverableErrorBoundary maxRetries={3}>
            <ResilientComponent data={data} />
        </RecoverableErrorBoundary>
    );
}
```

## Best Practices

### Error Boundary Placement:
- **Top-level**: Wrap your entire app for global error handling
- **Route-level**: Wrap route components to isolate route-specific errors
- **Component-level**: Wrap complex components that might fail
- **Avoid overuse**: Don't wrap every component - use strategically

### Error Handling Strategy:
- **Graceful degradation**: Show fallback UI instead of crashes
- **User communication**: Inform users about errors and recovery options
- **Logging**: Always log errors for debugging and monitoring
- **Recovery**: Provide ways for users to recover from errors

### Event Handler Error Handling:
- **Try-catch blocks**: Wrap risky operations in event handlers
- **User feedback**: Show error messages to users
- **Logging**: Log errors for monitoring
- **Graceful failure**: Don't crash the entire app

### Testing Error Boundaries:
```javascript
// Test error boundary behavior
const ProblematicComponent = () => {
    throw new Error('Test error');
};

// In your test
expect(() => {
    render(
        <ErrorBoundary>
            <ProblematicComponent />
        </ErrorBoundary>
    );
}).not.toThrow();
```

### Interview Tip:
*"Error boundaries catch errors in render, lifecycle methods, and constructors, but not in event handlers. For event handler errors, use try-catch blocks and provide user feedback. Place error boundaries strategically to prevent entire app crashes while allowing graceful error recovery."*