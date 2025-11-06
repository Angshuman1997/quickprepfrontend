# How do you handle authentication in Server Components?

## Question
How do you handle authentication in Server Components?

## Answer

Authentication in Next.js App Router Server Components requires a different approach compared to Pages Router. Server Components run on the server and don't have access to browser APIs like `localStorage` or cookies directly. You need to handle authentication at the server level using HTTP headers, cookies, or server-side sessions.

## Understanding Server Component Authentication

### Key Differences from Client Components

**Server Components:**
- Run on server during build/request time
- No access to browser APIs (`localStorage`, `sessionStorage`)
- Can access HTTP headers and cookies
- Cannot use client-side authentication libraries directly

**Client Components:**
- Run in browser
- Can access browser storage
- Can use client-side auth libraries
- Can manage authentication state with React hooks

## Method 1: HTTP-only Cookies with Server Actions

```javascript
// app/actions/auth.js
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function login(formData) {
    const email = formData.get('email');
    const password = formData.get('password');

    try {
        const response = await fetch(`${process.env.API_URL}/auth/login`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password })
        });

        if (!response.ok) {
            throw new Error('Invalid credentials');
        }

        const { token, user } = await response.json();

        // Set HTTP-only cookie
        cookies().set('auth-token', token, {
            httpOnly: true,
            secure: process.env.NODE_ENV === 'production',
            sameSite: 'lax',
            maxAge: 60 * 60 * 24 * 7 // 7 days
        });

        redirect('/dashboard');
    } catch (error) {
        throw new Error('Login failed');
    }
}

export async function logout() {
    cookies().delete('auth-token');
    redirect('/login');
}

export async function getAuthToken() {
    const cookieStore = cookies();
    return cookieStore.get('auth-token')?.value;
}
```

```javascript
// app/login/page.js
import { login } from '../actions/auth';

export default function LoginPage() {
    return (
        <div>
            <h1>Login</h1>
            <form action={login}>
                <input name="email" type="email" placeholder="Email" required />
                <input name="password" type="password" placeholder="Password" required />
                <button type="submit">Login</button>
            </form>
        </div>
    );
}
```

## Method 2: Server Component Authentication Helper

```javascript
// lib/auth.js
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export async function getServerSession() {
    try {
        const cookieStore = cookies();
        const token = cookieStore.get('auth-token')?.value;

        if (!token) {
            return null;
        }

        // Verify token with your auth service
        const response = await fetch(`${process.env.API_URL}/auth/verify`, {
            headers: {
                'Authorization': `Bearer ${token}`
            }
        });

        if (!response.ok) {
            return null;
        }

        const user = await response.json();
        return user;
    } catch (error) {
        console.error('Auth verification failed:', error);
        return null;
    }
}

export async function requireAuth() {
    const session = await getServerSession();

    if (!session) {
        redirect('/login');
    }

    return session;
}

export async function getUserFromToken(token) {
    try {
        const response = await fetch(`${process.env.API_URL}/auth/user`, {
            headers: {
                'Authorization': `Bearer ${token}`
            }
        });

        if (!response.ok) {
            throw new Error('Invalid token');
        }

        return await response.json();
    } catch (error) {
        throw new Error('Failed to get user data');
    }
}
```

```javascript
// app/dashboard/layout.js
import { requireAuth } from '../../lib/auth';

export default async function DashboardLayout({ children }) {
    const session = await requireAuth();

    return (
        <div>
            <header>
                <h1>Welcome, {session.user.name}!</h1>
                <LogoutButton />
            </header>
            <main>{children}</main>
        </div>
    );
}
```

## Method 3: Middleware for Route Protection

```javascript
// middleware.js
import { NextResponse } from 'next/server';

export function middleware(request) {
    const token = request.cookies.get('auth-token')?.value;

    // Public routes that don't require authentication
    const publicRoutes = ['/login', '/register', '/'];
    const isPublicRoute = publicRoutes.some(route =>
        request.nextUrl.pathname.startsWith(route)
    );

    // API routes that need different handling
    const isApiRoute = request.nextUrl.pathname.startsWith('/api');

    if (!isPublicRoute && !isApiRoute && !token) {
        return NextResponse.redirect(new URL('/login', request.url));
    }

    // Add user info to headers for API routes
    if (isApiRoute && token) {
        const requestHeaders = new Headers(request.headers);
        requestHeaders.set('x-user-token', token);

        return NextResponse.next({
            request: {
                headers: requestHeaders,
            },
        });
    }

    return NextResponse.next();
}

export const config = {
    matcher: [
        /*
         * Match all request paths except for the ones starting with:
         * - api (API routes)
         * - _next/static (static files)
         * - _next/image (image optimization files)
         * - favicon.ico (favicon file)
         */
        '/((?!api|_next/static|_next/image|favicon.ico).*)',
    ],
};
```

## Method 4: JWT with Server Components

```javascript
// lib/jwt.js
import jwt from 'jsonwebtoken';
import { cookies } from 'next/headers';

const JWT_SECRET = process.env.JWT_SECRET;

export async function verifyToken(token) {
    try {
        const decoded = jwt.verify(token, JWT_SECRET);
        return decoded;
    } catch (error) {
        return null;
    }
}

export async function createToken(payload) {
    return jwt.sign(payload, JWT_SECRET, { expiresIn: '7d' });
}

export async function getUserFromRequest() {
    const cookieStore = cookies();
    const token = cookieStore.get('auth-token')?.value;

    if (!token) {
        return null;
    }

    return await verifyToken(token);
}
```

```javascript
// app/api/auth/verify/route.js
import { getUserFromRequest } from '../../../../lib/jwt';
import { NextResponse } from 'next/server';

export async function GET() {
    try {
        const user = await getUserFromRequest();

        if (!user) {
            return NextResponse.json(
                { error: 'Unauthorized' },
                { status: 401 }
            );
        }

        return NextResponse.json({ user });
    } catch (error) {
        return NextResponse.json(
            { error: 'Internal server error' },
            { status: 500 }
        );
    }
}
```

## Method 5: Database Sessions with Server Components

```javascript
// lib/session.js
import { cookies } from 'next/headers';
import { randomBytes } from 'crypto';
import { prisma } from './db'; // Your database client

export async function createSession(userId) {
    const sessionId = randomBytes(32).toString('hex');

    await prisma.session.create({
        data: {
            id: sessionId,
            userId,
            expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
        }
    });

    cookies().set('session-id', sessionId, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        maxAge: 60 * 60 * 24 * 7
    });

    return sessionId;
}

export async function getSession() {
    const cookieStore = cookies();
    const sessionId = cookieStore.get('session-id')?.value;

    if (!sessionId) {
        return null;
    }

    const session = await prisma.session.findUnique({
        where: { id: sessionId },
        include: { user: true }
    });

    if (!session || session.expiresAt < new Date()) {
        return null;
    }

    return session;
}

export async function destroySession() {
    const cookieStore = cookies();
    const sessionId = cookieStore.get('session-id')?.value;

    if (sessionId) {
        await prisma.session.delete({
            where: { id: sessionId }
        });
    }

    cookies().delete('session-id');
}
```

```javascript
// app/dashboard/page.js
import { getSession } from '../../lib/session';

export default async function DashboardPage() {
    const session = await getSession();

    if (!session) {
        redirect('/login');
    }

    return (
        <div>
            <h1>Dashboard</h1>
            <p>Welcome, {session.user.name}!</p>
            <p>Email: {session.user.email}</p>
            <p>Role: {session.user.role}</p>
        </div>
    );
}
```

## Method 6: Third-party Auth Providers (NextAuth.js)

```javascript
// app/api/auth/[...nextauth]/route.js
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import { PrismaAdapter } from '@next-auth/prisma-adapter';
import { prisma } from '../../../../lib/db';

const handler = NextAuth({
    adapter: PrismaAdapter(prisma),
    providers: [
        GoogleProvider({
            clientId: process.env.GOOGLE_CLIENT_ID,
            clientSecret: process.env.GOOGLE_CLIENT_SECRET,
        }),
    ],
    callbacks: {
        async session({ session, user }) {
            session.user.id = user.id;
            return session;
        },
    },
});

export { handler as GET, handler as POST };
```

```javascript
// lib/auth.js
import { getServerSession } from 'next-auth';
import { authOptions } from '../api/auth/[...nextauth]/route';

export async function getSession() {
    return await getServerSession(authOptions);
}

export async function requireAuth() {
    const session = await getSession();

    if (!session) {
        redirect('/login');
    }

    return session;
}
```

```javascript
// app/dashboard/page.js
import { requireAuth } from '../../lib/auth';

export default async function DashboardPage() {
    const session = await requireAuth();

    return (
        <div>
            <h1>Dashboard</h1>
            <p>Welcome, {session.user.name}!</p>
            <img src={session.user.image} alt="Profile" />
        </div>
    );
}
```

## Method 7: Hybrid Approach (Server + Client)

```javascript
// app/auth/provider.js
'use client';

import { createContext, useContext, useEffect, useState } from 'react';

const AuthContext = createContext();

export function AuthProvider({ children }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    useEffect(() => {
        // Check authentication status on mount
        checkAuthStatus();
    }, []);

    const checkAuthStatus = async () => {
        try {
            const response = await fetch('/api/auth/status');
            if (response.ok) {
                const data = await response.json();
                setUser(data.user);
            }
        } catch (error) {
            console.error('Auth check failed:', error);
        } finally {
            setLoading(false);
        }
    };

    const login = async (credentials) => {
        const response = await fetch('/api/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(credentials)
        });

        if (response.ok) {
            const data = await response.json();
            setUser(data.user);
            return { success: true };
        }

        return { success: false, error: 'Login failed' };
    };

    const logout = async () => {
        await fetch('/api/auth/logout', { method: 'POST' });
        setUser(null);
    };

    return (
        <AuthContext.Provider value={{
            user,
            loading,
            login,
            logout
        }}>
            {children}
        </AuthContext.Provider>
    );
}

export function useAuth() {
    const context = useContext(AuthContext);
    if (!context) {
        throw new Error('useAuth must be used within AuthProvider');
    }
    return context;
}
```

```javascript
// app/dashboard/layout.js
import { requireAuth } from '../../lib/auth';
import { AuthProvider } from '../../auth/provider';

export default async function DashboardLayout({ children }) {
    // Server-side auth check
    await requireAuth();

    return (
        <AuthProvider>
            <div>
                <header>
                    <UserMenu />
                </header>
                <main>{children}</main>
            </div>
        </AuthProvider>
    );
}
```

## Method 8: API Route Authentication

```javascript
// app/api/auth/login/route.js
import { NextResponse } from 'next/server';
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';
import { prisma } from '../../../../lib/db';

export async function POST(request) {
    try {
        const { email, password } = await request.json();

        const user = await prisma.user.findUnique({
            where: { email }
        });

        if (!user) {
            return NextResponse.json(
                { error: 'Invalid credentials' },
                { status: 401 }
            );
        }

        const isValidPassword = await bcrypt.compare(password, user.password);

        if (!isValidPassword) {
            return NextResponse.json(
                { error: 'Invalid credentials' },
                { status: 401 }
            );
        }

        const token = jwt.sign(
            { userId: user.id, email: user.email },
            process.env.JWT_SECRET,
            { expiresIn: '7d' }
        );

        const response = NextResponse.json({
            user: {
                id: user.id,
                name: user.name,
                email: user.email
            }
        });

        response.cookies.set('auth-token', token, {
            httpOnly: true,
            secure: process.env.NODE_ENV === 'production',
            sameSite: 'lax',
            maxAge: 60 * 60 * 24 * 7
        });

        return response;
    } catch (error) {
        console.error('Login error:', error);
        return NextResponse.json(
            { error: 'Internal server error' },
            { status: 500 }
        );
    }
}
```

## Method 9: Role-Based Access Control

```javascript
// lib/rbac.js
export const ROLES = {
    ADMIN: 'admin',
    USER: 'user',
    MODERATOR: 'moderator'
};

export const PERMISSIONS = {
    READ_USERS: 'read:users',
    WRITE_USERS: 'write:users',
    DELETE_USERS: 'delete:users',
    READ_POSTS: 'read:posts',
    WRITE_POSTS: 'write:posts'
};

const rolePermissions = {
    [ROLES.ADMIN]: [
        PERMISSIONS.READ_USERS,
        PERMISSIONS.WRITE_USERS,
        PERMISSIONS.DELETE_USERS,
        PERMISSIONS.READ_POSTS,
        PERMISSIONS.WRITE_POSTS
    ],
    [ROLES.MODERATOR]: [
        PERMISSIONS.READ_USERS,
        PERMISSIONS.READ_POSTS,
        PERMISSIONS.WRITE_POSTS
    ],
    [ROLES.USER]: [
        PERMISSIONS.READ_POSTS,
        PERMISSIONS.WRITE_POSTS
    ]
};

export function hasPermission(userRole, permission) {
    const permissions = rolePermissions[userRole] || [];
    return permissions.includes(permission);
}

export function requirePermission(permission) {
    return async function (request) {
        const user = await getUserFromRequest(request);

        if (!user || !hasPermission(user.role, permission)) {
            return NextResponse.json(
                { error: 'Forbidden' },
                { status: 403 }
            );
        }

        return null; // Continue to next handler
    };
}
```

```javascript
// app/api/users/route.js
import { requirePermission, PERMISSIONS } from '../../../lib/rbac';

export async function GET(request) {
    const authError = await requirePermission(PERMISSIONS.READ_USERS)(request);
    if (authError) return authError;

    // Handle GET request
    const users = await prisma.user.findMany();
    return NextResponse.json({ users });
}

export async function POST(request) {
    const authError = await requirePermission(PERMISSIONS.WRITE_USERS)(request);
    if (authError) return authError;

    // Handle POST request
    const userData = await request.json();
    const user = await prisma.user.create({ data: userData });
    return NextResponse.json({ user });
}
```

## Method 10: Secure Headers and CSRF Protection

```javascript
// middleware.js
import { NextResponse } from 'next/server';

export function middleware(request) {
    // Add security headers
    const response = NextResponse.next();

    // Prevent CSRF attacks
    response.headers.set('X-Frame-Options', 'DENY');
    response.headers.set('X-Content-Type-Options', 'nosniff');
    response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');

    // CORS headers for API routes
    if (request.nextUrl.pathname.startsWith('/api')) {
        response.headers.set('Access-Control-Allow-Origin', process.env.ALLOWED_ORIGIN);
        response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
        response.headers.set('Access-Control-Allow-Headers', 'Content-Type, Authorization');
    }

    return response;
}
```

```javascript
// lib/csrf.js
import { cookies } from 'next/headers';
import { randomBytes } from 'crypto';

export async function generateCSRFToken() {
    const token = randomBytes(32).toString('hex');

    cookies().set('csrf-token', token, {
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production',
        sameSite: 'lax',
        maxAge: 60 * 60 // 1 hour
    });

    return token;
}

export async function validateCSRFToken(token) {
    const cookieStore = cookies();
    const storedToken = cookieStore.get('csrf-token')?.value;

    return token === storedToken;
}
```

## Best Practices

### 1. **Environment Variables**

```javascript
// .env.local
JWT_SECRET=your-super-secret-jwt-key
NEXTAUTH_SECRET=your-nextauth-secret
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
DATABASE_URL=your-database-url
API_URL=http://localhost:3001/api
ALLOWED_ORIGIN=http://localhost:3000
```

### 2. **Error Handling**

```javascript
// lib/auth-errors.js
export class AuthenticationError extends Error {
    constructor(message) {
        super(message);
        this.name = 'AuthenticationError';
        this.statusCode = 401;
    }
}

export class AuthorizationError extends Error {
    constructor(message) {
        super(message);
        this.name = 'AuthorizationError';
        this.statusCode = 403;
    }
}

export function handleAuthError(error) {
    if (error instanceof AuthenticationError) {
        return NextResponse.json(
            { error: error.message },
            { status: error.statusCode }
        );
    }

    if (error instanceof AuthorizationError) {
        return NextResponse.json(
            { error: error.message },
            { status: error.statusCode }
        );
    }

    console.error('Unexpected error:', error);
    return NextResponse.json(
        { error: 'Internal server error' },
        { status: 500 }
    );
}
```

### 3. **Testing Authentication**

```javascript
// __tests__/auth.test.js
import { getServerSession } from 'next-auth';
import { createMocks } from 'node-mocks-http';

describe('Authentication', () => {
    it('should return user session when authenticated', async () => {
        const { req, res } = createMocks({
            method: 'GET',
            cookies: {
                'next-auth.session-token': 'valid-session-token'
            }
        });

        const session = await getServerSession(req, res, authOptions);
        expect(session).toBeDefined();
        expect(session.user).toBeDefined();
    });

    it('should return null when not authenticated', async () => {
        const { req, res } = createMocks({
            method: 'GET'
        });

        const session = await getServerSession(req, res, authOptions);
        expect(session).toBeNull();
    });
});
```

### 4. **Performance Optimization**

```javascript
// lib/auth-cache.js
import { unstable_cache } from 'next/cache';

export const getCachedUser = unstable_cache(
    async (userId) => {
        const user = await prisma.user.findUnique({
            where: { id: userId }
        });
        return user;
    },
    ['user'],
    { revalidate: 300 } // Cache for 5 minutes
);

export const getCachedSession = unstable_cache(
    async (sessionId) => {
        const session = await prisma.session.findUnique({
            where: { id: sessionId },
            include: { user: true }
        });
        return session;
    },
    ['session'],
    { revalidate: 60 } // Cache for 1 minute
);
```

## Common Patterns and Anti-patterns

### ✅ Good Patterns

1. **Server-first authentication**: Always verify on server-side
2. **HTTP-only cookies**: Prevent XSS attacks
3. **Short-lived tokens**: Reduce impact of token theft
4. **Proper error handling**: Don't leak sensitive information
5. **Role-based access**: Implement granular permissions

### ❌ Anti-patterns

1. **Client-only auth**: Never trust client-side authentication
2. **Local storage tokens**: Vulnerable to XSS attacks
3. **Long-lived sessions**: Increase security risk
4. **Verbose error messages**: Can leak information about valid emails
5. **Global auth state**: Can cause unnecessary re-renders

## Real-World Example: E-commerce Dashboard

```javascript
// app/dashboard/layout.js
import { requireAuth } from '../../lib/auth';
import { hasPermission, PERMISSIONS } from '../../lib/rbac';

export default async function DashboardLayout({ children }) {
    const session = await requireAuth();

    const canViewAnalytics = hasPermission(session.user.role, PERMISSIONS.READ_ANALYTICS);
    const canManageUsers = hasPermission(session.user.role, PERMISSIONS.WRITE_USERS);

    return (
        <div>
            <nav>
                <Link href="/dashboard">Overview</Link>
                {canViewAnalytics && <Link href="/dashboard/analytics">Analytics</Link>}
                {canManageUsers && <Link href="/dashboard/users">Users</Link>}
            </nav>
            <main>{children}</main>
        </div>
    );
}

// app/dashboard/analytics/page.js
import { requireAuth } from '../../../lib/auth';
import { hasPermission, PERMISSIONS } from '../../../lib/rbac';

export default async function AnalyticsPage() {
    const session = await requireAuth();

    if (!hasPermission(session.user.role, PERMISSIONS.READ_ANALYTICS)) {
        redirect('/dashboard');
    }

    const analytics = await getAnalyticsData();

    return (
        <div>
            <h1>Analytics Dashboard</h1>
            <AnalyticsCharts data={analytics} />
        </div>
    );
}
```

### Interview Tip:
*"In Next.js App Router, handle authentication in Server Components using HTTP-only cookies, middleware for route protection, and server actions for auth operations. Always verify authentication on the server-side and use proper security headers to prevent common attacks."*