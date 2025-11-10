# What does JSX compile to under the hood?

## Question
What does JSX compile to under the hood?

## Answer

JSX is a syntax extension for JavaScript that allows you to write HTML-like code in your JavaScript files. Under the hood, JSX gets compiled to `React.createElement()` calls or equivalent functions, which create React elements that describe what should be rendered.

## JSX Compilation Process

### 1. **JSX Source Code**
```jsx
const element = (
    <div className="container">
        <h1>Hello, {name}!</h1>
        <p>Welcome to React</p>
    </div>
);
```

### 2. **Compiled JavaScript**
```javascript
const element = React.createElement(
    'div',
    { className: 'container' },
    React.createElement('h1', null, 'Hello, ', name, '!'),
    React.createElement('p', null, 'Welcome to React')
);
```

## React.createElement Function

`React.createElement()` creates a React element object with the following structure:

```javascript
{
    type: 'div',
    props: {
        className: 'container',
        children: [
            {
                type: 'h1',
                props: {
                    children: ['Hello, ', name, '!']
                }
            },
            {
                type: 'p',
                props: {
                    children: 'Welcome to React'
                }
            }
        ]
    },
    key: null,
    ref: null
}
```

### Parameters:
- **type**: String for HTML elements, function/class for components
- **props**: Object containing attributes and children
- **children**: Remaining arguments become children

## Compilation Examples

### 1. **Simple Element**
```jsx
// JSX
<div>Hello World</div>

// Compiles to
React.createElement('div', null, 'Hello World');
```

### 2. **Element with Props**
```jsx
// JSX
<button className="btn" onClick={handleClick}>
    Click me
</button>

// Compiles to
React.createElement(
    'button',
    {
        className: 'btn',
        onClick: handleClick
    },
    'Click me'
);
```

### 3. **Component Usage**
```jsx
// JSX
<UserProfile name={user.name} age={user.age} />

// Compiles to
React.createElement(UserProfile, {
    name: user.name,
    age: user.age
});
```

### 4. **Fragment**
```jsx
// JSX
<>
    <h1>Title</h1>
    <p>Description</p>
</>

// Compiles to
React.createElement(React.Fragment, null,
    React.createElement('h1', null, 'Title'),
    React.createElement('p', null, 'Description')
);
```

### 5. **Complex Nested Structure**
```jsx
// JSX
<div className="card">
    <img src={imageUrl} alt={title} />
    <div className="content">
        <h2>{title}</h2>
        <p>{description}</p>
        <button onClick={() => onEdit(id)}>Edit</button>
    </div>
</div>

// Compiles to
React.createElement('div', { className: 'card' },
    React.createElement('img', { src: imageUrl, alt: title }),
    React.createElement('div', { className: 'content' },
        React.createElement('h2', null, title),
        React.createElement('p', null, description),
        React.createElement('button', {
            onClick: () => onEdit(id)
        }, 'Edit')
    )
);
```

## Babel Transformation

JSX is typically transformed by Babel using the `@babel/plugin-transform-react-jsx` plugin.

### Classic Runtime (React 16 and earlier):
```javascript
// Input JSX
<div className="hello">Hello World</div>

// Output with classic runtime
React.createElement('div', { className: 'hello' }, 'Hello World');
```

### Automatic Runtime (React 17+):
```javascript
// Input JSX
<div className="hello">Hello World</div>

// Output with automatic runtime
import { jsx as _jsx } from 'react/jsx-runtime';

_jsx('div', { className: 'hello', children: 'Hello World' });
```

## Custom JSX Pragma

You can customize how JSX is compiled using the `/** @jsx */` pragma:

```javascript
/** @jsx myCreateElement */

// This JSX
<div>Hello</div>

// Compiles to
myCreateElement('div', null, 'Hello');

// Instead of React.createElement
```

## Real React Examples

### 1. **Custom JSX Implementation**

```javascript
// Custom createElement function
function createElement(type, props, ...children) {
    return {
        type,
        props: {
            ...props,
            children: children.length === 1 ? children[0] : children
        }
    };
}

// Custom JSX pragma
/** @jsx createElement */

function render(element, container) {
    if (typeof element === 'string') {
        container.appendChild(document.createTextNode(element));
        return;
    }

    const domElement = document.createElement(element.type);

    // Set attributes
    Object.keys(element.props || {}).forEach(key => {
        if (key !== 'children') {
            domElement.setAttribute(key, element.props[key]);
        }
    });

    // Render children
    if (element.props.children) {
        const children = Array.isArray(element.props.children)
            ? element.props.children
            : [element.props.children];

        children.forEach(child => {
            render(child, domElement);
        });
    }

    container.appendChild(domElement);
}

// Usage
const element = (
    <div className="greeting">
        <h1>Hello World</h1>
        <p>This is a custom JSX implementation</p>
    </div>
);

render(element, document.getElementById('root'));
```

### 2. **JSX in Build Tools**

```javascript
// Webpack + Babel configuration
module.exports = {
    module: {
        rules: [
            {
                test: /\.jsx?$/,
                use: {
                    loader: 'babel-loader',
                    options: {
                        presets: ['@babel/preset-env'],
                        plugins: [
                            ['@babel/plugin-transform-react-jsx', {
                                pragma: 'React.createElement', // default
                                pragmaFrag: 'React.Fragment',   // default
                                throwIfNamespace: false         // default
                            }]
                        ]
                    }
                }
            }
        ]
    }
};
```

### 3. **TypeScript JSX**

```typescript
// TypeScript JSX
interface Props {
    name: string;
    age: number;
}

function UserCard(props: Props) {
    return (
        <div className="user-card">
            <h2>{props.name}</h2>
            <p>Age: {props.age}</p>
        </div>
    );
}

// Compiles to (with React 17+ JSX transform)
import { jsx as _jsx } from 'react/jsx-runtime';

function UserCard(props) {
    return _jsx('div', {
        className: 'user-card',
        children: [
            _jsx('h2', { children: props.name }),
            _jsx('p', { children: `Age: ${props.age}` })
        ]
    });
}
```

### 4. **JSX and Performance**

```javascript
// Inefficient - creates new function on every render
function TodoList({ todos }) {
    return (
        <ul>
            {todos.map(todo => (
                <li key={todo.id} onClick={() => handleClick(todo.id)}>
                    {todo.text}
                </li>
            ))}
        </ul>
    );
}

// What it compiles to
function TodoList({ todos }) {
    return React.createElement('ul', null,
        todos.map(todo =>
            React.createElement('li', {
                key: todo.id,
                onClick: () => handleClick(todo.id) // New function each render!
            }, todo.text)
        )
    );
}

// Optimized version
function TodoList({ todos }) {
    const handleTodoClick = useCallback((id) => {
        handleClick(id);
    }, []);

    return (
        <ul>
            {todos.map(todo => (
                <li key={todo.id} onClick={() => handleTodoClick(todo.id)}>
                    {todo.text}
                </li>
            ))}
        </ul>
    );
}
```

### 5. **Conditional JSX**

```javascript
// JSX with conditionals
function StatusMessage({ isOnline, username }) {
    return (
        <div>
            {isOnline ? (
                <span className="online">{username} is online</span>
            ) : (
                <span className="offline">{username} is offline</span>
            )}
        </div>
    );
}

// Compiles to
function StatusMessage({ isOnline, username }) {
    return React.createElement('div', null,
        isOnline
            ? React.createElement('span', { className: 'online' },
                username, ' is online'
            )
            : React.createElement('span', { className: 'offline' },
                username, ' is offline'
            )
    );
}
```

### 6. **JSX Spread Operator**

```javascript
// JSX with spread
function Input(props) {
    return <input {...props} />;
}

// Compiles to
function Input(props) {
    return React.createElement('input', props);
}
```

## JSX Transform Evolution

### React 15 and earlier:
- Always used `React.createElement`
- Required importing React in every file

### React 16:
- Introduced `React.Fragment`
- Still used `React.createElement`

### React 17:
- **Automatic JSX Runtime**: No need to import React for JSX
- New JSX transform creates `_jsx()` calls instead of `React.createElement()`
- Smaller bundle size
- Better tree shaking

```javascript
// React 17+ automatic runtime
import { jsx as _jsx } from 'react/jsx-runtime';

// Instead of
import React from 'react';
React.createElement('div', null, 'Hello');

// It generates
_jsx('div', { children: 'Hello' });
```

## Common JSX Patterns and Their Compilation

### 1. **Self-Closing Tags**
```jsx
// JSX
<img src="logo.png" alt="Logo" />

// Compiles to
React.createElement('img', { src: 'logo.png', alt: 'Logo' });
```

### 2. **Boolean Attributes**
```jsx
// JSX
<input disabled required />

// Compiles to
React.createElement('input', { disabled: true, required: true });
```

### 3. **Children as Functions (Render Props)**
```jsx
// JSX
<DataProvider>
    {data => <div>{data.name}</div>}
</DataProvider>

// Compiles to
React.createElement(DataProvider, null,
    data => React.createElement('div', null, data.name)
);
```

### 4. **Custom Components with Children**
```jsx
// JSX
<Card>
    <h1>Title</h1>
    <p>Content</p>
</Card>

// Compiles to
React.createElement(Card, null,
    React.createElement('h1', null, 'Title'),
    React.createElement('p', null, 'Content')
);
```

## Debugging JSX Compilation

### 1. **Babel REPL**
You can use the Babel REPL to see how JSX compiles:
```javascript
// Input
<div className="hello">Hello {name}!</div>

// Output
React.createElement("div", {
    className: "hello"
}, "Hello ", name, "!");
```

### 2. **Build Tool Inspection**
Check your build output or use source maps to see compiled JSX.

### 3. **React DevTools**
Use React DevTools to inspect the component tree and see how JSX elements are structured.

## Best Practices

### 1. **Use Modern JSX Transform**
```javascript
// Enable automatic JSX runtime in babel.config.js
{
    "plugins": [
        ["@babel/plugin-transform-react-jsx", {
            "runtime": "automatic"
        }]
    ]
}
```

### 2. **Key Props for Lists**
```jsx
// Always add keys for list items
{todos.map(todo => (
    <li key={todo.id}>{todo.text}</li>
))}

// Compiles with key prop
React.createElement('li', { key: todo.id }, todo.text);
```

### 3. **Avoid Inline Functions**
```jsx
// Avoid - creates new function each render
<button onClick={() => handleClick(id)}>Click</button>

// Better - use callback
<button onClick={handleClick}>Click</button>
```

### Interview Tip:
*"JSX compiles to React.createElement() calls that create React element objects describing the UI. These elements have type, props, and children properties. Modern React uses an automatic JSX runtime that generates smaller, more efficient code without requiring React imports."*