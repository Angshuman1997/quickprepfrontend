# Difference between type and interface

## Question
Difference between type and interface.

## Answer

In TypeScript, both `type` and `interface` are used to define object shapes and contracts, but they have different capabilities and use cases. Understanding when to use each is crucial for writing maintainable TypeScript code.

## Basic Syntax

### Interface

```typescript
// Interface definition
interface User {
    id: number;
    name: string;
    email: string;
}

// Usage
const user: User = {
    id: 1,
    name: "John Doe",
    email: "john@example.com"
};
```

### Type Alias

```typescript
// Type alias definition
type User = {
    id: number;
    name: string;
    email: string;
};

// Usage
const user: User = {
    id: 1,
    name: "John Doe",
    email: "john@example.com"
};
```

## Key Differences

### 1. **Declaration Merging (Interface Only)**

```typescript
// ✅ Interfaces can merge
interface User {
    id: number;
    name: string;
}

interface User {
    email: string;  // Adds to existing interface
}

// Result: User has id, name, and email
const user: User = {
    id: 1,
    name: "John",
    email: "john@example.com"
};

// ❌ Types cannot merge
type User = {
    id: number;
    name: string;
};

// Error: Duplicate identifier 'User'
type User = {
    email: string;
};
```

**Use Case:** Declaration merging is useful for:
- Third-party library augmentation
- Splitting interface definitions across files
- Plugin/extension patterns

```typescript
// Example: Library augmentation
interface Window {
    myCustomProperty: string;
}

interface Window {
    anotherProperty: number;
}

// Now Window has both properties
window.myCustomProperty = "value";
window.anotherProperty = 42;
```

### 2. **Extending (Both Support, Different Syntax)**

```typescript
// Interface extending interface
interface Animal {
    name: string;
}

interface Dog extends Animal {
    breed: string;
}

// Type extending type
type Animal = {
    name: string;
};

type Dog = Animal & {
    breed: string;
};

// Interface extending type
type BaseEntity = {
    id: number;
    createdAt: Date;
};

interface User extends BaseEntity {
    name: string;
    email: string;
}

// Type extending interface (using intersection)
interface BaseEntity {
    id: number;
    createdAt: Date;
}

type User = BaseEntity & {
    name: string;
    email: string;
};
```

### 3. **Union Types (Type Only)**

```typescript
// ✅ Types support union types
type Status = "pending" | "approved" | "rejected";

type APIResponse = SuccessResponse | ErrorResponse;

// ❌ Interfaces don't support union types directly
// (You'd need to use type for this)
```

### 4. **Primitive Types and Complex Types (Type Only)**

```typescript
// ✅ Types can alias primitives
type UserId = number;
type Email = string;

// ✅ Types can create complex types
type Point = [number, number];  // Tuple
type Callback = (value: any) => void;
type Dictionary<T> = { [key: string]: T };

// ❌ Interfaces cannot do this
// interface UserId = number;  // Error
```

### 5. **Computed Property Names (Type Only)**

```typescript
// ✅ Types support computed properties
const propName = "dynamicKey";

type DynamicObject = {
    [propName]: string;  // Computed property name
    staticKey: number;
};

// ❌ Interfaces don't support computed property names
// interface DynamicObject {
//     [propName]: string;  // Error
// }
```

## When to Use Interface

### ✅ **Use Interfaces for:**

#### 1. **Object Shapes (Primary Use Case)**

```typescript
interface Product {
    id: number;
    name: string;
    price: number;
    category: string;
}

interface APIResponse<T> {
    data: T;
    status: number;
    message: string;
}
```

#### 2. **Classes Implementing Interfaces**

```typescript
interface Repository<T> {
    findById(id: number): Promise<T>;
    findAll(): Promise<T[]>;
    create(entity: T): Promise<T>;
    update(id: number, entity: T): Promise<T>;
    delete(id: number): Promise<void>;
}

class UserRepository implements Repository<User> {
    async findById(id: number): Promise<User> {
        // Implementation
        return {} as User;
    }

    // ... other methods
}
```

#### 3. **Declaration Merging Scenarios**

```typescript
// Useful for library types
interface ExpressRequest {
    user?: User;
}

// Later in another file
interface ExpressRequest {
    session?: Session;
}

// Now ExpressRequest has both user and session
```

#### 4. **Public API Definitions**

```typescript
// For libraries or public APIs
interface Config {
    apiUrl: string;
    timeout: number;
    retries: number;
}

interface SDK {
    connect(config: Config): Promise<void>;
    disconnect(): Promise<void>;
    send(data: any): Promise<Response>;
}
```

## When to Use Type

### ✅ **Use Types for:**

#### 1. **Union Types**

```typescript
type UserRole = "admin" | "user" | "moderator";

type APIStatus = "idle" | "loading" | "success" | "error";

type FormField = string | number | boolean | Date;
```

#### 2. **Primitive Type Aliases**

```typescript
type UserId = number;
type Email = string;
type Timestamp = number;

function getUser(id: UserId): Promise<User> {
    // Implementation
}
```

#### 3. **Complex Type Operations**

```typescript
type PartialUser = Partial<User>;
type RequiredUser = Required<User>;
type ReadonlyUser = Readonly<User>;
type PickedUser = Pick<User, "id" | "name">;
type OmittedUser = Omit<User, "password">;
```

#### 4. **Mapped Types**

```typescript
type Optional<T> = {
    [P in keyof T]?: T[P];
};

type Readonly<T> = {
    readonly [P in keyof T]: T[P];
};

type Nullable<T> = {
    [P in keyof T]: T[P] | null;
};
```

#### 5. **Conditional Types**

```typescript
type IsString<T> = T extends string ? true : false;

type FunctionReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type ArrayElementType<T> = T extends (infer U)[] ? U : never;
```

#### 6. **Template Literal Types**

```typescript
type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE";

type APIEndpoint = `/${string}`;

type EventName = `on${Capitalize<string>}`;
```

## Best Practices

### 1. **Prefer Interface for Object Shapes**

```typescript
// ✅ Good: Use interface for object contracts
interface User {
    id: number;
    name: string;
    email: string;
}

// ❌ Avoid: Using type for simple object shapes
type User = {
    id: number;
    name: string;
    email: string;
};
```

### 2. **Use Type for Unions and Primitives**

```typescript
// ✅ Good: Use type for unions
type Status = "pending" | "approved" | "rejected";

// ✅ Good: Use type for primitives
type UserId = number;
type Callback<T> = (value: T) => void;
```

### 3. **Consistent Naming Conventions**

```typescript
// Interfaces: PascalCase, often end with nouns
interface UserRepository {}
interface ProductService {}

// Types: PascalCase, descriptive names
type UserId = number;
type APIResponse<T> = { data: T; status: number };
type OptionalUser = Partial<User>;
```

### 4. **Interface for Public APIs, Type for Internal Logic**

```typescript
// Public API - use interface
interface DatabaseConfig {
    host: string;
    port: number;
    database: string;
    username: string;
    password: string;
}

// Internal logic - use type
type ConnectionStatus = "connecting" | "connected" | "disconnected" | "error";
type QueryResult<T> = { data: T[]; count: number };
```

## Advanced Patterns

### 1. **Hybrid Approach**

```typescript
// Use interface for base shape
interface BaseEntity {
    id: number;
    createdAt: Date;
    updatedAt: Date;
}

// Use type for variations
type CreateEntity = Omit<BaseEntity, 'id' | 'createdAt' | 'updatedAt'>;
type UpdateEntity = Partial<Omit<BaseEntity, 'id' | 'createdAt'>>;
type EntityResponse = BaseEntity & { computedField: string };
```

### 2. **Interface Extending Type**

```typescript
// Type for complex union
type ComponentProps = {
    variant: "primary" | "secondary";
    size: "small" | "medium" | "large";
};

// Interface extending type
interface ButtonProps extends ComponentProps {
    onClick: () => void;
    children: React.ReactNode;
    disabled?: boolean;
}
```

### 3. **Type-Only Imports/Exports**

```typescript
// Type-only imports (TypeScript 3.8+)
import type { User, APIResponse } from './types';

// This creates no runtime code
export type { User, APIResponse };
```

## Common Interview Questions

### Q: When should you use interface over type?

**A:** Use interfaces when:
- Defining object shapes that might be extended or merged
- Creating contracts for classes to implement
- Building public APIs or library definitions
- You need declaration merging capabilities

### Q: When should you use type over interface?

**A:** Use types when:
- Creating union types
- Aliasing primitive types
- Building complex type operations (mapped types, conditional types)
- You need template literal types or computed properties

### Q: Can you extend a type with an interface?

**A:** Yes! Interfaces can extend types using the `extends` keyword, and types can extend interfaces using intersection (`&`).

```typescript
type Base = { id: number };
interface Extended extends Base { name: string }

interface BaseInterface { value: string }
type ExtendedType = BaseInterface & { count: number }
```

## Migration Guide

### Converting Between Interface and Type

```typescript
// Interface to Type
interface User {
    name: string;
    age: number;
}

// Convert to type
type UserType = {
    name: string;
    age: number;
};

// Type to Interface (when possible)
type Status = "active" | "inactive";

// Convert to interface (limited use case)
interface StatusInterface {
    status: "active" | "inactive";
}
```

## Performance Considerations

### Compilation Speed

- Interfaces are generally faster to compile than complex types
- Simple type aliases have minimal performance impact
- Complex mapped types and conditional types can slow compilation

### IntelliSense and Developer Experience

- Interfaces provide better IntelliSense for object shapes
- Types offer better support for complex type operations
- Both provide excellent error messages and autocompletion

## Summary

| Feature | Interface | Type |
|---------|-----------|------|
| Declaration Merging | ✅ | ❌ |
| Extending | ✅ | ✅ (using `&`) |
| Union Types | ❌ | ✅ |
| Primitive Aliases | ❌ | ✅ |
| Classes Implementation | ✅ | ❌ |
| Declaration Merging | ✅ | ❌ |
| Computed Properties | ❌ | ✅ |
| Mapped Types | ❌ | ✅ |
| Conditional Types | ❌ | ✅ |

**Rule of Thumb:**
- Use `interface` for object shapes and public APIs
- Use `type` for unions, primitives, and complex type operations
- Prefer `interface` when declaration merging might be needed
- Use `type` when working with advanced TypeScript features

### Interview Tip:
*"Interfaces are best for object shapes and support declaration merging, while types excel at unions, primitives, and complex type operations. Use interfaces for public APIs and class contracts, types for internal logic and advanced TypeScript features."*