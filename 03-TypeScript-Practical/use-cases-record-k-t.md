# Use cases for Record<K, T>

## Question
Use cases for Record<K, T>

## Answer

`Record<K, T>` is a TypeScript utility type that creates an object type with keys of type `K` and values of type `T`. It's incredibly useful for creating dictionaries, maps, and configuration objects with consistent typing.

## Basic Usage

```typescript
// Record<KeyType, ValueType>
type StringDictionary = Record<string, string>;
type NumberDictionary = Record<string, number>;
type UserRoles = Record<string, 'admin' | 'user' | 'moderator'>;

// Usage
const colors: Record<string, string> = {
    primary: '#007bff',
    secondary: '#6c757d',
    success: '#28a745',
    danger: '#dc3545'
};

const userPermissions: Record<string, boolean> = {
    canEdit: true,
    canDelete: false,
    canPublish: true
};

const userRoles: UserRoles = {
    'user123': 'admin',
    'user456': 'user',
    'user789': 'moderator'
};
```

## Configuration Objects

### 1. **Theme Configuration**

```typescript
interface ThemeColors {
    primary: string;
    secondary: string;
    success: string;
    danger: string;
    warning: string;
    info: string;
    light: string;
    dark: string;
}

interface ThemeSpacing {
    xs: string;
    sm: string;
    md: string;
    lg: string;
    xl: string;
}

interface ThemeBreakpoints {
    xs: string;
    sm: string;
    md: string;
    lg: string;
    xl: string;
}

// Using Record for theme configuration
type ThemeConfig = {
    colors: Record<keyof ThemeColors, string>;
    spacing: Record<keyof ThemeSpacing, string>;
    breakpoints: Record<keyof ThemeBreakpoints, string>;
    fontSize: Record<string, string>; // Flexible keys
    fontWeight: Record<string, number>; // Flexible keys
};

const defaultTheme: ThemeConfig = {
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
    breakpoints: {
        xs: '0px',
        sm: '576px',
        md: '768px',
        lg: '992px',
        xl: '1200px'
    },
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
        medium: 500,
        bold: 700
    }
};

// Usage
function createTheme(overrides: Partial<ThemeConfig> = {}): ThemeConfig {
    return {
        ...defaultTheme,
        ...overrides,
        colors: { ...defaultTheme.colors, ...overrides.colors },
        spacing: { ...defaultTheme.spacing, ...overrides.spacing },
        breakpoints: { ...defaultTheme.breakpoints, ...overrides.breakpoints },
        fontSize: { ...defaultTheme.fontSize, ...overrides.fontSize },
        fontWeight: { ...defaultTheme.fontWeight, ...overrides.fontWeight }
    };
}
```

### 2. **API Configuration**

```typescript
type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';

interface APIEndpoint {
    url: string;
    method: HTTPMethod;
    requiresAuth: boolean;
    timeout?: number;
}

// API endpoints configuration
type APIConfig = Record<string, APIEndpoint>;

const apiConfig: APIConfig = {
    getUsers: {
        url: '/api/users',
        method: 'GET',
        requiresAuth: true
    },
    createUser: {
        url: '/api/users',
        method: 'POST',
        requiresAuth: true
    },
    getUser: {
        url: '/api/users/:id',
        method: 'GET',
        requiresAuth: true
    },
    updateUser: {
        url: '/api/users/:id',
        method: 'PUT',
        requiresAuth: true
    },
    deleteUser: {
        url: '/api/users/:id',
        method: 'DELETE',
        requiresAuth: true
    },
    login: {
        url: '/api/auth/login',
        method: 'POST',
        requiresAuth: false,
        timeout: 10000
    }
};

// API client using the configuration
class APIClient {
    constructor(private config: APIConfig) {}

    async request<T>(endpointKey: keyof APIConfig, params?: Record<string, string>): Promise<T> {
        const endpoint = this.config[endpointKey];
        let url = endpoint.url;

        // Replace URL parameters
        if (params) {
            Object.entries(params).forEach(([key, value]) => {
                url = url.replace(`:${key}`, value);
            });
        }

        const response = await fetch(url, {
            method: endpoint.method,
            headers: {
                'Content-Type': 'application/json',
                ...(endpoint.requiresAuth && { Authorization: `Bearer ${this.getToken()}` })
            },
            ...(endpoint.timeout && { signal: AbortSignal.timeout(endpoint.timeout) })
        });

        if (!response.ok) {
            throw new Error(`API request failed: ${response.status}`);
        }

        return response.json();
    }

    private getToken(): string {
        // Implementation to get auth token
        return localStorage.getItem('token') || '';
    }
}
```

## Data Structures and Caching

### 1. **In-Memory Cache**

```typescript
interface CacheEntry<T> {
    data: T;
    timestamp: number;
    ttl: number; // Time to live in milliseconds
}

// Generic cache using Record
class MemoryCache<T = any> {
    private cache: Record<string, CacheEntry<T>> = {};

    set(key: string, data: T, ttl = 5 * 60 * 1000): void { // 5 minutes default
        this.cache[key] = {
            data,
            timestamp: Date.now(),
            ttl
        };
    }

    get(key: string): T | null {
        const entry = this.cache[key];

        if (!entry) return null;

        if (Date.now() - entry.timestamp > entry.ttl) {
            delete this.cache[key];
            return null;
        }

        return entry.data;
    }

    has(key: string): boolean {
        return this.get(key) !== null;
    }

    delete(key: string): void {
        delete this.cache[key];
    }

    clear(): void {
        this.cache = {};
    }

    // Get all valid entries
    entries(): Record<string, T> {
        const validEntries: Record<string, T> = {};

        Object.entries(this.cache).forEach(([key, entry]) => {
            if (Date.now() - entry.timestamp <= entry.ttl) {
                validEntries[key] = entry.data;
            } else {
                delete this.cache[key];
            }
        });

        return validEntries;
    }
}

// Usage
const userCache = new MemoryCache<User>();

userCache.set('user123', { id: 123, name: 'John', email: 'john@example.com' });
const user = userCache.get('user123'); // Returns User or null
```

### 2. **Translation/Localization**

```typescript
type LanguageCode = 'en' | 'es' | 'fr' | 'de' | 'ja';

type TranslationKey =
    | 'welcome'
    | 'login'
    | 'logout'
    | 'save'
    | 'cancel'
    | 'error'
    | 'loading';

// Translation dictionary
type Translations = Record<LanguageCode, Record<TranslationKey, string>>;

const translations: Translations = {
    en: {
        welcome: 'Welcome',
        login: 'Login',
        logout: 'Logout',
        save: 'Save',
        cancel: 'Cancel',
        error: 'Error',
        loading: 'Loading...'
    },
    es: {
        welcome: 'Bienvenido',
        login: 'Iniciar sesión',
        logout: 'Cerrar sesión',
        save: 'Guardar',
        cancel: 'Cancelar',
        error: 'Error',
        loading: 'Cargando...'
    },
    fr: {
        welcome: 'Bienvenue',
        login: 'Se connecter',
        logout: 'Se déconnecter',
        save: 'Enregistrer',
        cancel: 'Annuler',
        error: 'Erreur',
        loading: 'Chargement...'
    },
    de: {
        welcome: 'Willkommen',
        login: 'Anmelden',
        logout: 'Abmelden',
        save: 'Speichern',
        cancel: 'Abbrechen',
        error: 'Fehler',
        loading: 'Laden...'
    },
    ja: {
        welcome: 'ようこそ',
        login: 'ログイン',
        logout: 'ログアウト',
        save: '保存',
        cancel: 'キャンセル',
        error: 'エラー',
        loading: '読み込み中...'
    }
};

// Translation hook
function useTranslation(language: LanguageCode) {
    const t = (key: TranslationKey): string => {
        return translations[language][key] || translations.en[key] || key;
    };

    return { t };
}

// Usage
function App() {
    const { t } = useTranslation('es');

    return (
        <div>
            <h1>{t('welcome')}</h1>
            <button>{t('login')}</button>
            <button>{t('save')}</button>
        </div>
    );
}
```

## Form Validation and Error Handling

### 1. **Form Validation Schema**

```typescript
type ValidationRule = (value: any) => string | null;

type ValidationRules = Record<string, ValidationRule[]>;

interface FormField {
    name: string;
    type: 'text' | 'email' | 'password' | 'number';
    label: string;
    required: boolean;
    validationRules: ValidationRule[];
}

// Form configuration using Record
type FormConfig = Record<string, FormField>;

const loginFormConfig: FormConfig = {
    email: {
        name: 'email',
        type: 'email',
        label: 'Email',
        required: true,
        validationRules: [
            (value) => !value ? 'Email is required' : null,
            (value) => !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) ? 'Invalid email format' : null
        ]
    },
    password: {
        name: 'password',
        type: 'password',
        label: 'Password',
        required: true,
        validationRules: [
            (value) => !value ? 'Password is required' : null,
            (value) => value.length < 8 ? 'Password must be at least 8 characters' : null
        ]
    }
};

// Form errors using Record
type FormErrors = Record<string, string[]>;

function validateForm(config: FormConfig, values: Record<string, any>): FormErrors {
    const errors: FormErrors = {};

    Object.entries(config).forEach(([fieldName, fieldConfig]) => {
        const value = values[fieldName];
        const fieldErrors: string[] = [];

        fieldConfig.validationRules.forEach(rule => {
            const error = rule(value);
            if (error) fieldErrors.push(error);
        });

        if (fieldErrors.length > 0) {
            errors[fieldName] = fieldErrors;
        }
    });

    return errors;
}

// Usage
const formValues = { email: 'invalid-email', password: '123' };
const errors = validateForm(loginFormConfig, formValues);
// errors: { email: ['Invalid email format'], password: ['Password must be at least 8 characters'] }
```

### 2. **Error Messages Dictionary**

```typescript
type ErrorCode =
    | 'VALIDATION_ERROR'
    | 'NETWORK_ERROR'
    | 'AUTHENTICATION_ERROR'
    | 'AUTHORIZATION_ERROR'
    | 'NOT_FOUND'
    | 'SERVER_ERROR';

type ErrorMessages = Record<ErrorCode, string>;

const errorMessages: ErrorMessages = {
    VALIDATION_ERROR: 'Please check your input and try again.',
    NETWORK_ERROR: 'Network connection failed. Please check your internet connection.',
    AUTHENTICATION_ERROR: 'Invalid credentials. Please check your username and password.',
    AUTHORIZATION_ERROR: 'You do not have permission to perform this action.',
    NOT_FOUND: 'The requested resource was not found.',
    SERVER_ERROR: 'An unexpected error occurred. Please try again later.'
};

// Error handling utility
function handleError(errorCode: ErrorCode): void {
    const message = errorMessages[errorCode];
    console.error(message);
    // Show toast notification, etc.
}

// API error mapping
function mapAPIErrorToCode(statusCode: number): ErrorCode {
    switch (statusCode) {
        case 400: return 'VALIDATION_ERROR';
        case 401: return 'AUTHENTICATION_ERROR';
        case 403: return 'AUTHORIZATION_ERROR';
        case 404: return 'NOT_FOUND';
        case 500: return 'SERVER_ERROR';
        default: return 'NETWORK_ERROR';
    }
}
```

## React State Management

### 1. **Reducer Actions**

```typescript
interface User {
    id: number;
    name: string;
    email: string;
}

// Action types
type UserActionType = 'SET_USERS' | 'ADD_USER' | 'UPDATE_USER' | 'DELETE_USER';

// Action payloads
type UserActionPayloads = {
    SET_USERS: User[];
    ADD_USER: User;
    UPDATE_USER: User;
    DELETE_USER: number;
};

// Action using Record
type UserAction = {
    [K in UserActionType]: {
        type: K;
        payload: UserActionPayloads[K];
    };
}[UserActionType];

// Reducer using the typed actions
function userReducer(state: User[], action: UserAction): User[] {
    switch (action.type) {
        case 'SET_USERS':
            return action.payload;

        case 'ADD_USER':
            return [...state, action.payload];

        case 'UPDATE_USER':
            return state.map(user =>
                user.id === action.payload.id ? action.payload : user
            );

        case 'DELETE_USER':
            return state.filter(user => user.id !== action.payload);

        default:
            return state;
    }
}

// Usage
const initialState: User[] = [];

const actions = {
    setUsers: (users: User[]): UserAction => ({ type: 'SET_USERS', payload: users }),
    addUser: (user: User): UserAction => ({ type: 'ADD_USER', payload: user }),
    updateUser: (user: User): UserAction => ({ type: 'UPDATE_USER', payload: user }),
    deleteUser: (id: number): UserAction => ({ type: 'DELETE_USER', payload: id })
};
```

### 2. **Component State with Dynamic Keys**

```typescript
interface FormState {
    values: Record<string, any>;
    errors: Record<string, string[]>;
    touched: Record<string, boolean>;
}

const initialFormState: FormState = {
    values: {},
    errors: {},
    touched: {}
};

// Custom hook for form state
function useFormState(initialValues: Record<string, any>) {
    const [state, setState] = useState<FormState>({
        values: initialValues,
        errors: {},
        touched: {}
    });

    const setValue = (field: string, value: any) => {
        setState(prev => ({
            ...prev,
            values: { ...prev.values, [field]: value }
        }));
    };

    const setError = (field: string, errors: string[]) => {
        setState(prev => ({
            ...prev,
            errors: { ...prev.errors, [field]: errors }
        }));
    };

    const setTouched = (field: string, touched = true) => {
        setState(prev => ({
            ...prev,
            touched: { ...prev.touched, [field]: touched }
        }));
    };

    const reset = () => {
        setState({
            values: initialValues,
            errors: {},
            touched: {}
        });
    };

    return {
        ...state,
        setValue,
        setError,
        setTouched,
        reset
    };
}

// Usage
function ContactForm() {
    const { values, errors, touched, setValue, setError, setTouched } = useFormState({
        name: '',
        email: '',
        message: ''
    });

    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault();
        // Validation logic
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                value={values.name}
                onChange={(e) => setValue('name', e.target.value)}
                onBlur={() => setTouched('name')}
            />
            {touched.name && errors.name && <span>{errors.name[0]}</span>}
            {/* Other fields */}
        </form>
    );
}
```

## Advanced Patterns

### 1. **Dynamic Object Keys with Constraints**

```typescript
// Record with constrained keys
type UserPreferences = Record<`pref_${string}`, boolean | string | number>;

const userPrefs: UserPreferences = {
    pref_theme: 'dark',
    pref_notifications: true,
    pref_language: 'en',
    pref_volume: 75
};

// Record with union of literal types
type ComponentVariants = 'button' | 'input' | 'select' | 'textarea';
type ComponentSizes = 'sm' | 'md' | 'lg';

type ComponentStyles = Record<ComponentVariants, Record<ComponentSizes, string>>;

const componentStyles: ComponentStyles = {
    button: {
        sm: 'px-2 py-1 text-sm',
        md: 'px-4 py-2 text-base',
        lg: 'px-6 py-3 text-lg'
    },
    input: {
        sm: 'px-2 py-1 text-sm',
        md: 'px-3 py-2 text-base',
        lg: 'px-4 py-3 text-lg'
    },
    select: {
        sm: 'px-2 py-1 text-sm',
        md: 'px-3 py-2 text-base',
        lg: 'px-4 py-3 text-lg'
    },
    textarea: {
        sm: 'px-2 py-1 text-sm',
        md: 'px-3 py-2 text-base',
        lg: 'px-4 py-3 text-lg'
    }
};
```

### 2. **Generic Dictionary Types**

```typescript
// Generic dictionary with key transformation
type Dictionary<T> = Record<string, T>;
type NumericDictionary<T> = Record<number, T>;

// Event handlers dictionary
type EventHandlers<T extends string> = Record<`on${Capitalize<T>}`, () => void>;

type ButtonEvents = EventHandlers<'click' | 'hover' | 'focus'>;
// Result: { onClick: () => void; onHover: () => void; onFocus: () => void }

// API responses dictionary
type APIResponses = Record<string, { data: any; loading: boolean; error: string | null }>;

const apiState: APIResponses = {
    users: { data: [], loading: false, error: null },
    posts: { data: [], loading: true, error: null },
    comments: { data: [], loading: false, error: 'Failed to load' }
};
```

## Best Practices

### 1. **Use Record for Configuration Objects**

```typescript
// ✅ Good: Structured configuration
type DatabaseConfig = Record<string, {
    host: string;
    port: number;
    database: string;
    ssl: boolean;
}>;

// ❌ Avoid: Loose object types
// type DatabaseConfig = { [key: string]: any };
```

### 2. **Constrain Keys When Possible**

```typescript
// ✅ Good: Constrained keys for type safety
type UserPermissions = Record<'read' | 'write' | 'delete' | 'admin', boolean>;

// ❌ Avoid: String keys without constraints
// type UserPermissions = Record<string, boolean>; // Too broad
```

### 3. **Use Record for Mappings and Dictionaries**

```typescript
// ✅ Good: Type-safe mappings
type StatusLabels = Record<'pending' | 'approved' | 'rejected', string>;
type PriorityColors = Record<'low' | 'medium' | 'high', string>;

// ✅ Good: Flexible but typed dictionaries
type Metadata = Record<string, string | number | boolean>;
```

### 4. **Combine Record with Other Utilities**

```typescript
// ✅ Good: Combine for complex types
type PartialRecord<K extends string, T> = Partial<Record<K, T>>;
type RequiredRecord<K extends string, T> = Required<Record<K, T>>;
type ReadonlyRecord<K extends string, T> = Readonly<Record<K, T>>;
```

## Common Interview Questions

### Q: When should you use Record<K, T> instead of a regular object type?

**A:** Use `Record<K, T>` when you need a dictionary-like object where all values have the same type, or when you want to ensure all keys of a certain type are present. It's better than `{ [key: string]: T }` because you can constrain the key types.

### Q: Can Record keys be union types?

**A:** Yes! `Record<'a' | 'b' | 'c', number>` creates a type that requires exactly those three keys with number values. This is perfect for configuration objects with known keys.

### Q: What's the difference between Record<K, T> and { [key: K]: T }?

**A:** They're equivalent, but `Record<K, T>` is more readable and is the preferred syntax. Both create an object type with keys of type K and values of type T.

### Q: How do you make some properties of a Record optional?

**A:** Use `Partial<Record<K, T>>` to make all properties optional, or create a custom type that combines required and optional keys.

## Summary

`Record<K, T>` is essential for:

1. **Configuration Objects**: Theme configs, API settings, app preferences
2. **Dictionaries/Maps**: Caches, translations, error messages
3. **State Management**: Form state, reducer actions, dynamic properties
4. **Type-Safe Mappings**: Status labels, color schemes, validation rules
5. **API Responses**: Structured data with consistent value types

**Key Benefits:**
- Type safety for dictionary-like objects
- Consistent value types across all keys
- IntelliSense support for known keys
- Prevention of typos in key names
- Flexible yet constrained typing

**Interview Tip:** "`Record<K, T>` creates an object type with keys of type K and values of type T. It's perfect for configuration objects, dictionaries, and mappings where you need consistent typing across all properties."