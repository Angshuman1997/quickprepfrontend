# How to extend an interface?

## Question
How to extend an interface?

## Answer

In TypeScript, extending interfaces allows you to create new interfaces that inherit properties and methods from existing ones. This promotes code reusability and maintains type safety. There are several ways to extend interfaces, each with different use cases.

## Basic Interface Extension

### Using `extends` Keyword

```typescript
// Base interface
interface Person {
    id: number;
    name: string;
    age: number;
}

// Extended interface
interface Employee extends Person {
    employeeId: string;
    department: string;
    salary: number;
}

// Usage
const employee: Employee = {
    id: 1,
    name: "John Doe",
    age: 30,
    employeeId: "EMP001",
    department: "Engineering",
    salary: 75000
};
```

## Multiple Interface Extension

```typescript
interface CanWalk {
    walk(): void;
}

interface CanSwim {
    swim(): void;
}

interface CanFly {
    fly(): void;
}

// Extend multiple interfaces
interface Duck extends CanWalk, CanSwim, CanFly {
    quack(): void;
}

class MallardDuck implements Duck {
    walk() {
        console.log("Walking...");
    }

    swim() {
        console.log("Swimming...");
    }

    fly() {
        console.log("Flying...");
    }

    quack() {
        console.log("Quack!");
    }
}
```

## Extending Interfaces with Additional Properties

### Adding Optional Properties

```typescript
interface User {
    id: number;
    name: string;
    email: string;
}

interface ExtendedUser extends User {
    phone?: string;
    address?: {
        street: string;
        city: string;
        zipCode: string;
    };
    preferences?: {
        theme: 'light' | 'dark';
        notifications: boolean;
    };
}

const user: ExtendedUser = {
    id: 1,
    name: "John Doe",
    email: "john@example.com",
    phone: "+1-555-0123",
    address: {
        street: "123 Main St",
        city: "Anytown",
        zipCode: "12345"
    }
};
```

### Adding Method Signatures

```typescript
interface Repository<T> {
    findById(id: number): Promise<T>;
    findAll(): Promise<T[]>;
    create(entity: T): Promise<T>;
}

interface UserRepository extends Repository<User> {
    findByEmail(email: string): Promise<User>;
    updatePassword(id: number, newPassword: string): Promise<void>;
    deactivateUser(id: number): Promise<void>;
}
```

## Extending Generic Interfaces

```typescript
// Generic base interface
interface Collection<T> {
    items: T[];
    add(item: T): void;
    remove(item: T): boolean;
    find(predicate: (item: T) => boolean): T | undefined;
}

// Extend with specific type
interface UserCollection extends Collection<User> {
    findByName(name: string): User[];
    findByEmail(email: string): User | undefined;
}

// Extend with additional generic parameter
interface PaginatedCollection<T> extends Collection<T> {
    pageSize: number;
    currentPage: number;
    totalPages: number;
    nextPage(): Promise<T[]>;
    previousPage(): Promise<T[]>;
}
```

## Interface Extension with Type Aliases

### Extending Type Aliases (Using Intersection Types)

```typescript
// Type alias
type BaseEntity = {
    id: number;
    createdAt: Date;
    updatedAt: Date;
};

// Extend type alias with interface
interface User extends BaseEntity {
    name: string;
    email: string;
    isActive: boolean;
}

// Or extend with another type alias
type ExtendedEntity = BaseEntity & {
    deletedAt?: Date;
    version: number;
};
```

## Advanced Extension Patterns

### Conditional Extension

```typescript
interface BaseProps {
    className?: string;
    style?: React.CSSProperties;
}

interface InputProps extends BaseProps {
    value: string;
    onChange: (value: string) => void;
    placeholder?: string;
    disabled?: boolean;
}

interface SelectProps extends BaseProps {
    value: string;
    onChange: (value: string) => void;
    options: Array<{ value: string; label: string }>;
    multiple?: boolean;
}

// Conditional props based on component type
type FormFieldProps<T extends 'input' | 'select'> =
    T extends 'input' ? InputProps : SelectProps;
```

### Extension with Method Overriding

```typescript
interface Logger {
    log(message: string): void;
    error(message: string): void;
}

interface AdvancedLogger extends Logger {
    // Override method with different signature
    log(message: string, level?: 'info' | 'warn' | 'error'): void;
    // Add new methods
    debug(message: string): void;
    setLevel(level: string): void;
}
```

## Real-World Examples

### 1. React Component Props Extension

```typescript
interface BaseComponentProps {
    className?: string;
    style?: React.CSSProperties;
    children?: React.ReactNode;
    onClick?: () => void;
}

interface ButtonProps extends BaseComponentProps {
    variant?: 'primary' | 'secondary' | 'danger';
    size?: 'small' | 'medium' | 'large';
    disabled?: boolean;
    loading?: boolean;
    onClick: () => void; // Override to make required
}

interface InputProps extends BaseComponentProps {
    value: string;
    onChange: (value: string) => void;
    placeholder?: string;
    type?: 'text' | 'email' | 'password';
    error?: string;
}

// Usage in components
const Button: React.FC<ButtonProps> = ({ children, onClick, variant = 'primary', ...props }) => {
    return (
        <button
            className={`btn btn-${variant}`}
            onClick={onClick}
            {...props}
        >
            {children}
        </button>
    );
};

const Input: React.FC<InputProps> = ({ value, onChange, error, ...props }) => {
    return (
        <div>
            <input
                value={value}
                onChange={(e) => onChange(e.target.value)}
                {...props}
            />
            {error && <span className="error">{error}</span>}
        </div>
    );
};
```

### 2. API Response Extension

```typescript
interface APIResponse {
    success: boolean;
    message: string;
    timestamp: Date;
}

interface UserResponse extends APIResponse {
    data: {
        id: number;
        name: string;
        email: string;
    };
}

interface ListResponse<T> extends APIResponse {
    data: T[];
    pagination: {
        page: number;
        limit: number;
        total: number;
        totalPages: number;
    };
}

interface ErrorResponse extends APIResponse {
    success: false;
    error: {
        code: string;
        details: string;
    };
}

// Usage
async function fetchUser(id: number): Promise<UserResponse | ErrorResponse> {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
}

async function fetchUsers(page = 1): Promise<ListResponse<User>> {
    const response = await fetch(`/api/users?page=${page}`);
    return response.json();
}
```

### 3. Database Model Extension

```typescript
interface BaseModel {
    id: number;
    createdAt: Date;
    updatedAt: Date;
}

interface SoftDeleteModel extends BaseModel {
    deletedAt?: Date;
}

interface UserModel extends SoftDeleteModel {
    name: string;
    email: string;
    password: string;
    role: 'admin' | 'user' | 'moderator';
    isActive: boolean;
}

interface PostModel extends SoftDeleteModel {
    title: string;
    content: string;
    authorId: number;
    published: boolean;
    tags: string[];
}

// Repository interfaces
interface BaseRepository<T extends BaseModel> {
    findById(id: number): Promise<T | null>;
    findAll(): Promise<T[]>;
    create(data: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<T>;
    update(id: number, data: Partial<T>): Promise<T>;
    delete(id: number): Promise<void>;
}

interface UserRepository extends BaseRepository<UserModel> {
    findByEmail(email: string): Promise<UserModel | null>;
    updatePassword(id: number, password: string): Promise<void>;
}

interface PostRepository extends BaseRepository<PostModel> {
    findByAuthor(authorId: number): Promise<PostModel[]>;
    findPublished(): Promise<PostModel[]>;
}
```

## Extension vs Declaration Merging

### Declaration Merging (Same Interface Name)

```typescript
// Declaration merging - adds to existing interface
interface Window {
    myCustomMethod(): void;
}

interface Window {
    anotherMethod(param: string): void;
}

// Result: Window interface has both methods
window.myCustomMethod();
window.anotherMethod("test");
```

### Extension (New Interface Name)

```typescript
interface BaseWindow {
    close(): void;
}

interface ExtendedWindow extends BaseWindow {
    myCustomMethod(): void;
}

// Result: Two separate interfaces
const baseWindow: BaseWindow = window;
const extendedWindow: ExtendedWindow = window as ExtendedWindow;
```

## Best Practices

### 1. **Use Extension for Inheritance**

```typescript
// ✅ Good: Clear inheritance hierarchy
interface Animal {
    name: string;
    move(): void;
}

interface Bird extends Animal {
    fly(): void;
}

interface Fish extends Animal {
    swim(): void;
}
```

### 2. **Prefer Composition Over Deep Hierarchies**

```typescript
// ✅ Good: Shallow hierarchy with composition
interface CanEat {
    eat(): void;
}

interface CanSleep {
    sleep(): void;
}

interface CanFly {
    fly(): void;
}

interface Bird extends CanEat, CanSleep, CanFly {
    layEggs(): void;
}
```

### 3. **Use Extension for API Evolution**

```typescript
// Version 1
interface APIClient {
    get(endpoint: string): Promise<any>;
    post(endpoint: string, data: any): Promise<any>;
}

// Version 2 - extend for backward compatibility
interface APIClientV2 extends APIClient {
    put(endpoint: string, data: any): Promise<any>;
    delete(endpoint: string): Promise<any>;
    setAuthToken(token: string): void;
}
```

### 4. **Avoid Circular Dependencies**

```typescript
// ❌ Bad: Circular reference
interface A extends B {
    b: B;
}

interface B extends A {
    a: A;
}

// ✅ Good: Use type aliases for circular references
interface A {
    b: B;
}

type B = {
    a: A;
};
```

## Common Interview Questions

### Q: What's the difference between extending interfaces and declaration merging?

**A:** Extension creates a new interface that inherits from the base interface. Declaration merging adds properties to an existing interface with the same name. Use extension when you want to create a subtype, use merging when you want to augment existing types (like third-party libraries).

### Q: Can you extend multiple interfaces?

**A:** Yes, you can extend multiple interfaces by separating them with commas: `interface C extends A, B { ... }`. The resulting interface will have all properties from both A and B.

### Q: Can an interface extend a type alias?

**A:** Yes, interfaces can extend type aliases using the `extends` keyword. However, type aliases cannot extend interfaces directly (you'd use intersection types instead).

### Q: When should you use interface extension vs type intersection?

**A:** Use interface extension when you want to create a named type hierarchy and need declaration merging. Use type intersections (`A & B`) when you want to combine types without creating a new named interface.

## Summary

Interface extension in TypeScript provides powerful ways to create type hierarchies and promote code reusability:

- **Basic Extension**: `interface B extends A { ... }`
- **Multiple Extension**: `interface C extends A, B { ... }`
- **Generic Extension**: `interface Repo<T> extends BaseRepo<T> { ... }`
- **Method Overriding**: Change signatures in extended interfaces
- **Property Addition**: Add new required or optional properties

**Key Benefits:**
- Type safety and inheritance
- Code reusability
- Backward compatibility
- Clear type hierarchies
- IntelliSense support

**Best Practices:**
- Keep hierarchies shallow
- Use composition over deep inheritance
- Prefer interfaces for object shapes
- Use extension for API evolution
- Avoid circular dependencies

**Interview Tip:** "Interface extension allows you to create type hierarchies where derived interfaces inherit all properties and methods from base interfaces, enabling code reuse while maintaining type safety. It's particularly useful for creating component prop interfaces, API response types, and domain model hierarchies."