# How do you type async functions and server actions?

## Question
How do you type async functions and server actions?

## Answer

Typing async functions and server actions in TypeScript requires understanding how to properly type asynchronous operations, error handling, and Next.js server actions. This involves Promise types, return types, and proper error handling patterns.

## Basic Async Function Typing

### Simple Async Function

```typescript
// Basic async function typing
async function fetchUser(id: number): Promise<User> {
    const response = await fetch(`/api/users/${id}`);

    if (!response.ok) {
        throw new Error(`Failed to fetch user: ${response.status}`);
    }

    const user: User = await response.json();
    return user;
}

// Usage
const user = await fetchUser(1); // Type: User
```

### Generic Async Function

```typescript
// Generic async function for API calls
async function apiRequest<T>(
    endpoint: string,
    options?: RequestInit
): Promise<T> {
    const response = await fetch(endpoint, options);

    if (!response.ok) {
        throw new Error(`API request failed: ${response.status}`);
    }

    return response.json();
}

// Usage with type inference
const user = await apiRequest<User>('/api/users/1');
const posts = await apiRequest<Post[]>('/api/posts');
```

## Advanced Async Function Patterns

### Async Function with Error Handling

```typescript
interface APIResponse<T> {
    success: boolean;
    data?: T;
    error?: string;
}

async function safeApiRequest<T>(
    endpoint: string,
    options?: RequestInit
): Promise<APIResponse<T>> {
    try {
        const response = await fetch(endpoint, options);

        if (!response.ok) {
            return {
                success: false,
                error: `HTTP ${response.status}: ${response.statusText}`
            };
        }

        const data: T = await response.json();
        return {
            success: true,
            data
        };
    } catch (error) {
        return {
            success: false,
            error: error instanceof Error ? error.message : 'Unknown error'
        };
    }
}

// Usage
const result = await safeApiRequest<User>('/api/users/1');
if (result.success) {
    console.log(result.data); // Type: User
} else {
    console.error(result.error); // Type: string
}
```

### Async Function with Timeout

```typescript
async function fetchWithTimeout<T>(
    url: string,
    timeoutMs: number = 5000
): Promise<T> {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

    try {
        const response = await fetch(url, {
            signal: controller.signal
        });

        clearTimeout(timeoutId);

        if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
        }

        return response.json();
    } catch (error) {
        clearTimeout(timeoutId);

        if (error instanceof Error && error.name === 'AbortError') {
            throw new Error('Request timeout');
        }

        throw error;
    }
}
```

## Next.js Server Actions Typing

### Basic Server Action

```typescript
// app/actions.ts
'use server';

import { z } from 'zod';

// Define input schema
const createUserSchema = z.object({
    name: z.string().min(1),
    email: z.string().email(),
    age: z.number().min(18)
});

type CreateUserInput = z.infer<typeof createUserSchema>;

export async function createUser(input: CreateUserInput): Promise<User> {
    // Validate input
    const validatedData = createUserSchema.parse(input);

    // Simulate database operation
    const newUser: User = {
        id: Date.now(),
        ...validatedData,
        createdAt: new Date()
    };

    // In real app, save to database
    // await db.users.create(newUser);

    return newUser;
}
```

### Server Action with Error Handling

```typescript
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

interface ActionResult<T = void> {
    success: boolean;
    data?: T;
    error?: string;
    fieldErrors?: Record<string, string[]>;
}

export async function createPost(formData: FormData): Promise<ActionResult<Post>> {
    try {
        const title = formData.get('title') as string;
        const content = formData.get('content') as string;
        const authorId = formData.get('authorId') as string;

        // Validation
        const errors: Record<string, string[]> = {};

        if (!title?.trim()) {
            errors.title = ['Title is required'];
        }

        if (!content?.trim()) {
            errors.content = ['Content is required'];
        }

        if (!authorId) {
            errors.authorId = ['Author is required'];
        }

        if (Object.keys(errors).length > 0) {
            return {
                success: false,
                fieldErrors: errors
            };
        }

        // Create post
        const newPost: Post = {
            id: Date.now().toString(),
            title: title.trim(),
            content: content.trim(),
            authorId,
            createdAt: new Date(),
            published: false
        };

        // Save to database (simulated)
        // await db.posts.create(newPost);

        // Revalidate and redirect
        revalidatePath('/posts');
        redirect(`/posts/${newPost.id}`);

        return {
            success: true,
            data: newPost
        };

    } catch (error) {
        console.error('Failed to create post:', error);
        return {
            success: false,
            error: 'Failed to create post. Please try again.'
        };
    }
}
```

### Typed Server Actions with Generics

```typescript
// lib/server-actions.ts
import { z } from 'zod';

type ServerActionResult<T = void> = {
    success: true;
    data: T;
} | {
    success: false;
    error: string;
    fieldErrors?: Record<string, string[]>;
};

function createServerAction<TInput, TResult>(
    schema: z.ZodSchema<TInput>,
    handler: (input: TInput) => Promise<TResult>
) {
    return async (input: TInput): Promise<ServerActionResult<TResult>> => {
        try {
            // Validate input
            const validatedInput = schema.parse(input);

            // Execute handler
            const result = await handler(validatedInput);

            return {
                success: true,
                data: result
            };
        } catch (error) {
            if (error instanceof z.ZodError) {
                return {
                    success: false,
                    error: 'Validation failed',
                    fieldErrors: error.flatten().fieldErrors
                };
            }

            console.error('Server action error:', error);
            return {
                success: false,
                error: 'An unexpected error occurred'
            };
        }
    };
}

// Usage
const createUserSchema = z.object({
    name: z.string().min(1),
    email: z.string().email(),
    password: z.string().min(8)
});

export const createUser = createServerAction(
    createUserSchema,
    async (input) => {
        // Create user logic
        const user = {
            id: Date.now().toString(),
            ...input,
            createdAt: new Date()
        };

        // Save to database
        return user;
    }
);
```

## Client-Side Usage of Server Actions

### Basic Usage

```typescript
// app/components/CreateUserForm.tsx
'use client';

import { createUser } from '@/app/actions';
import { useState } from 'react';

export function CreateUserForm() {
    const [isPending, setIsPending] = useState(false);
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        setIsPending(true);
        setErrors({});

        const result = await createUser({
            name: formData.get('name') as string,
            email: formData.get('email') as string,
            password: formData.get('password') as string
        });

        setIsPending(false);

        if (!result.success) {
            if (result.fieldErrors) {
                setErrors(result.fieldErrors);
            } else {
                alert(result.error);
            }
            return;
        }

        // Success - redirect or show success message
        alert('User created successfully!');
    }

    return (
        <form action={handleSubmit}>
            <div>
                <input name="name" placeholder="Name" required />
                {errors.name && <span>{errors.name[0]}</span>}
            </div>

            <div>
                <input name="email" type="email" placeholder="Email" required />
                {errors.email && <span>{errors.email[0]}</span>}
            </div>

            <div>
                <input name="password" type="password" placeholder="Password" required />
                {errors.password && <span>{errors.password[0]}</span>}
            </div>

            <button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create User'}
            </button>
        </form>
    );
}
```

### Using useTransition for Better UX

```typescript
// app/components/CreatePostForm.tsx
'use client';

import { createPost } from '@/app/actions';
import { useTransition } from 'react';

export function CreatePostForm() {
    const [isPending, startTransition] = useTransition();
    const [errors, setErrors] = useState<Record<string, string[]>>({});

    async function handleSubmit(formData: FormData) {
        startTransition(async () => {
            setErrors({});

            const result = await createPost(formData);

            if (!result.success) {
                if (result.fieldErrors) {
                    setErrors(result.fieldErrors);
                } else {
                    alert(result.error);
                }
            }
        });
    }

    return (
        <form action={handleSubmit}>
            <div>
                <input name="title" placeholder="Title" required />
                {errors.title && <span>{errors.title[0]}</span>}
            </div>

            <div>
                <textarea name="content" placeholder="Content" required />
                {errors.content && <span>{errors.content[0]}</span>}
            </div>

            <input name="authorId" type="hidden" value="user123" />

            <button type="submit" disabled={isPending}>
                {isPending ? 'Creating...' : 'Create Post'}
            </button>
        </form>
    );
}
```

## Advanced Server Action Patterns

### Server Action with File Upload

```typescript
// app/actions.ts
'use server';

import { writeFile } from 'fs/promises';
import { join } from 'path';

export async function uploadFile(formData: FormData): Promise<ServerActionResult<string>> {
    try {
        const file = formData.get('file') as File;

        if (!file) {
            return {
                success: false,
                error: 'No file provided'
            };
        }

        // Validate file type and size
        if (!file.type.startsWith('image/')) {
            return {
                success: false,
                error: 'Only image files are allowed'
            };
        }

        if (file.size > 5 * 1024 * 1024) { // 5MB
            return {
                success: false,
                error: 'File size must be less than 5MB'
            };
        }

        // Generate unique filename
        const filename = `${Date.now()}-${file.name}`;
        const filepath = join(process.cwd(), 'public', 'uploads', filename);

        // Convert File to Buffer and save
        const bytes = await file.arrayBuffer();
        const buffer = Buffer.from(bytes);
        await writeFile(filepath, buffer);

        return {
            success: true,
            data: `/uploads/${filename}`
        };

    } catch (error) {
        console.error('File upload error:', error);
        return {
            success: false,
            error: 'Failed to upload file'
        };
    }
}
```

### Batch Server Actions

```typescript
// app/actions.ts
'use server';

interface BatchResult<T> {
    successful: T[];
    failed: Array<{ item: T; error: string }>;
}

export async function batchCreateUsers(
    users: Array<{ name: string; email: string }>
): Promise<BatchResult<User>> {
    const successful: User[] = [];
    const failed: Array<{ item: typeof users[0]; error: string }> = [];

    // Process in parallel with limited concurrency
    const promises = users.map(async (userData) => {
        try {
            const user = await createUser(userData);
            successful.push(user);
        } catch (error) {
            failed.push({
                item: userData,
                error: error instanceof Error ? error.message : 'Unknown error'
            });
        }
    });

    await Promise.allSettled(promises);

    return { successful, failed };
}
```

## Error Handling Patterns

### Custom Error Types

```typescript
// lib/errors.ts
export class ValidationError extends Error {
    constructor(
        message: string,
        public fieldErrors: Record<string, string[]>
    ) {
        super(message);
        this.name = 'ValidationError';
    }
}

export class NotFoundError extends Error {
    constructor(resource: string, id: string) {
        super(`${resource} with id ${id} not found`);
        this.name = 'NotFoundError';
    }
}

export class UnauthorizedError extends Error {
    constructor(message = 'Unauthorized') {
        super(message);
        this.name = 'UnauthorizedError';
    }
}
```

### Typed Error Handling

```typescript
// app/actions.ts
'use server';

export async function updateUser(
    userId: string,
    updates: Partial<User>
): Promise<ServerActionResult<User>> {
    try {
        // Check if user exists
        const existingUser = await getUserById(userId);
        if (!existingUser) {
            throw new NotFoundError('User', userId);
        }

        // Check permissions (simplified)
        if (updates.role && !isAdmin(currentUser)) {
            throw new UnauthorizedError('Only admins can change user roles');
        }

        // Update user
        const updatedUser = await updateUserInDb(userId, updates);

        return {
            success: true,
            data: updatedUser
        };

    } catch (error) {
        if (error instanceof NotFoundError) {
            return {
                success: false,
                error: error.message
            };
        }

        if (error instanceof UnauthorizedError) {
            return {
                success: false,
                error: error.message
            };
        }

        if (error instanceof ValidationError) {
            return {
                success: false,
                error: error.message,
                fieldErrors: error.fieldErrors
            };
        }

        console.error('Unexpected error:', error);
        return {
            success: false,
            error: 'An unexpected error occurred'
        };
    }
}
```

## Best Practices

### 1. **Always Use Proper Return Types**

```typescript
// ✅ Good: Explicit return type
export async function getUser(id: string): Promise<User | null> {
    // Implementation
}

// ❌ Bad: Implicit any return type
export async function getUser(id: string) {
    // Implementation
}
```

### 2. **Handle Errors Appropriately**

```typescript
// ✅ Good: Typed error handling
export async function createUser(data: UserInput): Promise<Result<User>> {
    try {
        const user = await db.users.create(data);
        return { success: true, data: user };
    } catch (error) {
        return { success: false, error: 'Failed to create user' };
    }
}
```

### 3. **Use Zod for Validation**

```typescript
// ✅ Good: Schema-based validation
import { z } from 'zod';

const userSchema = z.object({
    name: z.string().min(1),
    email: z.string().email(),
    age: z.number().min(18)
});

export async function createUser(input: unknown): Promise<Result<User>> {
    const result = userSchema.safeParse(input);
    if (!result.success) {
        return { success: false, error: 'Invalid input', fieldErrors: result.error.flatten().fieldErrors };
    }

    // Use validated data
    const user = await db.users.create(result.data);
    return { success: true, data: user };
}
```

### 4. **Use Server Action Patterns**

```typescript
// ✅ Good: Consistent server action structure
type ServerAction<TInput, TResult> = (input: TInput) => Promise<
    | { success: true; data: TResult }
    | { success: false; error: string; fieldErrors?: Record<string, string[]> }
>;
```

## Interview Questions

### Q: How do you type async functions that might throw errors?

**A:** Use union types for return values: `Promise<T | Error>` or create result types like `{ success: boolean; data?: T; error?: string }`. Always handle both success and error cases.

### Q: What's the difference between typing server actions vs regular async functions?

**A:** Server actions run on the server and can access server-side resources, while regular async functions can run on client or server. Server actions have special Next.js typing requirements and should use the `'use server'` directive.

### Q: How do you handle validation errors in server actions?

**A:** Use Zod schemas for validation, return structured error responses with field-specific errors, and handle them appropriately on the client side with proper UI feedback.

### Q: Why use `useTransition` with server actions?

**A:** `useTransition` provides better UX by marking non-urgent updates as transitions, preventing the UI from freezing during server action execution and allowing for pending states.

## Summary

Typing async functions and server actions involves:

1. **Async Functions**: Use `Promise<T>` return types, handle errors with try/catch or Result types
2. **Server Actions**: Use `'use server'`, proper error handling, validation with Zod
3. **Client Integration**: Use `useTransition` for better UX, handle server action responses
4. **Best Practices**: Explicit typing, consistent error handling, schema validation

**Key Points:**
- Always specify return types for async functions
- Use Result types for operations that can fail
- Validate inputs with Zod schemas
- Handle errors consistently across client and server
- Use `useTransition` for server action calls

**Interview Tip:** "Async functions should return `Promise<T>` and handle errors with Result types. Server actions use `'use server'`, return structured responses with success/error states, and integrate with client components using `useTransition` for optimal UX."