# What are Portals? Practical example

## Question
What are Portals? Practical example.

## Answer

React Portals provide a way to render children into a DOM node that exists outside the DOM hierarchy of the parent component. This is particularly useful for components that need to break out of their container's boundaries, such as modals, tooltips, dropdowns, and overlays.

## What are Portals?

Portals allow you to render a component's children into a different part of the DOM tree while maintaining the component's position in the React tree. The portal's child components can still access React context and events bubble up through the React tree normally.

### Basic Syntax

```javascript
import { createPortal } from 'react-dom';

function MyComponent() {
    return createPortal(
        <div>Portal content</div>, // What to render
        document.getElementById('portal-root') // Where to render it
    );
}
```

## Why Use Portals?

### 1. **CSS Z-Index Issues**
Components rendered inside containers with `overflow: hidden` or low z-index can't appear above other elements.

### 2. **Event Bubbling**
Portals maintain React's event bubbling through the component tree, not the DOM tree.

### 3. **Accessibility**
Screen readers and other assistive technologies can properly navigate portal content.

### 4. **Styling Isolation**
Portals allow components to have their own styling context without inheriting parent styles.

## Portal Root Setup

First, add a portal root to your HTML:

```html
<!-- index.html -->
<div id="root"></div>
<div id="portal-root"></div>
```

Or create it dynamically:

```javascript
// In your app entry point
const portalRoot = document.createElement('div');
portalRoot.id = 'portal-root';
document.body.appendChild(portalRoot);
```

## Basic Portal Example

```javascript
import { createPortal } from 'react-dom';

function PortalExample() {
    return (
        <div>
            <h1>Regular content</h1>
            {createPortal(
                <div style={{
                    position: 'fixed',
                    top: '50%',
                    left: '50%',
                    transform: 'translate(-50%, -50%)',
                    background: 'white',
                    padding: '20px',
                    border: '1px solid black'
                }}>
                    This renders in portal-root!
                </div>,
                document.getElementById('portal-root')
            )}
        </div>
    );
}
```

## Practical Examples

### 1. **Modal Component**

```javascript
// Modal.js
import { createPortal } from 'react-dom';
import { useEffect } from 'react';

function Modal({ isOpen, onClose, children }) {
    useEffect(() => {
        if (isOpen) {
            // Prevent body scroll when modal is open
            document.body.style.overflow = 'hidden';
        } else {
            document.body.style.overflow = 'unset';
        }

        // Cleanup on unmount
        return () => {
            document.body.style.overflow = 'unset';
        };
    }, [isOpen]);

    if (!isOpen) return null;

    return createPortal(
        <div style={{
            position: 'fixed',
            top: 0,
            left: 0,
            right: 0,
            bottom: 0,
            backgroundColor: 'rgba(0, 0, 0, 0.5)',
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            zIndex: 1000
        }}>
            <div style={{
                background: 'white',
                padding: '20px',
                borderRadius: '8px',
                maxWidth: '500px',
                width: '90%',
                maxHeight: '80vh',
                overflow: 'auto'
            }}>
                <button
                    onClick={onClose}
                    style={{
                        float: 'right',
                        border: 'none',
                        background: 'none',
                        fontSize: '20px',
                        cursor: 'pointer'
                    }}
                >
                    ×
                </button>
                {children}
            </div>
        </div>,
        document.getElementById('portal-root')
    );
}

// Usage
function App() {
    const [isModalOpen, setIsModalOpen] = useState(false);

    return (
        <div>
            <button onClick={() => setIsModalOpen(true)}>
                Open Modal
            </button>

            <Modal isOpen={isModalOpen} onClose={() => setIsModalOpen(false)}>
                <h2>Modal Title</h2>
                <p>This modal renders in a portal!</p>
                <p>It can break out of parent containers with overflow hidden.</p>
            </Modal>
        </div>
    );
}
```

### 2. **Tooltip Component**

```javascript
// Tooltip.js
import { createPortal } from 'react-dom';
import { useState, useRef, useEffect } from 'react';

function Tooltip({ content, children, position = 'top' }) {
    const [isVisible, setIsVisible] = useState(false);
    const [tooltipPosition, setTooltipPosition] = useState({ top: 0, left: 0 });
    const triggerRef = useRef();
    const tooltipRef = useRef();

    useEffect(() => {
        if (isVisible && triggerRef.current && tooltipRef.current) {
            const triggerRect = triggerRef.current.getBoundingClientRect();
            const tooltipRect = tooltipRef.current.getBoundingClientRect();

            let top, left;

            switch (position) {
                case 'top':
                    top = triggerRect.top - tooltipRect.height - 8;
                    left = triggerRect.left + (triggerRect.width / 2) - (tooltipRect.width / 2);
                    break;
                case 'bottom':
                    top = triggerRect.bottom + 8;
                    left = triggerRect.left + (triggerRect.width / 2) - (tooltipRect.width / 2);
                    break;
                case 'left':
                    top = triggerRect.top + (triggerRect.height / 2) - (tooltipRect.height / 2);
                    left = triggerRect.left - tooltipRect.width - 8;
                    break;
                case 'right':
                    top = triggerRect.top + (triggerRect.height / 2) - (tooltipRect.height / 2);
                    left = triggerRect.right + 8;
                    break;
            }

            setTooltipPosition({ top, left });
        }
    }, [isVisible, position]);

    return (
        <>
            <span
                ref={triggerRef}
                onMouseEnter={() => setIsVisible(true)}
                onMouseLeave={() => setIsVisible(false)}
                style={{ cursor: 'help' }}
            >
                {children}
            </span>

            {isVisible && createPortal(
                <div
                    ref={tooltipRef}
                    style={{
                        position: 'fixed',
                        top: tooltipPosition.top,
                        left: tooltipPosition.left,
                        background: 'black',
                        color: 'white',
                        padding: '8px 12px',
                        borderRadius: '4px',
                        fontSize: '14px',
                        whiteSpace: 'nowrap',
                        zIndex: 9999,
                        pointerEvents: 'none'
                    }}
                >
                    {content}
                </div>,
                document.body
            )}
        </>
    );
}

// Usage
function App() {
    return (
        <div>
            <p>
                Hover over this <Tooltip content="This is a helpful tooltip!">text</Tooltip> to see a tooltip.
            </p>
            <p>
                This tooltip can appear <Tooltip content="Above the text" position="top">above</Tooltip> or{' '}
                <Tooltip content="Below the text" position="bottom">below</Tooltip> the trigger.
            </p>
        </div>
    );
}
```

### 3. **Dropdown Menu**

```javascript
// Dropdown.js
import { createPortal } from 'react-dom';
import { useState, useRef, useEffect } from 'react';

function Dropdown({ trigger, children, align = 'left' }) {
    const [isOpen, setIsOpen] = useState(false);
    const [menuPosition, setMenuPosition] = useState({ top: 0, left: 0 });
    const triggerRef = useRef();
    const menuRef = useRef();

    useEffect(() => {
        if (isOpen && triggerRef.current && menuRef.current) {
            const triggerRect = triggerRef.current.getBoundingClientRect();
            const menuRect = menuRef.current.getBoundingClientRect();

            const top = triggerRect.bottom + 4;
            let left;

            if (align === 'left') {
                left = triggerRect.left;
            } else if (align === 'right') {
                left = triggerRect.right - menuRect.width;
            } else { // center
                left = triggerRect.left + (triggerRect.width / 2) - (menuRect.width / 2);
            }

            setMenuPosition({ top, left });
        }
    }, [isOpen, align]);

    useEffect(() => {
        const handleClickOutside = (event) => {
            if (
                triggerRef.current &&
                !triggerRef.current.contains(event.target) &&
                menuRef.current &&
                !menuRef.current.contains(event.target)
            ) {
                setIsOpen(false);
            }
        };

        if (isOpen) {
            document.addEventListener('mousedown', handleClickOutside);
        }

        return () => {
            document.removeEventListener('mousedown', handleClickOutside);
        };
    }, [isOpen]);

    return (
        <>
            <div
                ref={triggerRef}
                onClick={() => setIsOpen(!isOpen)}
                style={{ cursor: 'pointer' }}
            >
                {trigger}
            </div>

            {isOpen && createPortal(
                <div
                    ref={menuRef}
                    style={{
                        position: 'fixed',
                        top: menuPosition.top,
                        left: menuPosition.left,
                        background: 'white',
                        border: '1px solid #ccc',
                        borderRadius: '4px',
                        boxShadow: '0 2px 8px rgba(0,0,0,0.1)',
                        zIndex: 1000,
                        minWidth: '200px'
                    }}
                >
                    {children}
                </div>,
                document.body
            )}
        </>
    );
}

// Usage
function App() {
    return (
        <div>
            <Dropdown
                trigger={<button>Menu ▼</button>}
                align="left"
            >
                <div style={{ padding: '8px 16px', cursor: 'pointer' }}>Profile</div>
                <div style={{ padding: '8px 16px', cursor: 'pointer' }}>Settings</div>
                <div style={{ padding: '8px 16px', cursor: 'pointer', borderTop: '1px solid #eee' }}>Logout</div>
            </Dropdown>
        </div>
    );
}
```

### 4. **Notification System**

```javascript
// NotificationContainer.js
import { createPortal } from 'react-dom';
import { useState, useEffect } from 'react';

const NotificationContext = React.createContext();

function NotificationProvider({ children }) {
    const [notifications, setNotifications] = useState([]);

    const addNotification = (message, type = 'info', duration = 5000) => {
        const id = Date.now();
        const notification = { id, message, type, duration };

        setNotifications(prev => [...prev, notification]);

        if (duration > 0) {
            setTimeout(() => {
                removeNotification(id);
            }, duration);
        }

        return id;
    };

    const removeNotification = (id) => {
        setNotifications(prev => prev.filter(n => n.id !== id));
    };

    return (
        <NotificationContext.Provider value={{ addNotification, removeNotification }}>
            {children}
            <NotificationContainer notifications={notifications} onRemove={removeNotification} />
        </NotificationContext.Provider>
    );
}

function NotificationContainer({ notifications, onRemove }) {
    if (notifications.length === 0) return null;

    return createPortal(
        <div style={{
            position: 'fixed',
            top: '20px',
            right: '20px',
            zIndex: 9999,
            maxWidth: '400px'
        }}>
            {notifications.map(notification => (
                <NotificationItem
                    key={notification.id}
                    notification={notification}
                    onRemove={onRemove}
                />
            ))}
        </div>,
        document.body
    );
}

function NotificationItem({ notification, onRemove }) {
    const [isVisible, setIsVisible] = useState(false);

    useEffect(() => {
        // Trigger animation
        setTimeout(() => setIsVisible(true), 10);
    }, []);

    const handleClose = () => {
        setIsVisible(false);
        setTimeout(() => onRemove(notification.id), 300); // Wait for animation
    };

    const getBackgroundColor = () => {
        switch (notification.type) {
            case 'success': return '#d4edda';
            case 'error': return '#f8d7da';
            case 'warning': return '#fff3cd';
            default: return '#d1ecf1';
        }
    };

    return (
        <div style={{
            background: getBackgroundColor(),
            border: `1px solid ${notification.type === 'success' ? '#c3e6cb' : '#bee5eb'}`,
            borderRadius: '4px',
            padding: '12px 16px',
            marginBottom: '8px',
            boxShadow: '0 2px 4px rgba(0,0,0,0.1)',
            transform: isVisible ? 'translateX(0)' : 'translateX(100%)',
            transition: 'transform 0.3s ease-out',
            display: 'flex',
            justifyContent: 'space-between',
            alignItems: 'center'
        }}>
            <span>{notification.message}</span>
            <button
                onClick={handleClose}
                style={{
                    background: 'none',
                    border: 'none',
                    fontSize: '18px',
                    cursor: 'pointer',
                    marginLeft: '12px'
                }}
            >
                ×
            </button>
        </div>
    );
}

// Usage
function App() {
    const { addNotification } = useContext(NotificationContext);

    return (
        <div>
            <button onClick={() => addNotification('Profile updated successfully!', 'success')}>
                Show Success
            </button>
            <button onClick={() => addNotification('Failed to save changes.', 'error')}>
                Show Error
            </button>
            <button onClick={() => addNotification('Please check your input.', 'warning')}>
                Show Warning
            </button>
        </div>
    );
}
```

### 5. **Context Menu**

```javascript
// ContextMenu.js
import { createPortal } from 'react-dom';
import { useState, useEffect, useRef } from 'react';

function ContextMenu({ children, menuItems }) {
    const [isVisible, setIsVisible] = useState(false);
    const [position, setPosition] = useState({ x: 0, y: 0 });
    const containerRef = useRef();
    const menuRef = useRef();

    useEffect(() => {
        const handleContextMenu = (e) => {
            e.preventDefault();
            const rect = containerRef.current.getBoundingClientRect();
            const x = e.clientX - rect.left;
            const y = e.clientY - rect.top;

            setPosition({ x, y });
            setIsVisible(true);
        };

        const handleClickOutside = (e) => {
            if (menuRef.current && !menuRef.current.contains(e.target)) {
                setIsVisible(false);
            }
        };

        const container = containerRef.current;
        if (container) {
            container.addEventListener('contextmenu', handleContextMenu);
            document.addEventListener('mousedown', handleClickOutside);
        }

        return () => {
            if (container) {
                container.removeEventListener('contextmenu', handleContextMenu);
            }
            document.removeEventListener('mousedown', handleClickOutside);
        };
    }, []);

    useEffect(() => {
        if (isVisible && menuRef.current) {
            const menuRect = menuRef.current.getBoundingClientRect();
            const viewportWidth = window.innerWidth;
            const viewportHeight = window.innerHeight;

            // Adjust position if menu would go off-screen
            let adjustedX = position.x;
            let adjustedY = position.y;

            if (position.x + menuRect.width > viewportWidth) {
                adjustedX = viewportWidth - menuRect.width - 10;
            }

            if (position.y + menuRect.height > viewportHeight) {
                adjustedY = viewportHeight - menuRect.height - 10;
            }

            setPosition({ x: adjustedX, y: adjustedY });
        }
    }, [isVisible, position]);

    return (
        <div ref={containerRef} style={{ width: '100%', height: '200px', background: '#f0f0f0' }}>
            {children}

            {isVisible && createPortal(
                <div
                    ref={menuRef}
                    style={{
                        position: 'fixed',
                        left: position.x,
                        top: position.y,
                        background: 'white',
                        border: '1px solid #ccc',
                        borderRadius: '4px',
                        boxShadow: '0 2px 8px rgba(0,0,0,0.1)',
                        zIndex: 1000,
                        minWidth: '150px'
                    }}
                >
                    {menuItems.map((item, index) => (
                        <div
                            key={index}
                            onClick={() => {
                                item.onClick();
                                setIsVisible(false);
                            }}
                            style={{
                                padding: '8px 16px',
                                cursor: 'pointer',
                                borderBottom: index < menuItems.length - 1 ? '1px solid #eee' : 'none'
                            }}
                        >
                            {item.label}
                        </div>
                    ))}
                </div>,
                document.body
            )}
        </div>
    );
}

// Usage
function App() {
    const menuItems = [
        { label: 'Copy', onClick: () => console.log('Copy clicked') },
        { label: 'Paste', onClick: () => console.log('Paste clicked') },
        { label: 'Delete', onClick: () => console.log('Delete clicked') }
    ];

    return (
        <ContextMenu menuItems={menuItems}>
            <p>Right-click anywhere in this area to see the context menu.</p>
        </ContextMenu>
    );
}
```

## Advanced Portal Patterns

### 1. **Portal with Event Bubbling**

```javascript
// Events still bubble through React tree, not DOM tree
function PortalWithEvents() {
    const [count, setCount] = useState(0);

    const handlePortalClick = () => {
        console.log('Portal clicked');
    };

    const handleParentClick = () => {
        console.log('Parent clicked');
        setCount(c => c + 1);
    };

    return (
        <div onClick={handleParentClick}>
            <p>Parent component (clicked {count} times)</p>

            {createPortal(
                <button onClick={handlePortalClick}>
                    Portal Button
                </button>,
                document.getElementById('portal-root')
            )}
        </div>
    );
}
```

### 2. **Conditional Portal Rendering**

```javascript
function ConditionalPortal({ shouldUsePortal, children }) {
    if (shouldUsePortal) {
        return createPortal(
            children,
            document.getElementById('portal-root')
        );
    }

    return children;
}
```

### 3. **Portal with Context**

```javascript
// Context works through portals
const ThemeContext = createContext('light');

function ThemedPortal() {
    return (
        <ThemeContext.Provider value="dark">
            <div>
                <p>Regular content (dark theme)</p>

                {createPortal(
                    <ThemeContext.Consumer>
                        {theme => <p>Portal content ({theme} theme)</p>}
                    </ThemeContext.Consumer>,
                    document.getElementById('portal-root')
                )}
            </div>
        </ThemeContext.Provider>
    );
}
```

## Portal Best Practices

### 1. **Portal Root Management**

```javascript
// Create portal root if it doesn't exist
function ensurePortalRoot() {
    let portalRoot = document.getElementById('portal-root');
    if (!portalRoot) {
        portalRoot = document.createElement('div');
        portalRoot.id = 'portal-root';
        document.body.appendChild(portalRoot);
    }
    return portalRoot;
}

function MyPortal({ children }) {
    const [portalRoot, setPortalRoot] = useState(null);

    useEffect(() => {
        setPortalRoot(ensurePortalRoot());
    }, []);

    if (!portalRoot) return null;

    return createPortal(children, portalRoot);
}
```

### 2. **Accessibility Considerations**

```javascript
function AccessibleModal({ isOpen, onClose, children }) {
    useEffect(() => {
        if (isOpen) {
            // Focus management
            const focusableElements = document.querySelectorAll(
                'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
            );
            const firstElement = focusableElements[0];
            const lastElement = focusableElements[focusableElements.length - 1];

            const handleTabKey = (e) => {
                if (e.key === 'Tab') {
                    if (e.shiftKey) {
                        if (document.activeElement === firstElement) {
                            lastElement.focus();
                            e.preventDefault();
                        }
                    } else {
                        if (document.activeElement === lastElement) {
                            firstElement.focus();
                            e.preventDefault();
                        }
                    }
                }
            };

            document.addEventListener('keydown', handleTabKey);
            firstElement?.focus();

            return () => {
                document.removeEventListener('keydown', handleTabKey);
            };
        }
    }, [isOpen]);

    // ... modal rendering with portal
}
```

### 3. **Performance Considerations**

```javascript
// Only create portal when needed
function LazyPortal({ isOpen, children }) {
    const [portalRoot, setPortalRoot] = useState(null);

    useEffect(() => {
        if (isOpen && !portalRoot) {
            const root = document.createElement('div');
            root.id = `portal-${Date.now()}`;
            document.body.appendChild(root);
            setPortalRoot(root);
        }

        return () => {
            if (portalRoot && !isOpen) {
                document.body.removeChild(portalRoot);
                setPortalRoot(null);
            }
        };
    }, [isOpen, portalRoot]);

    if (!isOpen || !portalRoot) return null;

    return createPortal(children, portalRoot);
}
```

## Common Portal Use Cases

1. **Modals and Dialogs** - Break out of parent containers
2. **Tooltips and Popovers** - Position relative to viewport
3. **Dropdowns and Selects** - Avoid clipping by parent containers
4. **Notifications and Toasts** - Global positioning
5. **Context Menus** - Absolute positioning
6. **Loading Overlays** - Full-screen coverage
7. **Image Galleries/Lightboxes** - Overlay content
8. **Chat Widgets** - Floating UI elements

## Testing Portals

```javascript
import { render, screen } from '@testing-library/react';

// Test that portal content renders
test('modal renders in portal', () => {
    render(<Modal isOpen={true}>Modal content</Modal>);

    // Content appears in document
    expect(screen.getByText('Modal content')).toBeInTheDocument();

    // Check portal root exists
    const portalRoot = document.getElementById('portal-root');
    expect(portalRoot).toBeInTheDocument();
});

// Test portal cleanup
test('portal cleans up on unmount', () => {
    const { unmount } = render(<Modal isOpen={true}>Content</Modal>);

    expect(document.getElementById('portal-root')).toBeInTheDocument();

    unmount();

    // Portal root might still exist, but content should be gone
    expect(screen.queryByText('Content')).not.toBeInTheDocument();
});
```

### Interview Tip:
*"Portals render children outside the parent component's DOM hierarchy while maintaining React's event bubbling and context. Use them for modals, tooltips, and dropdowns that need to break out of parent containers with overflow hidden or z-index issues."*