# How do Route Handlers work? (app/api/*/route.js)

## Question
How do Route Handlers work? (app/api/*/route.js)

## Answer

Route Handlers in Next.js App Router provide a way to create API endpoints using Web API Request and Response objects. They replace the API routes from the Pages Router and offer better performance, type safety, and integration with the App Router architecture.

## Basic Route Handler Structure

### Creating a Route Handler

```javascript
// app/api/users/route.js
import { NextResponse } from 'next/server';

export async function GET() {
    const users = await getUsersFromDB();

    return NextResponse.json({ users });
}

export async function POST(request) {
    const body = await request.json();
    const newUser = await createUser(body);

    return NextResponse.json({ user: newUser }, { status: 201 });
}
```

**File Location to API Mapping:**
- `app/api/users/route.js` → `GET/POST /api/users`
- `app/api/users/[id]/route.js` → `GET/PUT/DELETE /api/users/[id]`
- `app/api/posts/[...slug]/route.js` → `GET /api/posts/[...slug]`

## HTTP Methods

### Supported Methods

```javascript
// app/api/products/route.js
import { NextResponse } from 'next/server';

// GET - Retrieve resources
export async function GET(request) {
    const { searchParams } = new URL(request.url);
    const category = searchParams.get('category');

    const products = await getProducts({ category });
    return NextResponse.json({ products });
}

// POST - Create new resources
export async function POST(request) {
    const body = await request.json();
    const product = await createProduct(body);

    return NextResponse.json({ product }, { status: 201 });
}

// PUT - Update entire resource
export async function PUT(request) {
    const body = await request.json();
    const updatedProduct = await updateProduct(body.id, body);

    return NextResponse.json({ product: updatedProduct });
}

// PATCH - Partial update
export async function PATCH(request) {
    const body = await request.json();
    const updatedProduct = await updateProductPartial(body.id, body);

    return NextResponse.json({ product: updatedProduct });
}

// DELETE - Remove resource
export async function DELETE(request) {
    const { searchParams } = new URL(request.url);
    const id = searchParams.get('id');

    await deleteProduct(id);
    return NextResponse.json({ message: 'Product deleted' });
}
```

## Request Handling

### Reading Request Data

```javascript
// app/api/contact/route.js
import { NextResponse } from 'next/server';

export async function POST(request) {
    // 1. JSON body
    const jsonData = await request.json();

    // 2. Form data
    const formData = await request.formData();
    const name = formData.get('name');
    const file = formData.get('file');

    // 3. URL search parameters
    const { searchParams } = new URL(request.url);
    const page = searchParams.get('page') || '1';

    // 4. Headers
    const authToken = request.headers.get('authorization');

    // 5. Cookies
    const sessionId = request.cookies.get('sessionId');

    return NextResponse.json({ received: true });
}
```

### Request Object Properties

```javascript
export async function GET(request) {
    // URL and search params
    const url = request.url; // Full URL
    const { searchParams } = new URL(request.url);

    // Headers
    const userAgent = request.headers.get('user-agent');
    const contentType = request.headers.get('content-type');

    // Method
    const method = request.method; // 'GET', 'POST', etc.

    // Cookies
    const allCookies = request.cookies.getAll();
    const sessionCookie = request.cookies.get('session');

    // Body (for POST, PUT, PATCH)
    const body = request.body; // ReadableStream

    return NextResponse.json({ method, url, headers: Object.fromEntries(request.headers) });
}
```

## Response Handling

### NextResponse Methods

```javascript
// app/api/data/route.js
import { NextResponse } from 'next/server';

// JSON response
export async function GET() {
    return NextResponse.json({ data: 'Hello World' });
}

// Custom status
export async function POST() {
    return NextResponse.json(
        { error: 'Invalid data' },
        { status: 400 }
    );
}

// Custom headers
export async function GET() {
    return NextResponse.json(
        { data: 'Success' },
        {
            status: 200,
            headers: {
                'Cache-Control': 'public, max-age=300',
                'X-Custom-Header': 'Custom Value'
            }
        }
    );
}

// Redirect
export async function GET() {
    return NextResponse.redirect(new URL('/dashboard', request.url));
}

// Set cookies
export async function POST() {
    const response = NextResponse.json({ loggedIn: true });

    response.cookies.set('session', 'token123', {
        httpOnly: true,
        secure: true,
        maxAge: 60 * 60 * 24 // 24 hours
    });

    return response;
}
```

### Response Types

```javascript
// 1. JSON Response
export async function GET() {
    return NextResponse.json({ users: [] });
}

// 2. HTML Response
export async function GET() {
    return new Response('<h1>Hello World</h1>', {
        headers: { 'Content-Type': 'text/html' }
    });
}

// 3. File Response
export async function GET() {
    const file = await readFile('./data.json');
    return new Response(file, {
        headers: { 'Content-Type': 'application/json' }
    });
}

// 4. Stream Response
export async function GET() {
    const stream = createReadStream('./large-file.zip');
    return new Response(stream, {
        headers: { 'Content-Type': 'application/zip' }
    });
}

// 5. Empty Response
export async function DELETE() {
    await deleteResource();
    return new Response(null, { status: 204 });
}
```

## Dynamic Routes

### Dynamic Segments

```javascript
// app/api/users/[id]/route.js
import { NextResponse } from 'next/server';

export async function GET(request, { params }) {
    const { id } = params;
    const user = await getUserById(id);

    if (!user) {
        return NextResponse.json({ error: 'User not found' }, { status: 404 });
    }

    return NextResponse.json({ user });
}

export async function PUT(request, { params }) {
    const { id } = params;
    const updates = await request.json();

    const updatedUser = await updateUser(id, updates);
    return NextResponse.json({ user: updatedUser });
}

export async function DELETE(request, { params }) {
    const { id } = params;
    await deleteUser(id);

    return NextResponse.json({ message: 'User deleted' });
}
```

### Catch-All Routes

```javascript
// app/api/files/[...path]/route.js
import { NextResponse } from 'next/server';
import { readFile, writeFile } from 'fs/promises';
import path from 'path';

export async function GET(request, { params }) {
    const { path: filePath } = params;
    const fullPath = path.join(process.cwd(), 'files', ...filePath);

    try {
        const file = await readFile(fullPath);
        return new Response(file);
    } catch {
        return NextResponse.json({ error: 'File not found' }, { status: 404 });
    }
}

export async function PUT(request, { params }) {
    const { path: filePath } = params;
    const fullPath = path.join(process.cwd(), 'files', ...filePath);
    const content = await request.arrayBuffer();

    await writeFile(fullPath, Buffer.from(content));
    return NextResponse.json({ message: 'File saved' });
}
```

## Middleware Integration

### Route Handler with Middleware

```javascript
// middleware.js
import { NextResponse } from 'next/server';

export function middleware(request) {
    // Add CORS headers
    const response = NextResponse.next();
    response.headers.set('Access-Control-Allow-Origin', '*');
    response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    response.headers.set('Access-Control-Allow-Headers', 'Content-Type');

    return response;
}

export const config = {
    matcher: '/api/:path*'
};

// app/api/data/route.js
export async function GET() {
    // Middleware runs before this
    return NextResponse.json({ data: 'Protected data' });
}
```

## Error Handling

### Try-Catch Blocks

```javascript
// app/api/users/route.js
import { NextResponse } from 'next/server';

export async function GET() {
    try {
        const users = await getUsersFromDB();
        return NextResponse.json({ users });
    } catch (error) {
        console.error('Database error:', error);
        return NextResponse.json(
            { error: 'Internal server error' },
            { status: 500 }
        );
    }
}

export async function POST(request) {
    try {
        const body = await request.json();

        // Validation
        if (!body.name || !body.email) {
            return NextResponse.json(
                { error: 'Name and email are required' },
                { status: 400 }
            );
        }

        const user = await createUser(body);
        return NextResponse.json({ user }, { status: 201 });
    } catch (error) {
        if (error.code === 'DUPLICATE_KEY') {
            return NextResponse.json(
                { error: 'User already exists' },
                { status: 409 }
            );
        }

        return NextResponse.json(
            { error: 'Internal server error' },
            { status: 500 }
        );
    }
}
```

### Global Error Handling

```javascript
// lib/error-handler.js
export function handleApiError(error) {
    console.error('API Error:', error);

    if (error.name === 'ValidationError') {
        return { message: 'Invalid data provided', status: 400 };
    }

    if (error.name === 'UnauthorizedError') {
        return { message: 'Unauthorized access', status: 401 };
    }

    return { message: 'Internal server error', status: 500 };
}

// app/api/users/route.js
import { handleApiError } from '../../../lib/error-handler';

export async function POST(request) {
    try {
        const body = await request.json();
        const user = await createUser(body);

        return NextResponse.json({ user });
    } catch (error) {
        const { message, status } = handleApiError(error);
        return NextResponse.json({ error: message }, { status });
    }
}
```

## Authentication and Authorization

### JWT Authentication

```javascript
// lib/auth.js
import jwt from 'jsonwebtoken';

export function verifyToken(token) {
    try {
        return jwt.verify(token, process.env.JWT_SECRET);
    } catch {
        return null;
    }
}

// app/api/protected/route.js
import { NextResponse } from 'next/server';
import { verifyToken } from '../../../lib/auth';

export async function GET(request) {
    const authHeader = request.headers.get('authorization');

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }

    const token = authHeader.split(' ')[1];
    const user = verifyToken(token);

    if (!user) {
        return NextResponse.json({ error: 'Invalid token' }, { status: 401 });
    }

    return NextResponse.json({ data: 'Protected content', user });
}
```

### Session-Based Auth

```javascript
// app/api/auth/session/route.js
import { NextResponse } from 'next/server';
import { cookies } from 'next/headers';

export async function GET() {
    const cookieStore = cookies();
    const sessionId = cookieStore.get('sessionId');

    if (!sessionId) {
        return NextResponse.json({ error: 'No session' }, { status: 401 });
    }

    const session = await getSession(sessionId.value);

    if (!session) {
        return NextResponse.json({ error: 'Invalid session' }, { status: 401 });
    }

    return NextResponse.json({ user: session.user });
}
```

## Database Integration

### Direct Database Access

```javascript
// lib/db.js
import { Pool } from 'pg';

const pool = new Pool({
    connectionString: process.env.DATABASE_URL
});

export async function query(text, params) {
    const res = await pool.query(text, params);
    return res.rows;
}

// app/api/users/route.js
import { query } from '../../../lib/db';
import { NextResponse } from 'next/server';

export async function GET() {
    try {
        const users = await query('SELECT * FROM users');
        return NextResponse.json({ users });
    } catch (error) {
        return NextResponse.json({ error: 'Database error' }, { status: 500 });
    }
}

export async function POST(request) {
    try {
        const { name, email } = await request.json();
        const result = await query(
            'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
            [name, email]
        );

        return NextResponse.json({ user: result[0] }, { status: 201 });
    } catch (error) {
        return NextResponse.json({ error: 'Failed to create user' }, { status: 500 });
    }
}
```

### ORM Integration

```javascript
// lib/db.js
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// app/api/posts/route.js
import { NextResponse } from 'next/server';
import { prisma } from '../../../lib/db';

export async function GET() {
    const posts = await prisma.post.findMany({
        include: { author: true }
    });

    return NextResponse.json({ posts });
}

export async function POST(request) {
    const { title, content, authorId } = await request.json();

    const post = await prisma.post.create({
        data: { title, content, authorId }
    });

    return NextResponse.json({ post }, { status: 201 });
}
```

## File Upload Handling

### Single File Upload

```javascript
// app/api/upload/route.js
import { NextResponse } from 'next/server';
import { writeFile } from 'fs/promises';
import path from 'path';

export async function POST(request) {
    const data = await request.formData();
    const file = data.get('file');

    if (!file) {
        return NextResponse.json({ error: 'No file provided' }, { status: 400 });
    }

    const bytes = await file.arrayBuffer();
    const buffer = Buffer.from(bytes);

    const filename = Date.now() + '-' + file.name;
    const filepath = path.join(process.cwd(), 'public/uploads', filename);

    await writeFile(filepath, buffer);

    return NextResponse.json({ filename });
}
```

### Multiple File Upload

```javascript
// app/api/upload/bulk/route.js
import { NextResponse } from 'next/server';

export async function POST(request) {
    const data = await request.formData();
    const files = data.getAll('files');

    const uploadedFiles = [];

    for (const file of files) {
        const bytes = await file.arrayBuffer();
        const buffer = Buffer.from(bytes);

        const filename = Date.now() + '-' + file.name;
        const filepath = path.join(process.cwd(), 'uploads', filename);

        await writeFile(filepath, buffer);
        uploadedFiles.push({ originalName: file.name, filename });
    }

    return NextResponse.json({ files: uploadedFiles });
}
```

## Streaming and Large Data

### Streaming Responses

```javascript
// app/api/export/route.js
import { NextResponse } from 'next/server';

export async function GET() {
    const encoder = new TextEncoder();
    const stream = new ReadableStream({
        start(controller) {
            // Stream CSV data
            controller.enqueue(encoder.encode('id,name,email\n'));

            getUsersStream().on('data', (user) => {
                controller.enqueue(encoder.encode(`${user.id},${user.name},${user.email}\n`));
            }).on('end', () => {
                controller.close();
            });
        }
    });

    return new Response(stream, {
        headers: {
            'Content-Type': 'text/csv',
            'Content-Disposition': 'attachment; filename="users.csv"'
        }
    });
}
```

### Pagination

```javascript
// app/api/products/route.js
export async function GET(request) {
    const { searchParams } = new URL(request.url);
    const page = parseInt(searchParams.get('page') || '1');
    const limit = parseInt(searchParams.get('limit') || '10');
    const offset = (page - 1) * limit;

    const [products, total] = await Promise.all([
        getProducts({ limit, offset }),
        getProductsCount()
    ]);

    return NextResponse.json({
        products,
        pagination: {
            page,
            limit,
            total,
            pages: Math.ceil(total / limit)
        }
    });
}
```

## Caching and Performance

### Response Caching

```javascript
// app/api/data/route.js
export async function GET() {
    const data = await getExpensiveData();

    return NextResponse.json(data, {
        headers: {
            'Cache-Control': 'public, max-age=300', // Cache for 5 minutes
        }
    });
}
```

### Dynamic Revalidation

```javascript
// app/api/posts/route.js
export async function GET() {
    const posts = await getPostsFromCache();

    return NextResponse.json({ posts });
}

export async function POST(request) {
    const post = await createPost(await request.json());

    // Invalidate cache
    await invalidatePostsCache();

    return NextResponse.json({ post }, { status: 201 });
}
```

## Testing Route Handlers

### Unit Testing

```javascript
// __tests__/api/users/route.test.js
import { GET, POST } from '../../../app/api/users/route';
import { NextRequest } from 'next/server';

// Mock database functions
jest.mock('../../../lib/db', () => ({
    getUsers: jest.fn(() => Promise.resolve([{ id: 1, name: 'John' }])),
    createUser: jest.fn((user) => Promise.resolve({ id: 2, ...user }))
}));

test('GET returns users', async () => {
    const request = new NextRequest('http://localhost:3000/api/users');
    const response = await GET(request);

    expect(response.status).toBe(200);
    const data = await response.json();
    expect(data.users).toHaveLength(1);
});

test('POST creates user', async () => {
    const request = new NextRequest('http://localhost:3000/api/users', {
        method: 'POST',
        body: JSON.stringify({ name: 'Jane', email: 'jane@example.com' })
    });

    const response = await POST(request);

    expect(response.status).toBe(201);
    const data = await response.json();
    expect(data.user.name).toBe('Jane');
});
```

## Common Patterns

### RESTful API

```javascript
// app/api/resources/[id]/route.js
export async function GET(request, { params }) {
    const resource = await getResource(params.id);
    return resource
        ? NextResponse.json({ resource })
        : NextResponse.json({ error: 'Not found' }, { status: 404 });
}

export async function PUT(request, { params }) {
    const updates = await request.json();
    const resource = await updateResource(params.id, updates);
    return NextResponse.json({ resource });
}

export async function DELETE(request, { params }) {
    await deleteResource(params.id);
    return new Response(null, { status: 204 });
}
```

### GraphQL-like API

```javascript
// app/api/graphql/route.js
export async function POST(request) {
    const { query, variables } = await request.json();

    const result = await executeGraphQL(query, variables);

    return NextResponse.json(result);
}
```

### Webhook Handler

```javascript
// app/api/webhooks/stripe/route.js
export async function POST(request) {
    const signature = request.headers.get('stripe-signature');
    const body = await request.text();

    try {
        const event = stripe.webhooks.constructEvent(body, signature, endpointSecret);

        switch (event.type) {
            case 'payment_intent.succeeded':
                await handlePaymentSuccess(event.data.object);
                break;
            case 'payment_intent.payment_failed':
                await handlePaymentFailure(event.data.object);
                break;
        }

        return NextResponse.json({ received: true });
    } catch (error) {
        return NextResponse.json({ error: 'Webhook error' }, { status: 400 });
    }
}
```

## Best Practices

### 1. **Consistent Response Format**

```javascript
// ✅ Good: Consistent structure
export async function GET() {
    return NextResponse.json({
        success: true,
        data: users,
        meta: { count: users.length }
    });
}

export async function POST() {
    return NextResponse.json({
        success: true,
        data: newUser,
        message: 'User created successfully'
    }, { status: 201 });
}

// ❌ Bad: Inconsistent structure
export async function GET() {
    return NextResponse.json(users); // Just data
}

export async function POST() {
    return NextResponse.json({ user: newUser, created: true }); // Different structure
}
```

### 2. **Input Validation**

```javascript
// ✅ Good: Validate input
import { z } from 'zod';

const createUserSchema = z.object({
    name: z.string().min(1),
    email: z.string().email()
});

export async function POST(request) {
    const body = await request.json();

    const validation = createUserSchema.safeParse(body);
    if (!validation.success) {
        return NextResponse.json(
            { error: 'Invalid input', details: validation.error.issues },
            { status: 400 }
        );
    }

    const user = await createUser(validation.data);
    return NextResponse.json({ user });
}
```

### 3. **Error Handling**

```javascript
// ✅ Good: Comprehensive error handling
export async function POST(request) {
    try {
        const body = await request.json();
        const user = await createUser(body);

        return NextResponse.json({ user }, { status: 201 });
    } catch (error) {
        console.error('Create user error:', error);

        if (error.code === '23505') { // Unique constraint violation
            return NextResponse.json(
                { error: 'User already exists' },
                { status: 409 }
            );
        }

        return NextResponse.json(
            { error: 'Internal server error' },
            { status: 500 }
        );
    }
}
```

### 4. **Security Headers**

```javascript
// ✅ Good: Security headers
export async function GET() {
    return NextResponse.json(data, {
        headers: {
            'X-Content-Type-Options': 'nosniff',
            'X-Frame-Options': 'DENY',
            'X-XSS-Protection': '1; mode=block'
        }
    });
}
```

### Interview Tip:
*"Route Handlers in Next.js App Router (`app/api/*/route.js`) create API endpoints using Web API Request/Response objects, supporting HTTP methods like GET, POST, PUT, DELETE. They enable server-side processing, database integration, authentication, and provide better performance than Pages Router API routes through automatic code splitting and optimized bundling."*