# Use cases for Partial<T>

## Question
Use cases for Partial<T>

## Answer

`Partial<T>` is a TypeScript utility type that makes all properties of type `T` optional. It's incredibly useful for scenarios where you want to work with incomplete objects or allow partial updates. Here are practical use cases:

## Basic Usage

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    age: number;
    address: string;
}

// All properties become optional
type PartialUser = Partial<User>;

// Usage - can provide any subset of properties
const updateUser = (id: number, updates: PartialUser) => {
    // Implementation
};

// Valid calls
updateUser(1, { name: "John Doe" });
updateUser(2, { email: "john@example.com", age: 30 });
updateUser(3, { name: "Jane", email: "jane@example.com", age: 25, address: "123 Main St" });
updateUser(4, {}); // Empty object also valid
```

## React State Management

### 1. **useState with Complex Objects**

```typescript
interface FormState {
    name: string;
    email: string;
    password: string;
    confirmPassword: string;
    agreeToTerms: boolean;
    newsletter: boolean;
}

const initialState: FormState = {
    name: '',
    email: '',
    password: '',
    confirmPassword: '',
    agreeToTerms: false,
    newsletter: false
};

const RegistrationForm: React.FC = () => {
    const [formState, setFormState] = useState<FormState>(initialState);

    // Partial<FormState> allows updating any subset of fields
    const updateField = (updates: Partial<FormState>) => {
        setFormState(prev => ({ ...prev, ...updates }));
    };

    const handleNameChange = (name: string) => {
        updateField({ name });
    };

    const handleEmailChange = (email: string) => {
        updateField({ email });
    };

    const handleCheckboxChange = (field: keyof Pick<FormState, 'agreeToTerms' | 'newsletter'>) =>
        (checked: boolean) => {
            updateField({ [field]: checked });
        };

    return (
        <form>
            <input
                value={formState.name}
                onChange={(e) => handleNameChange(e.target.value)}
                placeholder="Name"
            />
            <input
                type="email"
                value={formState.email}
                onChange={(e) => handleEmailChange(e.target.value)}
                placeholder="Email"
            />
            <input
                type="password"
                value={formState.password}
                onChange={(e) => updateField({ password: e.target.value })}
                placeholder="Password"
            />
            <input
                type="password"
                value={formState.confirmPassword}
                onChange={(e) => updateField({ confirmPassword: e.target.value })}
                placeholder="Confirm Password"
            />
            <label>
                <input
                    type="checkbox"
                    checked={formState.agreeToTerms}
                    onChange={(e) => handleCheckboxChange('agreeToTerms')(e.target.checked)}
                />
                Agree to terms
            </label>
            <label>
                <input
                    type="checkbox"
                    checked={formState.newsletter}
                    onChange={(e) => handleCheckboxChange('newsletter')(e.target.checked)}
                />
                Subscribe to newsletter
            </label>
        </form>
    );
};
```

### 2. **Custom Hook for Form State**

```typescript
function useFormState<T extends Record<string, any>>(initialState: T) {
    const [state, setState] = useState<T>(initialState);

    const updateField = (updates: Partial<T>) => {
        setState(prev => ({ ...prev, ...updates }));
    };

    const reset = () => setState(initialState);

    const setValue = <K extends keyof T>(field: K, value: T[K]) => {
        updateField({ [field]: value } as Partial<T>);
    };

    return { state, updateField, reset, setValue };
}

// Usage
interface UserForm {
    name: string;
    email: string;
    age: number;
    bio: string;
}

const UserProfileForm: React.FC = () => {
    const { state, updateField, setValue, reset } = useFormState<UserForm>({
        name: '',
        email: '',
        age: 0,
        bio: ''
    });

    return (
        <form>
            <input
                value={state.name}
                onChange={(e) => setValue('name', e.target.value)}
                placeholder="Name"
            />
            <input
                value={state.email}
                onChange={(e) => setValue('email', e.target.value)}
                placeholder="Email"
            />
            <input
                type="number"
                value={state.age}
                onChange={(e) => setValue('age', Number(e.target.value))}
                placeholder="Age"
            />
            <textarea
                value={state.bio}
                onChange={(e) => setValue('bio', e.target.value)}
                placeholder="Bio"
            />
            <button type="button" onClick={reset}>Reset</button>
            <button type="button" onClick={() => updateField({ name: 'John', age: 25 })}>
                Quick Fill
            </button>
        </form>
    );
};
```

## API and Database Operations

### 1. **Update Operations**

```typescript
interface User {
    id: number;
    name: string;
    email: string;
    age: number;
    department: string;
    salary: number;
    hireDate: Date;
}

// Allow partial updates for user profiles
type UpdateUserRequest = Partial<User>;

class UserService {
    async updateUser(id: number, updates: UpdateUserRequest): Promise<User> {
        // Validate that at least one field is being updated
        if (Object.keys(updates).length === 0) {
            throw new Error('At least one field must be updated');
        }

        // Implementation
        const updatedUser = await db.users.update(id, updates);
        return updatedUser;
    }

    async patchUser(id: number, updates: UpdateUserRequest): Promise<User> {
        // Similar to update but allows partial updates
        return this.updateUser(id, updates);
    }
}

// Usage
const userService = new UserService();

// Update single field
await userService.updateUser(1, { name: "John Smith" });

// Update multiple fields
await userService.updateUser(2, {
    email: "john.smith@example.com",
    age: 31,
    department: "Engineering"
});

// Update with validation
await userService.updateUser(3, {
    salary: 85000,
    department: "Senior Engineering"
});
```

### 2. **Query Filters**

```typescript
interface Product {
    id: number;
    name: string;
    price: number;
    category: string;
    brand: string;
    inStock: boolean;
    rating: number;
}

// Allow filtering by any combination of fields
type ProductFilters = Partial<Product>;

class ProductService {
    async findProducts(filters: ProductFilters = {}): Promise<Product[]> {
        let query = db.products;

        // Apply filters dynamically
        if (filters.name) {
            query = query.where('name', 'like', `%${filters.name}%`);
        }
        if (filters.category) {
            query = query.where('category', filters.category);
        }
        if (filters.brand) {
            query = query.where('brand', filters.brand);
        }
        if (filters.price !== undefined) {
            query = query.where('price', '<=', filters.price);
        }
        if (filters.inStock !== undefined) {
            query = query.where('inStock', filters.inStock);
        }
        if (filters.rating !== undefined) {
            query = query.where('rating', '>=', filters.rating);
        }

        return query.find();
    }
}

// Usage
const productService = new ProductService();

// Find all products
const allProducts = await productService.findProducts();

// Find products by category
const electronics = await productService.findProducts({ category: 'Electronics' });

// Complex filter
const affordableLaptops = await productService.findProducts({
    category: 'Laptops',
    price: 1500,
    inStock: true,
    rating: 4.0
});
```

### 3. **Configuration Objects**

```typescript
interface AppConfig {
    apiUrl: string;
    timeout: number;
    retries: number;
    cacheEnabled: boolean;
    logLevel: 'debug' | 'info' | 'warn' | 'error';
    features: {
        darkMode: boolean;
        notifications: boolean;
        analytics: boolean;
    };
}

// Allow partial configuration for overrides
type ConfigOverrides = Partial<AppConfig>;

const defaultConfig: AppConfig = {
    apiUrl: 'https://api.example.com',
    timeout: 5000,
    retries: 3,
    cacheEnabled: true,
    logLevel: 'info',
    features: {
        darkMode: false,
        notifications: true,
        analytics: true
    }
};

function createConfig(overrides: ConfigOverrides = {}): AppConfig {
    return {
        ...defaultConfig,
        ...overrides,
        features: {
            ...defaultConfig.features,
            ...overrides.features
        }
    };
}

// Usage
const devConfig = createConfig({
    apiUrl: 'http://localhost:3000',
    logLevel: 'debug',
    features: {
        analytics: false
    }
});

const prodConfig = createConfig({
    timeout: 10000,
    retries: 5
});
```

## Component Props and Configuration

### 1. **Flexible Component Props**

```typescript
interface ChartProps {
    data: Array<{ x: number; y: number }>;
    type: 'line' | 'bar' | 'pie';
    width: number;
    height: number;
    title?: string;
    colors: string[];
    showLegend: boolean;
    showGrid: boolean;
    animation: boolean;
}

// Allow partial configuration for chart presets
type ChartOptions = Partial<ChartProps>;

const defaultChartProps: ChartProps = {
    data: [],
    type: 'line',
    width: 400,
    height: 300,
    colors: ['#007bff', '#28a745', '#dc3545'],
    showLegend: true,
    showGrid: true,
    animation: true
};

const Chart: React.FC<ChartOptions> = (options) => {
    const props = { ...defaultChartProps, ...options };

    // Render chart with props
    return <div className="chart">Chart: {props.type}</div>;
};

// Usage - only specify what you want to change
<Chart type="bar" width={600} height={400} />
<Chart data={dataPoints} showLegend={false} animation={false} />
<Chart
    type="pie"
    colors={['red', 'blue', 'green']}
    title="Sales Distribution"
/>
```

### 2. **Theme Configuration**

```typescript
interface Theme {
    colors: {
        primary: string;
        secondary: string;
        success: string;
        danger: string;
        warning: string;
        info: string;
        light: string;
        dark: string;
    };
    spacing: {
        xs: string;
        sm: string;
        md: string;
        lg: string;
        xl: string;
    };
    typography: {
        fontFamily: string;
        fontSize: {
            xs: string;
            sm: string;
            md: string;
            lg: string;
            xl: string;
        };
        fontWeight: {
            light: number;
            normal: number;
            bold: number;
        };
    };
    borderRadius: {
        none: string;
        sm: string;
        md: string;
        lg: string;
        full: string;
    };
}

// Allow partial theme overrides
type ThemeOverrides = Partial<Theme>;

const defaultTheme: Theme = {
    colors: {
        primary: '#007bff',
        secondary: '#6c757d',
        success: '#28a745',
        danger: '#dc3545',
        warning: '#ffc107',
        info: '#17a2b8',
        light: '#f8f9fa',
        dark: '#343a40'
    },
    spacing: {
        xs: '0.25rem',
        sm: '0.5rem',
        md: '1rem',
        lg: '1.5rem',
        xl: '3rem'
    },
    typography: {
        fontFamily: '"Helvetica Neue", Arial, sans-serif',
        fontSize: {
            xs: '0.75rem',
            sm: '0.875rem',
            md: '1rem',
            lg: '1.125rem',
            xl: '1.25rem'
        },
        fontWeight: {
            light: 300,
            normal: 400,
            bold: 700
        }
    },
    borderRadius: {
        none: '0',
        sm: '0.25rem',
        md: '0.375rem',
        lg: '0.5rem',
        full: '9999px'
    }
};

function createTheme(overrides: ThemeOverrides = {}): Theme {
    return {
        ...defaultTheme,
        ...overrides,
        colors: { ...defaultTheme.colors, ...overrides.colors },
        spacing: { ...defaultTheme.spacing, ...overrides.spacing },
        typography: {
            ...defaultTheme.typography,
            ...overrides.typography,
            fontSize: { ...defaultTheme.typography.fontSize, ...overrides.typography?.fontSize },
            fontWeight: { ...defaultTheme.typography.fontWeight, ...overrides.typography?.fontWeight }
        },
        borderRadius: { ...defaultTheme.borderRadius, ...overrides.borderRadius }
    };
}

// Usage
const darkTheme = createTheme({
    colors: {
        primary: '#bb86fc',
        light: '#121212',
        dark: '#ffffff'
    }
});

const compactTheme = createTheme({
    spacing: {
        xs: '0.125rem',
        sm: '0.25rem',
        md: '0.5rem',
        lg: '1rem',
        xl: '2rem'
    }
});
```

## Advanced Patterns

### 1. **Deep Partial for Nested Objects**

```typescript
// TypeScript doesn't have built-in DeepPartial, but you can create it
type DeepPartial<T> = {
    [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface NestedConfig {
    api: {
        baseUrl: string;
        timeout: number;
        retries: {
            count: number;
            delay: number;
        };
    };
    ui: {
        theme: string;
        layout: {
            sidebar: boolean;
            header: {
                fixed: boolean;
                height: number;
            };
        };
    };
}

type ConfigOverrides = DeepPartial<NestedConfig>;

const configOverrides: ConfigOverrides = {
    api: {
        retries: {
            count: 5 // Only override this deeply nested property
        }
    },
    ui: {
        layout: {
            header: {
                height: 80 // Override another nested property
            }
        }
    }
};
```

### 2. **Partial with Constraints**

```typescript
// At least one property must be provided
type AtLeastOne<T, U = { [K in keyof T]: Pick<T, K> }> = Partial<T> & U[keyof U];

// Usage
type UpdateUserAtLeastOne = AtLeastOne<User>;

const updateUser = (id: number, updates: UpdateUserAtLeastOne) => {
    // At least one property must be provided
};

// Valid
updateUser(1, { name: 'John' });
updateUser(2, { email: 'john@example.com', age: 30 });

// Invalid - empty object
// updateUser(3, {});
```

### 3. **Partial in Generic Constraints**

```typescript
function mergeObjects<T extends Record<string, any>>(
    target: T,
    source: Partial<T>
): T {
    return { ...target, ...source };
}

interface Settings {
    theme: string;
    language: string;
    notifications: boolean;
}

const defaultSettings: Settings = {
    theme: 'light',
    language: 'en',
    notifications: true
};

// Usage
const userSettings = mergeObjects(defaultSettings, {
    theme: 'dark' // Only override theme
});

const minimalSettings = mergeObjects(defaultSettings, {
    notifications: false // Only override notifications
});
```

## Best Practices

### 1. **Use Partial for Update Operations**

```typescript
// ✅ Good: Allow partial updates
interface UserService {
    updateUser(id: number, updates: Partial<User>): Promise<User>;
}

// ❌ Avoid: Require all fields for updates
interface BadUserService {
    updateUser(id: number, user: User): Promise<User>; // Forces updating all fields
}
```

### 2. **Combine with Other Utility Types**

```typescript
// ✅ Good: Combine utilities for specific use cases
type CreateUser = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;
type UpdateUser = Partial<Omit<User, 'id' | 'createdAt'>>;
type UserResponse = Omit<User, 'password'>;
```

### 3. **Use Partial for Configuration Objects**

```typescript
// ✅ Good: Flexible configuration
interface ComponentProps {
    required: string;
    optional?: string;
}

const Component: React.FC<Partial<ComponentProps> & Pick<ComponentProps, 'required'>> = (props) => {
    // required is guaranteed, others are optional
};
```

### 4. **Document Partial Usage**

```typescript
// ✅ Good: Document the intent
type FormUpdates = Partial<FormData>; // Allow updating any form fields
type FilterOptions = Partial<Entity>; // Allow filtering by any entity fields
type ConfigOverrides = Partial<Config>; // Allow overriding any config values
```

## Common Interview Questions

### Q: When should you use Partial<T> instead of making all properties optional manually?

**A:** Use `Partial<T>` when you want to make all properties optional based on an existing type. It ensures consistency if the base type changes and is more maintainable than manually marking each property as optional.

### Q: Can Partial work with nested objects?

**A:** `Partial<T>` only makes top-level properties optional. For deep partial objects, you need a custom `DeepPartial<T>` type that recursively applies Partial to nested objects.

### Q: What's the difference between Partial<T> and T | {}?

**A:** `Partial<T>` makes all properties optional but maintains the structure, while `T | {}` allows either the full type or an empty object. `Partial<T>` is more type-safe and specific.

### Q: How do you ensure at least one property is provided with Partial?

**A:** Create a custom type like `AtLeastOne<T>` that combines `Partial<T>` with a union of single-property types, ensuring at least one property is present.

## Summary

`Partial<T>` is essential for:

1. **Form State Management**: Allowing partial updates to form fields
2. **API Updates**: Supporting partial resource updates
3. **Configuration**: Enabling flexible configuration overrides
4. **Filters**: Allowing dynamic query filters
5. **Component Props**: Creating flexible component APIs

**Key Benefits:**
- Type safety with flexibility
- Reduced boilerplate code
- Better maintainability
- Intuitive API design

**Interview Tip:** "`Partial<T>` makes all properties of T optional, perfect for update operations, form state, and configuration objects where you want to allow modifying any subset of fields while maintaining type safety."