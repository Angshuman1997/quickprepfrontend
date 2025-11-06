# How do you type React component props?

## Question
How do you type React component props?

## Answer

Typing React component props in TypeScript is essential for type safety, better developer experience, and catching errors at compile time. There are several patterns and best practices for typing props effectively.

## Basic Component Props Typing

### Interface for Props

```typescript
interface ButtonProps {
    children: React.ReactNode;
    onClick: () => void;
    variant?: 'primary' | 'secondary' | 'danger';
    size?: 'small' | 'medium' | 'large';
    disabled?: boolean;
}

const Button: React.FC<ButtonProps> = ({
    children,
    onClick,
    variant = 'primary',
    size = 'medium',
    disabled = false
}) => {
    return (
        <button
            className={`btn btn-${variant} btn-${size}`}
            onClick={onClick}
            disabled={disabled}
        >
            {children}
        </button>
    );
};
```

### Type Alias for Props

```typescript
type InputProps = {
    value: string;
    onChange: (value: string) => void;
    placeholder?: string;
    type?: 'text' | 'email' | 'password';
    error?: string;
};

const Input: React.FC<InputProps> = ({
    value,
    onChange,
    placeholder,
    type = 'text',
    error
}) => {
    return (
        <div>
            <input
                type={type}
                value={value}
                placeholder={placeholder}
                onChange={(e) => onChange(e.target.value)}
                className={error ? 'input-error' : 'input'}
            />
            {error && <span className="error-message">{error}</span>}
        </div>
    );
};
```

## Advanced Props Patterns

### Generic Props

```typescript
interface SelectProps<T> {
    options: Array<{ value: T; label: string }>;
    value: T;
    onChange: (value: T) => void;
    placeholder?: string;
}

const Select = <T extends string | number>({
    options,
    value,
    onChange,
    placeholder
}: SelectProps<T>) => {
    return (
        <select
            value={value}
            onChange={(e) => onChange(e.target.value as T)}
        >
            {placeholder && (
                <option value="" disabled>
                    {placeholder}
                </option>
            )}
            {options.map((option) => (
                <option key={option.value} value={option.value}>
                    {option.label}
                </option>
            ))}
        </select>
    );
};

// Usage
<Select<string>
    options={[
        { value: 'apple', label: 'Apple' },
        { value: 'banana', label: 'Banana' }
    ]}
    value="apple"
    onChange={(value) => console.log(value)}
/>
```

### Props with Discriminated Unions

```typescript
type ButtonVariant = 'text' | 'contained' | 'outlined';

interface BaseButtonProps {
    children: React.ReactNode;
    onClick: () => void;
    disabled?: boolean;
}

interface TextButtonProps extends BaseButtonProps {
    variant: 'text';
    color?: 'primary' | 'secondary';
}

interface ContainedButtonProps extends BaseButtonProps {
    variant: 'contained';
    color?: 'primary' | 'secondary' | 'error';
    size?: 'small' | 'medium' | 'large';
}

interface OutlinedButtonProps extends BaseButtonProps {
    variant: 'outlined';
    color?: 'primary' | 'secondary' | 'error';
    borderWidth?: number;
}

type ButtonProps = TextButtonProps | ContainedButtonProps | OutlinedButtonProps;

const Button: React.FC<ButtonProps> = (props) => {
    const { variant, children, onClick, disabled } = props;

    const className = `btn btn-${variant}${
        'color' in props ? ` btn-${props.color}` : ''
    }${
        'size' in props ? ` btn-${props.size}` : ''
    }`;

    return (
        <button className={className} onClick={onClick} disabled={disabled}>
            {children}
        </button>
    );
};

// Usage
<Button variant="contained" color="primary" size="large" onClick={() => {}}>
    Click me
</Button>

<Button variant="text" color="secondary" onClick={() => {}}>
    Text button
</Button>
```

## Component Props with Children

### Typing Children

```typescript
interface CardProps {
    title: string;
    children: React.ReactNode; // Most flexible
}

const Card: React.FC<CardProps> = ({ title, children }) => {
    return (
        <div className="card">
            <h3 className="card-title">{title}</h3>
            <div className="card-content">{children}</div>
        </div>
    );
};

// Usage
<Card title="My Card">
    <p>This is some content</p>
    <button>Action</button>
</Card>
```

### Restricting Children Types

```typescript
interface TabProps {
    children: React.ReactElement<TabPanelProps>[];
}

interface TabPanelProps {
    title: string;
    children: React.ReactNode;
}

const TabPanel: React.FC<TabPanelProps> = ({ title, children }) => {
    // Implementation
    return <div>{children}</div>;
};

const Tabs: React.FC<TabProps> = ({ children }) => {
    const [activeTab, setActiveTab] = React.useState(0);

    return (
        <div>
            <div className="tab-headers">
                {children.map((child, index) => (
                    <button
                        key={index}
                        onClick={() => setActiveTab(index)}
                        className={activeTab === index ? 'active' : ''}
                    >
                        {child.props.title}
                    </button>
                ))}
            </div>
            <div className="tab-content">
                {children[activeTab]}
            </div>
        </div>
    );
};

// Usage
<Tabs>
    <TabPanel title="Tab 1">
        <p>Content for tab 1</p>
    </TabPanel>
    <TabPanel title="Tab 2">
        <p>Content for tab 2</p>
    </TabPanel>
</Tabs>
```

## Event Handler Props

### Basic Event Handlers

```typescript
interface FormProps {
    onSubmit: (data: FormData) => void;
    onCancel?: () => void;
}

const Form: React.FC<FormProps> = ({ onSubmit, onCancel }) => {
    const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
        e.preventDefault();
        const formData = new FormData(e.currentTarget);
        onSubmit(formData);
    };

    return (
        <form onSubmit={handleSubmit}>
            <input name="email" type="email" required />
            <input name="password" type="password" required />
            <button type="submit">Submit</button>
            {onCancel && <button type="button" onClick={onCancel}>Cancel</button>}
        </form>
    );
};
```

### Generic Event Handlers

```typescript
interface InputFieldProps<T = string> {
    value: T;
    onChange: (value: T) => void;
    type?: 'text' | 'number' | 'email';
}

const InputField = <T extends string | number>({
    value,
    onChange,
    type = 'text'
}: InputFieldProps<T>) => {
    const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        const newValue = type === 'number'
            ? (Number(e.target.value) as T)
            : (e.target.value as T);
        onChange(newValue);
    };

    return (
        <input
            type={type}
            value={value}
            onChange={handleChange}
        />
    );
};

// Usage
<InputField<string>
    value="hello"
    onChange={(value) => console.log(value)}
/>

<InputField<number>
    value={42}
    onChange={(value) => console.log(value)}
    type="number"
/>
```

## Props with Refs

### Forwarding Refs

```typescript
interface InputProps {
    placeholder?: string;
    type?: string;
}

const Input = React.forwardRef<HTMLInputElement, InputProps>(
    ({ placeholder, type = 'text' }, ref) => {
        return (
            <input
                ref={ref}
                type={type}
                placeholder={placeholder}
            />
        );
    }
);

// Usage
const inputRef = React.useRef<HTMLInputElement>(null);

<Input
    ref={inputRef}
    placeholder="Enter text"
    type="email"
/>
```

### Typing Ref Props

```typescript
interface AutoFocusInputProps {
    autoFocus?: boolean;
    inputRef?: React.RefObject<HTMLInputElement>;
}

const AutoFocusInput: React.FC<AutoFocusInputProps> = ({
    autoFocus = false,
    inputRef
}) => {
    const internalRef = React.useRef<HTMLInputElement>(null);
    const ref = inputRef || internalRef;

    React.useEffect(() => {
        if (autoFocus && ref.current) {
            ref.current.focus();
        }
    }, [autoFocus, ref]);

    return <input ref={ref} />;
};
```

## Higher-Order Component Props

### HOC with Props Modification

```typescript
interface WithLoadingProps {
    loading: boolean;
}

function withLoading<P extends object>(
    Component: React.ComponentType<P>
) {
    return function WithLoadingComponent(
        props: P & WithLoadingProps
    ) {
        const { loading, ...componentProps } = props;

        if (loading) {
            return <div>Loading...</div>;
        }

        return <Component {...(componentProps as P)} />;
    };
}

// Usage
interface UserListProps {
    users: User[];
    onUserClick: (user: User) => void;
}

const UserList: React.FC<UserListProps> = ({ users, onUserClick }) => {
    return (
        <ul>
            {users.map(user => (
                <li key={user.id} onClick={() => onUserClick(user)}>
                    {user.name}
                </li>
            ))}
        </ul>
    );
};

const UserListWithLoading = withLoading(UserList);

// Now UserListWithLoading expects: users, onUserClick, loading
<UserListWithLoading
    users={users}
    onUserClick={handleUserClick}
    loading={isLoading}
/>
```

## Component Props with Context

### Context Consumer Props

```typescript
interface Theme {
    primaryColor: string;
    secondaryColor: string;
    fontSize: 'small' | 'medium' | 'large';
}

const ThemeContext = React.createContext<Theme>({
    primaryColor: '#007bff',
    secondaryColor: '#6c757d',
    fontSize: 'medium'
});

interface ThemedButtonProps {
    children: React.ReactNode;
    variant?: 'primary' | 'secondary';
}

const ThemedButton: React.FC<ThemedButtonProps> = ({
    children,
    variant = 'primary'
}) => {
    const theme = React.useContext(ThemeContext);

    const style = {
        backgroundColor: variant === 'primary' ? theme.primaryColor : theme.secondaryColor,
        fontSize: theme.fontSize === 'small' ? '12px' :
                 theme.fontSize === 'medium' ? '14px' : '16px'
    };

    return (
        <button style={style}>
            {children}
        </button>
    );
};
```

## Advanced Patterns

### Props with Conditional Rendering

```typescript
interface ModalProps {
    isOpen: boolean;
    onClose: () => void;
    title?: string;
    children: React.ReactNode;
    size?: 'small' | 'medium' | 'large';
    showCloseButton?: boolean;
}

const Modal: React.FC<ModalProps> = ({
    isOpen,
    onClose,
    title,
    children,
    size = 'medium',
    showCloseButton = true
}) => {
    if (!isOpen) return null;

    return (
        <div className="modal-overlay" onClick={onClose}>
            <div
                className={`modal modal-${size}`}
                onClick={(e) => e.stopPropagation()}
            >
                {(title || showCloseButton) && (
                    <div className="modal-header">
                        {title && <h3>{title}</h3>}
                        {showCloseButton && (
                            <button onClick={onClose}>×</button>
                        )}
                    </div>
                )}
                <div className="modal-content">
                    {children}
                </div>
            </div>
        </div>
    );
};
```

### Props with Default Values

```typescript
interface PaginationProps {
    currentPage: number;
    totalPages: number;
    onPageChange: (page: number) => void;
    showFirstLast?: boolean;
    maxVisiblePages?: number;
}

const Pagination: React.FC<PaginationProps> = ({
    currentPage,
    totalPages,
    onPageChange,
    showFirstLast = true,
    maxVisiblePages = 5
}) => {
    // Implementation
    return <div>Pagination component</div>;
};
```

## Best Practices

### 1. **Use Interfaces for Complex Props**

```typescript
// ✅ Good: Clear and extensible
interface UserCardProps {
    user: User;
    onEdit?: (user: User) => void;
    onDelete?: (user: User) => void;
    showActions?: boolean;
}

// ❌ Avoid: Inline types for complex props
const UserCard: React.FC<{
    user: User;
    onEdit?: (user: User) => void;
    onDelete?: (user: User) => void;
    showActions?: boolean;
}> = (props) => { ... }
```

### 2. **Make Props Optional When Appropriate**

```typescript
// ✅ Good: Optional props with defaults
interface ButtonProps {
    children: React.ReactNode;
    onClick: () => void;
    variant?: 'primary' | 'secondary'; // Optional with default
    disabled?: boolean; // Optional
}

// ❌ Avoid: Too many required props
interface ButtonProps {
    children: React.ReactNode;
    onClick: () => void;
    variant: 'primary' | 'secondary'; // Required
    disabled: boolean; // Required
}
```

### 3. **Use Discriminated Unions for Variant Props**

```typescript
// ✅ Good: Type-safe variants
type ButtonProps =
    | { variant: 'text'; color?: 'primary' | 'secondary' }
    | { variant: 'contained'; color?: 'primary' | 'secondary' | 'error'; size?: 'small' | 'medium' | 'large' }
    | { variant: 'outlined'; color?: 'primary' | 'secondary' | 'error'; borderWidth?: number };

// ❌ Avoid: Unsafe variants
interface ButtonProps {
    variant: 'text' | 'contained' | 'outlined';
    color?: string; // Too broad
    size?: string; // Too broad
}
```

### 4. **Extract Common Props**

```typescript
// ✅ Good: Reusable base props
interface BaseInputProps {
    disabled?: boolean;
    required?: boolean;
    className?: string;
}

interface TextInputProps extends BaseInputProps {
    value: string;
    onChange: (value: string) => void;
    type?: 'text' | 'email' | 'password';
}

interface NumberInputProps extends BaseInputProps {
    value: number;
    onChange: (value: number) => void;
    min?: number;
    max?: number;
}
```

### 5. **Use Generic Constraints**

```typescript
// ✅ Good: Constrained generics
interface DataTableProps<T extends { id: string | number }> {
    data: T[];
    columns: Array<{
        key: keyof T;
        header: string;
        render?: (value: T[keyof T], item: T) => React.ReactNode;
    }>;
}

// ❌ Avoid: Unconstrained generics
interface DataTableProps<T> {
    data: T[];
    columns: Array<{
        key: string; // Type unsafe
        header: string;
    }>;
}
```

## Common Interview Questions

### Q: What's the difference between typing props with interface vs type alias?

**A:** Interfaces are better for object shapes and support declaration merging. Type aliases are more flexible for unions, primitives, and complex types. Use interfaces for component props unless you need advanced type features.

### Q: How do you type optional props with default values?

**A:** Make the prop optional in the interface (`prop?: Type`) and provide a default value in the component destructuring (`{ prop = defaultValue }`).

### Q: How do you handle props that can be different types?

**A:** Use union types (`string | number`) or discriminated unions for complex variant props. For example: `{ variant: 'text'; color?: string } | { variant: 'contained'; color?: string; size?: string }`.

### Q: Should you use React.FC for typing components?

**A:** It's optional but provides some benefits like automatic children typing. However, it's being phased out in favor of direct function typing. Use `React.FC<Props>` or `React.ComponentType<Props>`.

## Summary

Typing React component props involves:

1. **Basic Props**: Use interfaces or type aliases for prop shapes
2. **Advanced Patterns**: Generics, discriminated unions, conditional props
3. **Event Handlers**: Proper typing for React synthetic events
4. **Children**: `React.ReactNode` for flexible content, `React.ReactElement` for restrictions
5. **Refs**: `React.forwardRef` and `React.RefObject` for ref forwarding
6. **Best Practices**: Optional props with defaults, discriminated unions for variants, reusable base interfaces

**Key Principles:**
- Make props as specific as possible for type safety
- Use optional props with sensible defaults
- Prefer interfaces for object props, types for complex patterns
- Use discriminated unions for variant components
- Extract common props into reusable interfaces

**Interview Tip:** "React component props should be typed with interfaces for object shapes, using optional props with default values. For variant components, use discriminated unions to ensure type safety across different prop combinations."