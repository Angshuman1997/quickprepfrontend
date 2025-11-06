# Use cases for Pick<T>

## Question
Use cases for Pick<T>

## Answer

`Pick<T, K>` is a TypeScript utility type that creates a new type by selecting specific properties `K` from type `T`. It's the opposite of `Omit<T, K>` and is incredibly useful when you only need a subset of properties from an existing type.

## Basic Usage

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    age: number;
    address: string;
    phone: string;
    createdAt: Date;
}

// Pick specific properties
type UserBasicInfo = Pick<User, 'id' | 'name' | 'email'>;
type UserContactInfo = Pick<User, 'email' | 'phone'>;
type UserPersonalInfo = Pick<User, 'name' | 'age' | 'address'>;

// Usage
const basicInfo: UserBasicInfo = {
    id: 1,
    name: "John Doe",
    email: "john@example.com"
};

const contactInfo: UserContactInfo = {
    email: "john@example.com",
    phone: "+1-555-0123"
};
```

## API Response Types

### 1. **Public API Responses**

```typescript
interface UserEntity {
    id: string;
    name: string;
    email: string;
    passwordHash: string;
    salt: string;
    role: string;
    createdAt: Date;
    updatedAt: Date;
    lastLoginAt: Date;
    isActive: boolean;
    profilePictureUrl?: string;
}

// Public profile (safe to expose)
type PublicUserProfile = Pick<UserEntity, 'id' | 'name' | 'email' | 'profilePictureUrl'>;

// User summary for lists
type UserSummary = Pick<UserEntity, 'id' | 'name' | 'email' | 'role' | 'isActive'>;

// Admin view (more fields for admins)
type AdminUserView = Pick<UserEntity,
    'id' | 'name' | 'email' | 'role' | 'isActive' | 'createdAt' | 'lastLoginAt'>;

// Login response
type LoginResponse = Pick<UserEntity, 'id' | 'name' | 'email' | 'role'>;

// Usage in API handlers
const getPublicProfile = (user: UserEntity): PublicUserProfile => ({
    id: user.id,
    name: user.name,
    email: user.email,
    profilePictureUrl: user.profilePictureUrl
});

const getUserSummary = (user: UserEntity): UserSummary => ({
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
    isActive: user.isActive
});
```

### 2. **Database Query Projections**

```typescript
interface Product {
    id: number;
    name: string;
    description: string;
    price: number;
    category: string;
    brand: string;
    sku: string;
    stockQuantity: number;
    weight: number;
    dimensions: {
        length: number;
        width: number;
        height: number;
    };
    tags: string[];
    createdAt: Date;
    updatedAt: Date;
}

// Product list view (compact)
type ProductListItem = Pick<Product, 'id' | 'name' | 'price' | 'category' | 'brand'>;

// Product details view
type ProductDetails = Pick<Product,
    'id' | 'name' | 'description' | 'price' | 'category' | 'brand' | 'stockQuantity'>;

// Search result
type ProductSearchResult = Pick<Product, 'id' | 'name' | 'price' | 'category' | 'tags'>;

// Inventory management
type InventoryItem = Pick<Product, 'id' | 'name' | 'sku' | 'stockQuantity'>;

// Shipping calculation
type ShippableProduct = Pick<Product, 'id' | 'name' | 'weight' | 'dimensions'>;

class ProductService {
    async getProductList(): Promise<ProductListItem[]> {
        // Query only needed fields for performance
        return db.products.find({}, {
            id: 1,
            name: 1,
            price: 1,
            category: 1,
            brand: 1
        });
    }

    async getProductDetails(id: number): Promise<ProductDetails | null> {
        const product = await db.products.findById(id);
        if (!product) return null;

        return {
            id: product.id,
            name: product.name,
            description: product.description,
            price: product.price,
            category: product.category,
            brand: product.brand,
            stockQuantity: product.stockQuantity
        };
    }
}
```

## React Component Props

### 1. **Inheriting Specific HTML Attributes**

```typescript
interface ButtonProps extends Pick<React.ButtonHTMLAttributes<HTMLButtonElement>,
    'onClick' | 'disabled' | 'type' | 'className'> {
    variant?: 'primary' | 'secondary' | 'danger';
    size?: 'small' | 'medium' | 'large';
    loading?: boolean;
}

const Button: React.FC<ButtonProps> = ({
    variant = 'primary',
    size = 'medium',
    loading = false,
    children,
    disabled,
    className,
    ...props
}) => {
    const buttonClass = `btn btn-${variant} btn-${size} ${loading ? 'loading' : ''} ${className || ''}`;

    return (
        <button
            className={buttonClass}
            disabled={disabled || loading}
            {...props}
        >
            {loading ? 'Loading...' : children}
        </button>
    );
};

// Usage - inherits onClick, disabled, type, className from button element
<Button
    variant="primary"
    size="large"
    onClick={() => console.log('Clicked!')}
    disabled={false}
    type="submit"
    className="custom-button"
>
    Submit
</Button>
```

### 2. **Form Input Components**

```typescript
interface InputProps extends Pick<React.InputHTMLAttributes<HTMLInputElement>,
    'placeholder' | 'disabled' | 'required' | 'className' | 'autoFocus'> {
    label?: string;
    error?: string;
    helperText?: string;
    startAdornment?: React.ReactNode;
    endAdornment?: React.ReactNode;
}

const Input: React.FC<InputProps> = ({
    label,
    error,
    helperText,
    startAdornment,
    endAdornment,
    className,
    ...inputProps
}) => {
    const inputClass = `input ${error ? 'input-error' : ''} ${className || ''}`;

    return (
        <div className="input-wrapper">
            {label && <label className="input-label">{label}</label>}
            <div className="input-adornment-wrapper">
                {startAdornment && (
                    <div className="input-start-adornment">{startAdornment}</div>
                )}
                <input className={inputClass} {...inputProps} />
                {endAdornment && (
                    <div className="input-end-adornment">{endAdornment}</div>
                )}
            </div>
            {error && <span className="input-error-text">{error}</span>}
            {helperText && !error && (
                <span className="input-helper-text">{helperText}</span>
            )}
        </div>
    );
};

// Usage - inherits placeholder, disabled, required, className, autoFocus
<Input
    label="Email"
    type="email"
    placeholder="Enter your email"
    required
    autoFocus
    error={emailError}
    helperText="We'll never share your email"
/>
```

### 3. **Table Column Configuration**

```typescript
interface ColumnConfig<T> {
    key: keyof T;
    header: string;
    sortable?: boolean;
    width?: number;
    render?: (value: T[keyof T], row: T) => React.ReactNode;
}

interface TableProps<T> {
    data: T[];
    columns: ColumnConfig<T>[];
    selectedRows?: T[];
    onRowSelect?: (row: T) => void;
}

const Table = <T extends Record<string, any>>({
    data,
    columns,
    selectedRows = [],
    onRowSelect
}: TableProps<T>) => {
    return (
        <table>
            <thead>
                <tr>
                    {columns.map(column => (
                        <th key={String(column.key)} style={{ width: column.width }}>
                            {column.header}
                        </th>
                    ))}
                </tr>
            </thead>
            <tbody>
                {data.map((row, index) => (
                    <tr key={index} onClick={() => onRowSelect?.(row)}>
                        {columns.map(column => (
                            <td key={String(column.key)}>
                                {column.render
                                    ? column.render(row[column.key], row)
                                    : String(row[column.key])
                                }
                            </td>
                        ))}
                    </tr>
                ))}
            </tbody>
        </table>
    );
};

// Usage with User type
interface User {
    id: number;
    name: string;
    email: string;
    role: string;
    department: string;
    salary: number;
    hireDate: Date;
}

const userColumns: ColumnConfig<User>[] = [
    { key: 'name', header: 'Name', sortable: true },
    { key: 'email', header: 'Email' },
    { key: 'role', header: 'Role' },
    { key: 'department', header: 'Department', sortable: true },
    {
        key: 'salary',
        header: 'Salary',
        render: (value) => `$${Number(value).toLocaleString()}`
    }
];

<Table<User>
    data={users}
    columns={userColumns}
    onRowSelect={(user) => console.log('Selected:', user)}
/>
```

## Form and Validation Types

### 1. **Form Field Selection**

```typescript
interface RegistrationForm {
    firstName: string;
    lastName: string;
    email: string;
    password: string;
    confirmPassword: string;
    agreeToTerms: boolean;
    newsletter: boolean;
    dateOfBirth: Date;
    phoneNumber: string;
}

// Required fields for registration
type RequiredRegistrationFields = Pick<RegistrationForm,
    'firstName' | 'lastName' | 'email' | 'password' | 'agreeToTerms'>;

// Optional fields
type OptionalRegistrationFields = Pick<RegistrationForm,
    'confirmPassword' | 'newsletter' | 'dateOfBirth' | 'phoneNumber'>;

// Login form (subset of registration)
type LoginForm = Pick<RegistrationForm, 'email' | 'password'>;

// Profile update form
type ProfileUpdateForm = Pick<RegistrationForm,
    'firstName' | 'lastName' | 'dateOfBirth' | 'phoneNumber'>;

// Password change form
type PasswordChangeForm = Pick<RegistrationForm, 'password' | 'confirmPassword'>;

interface FormComponentProps<T> {
    initialValues: T;
    onSubmit: (values: T) => Promise<void>;
    validationSchema?: Record<keyof T, (value: any) => string | null>;
}

const RegistrationFormComponent: React.FC<FormComponentProps<RequiredRegistrationFields & Partial<OptionalRegistrationFields>>> = ({
    initialValues,
    onSubmit,
    validationSchema
}) => {
    // Form implementation
    return <form>Registration Form</form>;
};
```

### 2. **Validation Rules**

```typescript
interface ValidationRules {
    required: boolean;
    minLength?: number;
    maxLength?: number;
    pattern?: RegExp;
    custom?: (value: any) => string | null;
}

// Field-specific validation rules
type NameValidation = Pick<ValidationRules, 'required' | 'minLength' | 'maxLength'>;
type EmailValidation = Pick<ValidationRules, 'required' | 'pattern'>;
type PasswordValidation = Pick<ValidationRules, 'required' | 'minLength' | 'pattern'>;
type AgeValidation = Pick<ValidationRules, 'required' | 'custom'>;

interface FormFieldConfig {
    name: string;
    type: 'text' | 'email' | 'password' | 'number';
    label: string;
    placeholder?: string;
    validation: ValidationRules;
}

const formFieldConfigs: FormFieldConfig[] = [
    {
        name: 'firstName',
        type: 'text',
        label: 'First Name',
        validation: {
            required: true,
            minLength: 2,
            maxLength: 50
        } as NameValidation
    },
    {
        name: 'email',
        type: 'email',
        label: 'Email',
        validation: {
            required: true,
            pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/
        } as EmailValidation
    },
    {
        name: 'password',
        type: 'password',
        label: 'Password',
        validation: {
            required: true,
            minLength: 8,
            pattern: /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/
        } as PasswordValidation
    }
];
```

## Advanced Patterns

### 1. **Dynamic Property Selection**

```typescript
// Create a type that picks properties based on a condition
type SelectProperties<T, K extends keyof T> = Pick<T, K>;

type UserDisplayFields = 'name' | 'email' | 'role';
type UserDisplay = SelectProperties<User, UserDisplayFields>;

// More advanced: conditional picking
type PickIfString<T, K extends keyof T> = Pick<T, {
    [P in K]: T[P] extends string ? P : never;
}[K]>;

type StringFieldsOnly = PickIfString<User, keyof User>;
// Result: { name: string; email: string; role: string; ... }
```

### 2. **API Response Shaping**

```typescript
interface FullAPIResponse<T> {
    data: T;
    meta: {
        page: number;
        limit: number;
        total: number;
        totalPages: number;
    };
    links: {
        self: string;
        next?: string;
        prev?: string;
        first: string;
        last: string;
    };
}

// Different response shapes for different endpoints
type ListResponse<T> = Pick<FullAPIResponse<T>, 'data' | 'meta' | 'links'>;
type SingleResponse<T> = Pick<FullAPIResponse<T>, 'data' | 'links'>;
type CreateResponse<T> = Pick<FullAPIResponse<T>, 'data' | 'links'>;
type UpdateResponse<T> = Pick<FullAPIResponse<T>, 'data' | 'links'>;
type DeleteResponse = Pick<FullAPIResponse<null>, 'links'>;

const apiClient = {
    getUsers: (page = 1): Promise<ListResponse<User[]>> => {
        // Implementation
        return Promise.resolve({} as ListResponse<User[]>);
    },

    getUser: (id: number): Promise<SingleResponse<User>> => {
        // Implementation
        return Promise.resolve({} as SingleResponse<User>);
    },

    createUser: (user: CreateUserRequest): Promise<CreateResponse<User>> => {
        // Implementation
        return Promise.resolve({} as CreateResponse<User>);
    },

    deleteUser: (id: number): Promise<DeleteResponse> => {
        // Implementation
        return Promise.resolve({} as DeleteResponse);
    }
};
```

### 3. **Component Variants with Shared Props**

```typescript
interface BaseCardProps {
    title: string;
    children: React.ReactNode;
    className?: string;
    onClick?: () => void;
}

type CardVariant = 'elevated' | 'outlined' | 'filled';

interface ElevatedCardProps extends Pick<BaseCardProps, 'title' | 'children' | 'className'> {
    elevation: 1 | 2 | 3 | 4 | 5;
}

interface OutlinedCardProps extends Pick<BaseCardProps, 'title' | 'children' | 'className'> {
    borderWidth: 'thin' | 'medium' | 'thick';
    borderColor?: string;
}

interface FilledCardProps extends Pick<BaseCardProps, 'title' | 'children' | 'className'> {
    backgroundColor?: string;
    textColor?: string;
}

type CardProps =
    | (ElevatedCardProps & { variant: 'elevated' })
    | (OutlinedCardProps & { variant: 'outlined' })
    | (FilledCardProps & { variant: 'filled' });

const Card: React.FC<CardProps> = (props) => {
    const { variant, title, children, className } = props;

    const cardClass = `card card-${variant} ${className || ''}`;

    return (
        <div className={cardClass}>
            <h3 className="card-title">{title}</h3>
            <div className="card-content">{children}</div>
        </div>
    );
};

// Usage
<Card
    variant="elevated"
    title="Elevated Card"
    elevation={3}
>
    Content
</Card>

<Card
    variant="outlined"
    title="Outlined Card"
    borderWidth="medium"
    borderColor="blue"
>
    Content
</Card>
```

## Best Practices

### 1. **Use Pick for API Contracts**

```typescript
// ✅ Good: Clear API boundaries
type PublicUser = Pick<User, 'id' | 'name' | 'email'>;
type UserCreate = Pick<User, 'name' | 'email' | 'password'>;
type UserUpdate = Partial<Pick<User, 'name' | 'email' | 'role'>>;

// ❌ Avoid: Exposing internal types directly
// type PublicUser = User; // Exposes all fields including password
```

### 2. **Combine with Other Utilities**

```typescript
// ✅ Good: Combine utilities for complex types
type EditableUserFields = Partial<Pick<User, 'name' | 'email' | 'phone'>>;
type RequiredUserFields = Pick<User, 'name' | 'email'>;
type OptionalUserFields = Partial<Omit<User, 'id' | 'createdAt'>>;
```

### 3. **Use Pick for Component Composition**

```typescript
// ✅ Good: Selective inheritance
interface CustomInputProps extends Pick<React.InputHTMLAttributes<HTMLInputElement>,
    'placeholder' | 'disabled' | 'required' | 'className'> {
    label?: string;
    error?: string;
}

// ❌ Avoid: Unnecessary inheritance
// interface CustomInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
//     label?: string;
//     error?: string;
// } // Inherits all input props, even irrelevant ones
```

### 4. **Document Your Picks**

```typescript
// ✅ Good: Document the purpose
type UserProfile = Pick<User, 'id' | 'name' | 'email' | 'avatar'>; // Public profile data
type UserCredentials = Pick<User, 'email' | 'password'>; // Login credentials
type UserPermissions = Pick<User, 'id' | 'role' | 'permissions'>; // Access control
```

## Common Interview Questions

### Q: When should you use Pick<T, K> instead of defining a new interface?

**A:** Use `Pick` when you want to reuse properties from an existing type. It ensures consistency when the base type changes and reduces duplication. Define a new interface when the subset represents a meaningful domain concept.

### Q: What's the difference between Pick and Omit?

**A:** `Pick<T, K>` selects specific properties K from T, while `Omit<T, K>` removes properties K from T. They're complementary operations - `Pick<T, K>` is equivalent to `Omit<T, Exclude<keyof T, K>>`.

### Q: Can Pick work with union types?

**A:** Yes, but it picks from each member of the union. For complex cases, you might need different approaches depending on your requirements.

### Q: How do you pick properties conditionally?

**A:** You can create custom utility types that pick properties based on conditions, like picking only string properties or properties matching certain criteria.

## Summary

`Pick<T, K>` is essential for:

1. **API Response Shaping**: Creating different views of the same data
2. **Component Props**: Selective inheritance from HTML elements
3. **Form Types**: Defining specific form field subsets
4. **Type Composition**: Building complex types from existing ones
5. **Data Projections**: Optimizing database queries and responses

**Key Benefits:**
- Type safety with existing types
- Reduced code duplication
- Clear API boundaries
- Better maintainability
- Performance optimization (database projections)

**Interview Tip:** "`Pick<T, K>` selects specific properties from a type, perfect for API responses, component props, and creating different data views while maintaining type safety and consistency."