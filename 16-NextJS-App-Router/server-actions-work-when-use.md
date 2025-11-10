# How do Server Actions work? When do you use them?

## Question
How do Server Actions work? When do you use them?

## Answer

Server Actions in Next.js App Router allow you to run server-side code directly from client components. They provide a way to handle form submissions, data mutations, and other server operations without needing separate API routes, while maintaining progressive enhancement and security.

## Basic Server Action Structure

### Creating a Server Action

```javascript
// app/actions.js
'use server';

export async function createUser(formData) {
    const name = formData.get('name');
    const email = formData.get('email');

    // Server-side logic
    const user = await db.user.create({
        data: { name, email }
    });

    // Revalidate cache
    revalidatePath('/users');

    return { success: true, user };
}
```

**Key Features:**
- `'use server'` directive marks server-only functions
- Can be called from client components
- Run on the server with full Node.js access
- Automatic request/response handling

## How Server Actions Work

### Client-Server Communication

```javascript
// Client component calling server action
'use client';

import { createUser } from './actions';

export function UserForm() {
    return (
        <form action={createUser}>
            <input name="name" required />
            <input name="email" type="email" required />
            <button type="submit">Create User</button>
        </form>
    );
}

// Server action execution
// 1. Form submits to server
// 2. Server action runs with form data
// 3. Response sent back to client
// 4. Page revalidated if needed
```

### Progressive Enhancement

```javascript
// Server actions work without JavaScript
<form action="/create-user" method="post">
    <input name="name" />
    <input name="email" />
    <button type="submit">Create User</button>
</form>

// With JavaScript, it becomes a server action call
// Without JavaScript, it falls back to form submission
```

## Server Action Patterns

### 1. **Form Actions**

```javascript
// app/actions/userActions.js
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createUser(formData) {
    const name = formData.get('name');
    const email = formData.get('email');

    // Validation
    if (!name || !email) {
        throw new Error('Name and email are required');
    }

    try {
        const user = await prisma.user.create({
            data: { name, email }
        });

        revalidatePath('/users');
        redirect('/users');
    } catch (error) {
        throw new Error('Failed to create user');
    }
}

export async function updateUser(userId, formData) {
    const name = formData.get('name');
    const email = formData.get('email');

    const updatedUser = await prisma.user.update({
        where: { id: userId },
        data: { name, email }
    });

    revalidatePath('/users');
    revalidatePath(`/users/${userId}`);

    return updatedUser;
}
```

### 2. **Inline Server Actions**

```javascript
// app/components/UserForm.js
'use client';

export function UserForm() {
    async function createUser(formData) {
        'use server';

        const name = formData.get('name');
        const email = formData.get('email');

        const user = await prisma.user.create({
            data: { name, email }
        });

        revalidatePath('/users');
    }

    return (
        <form action={createUser}>
            <input name="name" />
            <input name="email" />
            <button>Create User</button>
        </form>
    );
}
```

### 3. **Server Actions with Client State**

```javascript
// app/components/Counter.js
'use client';

import { incrementCounter } from '../actions';

export function Counter({ initialCount }) {
    const [count, setCount] = useState(initialCount);
    const [pending, setPending] = useState(false);

    async function handleIncrement() {
        setPending(true);

        try {
            const newCount = await incrementCounter();
            setCount(newCount);
        } finally {
            setPending(false);
        }
    }

    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={handleIncrement} disabled={pending}>
                {pending ? 'Incrementing...' : 'Increment'}
            </button>
        </div>
    );
}

// app/actions.js
'use server';

let counter = 0;

export async function incrementCounter() {
    counter += 1;
    return counter;
}
```

## When to Use Server Actions

### ✅ **Use Server Actions for:**

#### 1. **Form Submissions**
```javascript
// Perfect for form handling
export async function submitContactForm(formData) {
    const data = Object.fromEntries(formData);
    await sendEmail(data);
    redirect('/thank-you');
}
```

#### 2. **Data Mutations**
```javascript
// Creating, updating, deleting data
export async function deletePost(postId) {
    await prisma.post.delete({ where: { id: postId } });
    revalidatePath('/posts');
}
```

#### 3. **Authentication**
```javascript
// Login, logout, registration
export async function signIn(credentials) {
    const user = await authenticateUser(credentials);
    // Set session cookie
    redirect('/dashboard');
}
```

#### 4. **File Uploads**
```javascript
// Handle file processing on server
export async function uploadFile(formData) {
    const file = formData.get('file');
    const buffer = await file.arrayBuffer();

    // Process file on server
    const result = await processImage(Buffer.from(buffer));
    return result;
}
```

#### 5. **Cache Invalidation**
```javascript
// Revalidate specific paths or tags
export async function updateSettings(settings) {
    await saveSettings(settings);
    revalidateTag('settings');
    revalidatePath('/settings');
}
```

### ❌ **Don't Use Server Actions for:**

#### 1. **Data Fetching (Use Server Components)**
```javascript
// ❌ Bad: Using server action for data fetching
export async function getUsers() {
    return await prisma.user.findMany();
}

// ✅ Good: Use server component
export default async function UsersPage() {
    const users = await prisma.user.findMany();
    return <UserList users={users} />;
}
```

#### 2. **Real-time Updates (Use WebSockets/API)**
```javascript
// ❌ Bad: Polling with server actions
export async function getLiveData() {
    // This creates unnecessary requests
    return await fetchLiveData();
}
```

#### 3. **Heavy Computations (Consider API Routes)**
```javascript
// For very heavy operations, API routes might be better
export async function processLargeDataset(data) {
    // Very expensive operation
    const result = await heavyComputation(data);
    return result;
}
```

## Server Action Security

### Automatic Security Features

```javascript
// 1. Server actions are protected from CSRF
// 2. Input is automatically sanitized
// 3. Server code never reaches the client

// 2. Input validation
export async function createPost(formData) {
    const title = formData.get('title');
    const content = formData.get('content');

    // Server-side validation
    if (!title || title.length < 3) {
        throw new Error('Title must be at least 3 characters');
    }

    // Safe to use directly (no SQL injection)
    const post = await prisma.post.create({
        data: { title, content }
    });

    return post;
}
```

### Custom Security

```javascript
// app/actions/secureActions.js
'use server';

import { auth } from '../lib/auth';

export async function deleteUser(userId) {
    const session = await auth();

    if (!session || session.user.role !== 'admin') {
        throw new Error('Unauthorized');
    }

    await prisma.user.delete({ where: { id: userId } });
    revalidatePath('/users');
}
```

## Error Handling

### Server Action Errors

```javascript
// app/components/UserForm.js
'use client';

import { createUser } from '../actions';
import { useState } from 'react';

export function UserForm() {
    const [error, setError] = useState(null);

    async function handleSubmit(formData) {
        try {
            setError(null);
            await createUser(formData);
        } catch (error) {
            setError(error.message);
        }
    }

    return (
        <form action={handleSubmit}>
            {error && <div className="error">{error}</div>}
            <input name="name" />
            <input name="email" />
            <button>Create User</button>
        </form>
    );
}
```

### Error Boundaries

```javascript
// app/components/ErrorBoundary.js
'use client';

import { useState } from 'react';

export function ErrorBoundary({ children }) {
    const [error, setError] = useState(null);

    if (error) {
        return <div>Error: {error.message}</div>;
    }

    return (
        <ErrorBoundary onError={setError}>
            {children}
        </ErrorBoundary>
    );
}

// Usage
<ErrorBoundary>
    <UserForm />
</ErrorBoundary>
```

## Pending States

### useFormStatus Hook

```javascript
// app/components/SubmitButton.js
'use client';

import { useFormStatus } from 'react-dom';

export function SubmitButton() {
    const { pending } = useFormStatus();

    return (
        <button type="submit" disabled={pending}>
            {pending ? 'Submitting...' : 'Submit'}
        </button>
    );
}

// app/components/UserForm.js
import { SubmitButton } from './SubmitButton';

export function UserForm() {
    return (
        <form action={createUser}>
            <input name="name" />
            <input name="email" />
            <SubmitButton />
        </form>
    );
}
```

### Custom Pending States

```javascript
// app/components/FormWithStatus.js
'use client';

import { useTransition } from 'react';
import { createUser } from '../actions';

export function FormWithStatus() {
    const [isPending, startTransition] = useTransition();

    async function handleSubmit(formData) {
        startTransition(async () => {
            await createUser(formData);
        });
    }

    return (
        <form action={handleSubmit}>
            <input name="name" disabled={isPending} />
            <input name="email" disabled={isPending} />
            <button disabled={isPending}>
                {isPending ? 'Creating...' : 'Create User'}
            </button>
        </form>
    );
}
```

## Revalidation and Cache Management

### Path Revalidation

```javascript
// app/actions/postActions.js
'use server';

import { revalidatePath } from 'next/cache';

export async function createPost(formData) {
    const title = formData.get('title');
    const content = formData.get('content');

    const post = await prisma.post.create({
        data: { title, content }
    });

    // Revalidate the posts page
    revalidatePath('/posts');

    // Revalidate the home page if it shows recent posts
    revalidatePath('/');

    return post;
}
```

### Tag-Based Revalidation

```javascript
// app/actions/cacheActions.js
'use server';

import { revalidateTag } from 'next/cache';

export async function updateSettings(settings) {
    await saveSettings(settings);

    // Revalidate all pages that use 'settings' tag
    revalidateTag('settings');
}

// In a server component
export default async function SettingsPage() {
    const settings = await fetch('/api/settings', {
        next: { tags: ['settings'] }
    });

    return <SettingsForm initialSettings={settings} />;
}
```

## Server Actions vs API Routes

### When to Use Server Actions

```javascript
// ✅ Server Actions for:
// - Form submissions
// - Simple mutations
// - Progressive enhancement
// - Tight client-server coupling

export async function updateProfile(formData) {
    'use server';

    const user = await updateUser(formData);
    revalidatePath('/profile');
    return user;
}
```

### When to Use API Routes

```javascript
// ✅ API Routes for:
// - Complex business logic
// - External API integrations
// - File processing
// - Real-time features

// app/api/complex-operation/route.js
export async function POST(request) {
    const data = await request.json();

    // Complex multi-step operation
    const result1 = await step1(data);
    const result2 = await step2(result1);
    const finalResult = await step3(result2);

    return NextResponse.json(finalResult);
}
```

## Advanced Patterns

### 1. **Optimistic Updates**

```javascript
// app/components/OptimisticLike.js
'use client';

import { useOptimistic } from 'react';
import { likePost } from '../actions';

export function OptimisticLike({ postId, initialLikes }) {
    const [optimisticLikes, addOptimisticLike] = useOptimistic(
        initialLikes,
        (state) => state + 1
    );

    async function handleLike() {
        addOptimisticLike();
        await likePost(postId);
    }

    return (
        <button onClick={handleLike}>
            ❤️ {optimisticLikes}
        </button>
    );
}
```

### 2. **Server Actions with Redirects**

```javascript
// app/actions/authActions.js
'use server';

import { redirect } from 'next/navigation';
import { cookies } from 'next/headers';

export async function signIn(formData) {
    const email = formData.get('email');
    const password = formData.get('password');

    const user = await authenticateUser(email, password);

    if (!user) {
        throw new Error('Invalid credentials');
    }

    // Set session
    cookies().set('session', user.sessionToken);

    redirect('/dashboard');
}

export async function signOut() {
    cookies().delete('session');
    redirect('/');
}
```

### 3. **Batch Operations**

```javascript
// app/actions/batchActions.js
'use server';

import { revalidatePath } from 'next/cache';

export async function deleteMultiplePosts(postIds) {
    await prisma.post.deleteMany({
        where: { id: { in: postIds } }
    });

    revalidatePath('/posts');
    return { deletedCount: postIds.length };
}

// Client usage
async function handleBulkDelete() {
    const selectedIds = getSelectedPostIds();
    await deleteMultiplePosts(selectedIds);
}
```

## Testing Server Actions

### Unit Testing

```javascript
// __tests__/actions.test.js
import { createUser } from '../app/actions';

// Mock dependencies
jest.mock('../lib/prisma', () => ({
    user: {
        create: jest.fn()
    }
}));

test('createUser creates a user', async () => {
    const formData = new FormData();
    formData.append('name', 'John');
    formData.append('email', 'john@example.com');

    const result = await createUser(formData);

    expect(result.success).toBe(true);
    expect(result.user.name).toBe('John');
});
```

### Integration Testing

```javascript
// Test with form submission
test('form submission calls server action', async () => {
    render(<UserForm />);

    fireEvent.change(screen.getByLabelText('Name'), {
        target: { value: 'John' }
    });

    fireEvent.click(screen.getByText('Create User'));

    await waitFor(() => {
        expect(mockCreateUser).toHaveBeenCalledWith(
            expect.any(FormData)
        );
    });
});
```

## Performance Considerations

### Server Action Optimization

```javascript
// ✅ Good: Efficient server actions
export async function quickUpdate(formData) {
    const id = formData.get('id');
    const field = formData.get('field');
    const value = formData.get('value');

    // Direct database update
    await prisma.user.update({
        where: { id },
        data: { [field]: value }
    });

    revalidateTag('users');
}

// ❌ Bad: Inefficient operations
export async function slowUpdate(formData) {
    // Unnecessary data fetching
    const user = await prisma.user.findUnique({ where: { id: formData.get('id') } });

    // Multiple database calls
    await prisma.user.update({ where: { id: user.id }, data: { ... } });
    await prisma.auditLog.create({ data: { ... } });
    await sendNotification(user.email);
}
```

## Best Practices

### 1. **Keep Server Actions Focused**

```javascript
// ✅ Good: Single responsibility
export async function updateUserEmail(userId, newEmail) {
    await prisma.user.update({
        where: { id: userId },
        data: { email: newEmail }
    });

    revalidatePath('/profile');
}

// ❌ Bad: Multiple responsibilities
export async function updateUser(formData) {
    // Validation, database, email, cache, redirect
    const data = validateForm(formData);
    await updateDB(data);
    await sendEmail(data);
    revalidatePath('/users');
    redirect('/success');
}
```

### 2. **Proper Error Handling**

```javascript
// ✅ Good: Descriptive errors
export async function createPost(formData) {
    try {
        const title = formData.get('title');

        if (!title || title.length < 5) {
            throw new Error('Title must be at least 5 characters');
        }

        const post = await prisma.post.create({
            data: { title, content: formData.get('content') }
        });

        return post;
    } catch (error) {
        // Log error for debugging
        console.error('Create post error:', error);

        // Throw user-friendly error
        throw new Error('Failed to create post. Please try again.');
    }
}
```

### 3. **Use Appropriate Revalidation**

```javascript
// ✅ Good: Targeted revalidation
export async function updatePost(postId, formData) {
    await prisma.post.update({
        where: { id: postId },
        data: Object.fromEntries(formData)
    });

    // Revalidate specific paths
    revalidatePath('/posts');
    revalidatePath(`/posts/${postId}`);
}

// ❌ Bad: Over-revalidation
export async function updatePost(postId, formData) {
    await updatePostInDB(postId, formData);

    // Revalidates everything
    revalidatePath('/', 'layout');
}
```

### Interview Tip:
*"Server Actions in Next.js App Router allow client components to call server-side functions directly using the 'use server' directive. They're ideal for form submissions, data mutations, and operations requiring server-side processing, providing progressive enhancement, automatic security, and seamless cache revalidation without separate API routes."*