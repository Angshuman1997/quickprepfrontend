# What is forwardRef and when would you use it?

## Question
What is forwardRef and when would you use it?

## Answer

`forwardRef` is a React utility function that allows components to forward refs to their child components. This is particularly useful when you need to access DOM nodes or React component instances from parent components, especially when creating reusable component libraries or when working with higher-order components.

## What is forwardRef?

In React, refs are typically used to access DOM nodes or React component instances. However, when you create a component that wraps another component, the ref passed from the parent won't automatically reach the inner component. `forwardRef` solves this by allowing the component to receive a ref and forward it to a child component.

### Basic Syntax

```javascript
import { forwardRef } from 'react';

const MyComponent = forwardRef((props, ref) => {
    return <div ref={ref}>{props.children}</div>;
});

// Usage
function App() {
    const myRef = useRef();

    return <MyComponent ref={myRef}>Hello World</MyComponent>;
}
```

## When to Use forwardRef

### 1. **Creating Reusable UI Components**

When building component libraries where consumers need DOM access:

```javascript
// Button component that forwards refs
const Button = forwardRef(({ children, ...props }, ref) => {
    return (
        <button
            ref={ref}
            className="btn btn-primary"
            {...props}
        >
            {children}
        </button>
    );
});

// Usage in parent component
function Form() {
    const buttonRef = useRef();

    const handleSubmit = () => {
        // Direct access to button DOM node
        buttonRef.current.click();
    };

    return (
        <div>
            <Button ref={buttonRef} onClick={handleSubmit}>
                Submit
            </Button>
        </div>
    );
}
```

### 2. **Higher-Order Components (HOCs)**

When wrapping components with additional functionality:

```javascript
function withLogging(WrappedComponent) {
    const WithLogging = forwardRef((props, ref) => {
        useEffect(() => {
            console.log('Component mounted');
            return () => console.log('Component unmounted');
        }, []);

        return <WrappedComponent {...props} ref={ref} />;
    });

    WithLogging.displayName = `withLogging(${WrappedComponent.displayName || WrappedComponent.name})`;

    return WithLogging;
}

// Usage
const LoggedButton = withLogging(Button);

function App() {
    const buttonRef = useRef();

    return (
        <LoggedButton ref={buttonRef}>
            Click me
        </LoggedButton>
    );
}
```

### 3. **Custom Input Components**

For form libraries where parent components need input focus control:

```javascript
const Input = forwardRef(({
    label,
    error,
    ...props
}, ref) => {
    return (
        <div className="input-group">
            {label && <label>{label}</label>}
            <input
                ref={ref}
                className={`input ${error ? 'input-error' : ''}`}
                {...props}
            />
            {error && <span className="error-message">{error}</span>}
        </div>
    );
});

// Usage
function LoginForm() {
    const emailRef = useRef();
    const passwordRef = useRef();

    const handleSubmit = (e) => {
        e.preventDefault();

        // Focus first invalid field
        if (!emailRef.current.value) {
            emailRef.current.focus();
            return;
        }

        if (!passwordRef.current.value) {
            passwordRef.current.focus();
            return;
        }

        // Submit form
    };

    return (
        <form onSubmit={handleSubmit}>
            <Input
                ref={emailRef}
                label="Email"
                type="email"
                placeholder="Enter your email"
            />
            <Input
                ref={passwordRef}
                label="Password"
                type="password"
                placeholder="Enter your password"
            />
            <button type="submit">Login</button>
        </form>
    );
}
```

### 4. **Animation Libraries Integration**

When using animation libraries that need DOM access:

```javascript
const AnimatedDiv = forwardRef((props, ref) => {
    return (
        <div
            ref={ref}
            style={{
                transition: 'all 0.3s ease',
                ...props.style
            }}
            {...props}
        >
            {props.children}
        </div>
    );
});

// Usage with animation library
function AnimatedList() {
    const itemRefs = useRef([]);

    useEffect(() => {
        // Animate items in sequence
        itemRefs.current.forEach((ref, index) => {
            if (ref) {
                setTimeout(() => {
                    ref.style.opacity = '1';
                    ref.style.transform = 'translateY(0)';
                }, index * 100);
            }
        });
    }, []);

    const items = ['Item 1', 'Item 2', 'Item 3'];

    return (
        <div>
            {items.map((item, index) => (
                <AnimatedDiv
                    key={item}
                    ref={(el) => itemRefs.current[index] = el}
                    style={{
                        opacity: 0,
                        transform: 'translateY(20px)'
                    }}
                >
                    {item}
                </AnimatedDiv>
            ))}
        </div>
    );
}
```

## Advanced forwardRef Patterns

### 1. **Multiple Refs Forwarding**

```javascript
// Forwarding to multiple elements
const ComplexComponent = forwardRef((props, ref) => {
    const internalRef = useRef();

    useImperativeHandle(ref, () => ({
        focus: () => internalRef.current.focus(),
        scrollToTop: () => internalRef.current.scrollTo(0, 0),
        getValue: () => internalRef.current.value
    }));

    return (
        <div>
            <input ref={internalRef} {...props} />
        </div>
    );
});

// Usage
function App() {
    const complexRef = useRef();

    const handleFocus = () => {
        complexRef.current.focus();
    };

    const handleScroll = () => {
        complexRef.current.scrollToTop();
    };

    return (
        <div>
            <ComplexComponent ref={complexRef} />
            <button onClick={handleFocus}>Focus Input</button>
            <button onClick={handleScroll}>Scroll to Top</button>
        </div>
    );
}
```

### 2. **Conditional Ref Forwarding**

```javascript
const ConditionalForward = forwardRef((props, ref) => {
    const { shouldForward, children, ...otherProps } = props;

    if (shouldForward) {
        return React.cloneElement(children, {
            ...otherProps,
            ref
        });
    }

    return React.cloneElement(children, otherProps);
});

// Usage
function App() {
    const inputRef = useRef();

    return (
        <ConditionalForward shouldForward={true} ref={inputRef}>
            <input placeholder="This will get the ref" />
        </ConditionalForward>
    );
}
```

### 3. **Ref Forwarding with TypeScript**

```typescript
interface ButtonProps {
    children: React.ReactNode;
    onClick?: () => void;
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
    ({ children, onClick }, ref) => {
        return (
            <button ref={ref} onClick={onClick}>
                {children}
            </button>
        );
    }
);

// Usage with TypeScript
function App() {
    const buttonRef = useRef<HTMLButtonElement>(null);

    const handleClick = () => {
        if (buttonRef.current) {
            buttonRef.current.style.backgroundColor = 'red';
        }
    };

    return (
        <Button ref={buttonRef} onClick={handleClick}>
            Click me
        </Button>
    );
}
```

### 4. **Ref Forwarding in Compound Components**

```javascript
const SelectContext = createContext();

const Select = ({ children, value, onChange }) => {
    return (
        <SelectContext.Provider value={{ value, onChange }}>
            <div className="select">
                {children}
            </div>
        </SelectContext.Provider>
    );
};

const SelectTrigger = forwardRef((props, ref) => {
    const { value, onChange } = useContext(SelectContext);

    return (
        <button
            ref={ref}
            className="select-trigger"
            onClick={() => onChange(!value)}
            {...props}
        >
            {value ? 'Open' : 'Closed'}
        </button>
    );
});

const SelectContent = forwardRef((props, ref) => {
    const { value } = useContext(SelectContext);

    return value ? (
        <div ref={ref} className="select-content" {...props}>
            Select content
        </div>
    ) : null;
});

Select.Trigger = SelectTrigger;
Select.Content = SelectContent;

// Usage
function App() {
    const [isOpen, setIsOpen] = useState(false);
    const triggerRef = useRef();
    const contentRef = useRef();

    return (
        <Select value={isOpen} onChange={setIsOpen}>
            <Select.Trigger ref={triggerRef} />
            <Select.Content ref={contentRef} />
        </Select>
    );
}
```

## Real React Examples

### 1. **Custom Form Components**

```javascript
// Custom form components that forward refs
const TextField = forwardRef(({
    label,
    error,
    helperText,
    ...props
}, ref) => {
    const [isFocused, setIsFocused] = useState(false);

    return (
        <div className="text-field">
            {label && (
                <label className={`label ${isFocused ? 'focused' : ''}`}>
                    {label}
                </label>
            )}
            <input
                ref={ref}
                className={`input ${error ? 'error' : ''}`}
                onFocus={() => setIsFocused(true)}
                onBlur={() => setIsFocused(false)}
                {...props}
            />
            {error && <span className="error-text">{error}</span>}
            {helperText && !error && (
                <span className="helper-text">{helperText}</span>
            )}
        </div>
    );
});

const SelectField = forwardRef(({
    label,
    options,
    placeholder,
    ...props
}, ref) => {
    const [isOpen, setIsOpen] = useState(false);

    return (
        <div className="select-field">
            {label && <label>{label}</label>}
            <div className="select-wrapper">
                <select
                    ref={ref}
                    className="select"
                    onFocus={() => setIsOpen(true)}
                    onBlur={() => setIsOpen(false)}
                    {...props}
                >
                    {placeholder && (
                        <option value="" disabled>
                            {placeholder}
                        </option>
                    )}
                    {options.map(option => (
                        <option key={option.value} value={option.value}>
                            {option.label}
                        </option>
                    ))}
                </select>
                <span className="select-arrow">▼</span>
            </div>
        </div>
    );
});

// Usage in a form
function ContactForm() {
    const nameRef = useRef();
    const emailRef = useRef();
    const countryRef = useRef();

    const handleSubmit = (e) => {
        e.preventDefault();

        // Validate and focus first invalid field
        if (!nameRef.current.value) {
            nameRef.current.focus();
            return;
        }

        if (!emailRef.current.value) {
            emailRef.current.focus();
            return;
        }

        console.log('Form submitted');
    };

    return (
        <form onSubmit={handleSubmit}>
            <TextField
                ref={nameRef}
                label="Name"
                placeholder="Enter your name"
                required
            />

            <TextField
                ref={emailRef}
                label="Email"
                type="email"
                placeholder="Enter your email"
                required
            />

            <SelectField
                ref={countryRef}
                label="Country"
                placeholder="Select your country"
                options={[
                    { value: 'us', label: 'United States' },
                    { value: 'ca', label: 'Canada' },
                    { value: 'uk', label: 'United Kingdom' }
                ]}
            />

            <button type="submit">Submit</button>
        </form>
    );
}
```

### 2. **Virtualized List Component**

```javascript
const VirtualizedList = forwardRef(({
    items,
    itemHeight,
    containerHeight,
    renderItem
}, ref) => {
    const [scrollTop, setScrollTop] = useState(0);
    const containerRef = useRef();

    useImperativeHandle(ref, () => ({
        scrollToIndex: (index) => {
            if (containerRef.current) {
                containerRef.current.scrollTop = index * itemHeight;
            }
        },
        getVisibleRange: () => {
            const start = Math.floor(scrollTop / itemHeight);
            const end = Math.min(
                start + Math.ceil(containerHeight / itemHeight),
                items.length - 1
            );
            return { start, end };
        }
    }));

    const visibleItems = useMemo(() => {
        const start = Math.floor(scrollTop / itemHeight);
        const end = Math.min(
            start + Math.ceil(containerHeight / itemHeight) + 1,
            items.length
        );

        return items.slice(start, end).map((item, index) => ({
            item,
            index: start + index
        }));
    }, [items, scrollTop, itemHeight, containerHeight]);

    const handleScroll = (e) => {
        setScrollTop(e.target.scrollTop);
    };

    return (
        <div
            ref={containerRef}
            style={{
                height: containerHeight,
                overflow: 'auto'
            }}
            onScroll={handleScroll}
        >
            <div style={{ height: items.length * itemHeight }}>
                <div
                    style={{
                        transform: `translateY(${Math.floor(scrollTop / itemHeight) * itemHeight}px)`
                    }}
                >
                    {visibleItems.map(({ item, index }) => (
                        <div
                            key={index}
                            style={{ height: itemHeight }}
                        >
                            {renderItem(item, index)}
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
});

// Usage
function App() {
    const listRef = useRef();
    const items = Array.from({ length: 1000 }, (_, i) => `Item ${i + 1}`);

    const scrollToItem = () => {
        listRef.current.scrollToIndex(500);
    };

    return (
        <div>
            <button onClick={scrollToItem}>Scroll to Item 500</button>
            <VirtualizedList
                ref={listRef}
                items={items}
                itemHeight={50}
                containerHeight={400}
                renderItem={(item, index) => (
                    <div style={{ padding: '10px', border: '1px solid #ccc' }}>
                        {item}
                    </div>
                )}
            />
        </div>
    );
}
```

### 3. **Draggable Component**

```javascript
const Draggable = forwardRef(({
    children,
    onDragStart,
    onDragEnd,
    ...props
}, ref) => {
    const [isDragging, setIsDragging] = useState(false);
    const [dragOffset, setDragOffset] = useState({ x: 0, y: 0 });
    const dragRef = useRef();

    useImperativeHandle(ref, () => ({
        resetPosition: () => {
            if (dragRef.current) {
                dragRef.current.style.transform = 'translate(0, 0)';
            }
        },
        getPosition: () => {
            return dragOffset;
        }
    }));

    const handleMouseDown = (e) => {
        setIsDragging(true);
        const startX = e.clientX - dragOffset.x;
        const startY = e.clientY - dragOffset.y;

        const handleMouseMove = (e) => {
            const newX = e.clientX - startX;
            const newY = e.clientY - startY;
            setDragOffset({ x: newX, y: newY });

            if (dragRef.current) {
                dragRef.current.style.transform = `translate(${newX}px, ${newY}px)`;
            }
        };

        const handleMouseUp = () => {
            setIsDragging(false);
            document.removeEventListener('mousemove', handleMouseMove);
            document.removeEventListener('mouseup', handleMouseUp);
            onDragEnd?.(dragOffset);
        };

        document.addEventListener('mousemove', handleMouseMove);
        document.addEventListener('mouseup', handleMouseUp);
        onDragStart?.();
    };

    return (
        <div
            ref={dragRef}
            style={{
                cursor: isDragging ? 'grabbing' : 'grab',
                userSelect: 'none'
            }}
            onMouseDown={handleMouseDown}
            {...props}
        >
            {children}
        </div>
    );
});

// Usage
function App() {
    const dragRef = useRef();

    const handleDragStart = () => {
        console.log('Drag started');
    };

    const handleDragEnd = (position) => {
        console.log('Drag ended at:', position);
    };

    const resetPosition = () => {
        dragRef.current.resetPosition();
    };

    return (
        <div>
            <button onClick={resetPosition}>Reset Position</button>
            <Draggable
                ref={dragRef}
                onDragStart={handleDragStart}
                onDragEnd={handleDragEnd}
            >
                <div style={{
                    width: '100px',
                    height: '100px',
                    background: 'blue',
                    color: 'white',
                    display: 'flex',
                    alignItems: 'center',
                    justifyContent: 'center'
                }}>
                    Drag me!
                </div>
            </Draggable>
        </div>
    );
}
```

### 4. **Focus Trap Component**

```javascript
const FocusTrap = forwardRef(({
    children,
    autoFocus = true,
    ...props
}, ref) => {
    const containerRef = useRef();
    const firstFocusableRef = useRef();
    const lastFocusableRef = useRef();

    useImperativeHandle(ref, () => ({
        focusFirst: () => firstFocusableRef.current?.focus(),
        focusLast: () => lastFocusableRef.current?.focus()
    }));

    useEffect(() => {
        if (!containerRef.current) return;

        const focusableElements = containerRef.current.querySelectorAll(
            'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        );

        if (focusableElements.length > 0) {
            firstFocusableRef.current = focusableElements[0];
            lastFocusableRef.current = focusableElements[focusableElements.length - 1];

            if (autoFocus) {
                firstFocusableRef.current.focus();
            }
        }

        const handleKeyDown = (e) => {
            if (e.key === 'Tab') {
                if (e.shiftKey) {
                    if (document.activeElement === firstFocusableRef.current) {
                        lastFocusableRef.current.focus();
                        e.preventDefault();
                    }
                } else {
                    if (document.activeElement === lastFocusableRef.current) {
                        firstFocusableRef.current.focus();
                        e.preventDefault();
                    }
                }
            }
        };

        document.addEventListener('keydown', handleKeyDown);

        return () => {
            document.removeEventListener('keydown', handleKeyDown);
        };
    }, [autoFocus]);

    return (
        <div ref={containerRef} {...props}>
            {children}
        </div>
    );
});

// Usage in modal
const Modal = ({ isOpen, onClose, children }) => {
    const focusTrapRef = useRef();

    useEffect(() => {
        if (isOpen) {
            focusTrapRef.current?.focusFirst();
        }
    }, [isOpen]);

    if (!isOpen) return null;

    return createPortal(
        <div className="modal-overlay">
            <FocusTrap ref={focusTrapRef} className="modal-content">
                <button onClick={onClose}>×</button>
                {children}
            </FocusTrap>
        </div>,
        document.body
    );
};
```

## Best Practices

### 1. **Display Name for Debugging**

```javascript
const MyComponent = forwardRef((props, ref) => {
    return <div ref={ref}>{props.children}</div>;
});

MyComponent.displayName = 'MyComponent'; // Helps with debugging
```

### 2. **TypeScript Support**

```typescript
interface MyComponentProps {
    children: React.ReactNode;
}

const MyComponent = forwardRef<HTMLDivElement, MyComponentProps>(
    ({ children }, ref) => {
        return <div ref={ref}>{children}</div>;
    }
);

MyComponent.displayName = 'MyComponent';
```

### 3. **Conditional Forwarding**

```javascript
const SmartInput = forwardRef((props, ref) => {
    const { as: Component = 'input', ...otherProps } = props;

    return <Component ref={ref} {...otherProps} />;
});

// Can be used as different elements while maintaining ref forwarding
<SmartInput as="input" ref={inputRef} />
<SmartInput as="textarea" ref={textareaRef} />
```

### 4. **useImperativeHandle for Custom API**

```javascript
const VideoPlayer = forwardRef((props, ref) => {
    const videoRef = useRef();

    useImperativeHandle(ref, () => ({
        play: () => videoRef.current.play(),
        pause: () => videoRef.current.pause(),
        getCurrentTime: () => videoRef.current.currentTime,
        setCurrentTime: (time) => videoRef.current.currentTime = time
    }));

    return <video ref={videoRef} {...props} />;
});

// Usage
function App() {
    const videoRef = useRef();

    return (
        <div>
            <VideoPlayer ref={videoRef} src="video.mp4" />
            <button onClick={() => videoRef.current.play()}>Play</button>
            <button onClick={() => videoRef.current.pause()}>Pause</button>
        </div>
    );
}
```

## Common Mistakes

### 1. **Forgetting to Use forwardRef**

```javascript
// ❌ Wrong: Ref won't be forwarded
const MyComponent = ({ children }) => {
    return <div>{children}</div>;
};

// ✅ Correct: Use forwardRef
const MyComponent = forwardRef((props, ref) => {
    return <div ref={ref}>{props.children}</div>;
});
```

### 2. **Incorrect Ref Assignment**

```javascript
// ❌ Wrong: Assigning ref to wrong element
const MyComponent = forwardRef((props, ref) => {
    return (
        <div>
            <input ref={ref} /> {/* This is correct */}
        </div>
    );
});

// ✅ Correct: Forward to the actual interactive element
const MyComponent = forwardRef((props, ref) => {
    return <input ref={ref} {...props} />;
});
```

### 3. **Missing Dependencies in useImperativeHandle**

```javascript
const MyComponent = forwardRef((props, ref) => {
    const [count, setCount] = useState(0);

    useImperativeHandle(ref, () => ({
        getCount: () => count, // Stale closure if count changes
        increment: () => setCount(c => c + 1)
    })); // Missing count in dependencies

    return <div>Count: {count}</div>;
});

// ✅ Fixed: Include dependencies
useImperativeHandle(ref, () => ({
    getCount: () => count,
    increment: () => setCount(c => c + 1)
}), [count]);
```

## Testing forwardRef Components

```javascript
import { render, screen } from '@testing-library/react';

// Test ref forwarding
test('forwards ref correctly', () => {
    const ref = { current: null };
    render(<Button ref={ref}>Click me</Button>);

    expect(ref.current).toBeInstanceOf(HTMLButtonElement);
});

// Test imperative methods
test('exposes imperative methods', () => {
    const ref = { current: null };
    render(<CustomInput ref={ref} />);

    expect(typeof ref.current.focus).toBe('function');
    expect(typeof ref.current.getValue).toBe('function');
});
```

### Interview Tip:
*"forwardRef allows components to forward refs to child components, enabling parent components to access DOM nodes or imperative methods. Use it when building reusable UI libraries, HOCs, or when parent components need direct DOM access to child elements."*